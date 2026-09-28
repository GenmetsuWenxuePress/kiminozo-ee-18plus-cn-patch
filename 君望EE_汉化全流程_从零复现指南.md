# 《君望》EE 简体中文汉化 · 从零复现全流程指南

> **适用游戏**：Steam《Kimi ga Nozomu Eien ~Enhanced Edition~》（appid 1777440）＋ DLC《Another Episode Collection+》（appid 3112140）
> **文档性质**：完整复现指南 —— 记录我们**实际走通并实机验收过**的全部流程。你可以照着从零做起。
> **配套文档**：《君望EE_汉化问题与解决_全档案.md》—— 过程中遇到的全部坑与解法（含已证伪路线）。**开工前建议先扫一遍它**，能帮你省掉几天的弯路。
> **整理时间**：2026-09-28 ｜ 所有数据均来自实际工程产物（哈希、尺寸、计数均可对拍）

---

## 0. 导读

### 0.1 这份文档是什么

把一款商业游戏（fuzz 引擎）从「零知识」做到「全中文可发布补丁」的完整工程记录。包含：

1. 游戏容器格式的逆向成果（字节级规格 + 可复用工具）；
2. 从提取、翻译、质检、回写到容器重建的完整管线（含全部脚本）；
3. 中文渲染（字体层）的做法与关键陷阱；
4. 与 18+ 内容补丁（EER 层）、DLC 的整合方法；
5. 装机、验收、打包发布的流程。

### 0.2 怎么用这份文档

- **按章节顺序做**。每一章都有【产出物】与【验收标准】，没达到验收标准不要往下走。
- 遇到任何报错/异常，先去配套文档《问题与解决全档案》按症状查——你踩的坑大概率我们踩过。
- 全部脚本都是完整可用的，直接复制成文件即可运行（注意修改脚本顶部的 `BASE` 路径）。
- 数据表（哈希/尺寸/计数）在附录 B，随时对拍。

### 0.3 你需要的基础与工具

- 会用命令行（本文以 **Windows + WSL2 Ubuntu** 为例；纯 Windows 也可以，把路径风格换一下即可）；
- 会跑 Python 3 脚本（不需要会写，但需要能改路径、看报错）；
- 会用 AI 翻译通道（我们用了 Google Gemini 直连 API；你也可以用任何兼容的 LLM 服务，管线是通道无关的）；
- 一台能跑游戏的 Windows 机器（用于装机验收）。

### 0.4 工作量参考（单人 + AI 辅助）

| 阶段 | 内容 | 耗时参考 |
|---|---|---|
| 逆向与格式 | 容器格式破译 + 工具成型 | 2-3 天（难点在首次破译；有本文档后可跳过） |
| 语料与单元 | 提取 452 个剧本 + 去重建库 | 半天 |
| 翻译 | 79,256 单元全量机翻（并发 8） | 约 1.5 小时（不含重试与修复） |
| 质检与修复 | 控制符修复 + 抽样复核 + 术语 | 1 天（含多波次复核） |
| 字体层 | 方框攻坚（本项目最大难点） | 2-3 天（首次） |
| 组装与装机 | 容器合并 + 装机验收 | 半天 |
| DLC | 独立一套（流程同本体） | 1 天 |
| 安装器打包 | 单文件 exe（可选） | 1-2 天 |

> 核心结论：**格式与字体是全部难点；翻译本身是最省事的一步**（AI 批量 + 质检回路即可）。

---

## 1. 成果与总览

### 1.1 我们最终做出来的东西

**单文件安装器**（1.1GB，全内嵌），含三个**独立可选**组件：

| 组件 | 内容 | 依赖 |
|---|---|---|
| 18+ 补丁（EER 层） | 第三方民间补丁「Kiminozo EER Patch v1.0」三容器原样 | 无 |
| 本体汉化 | `patch00m.bin`（v20：EER 文本层汉化 + 中文字体 + 全量翻译） | 无（完全独立；不装 EER 层时 H 场景可能缺图） |
| DLC 汉化 | `pack00m.bin` + `pack.bin`（DLC 文本汉化 + 字体注入） | 需先装 DLC（安装器自动检测） |

发布形态（GitHub Release，供参考）：

- 仓库：`https://github.com/GenmetsuWenxuePress/kiminozo-ee-18plus-cn-patch`
- 安装器 + 6 个补丁文件分别可下载；校验表见附录 B。

### 1.2 技术路线总图

```
[Steam 英文版游戏（干净基底）]
        │
        ▼
① 容器格式逆向 ─────────────────────────────────────────────┐
    · FPD 容器（patchNNx.bin / pack.bin）：XOR 加密 + 条目索引 │
    · egpack 剧本：记录流 + 11 语言槽 + 控制符                 │
    · EPK：uistring 等 UI 串（文件名 stem 为密钥）             │
        │                                                    │
        ▼                                                    │
② 全量语料提取（452 个 egpack：本体 220 + EER 232）            │
        │                                                    │
        ▼                                                    │
③ 翻译单元库（去重 + 上下文 + 18+/普通分流）                    │
    · 114,329 条记录 → 79,256 唯一单元                        │
        │                                                    │
        ▼                                                    │
④ AI 批量翻译 + 控制符质检 + 修复回路（100% 硬码合规）          │
        │                                                    │
        ▼                                                    │
⑤ 译文回写（en 槽 ← 中文）+ 容器重建（外科手术式，含 0x08 同步）│
        │                                                    │
        ▼                                                    │
⑥ 字体层（中文渲染）★最大难点                                  │
    · mimic 字体（Noto 字形集 + 原字体身份拟态）                │
    · 方框根因 = 字体条目数过多污染字体池（v19/v20 单变量实证）  │
        │                                                    │
        ▼                                                    │
⑦ 与 EER 18+ 层整合（文本层汉化合并；图形/视频层原样）          │
        │                                                    │
        ▼                                                    │
⑧ 装机验收（备份 + 覆盖 + 哈希校验 + 实机试玩）                 │
        │                                                    │
        ▼                                                    │
⑨ DLC 汉化（独立一套，同流程）→ ⑩ 单文件安装器 → ⑪ 发布        │
```

### 1.3 关键数据速览

| 项目 | 数值 |
|---|---|
| 提取剧本 | 452 个 egpack（本体 220 + EER 232） |
| 记录数 → 唯一单元 | 114,329 → **79,256**（去重 30.7%） |
| 单元构成 | 普通 75,379（223 万字符）+ 成人 3,877（11.7 万字符） |
| 翻译完成度 | **100%**（79,256/79,256） |
| 控制符硬码合规 | 100%（全量 75,379 条终验 0 不一致） |
| 复检覆盖 | 26,700/75,379（35.4%）→ 671 发现 → 修复 705 key/934 记录 |
| 字体条目（出厂） | 3 个（UDKakugo-mimic / beatfont1-mimic / BIZ-mimic） |
| 本体汉化容器 | `patch00m.bin` v20 = **52,435,254 B**，sha256 `EAF2B9924482EE53...` |
| DLC 汉化容器 | `pack00m.bin` 37,256,314 B + `pack.bin` 399,507,242 B |
| 安装器 | 1,100,177,375 B（内嵌全部载荷） |

---

## 2. 游戏与容器结构（开工前必须搞清的事）

### 2.1 安装目录结构（Steam 版）

```
<Steam 库>/steamapps/common/Kimi ga Nozomu Eien/
├─ kiminozs/                          ← 游戏本体（主程序目录）
│  ├─ kiminozs-win64vc14-release.exe  ← 主程序（fuzz 引擎）
│  ├─ erc_nospfx.dll                  ← 引擎核心 DLL（文本读取逻辑在此）
│  └─ obb/                            ← 资源与补丁容器目录 ★
│     ├─ pack.bin                     ← 本体资源包（原始字体在这里）
│     └─ （补丁容器也放这里：patch00d.bin / patch00m.bin / patch01m.bin）
│
├─ kiminoaz/                          ← DLC《Another Episode Collection+》（独立目录）
│  ├─ prog.ico
│  └─ obb/
│     ├─ pack.bin                     ← DLC 资源包（含 DLC 字体）
│     └─ （DLC 补丁容器放这里：pack00m.bin）
│
└─ （其他 Steam 文件略）
```

**要点**：
- 本体与 DLC 是**两套独立目录**（`kiminozs` / `kiminoaz`），各自的补丁互不影响；
- 补丁容器命名规律：`patch00d.bin`（图形层）、`patch00m.bin`（文本/杂项层）、`patch01m.bin`（视频层）、`pack00m.bin`（小补丁层）——数字+字母的组合是引擎的加载优先级机制（m = message/text 类、d = data/graphics 类）；
- 我们的汉化**不改动任何原有文件**，全部以**新增/替换补丁容器**的方式实现（安装器负责备份原文件）。

### 2.2 三层容器体系

游戏的资源全部装在"容器"里，从外到内三层：

```
第一层：FPD 容器（.bin 文件）
   ├─ patch00d.bin   326,230,153 B（EER 图形层；916 webp + 3,712 fcd + pso/vso/png/gut）
   ├─ patch00m.bin    31,566,760 B（EER 文本层；4,964 条目 = 232 剧本 + 4,716 资源 XML + 8 UI串 + 8 JSON）
   ├─ patch01m.bin   252,576,737 B（EER 视频层；3 个开场 ogv）
   └─ pack.bin        （本体/DLC 资源：字体、UI 等）
        │
        │  每个条目 = 一个文件（路径 + 内容）
        ▼
第二层：条目格式
   ├─ *.egpack  → 剧本容器（加密：整文件 XOR 64KB 密钥流）
   ├─ *.epk     → 加密资源（如 uistring.epk，密钥 = 文件名 stem）
   ├─ *.xml     → 场景/语音/画廊定义（明文）
   ├─ *.otf/.ttc→ 字体（明文）
   └─ *.json    → 配置（明文）
        │
        ▼
第三层：egpack 内部
   └─ 记录流：每条记录 = 11 个「语言槽」token + 控制符
        · jp 槽 = 日文原文（权威）
        · en 槽 = 英文本地化（游戏在 EN 语言下实际读取的槽）★汉化写入目标
        · 其余 9 个语言槽为空
```

### 2.3 语言槽机制（最关键的游戏机制之一）

游戏引擎读取剧本时，按**当前语言设置**选择槽位。Steam 版游戏**只有英语**（读 `en` 槽），所以：

- **汉化的本质 = 把中文写进 `en` 槽**（不是 jp，不是新增槽）；
- 槽标识 = **语言名的 CRC32（小端序 4 字节）**，全 11 槽对照表：

| 语言 | 槽标识（hex） | 语言 | 槽标识（hex） |
|---|---|---|---|
| zh_hans | `629d650e` | zh_hant | `c1080190` |
| pt | `2cde8e39` | es | `9bad5f90` |
| pt_br | `a1508d55` | it | `34778ea2` |
| de | `8b29907d` | id | `506739bf` |
| jp | `eee0ce8e` | fr | `cece75cc` |
| en | `42c159f3` | | |

> 验证方法（Python）：`import zlib, struct; struct.pack('<I', zlib.crc32(b'zh_hans') & 0xffffffff).hex()` → `629d650e` ✓

### 2.4 UI 字符串（uistring.epk）

菜单/按钮文字不在 egpack 里，而在 `root/assets/data/locale/en/epk/uistring.epk`：

- EPK 格式，**加密密钥 = 文件名 stem**（`uistring`）；
- 内含 47 条显示串（返回/确定/取消/第一章/开始遥线/直至化作回忆…）；
- 汉化 = 解密 → 翻译 → 加密回灌（往返验证通过后放回容器）。
- 注意：`root` 版是日文默认（保留不动）；官方 `ck` 槽全是 `LC_*` 占位符（从未提供中文），所以我们用的是 `en` 槽的 uistring。

### 2.5 补丁层的加载机制

- 引擎启动时扫描 `obb/` 目录，按**命名优先级**加载补丁容器（`patch00d` / `patch00m` / `patch01m` / `pack00m`）；
- 补丁容器里的条目**按路径覆盖**同名条目（同路径 = 替换）；
- 因此汉化的完整实现 = **同机制重建一个补丁容器**（结构、加密、长度字段全部保持），条目路径与官方一致，内容 = 原文 + 我们的差异（翻译 + 字体）。
- 装到游戏里 = 把 `patch00m.bin` 放进 `kiminozs/obb/`；卸载 = 删掉/还原备份。

---

## 3. 环境与工具准备

### 3.1 推荐工作环境

| 角色 | 环境 | 用途 |
|---|---|---|
| 开发/构建机 | Windows + WSL2 Ubuntu | 全部脚本运行（Python）、容器构建 |
| 游戏机 | Windows（跑 Steam 游戏） | 装机、实机验收（可以是同一台） |

> 我们的分工：WSL2 里跑 Python 管线；构建产物通过 `scp` 传到游戏机；装机用 PowerShell 脚本。你也可以全在一台机器上做（把 WSL 路径换成 Windows 路径即可）。

### 3.2 必需资产清单

| 资产 | 获取方式 | 说明 |
|---|---|---|
| 游戏本体 | Steam 购买安装 | 英文版；appid 1777440 |
| DLC（如需） | Steam 购买安装 | appid 3112140 |
| **FSNr_tools** | GitHub `kurikomoe/FSNr_tools` | 社区工具集（我们用它解密容器；也可仅参考其格式文档） |
| **decryptKey.bin** | 同上仓库 `scripts/` 子目录 | **64KB XOR 密钥流**（容器解密必需！） |
| Python 3.10+ | 系统安装 | 管线语言 |
| fontTools | `pip install fonttools` | 字体处理（子集化/拟态） |
| EER 官方补丁 | `https://files.kiminozo.life/Patches/Kiminozo_EER_Patch_EN_Steam_1.0.zip` | 18+ 内容层（文本/图形/视频三容器） |

> ⚠️ **decryptKey.bin 的坑**（详见问题档案 §1.1）：它是解密的命根子。拿到后**立刻复制进项目目录**，别只放在临时目录——我们的工作目录被系统清理过一次，密钥丢失导致构建中断，只能重新从 GitHub 拉取。校验方法：64KB，首字节 `46 1f 3c 08 76 0f 12 8d`。

### 3.3 解密与容器侦察（FSNr_tools 用法）

FSNr_tools 是 C++ 工具集（需要自行编译，或参考其源码理解格式）。**更省事的路径**：直接用我们本文档附录里的 `fpd_tool.py`（Python 重实现，已覆盖全部所需功能）：

```python
# 核心 API（完整源码见 §4.4）
read_fpd(path)      # → (ver, entries)：读取 FPD 容器（自动解密 + 解压索引）
build_fpd(entries)  # → bytes：重建 FPD 容器（逐条 XOR 加密）
xor(data)           # 用 decryptKey.bin 做整文件 XOR 解密
eg_tokens(d)        # 解析 egpack 记录流（正则版，仅用于"读"）
eg_records(toks)    # 按记录分组
eg_rebuild(d, toks, tail)  # 重建 egpack（正则版）
```

> ⚠️ **重建 egpack 必须用「结构式解析器」**（§4.5 + 问题档案 §1.4）——正则版有"幽灵 token"缺陷，会把文本片段复制成多余 token。出厂包实测带 41 个幽灵（良性但脏）。**新项目从第一天就用结构式。**

### 3.4 工作目录结构（建议照抄）

```
kiminozo/                          ← 项目根
├─ fsnr/                           ← FSNr_tools 工作目录
│  ├─ decryptKey.bin               ← 64KB 密钥（★立即备份）
│  └─ （FSNr_tools 源码/二进制）
├─ fpd_tool.py                     ← 核心容器工具（§4.4 全文）
├─ corpus/                         ← 提取出的原始语料
│  ├─ base/                        ← 本体 220 个 egpack（存储态=加密）
│  └─ eer/                         ← EER 232 个 egpack
├─ corpus_manifest.json            ← 语料清单（idx → path 映射）
├─ pipeline/                       ← 全部管线脚本
│  ├─ build_units.py               ← 单元库构建
│  ├─ translate.py                 ← 翻译引擎
│  ├─ fix_codes.py                 ← 控制符修复
│  ├─ fix_ws.py                    ← 空白符归一化
│  ├─ inject.py                    ← 译文回写
│  ├─ merge_obb.py                 ← 容器合并
│  └─ make_mimics.py               ← 字体拟态生成
├─ out/                            ← 全部产物
│  ├─ units.jsonl                  ← 翻译单元库
│  ├─ translations/{normal,adult}.jsonl   ← 翻译结果
│  ├─ translations_final/          ← 终态（含修复）
│  ├─ patch00m_v20.bin             ← 出厂汉化容器
│  └─ ...
├─ fonts_cn/                       ← 中文拟态字体
├─ eer_obb/                        ← EER 官方容器副本
│  ├─ patch00m.bin                 ← 汉化的 base（文本层）
│  └─ patch00d.bin / patch01m.bin  ← 原样打包（无文本）
└─ build/                          ← 中间产物（回写后的剧本）
   └─ eer_cn/                      ← 回写后的 232 个 egpack
```

### 3.5 Python 依赖

```bash
pip install fonttools zlib  # zlib 是标准库，无需装；主要就是 fonttools
# 其余全部用标准库（urllib / json / struct / hashlib / concurrent.futures）
```

---

（续下页：§4 容器格式字节级规格 + 完整工具源码）

---

## 4. 容器格式字节级规格（逆向成果，全部经字节级验证）

> 这一章是全部工程的地基。**看懂它，你就能读写游戏的全部资源**；看不懂也没关系——用 §4.4 的工具即可，但建议至少通读一遍，出问题时能定位。

### 4.1 FPD 容器格式（`patchNNx.bin` / `pack.bin`）

```
┌─────────────────────────────────────────────────────────────┐
│ 偏移 0x00: "FPD\0"（4 字节魔数）                              │
│ 偏移 0x04: version（大端 u32，实测 2）                        │
│ 偏移 0x08: 条目数 count（大端 u64）                           │
│ 偏移 0x10: 索引区结束偏移 ebs（大端 u64）                      │
│ 偏移 0x18~0x37: 保留（32 字节 0x00）                          │
│ 偏移 0x38 (=56, HDR): 索引区（XOR 加密）                      │
│   ├─ count × 32 字节条目记录（每条：s_off / off / size / usz  │
│   │   全为大端 u64；s_off=字符串表内路径偏移，off=数据区偏移，  │
│   │   size=存储字节数，usz=解压后明文长度[0=未压缩]）           │
│   └─ zlib 压缩的字符串表（各条目路径，\0 分隔）                 │
│ 偏移 ebs: 数据区（每条目独立 XOR 加密；usz>0 时先 zlib 再 XOR）│
└─────────────────────────────────────────────────────────────┘
```

**加密机制**：
- 全容器用 **64KB 密钥流**（`decryptKey.bin`）做 XOR；
- **索引区与每个数据条目各自从密钥偏移 0 开始**（不是全文件连续 XOR！）；
- 条目数据：`usz=0` → 原样 XOR；`usz>0` → `zlib.compress` 后 XOR（usz 存明文长度）。

**重建规则（关键，错一处游戏就黑屏/拒载）**：
1. 字符串表尾部要 **16 字节对齐**（用 `\x00` 补齐）——实证：缺失补齐的容器在游戏内黑屏（引擎依赖该对齐）；
2. 头部 `ebs` 字段必须精确重算；
3. 数据区偏移 `off` 累计推进。

> ⚠️ 另一个容易踩的坑：EER 官方包的条目**不是从密钥偏移 0 起 XOR**——需要暴力定位偏移（见问题档案 §1.2）。而游戏本体与自建容器都是从 0 起。

### 4.2 egpack 剧本格式（记录流）

```
┌─────────────────────────────────────────────────────────────┐
│ 偏移 0x00: "EPK\0"                                            │
│ 偏移 0x08: 文件总长（小端 u32）★ 改内容后必须同步！否则拒载    │
│ 偏移 0x40: 记录流开始                                         │
│   └─ token 序列：                                             │
│        \x87 + 4B槽标识 + \xa6 + [值字节] + \x00               │
│      · 记录分隔符 \x85\xdb 出现在每条记录首个 token 的 gap 里  │
│      · gap = 上一个 token 结束到本 token 之间的原始字节        │
│      · 文件尾 = 3 字节 tail                                   │
│ 每条记录 = 11 个槽（id/jp/en 恒有值 + 8 个空语言槽，各 7B）     │
└─────────────────────────────────────────────────────────────┘
```

- **值（value）= 到下一个 `\x00` 为止的原始字节**（UTF-8 文本 + 控制符混合）；
- **控制符**：`\w`（停顿）、`\p`（分页）、`\n`（换行）、`\k`、`\f` —— 以**反斜杠+字母**的明文形式存在于文本里，翻译时必须原样保留（数量与顺序）；
- 头 0x40 起还有记录数等信息（如 `a8 + u16 LE 记录数 + db`，实测 `a8 07 27 db` = 9,991 条）——**不是所有文件都同构**，读的时候以 token 流为准。

### 4.3 EPK 格式（uistring 等 UI 串）

- 加密密钥 = **文件名 stem**（如 `uistring.epk` → 密钥 `uistring`）；
- 解密后是键值文本表；汉化 = 解密 → 替换显示串 → 加密回灌 → 往返验证。

### 4.4 完整工具源码：`fpd_tool.py`

> 这是我们的核心工具（Python 重实现，覆盖读/写/改全部功能）。**直接使用**，注意把 `KEY_PATH` 改成你的 decryptKey.bin 路径。

```python
#!/usr/bin/env python3
"""fuzz 引擎 FPD/egpack 工具链：读、改、写（只动内容，不动结构）"""
import struct, zlib, re, os, sys, glob

KEY_PATH = './fsnr/decryptKey.bin'          # ★ 改成你的路径
KEY = open(KEY_PATH, 'rb').read()
HDR = 56
SLOT = re.compile(rb'\x87....\xa6', re.S)
LANGS = {'zh_hans':'629d650e','pt':'2cde8e39','pt_br':'a1508d55','de':'8b29907d','jp':'eee0ce8e',
         'zh_hant':'c1080190','es':'9bad5f90','it':'34778ea2','id':'506739bf','fr':'cece75cc','en':'42c159f3'}

def xor(b, off=0):
    return bytes(x ^ KEY[(off + i) % len(KEY)] for i, x in enumerate(b))

# ---------- FPD ----------
def read_fpd(path):
    d = open(path, 'rb').read()
    assert d[:4] == b'FPD\0', '非 FPD: %r' % d[:8]
    ver, = struct.unpack_from('>I', d, 4)
    count, = struct.unpack_from('>Q', d, 8)
    ebs, = struct.unpack_from('>Q', d, 16)
    idx = xor(d[HDR:ebs])
    ents = []
    for i in range(count):
        s_off, off, size, usz = struct.unpack_from('>QQQQ', idx, i * 32)
        ents.append(dict(s_off=s_off, off=off, size=size, usz=usz))
    strtab = zlib.decompress(idx[count * 32:])
    for e in ents:
        j = strtab.find(b'\x00', e['s_off'])
        e['path'] = strtab[e['s_off']:j].decode('utf-8')
        e['stored'] = d[ebs + e['off']: ebs + e['off'] + e['size']]
        raw = xor(e['stored'])
        e['plain'] = zlib.decompress(raw) if e['usz'] > 0 else raw
    return ver, ents

def enc_entry(plain, usz=0):
    """usz=0 → 原样 XOR；usz>0 → zlib 后 XOR（usz 存明文长度）"""
    if usz > 0:
        return xor(zlib.compress(plain)), len(plain)
    return xor(plain), 0

def build_fpd(ents, ver=2):
    """ents: [dict(path=..., plain=明文)] 或 [dict(path=..., stored=存储字节, usz=...)]"""
    recs, data, so, off = b'', bytearray(), 0, 0
    for e in ents:
        p = e['path'].encode('utf-8')
        if 'stored' in e:
            sb, usz = e['stored'], e.get('usz', 0)
        else:
            sb, usz = enc_entry(e['plain'], e.get('usz', 0))
        recs += struct.pack('>QQQQ', so, off, len(sb), usz)
        data += sb
        off += len(sb); so += len(p) + 1
    strtab = b''.join(e['path'].encode('utf-8') + b'\x00' for e in ents)
    # ⚠ 关键：官方容器字符串表按 16 字节对齐（尾部 \\x00 补齐）。
    # 实证：缺失补齐的容器在游戏内黑屏（引擎校验/依赖该对齐）。勿删！
    strtab += b'\x00' * ((-len(strtab)) % 16)
    idx = recs + zlib.compress(strtab)
    ebs = HDR + len(idx)
    head = b'FPD\0' + struct.pack('>I', ver) + struct.pack('>Q', len(ents)) + struct.pack('>Q', ebs) + b'\x00' * 32
    return head + xor(idx) + bytes(data)

# ---------- egpack ----------
def eg_tokens(d):
    toks, prev = [], 0x40
    for m in SLOT.finditer(d, 0x40):
        gap = d[prev:m.start()]
        i = m.end()
        if d[i:i+1] == b'\x00':
            val = b''; i += 1
        else:
            j = d.find(b'\x00', i)
            val = d[i:j]; i = j + 1
        toks.append([gap, m.group()[1:5], val])
        prev = i
    return toks, d[prev:]

def eg_rebuild(d, toks, tail):
    out = bytearray(d[:0x40])
    for gap, slot, val in toks:
        out += gap + b'\x87' + slot + b'\xa6' + val + b'\x00'
    return bytes(out) + tail

def eg_records(toks):
    recs, cur = [], []
    for t in toks:
        if t[0]:
            if cur: recs.append(cur)
            cur = [t]
        else: cur.append(t)
    if cur: recs.append(cur)
    return recs

def eg_set(d, rec_idx, lang, text):
    toks, tail = eg_tokens(d)
    recs = eg_records(toks)
    want = bytes.fromhex(LANGS[lang])
    hit = [t for t in recs[rec_idx] if t[1] == want]
    assert hit, '记录 %d 无 %s 槽' % (rec_idx, lang)
    hit[0][2] = text.encode('utf-8')
    out = bytearray(eg_rebuild(d, toks, tail))
    struct.pack_into('<I', out, 8, len(out))   # 0x08 = 文件总长，必须同步更新（否则引擎拒载）
    return bytes(out)

if __name__ == '__main__':
    print('模块自检: KEY %d B' % len(KEY))```

### 4.5 ⚠️ 重建必须用「结构式解析器」（不要用上面的正则版重建）

上面的 `eg_tokens/eg_rebuild` 是**正则扫描版**：它用 `re.finditer(rb'\x87....\xa6')` 全局找 token。**当文本值内部恰好含 `\x87....\xa6` 字节序列时**（CJK UTF-8 字节流的自然巧合，比如「采取妥…」的字节 `87 E5 8F 96 E5 A6`），会产生**幽灵 token**：重建时文本片段被复制成多余 token（+18~+484 字节/处，不幂等）。

> 实测：出厂 v20 容器里共有 **41 个幽灵 token**（涉及 25 个文件；theater 8 个最多）。它们**对游戏无害**（值完好、引擎忽略多余 token——实机全通过），但属于脏数据。**新项目请从第一天就用结构式解析器**：

```python
import struct

def eg_parse_struct(d, start=0x40):
    """结构式位置驱动解析器（正确做法）。返回 (tokens, tail)；token = (gap, slot4, val)。"""
    pos = start; toks = []
    while True:
        i = d.find(b'\x87', pos)
        while i >= 0 and (i + 5 >= len(d) or d[i+5] != 0xa6):
            i = d.find(b'\x87', i + 1)      # 值内/杂散 0x87：只挪搜索位，gap 起点 pos 不动
        if i < 0: break
        j = d.find(b'\x00', i + 6)
        if j < 0: break
        toks.append((d[pos:i], d[i+1:i+5], d[i+6:j]))
        pos = j + 1
    return toks, d[pos:]                    # tail（实测 3 字节）

def eg_build_struct(d, toks, tail):
    out = bytearray(d[:0x40])
    for gap, slot, val in toks:
        out += gap + b'\x87' + slot + b'\xa6' + val + b'\x00'
    out += tail
    struct.pack_into('<I', out, 8, len(out))    # 头部总长字段（偏移 8，LE u32）必须同步
    return bytes(out)
```

**出货强制门槛（双检验，未过不得出货）**：
1. **往返逐字节**：`eg_build_struct(d, *eg_parse_struct(d)) == d`（对未修改文件）；
2. **幂等**：同一输入重建两次输出相同；
3. 修改重建后，与原始文件的 diff **只落在预期槽位**（如仅 en 槽的文本值）。

> 为什么结构式是对的：它「位置驱动、逐段消费」——每字节恰好消费一次并原样保留，文本值内的幽灵序列会被当作值的一部分吞掉，不会重发。详细原理与实测数据见问题档案 §1.4。

---

## 5. 语料提取（Step ①）

### 5.1 提取什么

游戏的全部剧本 = 容器里的 `.egpack` 条目，两个来源：

| 来源 | 容器 | 数量 | 用途 |
|---|---|---|---|
| 本体 | 游戏 `kiminozs/obb/pack.bin` | 220 个 | 对照参考 + 共享台词 |
| EER | `eer_obb/patch00m.bin`（EER 官方） | 232 个 | **汉化的 base**（含 18+ 剧情） |

> 为什么汉化基于 EER 层：EER 的 232 个剧本覆盖本体全部剧情并追加/改写了 18+ 内容；最终补丁容器 = EER 文本层 + 我们的翻译，装进游戏即可（无论是否另装 EER 图形/视频层）。

### 5.2 提取方法

```python
#!/usr/bin/env python3
"""提取语料：从容器中导出全部 .egpack，建立 corpus/ 与 corpus_manifest.json"""
import json, os, importlib.util
spec = importlib.util.spec_from_file_location('ft', 'fpd_tool.py')
ft = importlib.util.module_from_spec(spec); spec.loader.exec_module(ft)

BASE = '.'   # 项目根
man = []
def dump(container, tag, outdir):
    ver, ents = ft.read_fpd(container)
    n = 0
    os.makedirs(outdir, exist_ok=True)
    for e in ents:
        if not e['path'].endswith('.egpack'):
            continue
        # 注意：corpus 里存"存储态"（未 XOR 的原始字节），使用时再解密
        open(f'{outdir}/{n:03d}.egpack', 'wb').write(e['stored'])
        man.append(dict(tag=tag, idx=n, path=e['path'], size=len(e['stored']), usz=e.get('usz', 0)))
        n += 1
    print(f'{tag}: {n} 个 egpack ← {container}')
    return n

dump(f'{BASE}/game_obb/pack.bin',  'base', f'{BASE}/corpus/base')   # 本体 220
dump(f'{BASE}/eer_obb/patch00m.bin', 'eer', f'{BASE}/corpus/eer')   # EER 232
json.dump(man, open(f'{BASE}/corpus_manifest.json', 'w'), ensure_ascii=False, indent=1)
print(f'manifest: {len(man)} 条（base 220 在前，eer 232 在后——idx 220 起）')
```

**验收标准**：
- 本体 220 个、EER 232 个，共 **452**；
- `corpus_manifest.json` 452 条，`tag/idx/path/size/usz` 五字段；
- **顺序敏感**：本体在前（idx 0-219）、EER 在后（idx 220-451 的数组位置）——后续所有脚本依赖此约定（`man[220:]` = EER 集合）。

> 踩坑提醒：EER 官方包条目不是从密钥偏移 0 起 XOR（见问题档案 §1.2）——如果提取出来解密报错，先查这个。

---

## 6. 翻译单元构建（Step ②）

### 6.1 从记录到"单元"

- 遍历全部 EER 232 个剧本的**记录**（每条记录 = 一条台词/一行文本）；
- 每条记录取 `jp` 槽（日文原文）与 `en` 槽（英文参考）；
- **单元键 = `sha1(jp)[:12]`**（同一条日文台词全游戏只翻译一次——重复台词自动去重）；
- 附带 **prev/next 上下文**（前一条/后一条的日文）——供翻译模型理解语境；
- **分流**：`adult`（EER 独有的 11 个文件：`27水月奴隷ルート_*` ×10 + `theater.egpack`）vs `normal`（其余）——纯**结构性判定（文件边界），零分类器**。

### 6.2 单元库统计（我们的实际数据）

| 项 | 数值 |
|---|---|
| 记录总数 | 114,329 |
| 唯一单元 | **79,256**（去重 30.7%） |
| 普通集 | 75,379（含共享台词 6,862）→ 223 万字符 |
| 成人集 | 3,877 → 11.7 万字符 |

### 6.3 完整脚本：`pipeline/build_units.py`

```python
#!/usr/bin/env python3
"""构建翻译单元库：去重 + 上下文 + 18+/普通分流
输出：
  out/units.jsonl       每行 {key, jp, en, prev, next, n, refs, set}   (n=出现次数, set=normal|adult)
  out/units_stats.json  统计
  out/files.json        文件索引 → {path, set}
"""
import json, hashlib, importlib.util, re, sys
from collections import defaultdict

BASE = '.'   # 项目根
spec = importlib.util.spec_from_file_location('ft', f'{BASE}/fpd_tool.py')
ft = importlib.util.module_from_spec(spec); spec.loader.exec_module(ft)
JP, EN = 'eee0ce8e', '42c159f3'

man = json.load(open(f'{BASE}/corpus_manifest.json'))
eer = [(i, man[220 + i]['path']) for i in range(232)]

# 18+ 文件集：EER 独有内容（水月奴隶路线 ×10 + theater）
ADULT_MARK = ('奴隷ルート', 'theater.egpack')
def is_adult(path): return any(m in path for m in ADULT_MARK)

files = [dict(idx=i, path=p, set=('adult' if is_adult(p) else 'normal')) for i, p in eer]
json.dump(files, open(f'{BASE}/out/files.json', 'w'), ensure_ascii=False, indent=1)

def key_of(jp): return hashlib.sha1(jp.encode()).hexdigest()[:12]

units = {}   # key → dict
order = []   # 保持出现顺序
for f in files:
    d = ft.xor(open(f"{BASE}/corpus/eer/{f['idx']:03d}.egpack", 'rb').read())   # 存储态 → 解密
    toks, _ = ft.eg_tokens(d)
    recs = ft.eg_records(toks)
    parsed = []
    for r in recs:
        m = {t[1].hex(): t[2].decode('utf-8', 'replace') for t in r}
        parsed.append((m.get(JP, ''), m.get(EN, '')))
    for ri, (jp, en) in enumerate(parsed):
        if not jp:
            continue
        k = key_of(jp)
        if k not in units:
            units[k] = dict(key=k, jp=jp, en=en,
                            prev=parsed[ri-1][0] if ri else '',
                            next=parsed[ri+1][0] if ri + 1 < len(parsed) else '',
                            n=0, refs=[], adult_only=True)
            order.append(k)
        u = units[k]
        u['n'] += 1
        u['refs'].append(f"{f['idx']:03d}-{ri}")
        if f['set'] != 'adult':
            u['adult_only'] = False

with open(f'{BASE}/out/units.jsonl', 'w', encoding='utf-8') as fh:
    for k in order:
        u = units[k]
        u['set'] = 'adult' if u['adult_only'] else 'normal'
        fh.write(json.dumps(u, ensure_ascii=False) + '\n')

tot = len(units)
n_adult = sum(1 for u in units.values() if u['set'] == 'adult')
recs_total = sum(u['n'] for u in units.values())
stats = dict(records=recs_total, unique=tot, dedupe_pct=round(100 * (1 - tot / recs_total), 1),
             normal=tot - n_adult, adult=n_adult,
             adult_chars=sum(len(u['jp']) for u in units.values() if u['set'] == 'adult'),
             normal_chars=sum(len(u['jp']) for u in units.values() if u['set'] == 'normal'))
json.dump(stats, open(f'{BASE}/out/units_stats.json', 'w'), ensure_ascii=False, indent=1)
print(json.dumps(stats, ensure_ascii=False, indent=1))
print('文件集:', {s: sum(1 for f in files if f['set'] == s) for s in ('normal', 'adult')})```

**产出物**：`out/units.jsonl`（79,256 行）+ `out/files.json` + `out/units_stats.json`。
**验收标准**：stats 与上表一致（±微小）；`refs` 字段记录每个单元在哪些文件/记录出现过（回写与修复都靠它定位）。

---

## 7. 翻译管线（Step ③）

### 7.1 通道选型（实测数据，2026-09）

| 通道 | 模型 | 实测速度 | 全量推算 | 备注 |
|---|---|---|---|---|
| **Google 直连 API** ★ | gemini-3.8-flash | 并发 8 → 160 条/8.8s，0 错误 | **≈1.2 h** | 需代理；我们最终用它跑完全量 |
| agy CLI | Gemini 3.8 Flash (High) | 71-79s/批，并发 2 | ≈43.7 h | 预付费免费但慢 |
| 本地 Ollama | qwen38-fast | 11.9 tok/s | 成人集 ≈4.3 h | 无审查、免费；用于兜底 4 条 |
| opencode-go | deepseek-v4-pro | 并发 4 无 429 | — | 需 `x-opencode-session` 头 |

> 结论：**并发 8 的官方 API 直连 = 最快**；本地模型适合兜底（敏感内容被 API 拦截时）。管线与通道解耦，你可以换成任何兼容服务。

### 7.2 系统提示词（SYS）全文 ★

> 这是翻译质量的核心。术语表 + 控制符规则 + 输出规范，缺一不可。

```
你是资深的日译中游戏本地化译者，正在翻译视觉小说《君が望む永遠》(Kimi ga Nozomu Eien) 的简体中文版。
规则：
1. 忠实原文，中文自然、口语化，符合角色语气。露骨/成人内容照实翻译，不回避、不净化、不省略。
2. 控制符 \w \p \n \k \f 必须原样保留，**数量与顺序完全一致**：\w=停顿（译文放在对应位置的自然停顿处，如「…」「，」附近，不得合并、不得省略、不得新增）；\n=换行、\p=分页、\k=标记，位置数量都不能动。
3. 引号用「」，句末标点用中文全角。省略号用「…」。
4. 不得残留日文假名（专有名词一律按下方术语表）。
5. 英文参考译文可能与日文有出入，以日文为准。
6. 只输出 JSON：{"items":[{"id":"<与输入相同的id>","zh":"<中文译文>"}]}，不输出任何其他文字。

【术语表·人名（必须严格一致，一律简体字形）】
鳴海孝之→鸣海孝之（鳴海→鸣海）；孝之→孝之
速瀬水月→速濑水月（速瀬→速濑）；水月→水月
涼宮遙→凉宫遥（涼宮→凉宫）；遙→遥（注意是简体「遥」，不得写「遙」）
涼宮茜→凉宫茜；茜→茜
平慎二→平慎二；慎二→慎二
穂村愛美→穗村爱美（穂村→穗村）；愛美→爱美
玉野まゆ→玉野真由；まゆ→真由
大空寺あゆ→大空寺亚由；あゆ→亚由
天川蛍→天川萤（天川→天川）；蛍→萤
星乃文緒→星乃文绪；文緒→文绪
香月モトコ→香月素子（她是医生）；モトコ→素子
マナマナ→真奈真奈（爱美的昵称）；千鶴→千鹤；勲→勋；健さん→健哥（店长）；真智子→真智子

【称呼规则（同一角色对同一人的称呼全文保持一致）】
〜さん→学生之间「〜同学」；成年社会人「〜女士/〜先生」；熟络亲切时「〜哥/〜姐」
〜くん→「〜君」；〜ちゃん→「小〜」；〜先輩→「〜前辈」
〜先生→老师（教师）/ 医生（香月素子等医护）；〜様→「〜小姐 / 〜大人」（按语境）
姉さん/お姉ちゃん→姐姐；お父さん/お母さん→爸爸/妈妈；おじさん→大叔；おばさん→阿姨
呼び捨て（无后缀直呼）→直接叫名字，不加任何后缀

【18+ 术语统一（自然直译，禁止生造/生硬婉辞）】中に出す→内射；外に出す→外射；アナル→后庭；アソコ→私处（露骨处可「小穴」）；飲ませる→吞精；おっぱい→胸部（露骨处可「奶子」）；イく→高潮；膣→阴道；クリトリス→阴蒂；モノ／イチモツ／一物／それ／アレ（指代男性性器）→肉棒（露骨叙述）或阴茎（中性叙述）——**严禁「分身」「那话儿」「阳物」「肉茎」「尘根」「龙根」等生造或生硬译法**
【输出规范】译文必须是纯简体中文：严禁保留任何日文原文（含促音「っ」「ッ」、助词、未译片段、日文汉字词）；人名/爱称一律按上述术语表与称呼规则翻译；标点使用中文标点。
```

> **称谓政策（重要）**：称谓**不做全局一刀切**——同一角色在不同场景/情境下的称呼本就不同（学生时代叫「同学」、成年后叫「女士」都是对的）。全局扫描只用于**发现可疑偏差**，修复只针对**判定为错误的条目**。已拍板固定的术语：餐厅名=**天空神殿**、文绪昵称=**文绪酱**、绘本名=**玛雅乌尔**。

### 7.3 完整脚本：`pipeline/translate.py`（翻译引擎）

> 设计要点：**断点续跑**（已完成的 key 自动跳过）+ **批次/并发** + **指数退避重试** + **控制符质检**（每条翻译结果立即校验硬码）。

```python
#!/usr/bin/env python3
"""量产翻译引擎：分流 + 多通道 + 断点续跑 + 控制符质检
用法:
  python3 pipeline/translate.py --set normal --channel google --concurrency 4 --batch 20 [--limit N]
  python3 pipeline/translate.py --set adult  --channel local  --concurrency 1 --batch 10 [--limit N]

结果: out/translations/{set}.jsonl  (每行 {key, zh, ch, ok, ts})   断点续跑自动跳过已完成 key
失败: out/translations/{set}.failed.jsonl
"""
import json, time, re, os, sys, subprocess, hashlib, argparse, random, urllib.request, urllib.error
from concurrent.futures import ThreadPoolExecutor, as_completed

BASE = '.'   # 项目根
JOB = f'{BASE}/out/translations'

def load_env():
    ef = {}
    for line in open(os.path.expanduser('~/.hermes/.env')):   # ★ 改成你的密钥文件
        line = line.strip()
        if '=' in line and not line.startswith('#'):
            k, v = line.split('=', 1); ef[k.strip()] = v.strip().strip('"').strip("'")
    return ef
ENV = load_env()
PROXY = {'http': 'http://127.0.0.1:7897', 'https': 'http://127.0.0.1:7897'}   # ★ 代理按需

SYS = """（此处粘贴 §7.2 的完整 SYS 文本）"""

def json_fix(text):
    s = text[text.find('{'): text.rfind('}') + 1]
    try:
        return json.loads(s)
    except json.JSONDecodeError:
        return json.loads(re.sub(r'(?<!\\)\\(?!\\)', r'\\\\', s), strict=False)

def codes(t): return re.findall(r'\\[a-z]', t)
HARD = {'p', 'n', 'k', 'f'}   # 硬码：位置/数量必须一致
def codes_hard(t): return [c for c in codes(t) if c[1] in HARD]

# ---------------- 通道实现 ----------------
def ch_google(items, model='gemini-3.8-flash'):
    body = {'systemInstruction': {'parts': [{'text': SYS}]},
            'contents': [{'role': 'user', 'parts': [{'text': json.dumps(dict(items=items), ensure_ascii=False)}]}],
            'generationConfig': {'temperature': 0.3, 'responseMimeType': 'application/json'}}
    url = f'https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={ENV["GOOGLE_API_KEY"]}'
    opener = urllib.request.build_opener(urllib.request.ProxyHandler(PROXY))
    req = urllib.request.Request(url, data=json.dumps(body).encode(), headers={'Content-Type': 'application/json'})
    with opener.open(req, timeout=300) as r:
        d = json.loads(r.read())
    return json_fix(d['candidates'][0]['content']['parts'][-1]['text'])['items']

def ch_agy(items):
    rules = SYS + '\n输入：\n' + json.dumps(dict(items=items), ensure_ascii=False)
    e2 = os.environ.copy(); e2['HTTP_PROXY'] = e2['HTTPS_PROXY'] = 'http://127.0.0.1:7897'
    p = subprocess.run(['agy', '--model', 'Gemini 3.8 Flash (High)', '--dangerously-skip-permissions',
                        '-p', rules, '--print-timeout', '7m'], capture_output=True, text=True, env=e2, timeout=500)
    return json_fix(p.stdout)['items']

def ch_local(items):
    body = {'model': 'qwen38-fast', 'stream': False, 'format': 'json',
            'options': {'num_ctx': 8192, 'temperature': 0.4},
            'messages': [{'role': 'system', 'content': SYS},
                         {'role': 'user', 'content': json.dumps(dict(items=items), ensure_ascii=False)}]}
    req = urllib.request.Request('http://127.0.0.1:11500/api/chat', data=json.dumps(body).encode(),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=1800) as r:
        d = json.loads(r.read())
    return json_fix(d['message']['content'])['items']

def ch_go(items, model='deepseek-v4-pro'):
    body = dict(model=model, temperature=0.3,
                messages=[dict(role='system', content=SYS),
                          dict(role='user', content=json.dumps(dict(items=items), ensure_ascii=False))])
    req = urllib.request.Request('https://opencode.ai/zen/go/v1/chat/completions',
        data=json.dumps(body).encode(),
        headers={'Authorization': f'Bearer {ENV["OPENCODE_GO_API_KEY"]}', 'Content-Type': 'application/json',
                 'x-opencode-session': f'ses_{int(time.time())}_{random.randint(0,99999)}',
                 'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'})
    with urllib.request.urlopen(req, timeout=300) as r:
        d = json.loads(r.read())
    return json_fix(d['choices'][0]['message']['content'])['items']

CHANNEL = {'google': ch_google, 'agy': ch_agy, 'local': ch_local, 'go': ch_go}

def call_with_retry(ch, items, tries=5):
    for a in range(tries):
        try:
            got = ch(items)
            return {it['id']: it['zh'] for it in got}, None
        except Exception as e:
            code = getattr(e, 'code', None) or (e.args[0] if isinstance(e, urllib.error.HTTPError) else None)
            if a == tries - 1:
                return {}, f'{type(e).__name__}:{str(e)[:90]}'
            time.sleep(min(60, 3 * (2 ** a)) + random.random() * 3)
    return {}, 'unreachable'

# ---------------- 主流程 ----------------
def main():
    ap = argparse.ArgumentParser()
    ap.add_argument('--set', choices=['normal', 'adult'], required=True)
    ap.add_argument('--channel', choices=list(CHANNEL), required=True)
    ap.add_argument('--model', default='')
    ap.add_argument('--concurrency', type=int, default=4)
    ap.add_argument('--batch', type=int, default=20)
    ap.add_argument('--limit', type=int, default=0, help='只跑前 N 条（试跑）')
    ap.add_argument('--keys', default='', help='定点重译：逗号分隔 key 列表（强制覆盖已有译文）')
    args = ap.parse_args()

    units = [json.loads(l) for l in open(f'{BASE}/out/units.jsonl', encoding='utf-8')]
    units = [u for u in units if u['set'] == args.set]
    done = set()
    rp = f'{JOB}/{args.set}.jsonl'
    if os.path.exists(rp):
        for l in open(rp, encoding='utf-8'):
            try: done.add(json.loads(l)['key'])
            except Exception: pass
    todo = [u for u in units if u['key'] not in done]
    if args.keys:
        kl = set(k.strip() for k in args.keys.split(',') if k.strip())
        todo = [u for u in units if u['key'] in kl]
    if args.limit: todo = todo[:args.limit]
    print(f'[{args.set}/{args.channel}] 单元 {len(units)} | 已完成 {len(done)} | 本次 {len(todo)}', flush=True)
    if not todo: return

    batches = [todo[i:i + args.batch] for i in range(0, len(todo), args.batch)]
    fh = open(rp, 'a', encoding='utf-8')
    ffail = open(f'{JOB}/{args.set}.failed.jsonl', 'a', encoding='utf-8')
    t0 = time.time(); nok = nfail = ncode = nsoft = 0
    ch = CHANNEL[args.channel]
    if args.model:
        ch = (lambda items, _m=args.model, _c=ch: _c(items, model=_m))

    def work(b):
        items = [dict(id=u['key'], jp=u['jp'], en=u['en']) for u in b]
        res, err = call_with_retry(ch, items)
        out = []
        for u in b:
            zh = res.get(u['key'], '')
            ok = bool(zh) and codes_hard(zh) == codes_hard(u['jp'])
            soft = bool(zh) and codes(zh) == codes(u['jp'])
            out.append(dict(key=u['key'], zh=zh, ch=args.channel, ok=ok, soft_ok=soft, ts=int(time.time())))
        return out, err

    with ThreadPoolExecutor(args.concurrency) as ex:
        futs = [ex.submit(work, b) for b in batches]
        for i, fu in enumerate(as_completed(futs), 1):
            out, err = fu.result()
            for r in out:
                if r['ok']: nok += 1
                elif not r['zh']: nfail += 1
                else: ncode += 1
                if r['zh'] and not r.get('soft_ok', True): nsoft += 1
                (fh if r['zh'] else ffail).write(json.dumps(r, ensure_ascii=False) + '\n')
            fh.flush(); ffail.flush()
            if i % 5 == 0 or i == len(batches):
                el = time.time() - t0
                eta = el / i * (len(batches) - i)
                print(f'  批 {i}/{len(batches)} | ok={nok} 硬码异常={ncode} 软码差异={nsoft} 失败={nfail} | '
                      f'{el:.0f}s 已用, ETA {eta/60:.1f}min {("| " + err) if err else ""}', flush=True)
    print(f'完成: ok={nok} 硬码异常={ncode} 软码差异={nsoft} 失败={nfail} 用时 {(time.time()-t0)/60:.1f}min', flush=True)

if __name__ == '__main__':
    main()```

### 7.4 质检体系（关键设计）

| 类别 | 定义 | 要求 |
|---|---|---|
| **硬码** | `\p \n \k \f` | **数量与顺序必须完全一致**（位置不能动） |
| **软码** | `\w`（停顿） | 数量尽量一致，位置可就近调整（容忍） |

- 每条翻译结果**立即计算** `ok`（硬码合规）与 `soft_ok`（全码合规），写入结果文件；
- **主通道首过硬码合规率 ≈85%**（失败模式 = 丢 `\w`/`\n`）；
- 不合规的走**修复通道**（§7.5），修复率 13/15，余量手工收尾 → **100%**。

### 7.5 修复脚本

**`pipeline/fix_codes.py`**（控制符修复：带"控制符清单"重译）：

```python
#!/usr/bin/env python3
"""控制符修复通道：对 ok=false 的单元带"控制符清单"重译并复检
用法: python3 pipeline/fix_codes.py --set normal --channel google
"""
import json, re, time, argparse, urllib.request, importlib.util
from concurrent.futures import ThreadPoolExecutor

BASE = '.'
spec = importlib.util.spec_from_file_location('tp', f'{BASE}/pipeline/translate.py')
tp = importlib.util.module_from_spec(spec); spec.loader.exec_module(tp)

FIX_SYS = tp.SYS + """
【本批强化规则】每条输入附带 codes 字段 = 原文控制符序列（按出现顺序）。
你的译文《必须》原样包含完全相同的控制符序列：数量一致、顺序一致、种类一致，一个都不能少、不能多。
自检：译文中的控制符数量必须等于 codes 中的数量。"""

def google_call(items, sysmsg, temp=0.2):
    body = {'systemInstruction': {'parts': [{'text': sysmsg}]},
            'contents': [{'role': 'user', 'parts': [{'text': json.dumps(dict(items=items), ensure_ascii=False)}]}],
            'generationConfig': {'temperature': temp, 'responseMimeType': 'application/json'}}
    url = f'https://generativelanguage.googleapis.com/v1beta/models/gemini-3.8-flash:generateContent?key={tp.ENV["GOOGLE_API_KEY"]}'
    opener = urllib.request.build_opener(urllib.request.ProxyHandler(tp.PROXY))
    req = urllib.request.Request(url, data=json.dumps(body).encode(), headers={'Content-Type': 'application/json'})
    with opener.open(req, timeout=300) as r:
        return json.loads(r.read())['candidates'][0]['content']['parts'][-1]['text']

def go_call(items, sysmsg, model='deepseek-v4-pro', temp=0.2):
    body = dict(model=model, temperature=temp,
                messages=[dict(role='system', content=sysmsg),
                          dict(role='user', content=json.dumps(dict(items=items), ensure_ascii=False))])
    req = urllib.request.Request('https://opencode.ai/zen/go/v1/chat/completions',
        data=json.dumps(body).encode(),
        headers={'Authorization': f'Bearer {tp.ENV["OPENCODE_GO_API_KEY"]}', 'Content-Type': 'application/json',
                 'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'})
    with urllib.request.urlopen(req, timeout=300) as r:
        d = json.loads(r.read())
    return tp.json_fix(d['choices'][0]['message']['content'])['items']

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument('--set', default='normal')
    ap.add_argument('--channel', default='google')
    ap.add_argument('--model', default='')
    ap.add_argument('--batch', type=int, default=10)
    ap.add_argument('--concurrency', type=int, default=4)
    args = ap.parse_args()

    units = {json.loads(l)['key']: json.loads(l) for l in open(f'{BASE}/out/units.jsonl', encoding='utf-8')}
    rp = f'{BASE}/out/translations/{args.set}.jsonl'
    entries = [json.loads(l) for l in open(rp, encoding='utf-8')]
    bad = [e for e in entries if not e['ok'] and e['zh']]
    print(f'待修复 {len(bad)} 条', flush=True)
    if not bad:
        return

    def work(b):
        items = [dict(id=u['key'], jp=u['jp'], en=u['en'], codes=' '.join(tp.codes(u['jp'])))
                 for u in (units[e['key']] for e in b)]
        for a in range(5):
            try:
                if args.channel == 'google':
                    txt = google_call(items, FIX_SYS)
                    got = tp.json_fix(txt)['items']
                elif args.channel == 'go':
                    got = go_call(items, FIX_SYS, model=args.model or 'deepseek-v4-pro')
                elif args.channel == 'agy':
                    got = tp.ch_agy(items)
                else:
                    got = tp.CHANNEL[args.channel](items)
                return b, {it['id']: it['zh'] for it in got}, None
            except Exception as e:
                if a == 4:
                    return b, {}, f'{type(e).__name__}:{str(e)[:80]}'
                time.sleep(min(30, 3 * 2 ** a))
    fixed = 0; still = []
    batches = [bad[i:i + args.batch] for i in range(0, len(bad), args.batch)]
    with ThreadPoolExecutor(args.concurrency) as ex:
        for b, res, err in ex.map(work, batches):
            nf = 0
            for e in b:
                u = units[e['key']]
                zh = res.get(e['key'], '')
                if zh and tp.codes_hard(zh) == tp.codes_hard(u['jp']):
                    e['zh'] = zh; e['ok'] = True; e['soft_ok'] = (tp.codes(zh) == tp.codes(u['jp'])); e['fixed'] = 1; fixed += 1; nf += 1
                else:
                    e['zh2'] = zh; still.append(e['key'])
            print(f'   批: 修复 {nf}/{len(b)} {err or ""}', flush=True)
    with open(rp, 'w', encoding='utf-8') as fh:
        for e in entries:
            fh.write(json.dumps(e, ensure_ascii=False) + '\n')
    print(f'修复成功 {fixed}/{len(bad)} | 仍异常 {len(still)}: {still[:10]}', flush=True)

if __name__ == '__main__':
    main()```

**`pipeline/fix_ws.py`**（空白符归一化：真实换行 → `\n` 转义，不调 API 的确定性修复）：

```python
#!/usr/bin/env python3
"""fix_ws.py — 归一化译文中的真实空白符（不调 API，确定性修复）
用法: python3 fix_ws.py <normal|adult>
规则:
  - 真实换行(0x0A/0x0D) → 转为两字符转义 \\n（仅当该条 JP 的 \\n 数还有余量）
  - 超出余量的多余换行 → 删除（并计数报告）
  - \\t → 删除；\\r → 先并入换行处理
  - 修复后重算 ok(硬码) / soft_ok(全码)
安全: 运行中的结果文件请等该集跑完后再处理（本脚本会重写整个文件）。
"""
import json, re, sys, os

BASE = '.'
name = sys.argv[1]
assert name in ('normal', 'adult')
rp = f'{BASE}/out/translations/{name}.jsonl'
units = {json.loads(l)['key']: json.loads(l) for l in open(f'{BASE}/out/units.jsonl', encoding='utf-8')}

def codes(t): return re.findall(r'\\[a-z]', t)
HARD = {'p', 'n', 'k', 'f'}
def codes_hard(t): return [c for c in codes(t) if c[1] in HARD]

rows = [json.loads(l) for l in open(rp, encoding='utf-8')]
nfix = nadd = ndrop = 0
for r in rows:
    zh = r.get('zh') or ''
    if not zh or not any(ch in zh for ch in ('\n', '\r', '\t')):
        continue
    u = units.get(r['key'])
    if not u:
        continue
    zh2 = zh.replace('\r\n', '\n').replace('\r', '\n').replace('\t', '')
    room = max(0, u['jp'].count('\\n') - zh2.count('\\n'))
    real = zh2.count('\n')
    conv = min(room, real)
    parts = zh2.split('\n')
    new = parts[0] + ''.join(('\\n' if i < conv else '') + p for i, p in enumerate(parts[1:]))
    nadd += conv
    ndrop += real - conv
    r['zh'] = new
    r['ok'] = codes_hard(new) == codes_hard(u['jp'])
    r['soft_ok'] = codes(new) == codes(u['jp'])
    nfix += 1

tmp = rp + '.tmp'
with open(tmp, 'w', encoding='utf-8') as f:
    for r in rows:
        f.write(json.dumps(r, ensure_ascii=False) + '\n')
os.replace(tmp, rp)
ok = sum(1 for r in rows if r.get('zh') and r.get('ok'))
soft = sum(1 for r in rows if r.get('zh') and not r.get('soft_ok'))
print(f'{name}: 修复 {nfix} 条（补 \\n 转义 {nadd} 个，删多余换行 {ndrop} 个）| 总 {len(rows)} 条，硬码ok {ok}，软码差异 {soft}')```

### 7.6 运行实录（我们的数据，供对照）

- **成人集**：3,877/3,877 完成（7.2 分钟；硬码 100% 干净；40 条批量级拦截 → `--batch 1` 逐条重试成功；2 条真实换行 → fix_ws 归一化）；
- **普通集**：75,379 条，并发 8，约 55 分钟；硬码异常 0.4% → 修复回路 → 终验 **75,379 条全查 0 不一致**；
- 关键教训：**批量级偶发拦截**（一条敏感行 → 整批 `KeyError:'candidates'`）→ 失败重试必须 `--batch 1`；**真实换行 vs `\n` 转义**（模型偶发输出 0x0A）→ fix_ws 确定性归一化（见问题档案 §2.2/§2.3）。

### 7.7 收尾（残留清零）

终态前做全库扫描并清零：促音（っ/ッ）残留、繁体字形残留、未译片段、注释记录、缺 `\n`、小假名等。我们最终做到**全库零假名/零繁体残留**。扫描方法：

```python
import json, re
jp_kana = re.compile(r'[\u3040-\u309f\u30a0-\u30ff]')       # 假名
trad    = ['遙','瀬','愛','宮','緒','鳴','穗','涼']           # 常见繁体/日文汉字残留字形
for line in open('out/translations_final/normal.jsonl', encoding='utf-8'):
    t = json.loads(line)
    zh = t.get('zh', '')
    if jp_kana.search(zh) or any(c in zh for c in trad):
        print(t['key'], repr(zh[:60]))
```

---

（续下页：§8 回写与容器重建 + §9 字体层 + §10-13 整合/装机/发布 + 附录）

---

## 8. 译文回写与容器重建（Step ④）

### 8.1 回写原理（inject）

- 逐文件读取 EER 剧本（`corpus/eer/NNN.egpack`，存储态 → `ft.xor()` 解密）；
- 解析记录，取每条的 `jp` 槽算 `key`，从译文库查 `zh`，**写入 `en` 槽**（`en_tok[2] = zh.encode()`）；
- 重建文件 + **同步 `0x08` 总长字段** + **回读自检**（断言：写入 N 条 = 回读命中 N 条）；
- 输出到 `build/eer_cn/NNN.egpack`。

### 8.2 完整脚本：`pipeline/inject.py`

```python
#!/usr/bin/env python3
"""译文回写：把 zh 写进每条记录的 en 槽（游戏读 en 槽），输出到 build/eer_cn/
用法: python3 pipeline/inject.py [--all]   (默认只写有译文的；--all 全量输出)
"""
import json, hashlib, importlib.util, os, struct, sys
from collections import Counter

BASE = '.'
spec = importlib.util.spec_from_file_location('ft', f'{BASE}/fpd_tool.py')
ft = importlib.util.module_from_spec(spec); spec.loader.exec_module(ft)
JP, EN = bytes.fromhex('eee0ce8e'), bytes.fromhex('42c159f3')

def key_of(jp): return hashlib.sha1(jp.encode()).hexdigest()[:12]

def main():
    trans = {}
    bad = []
    for s in ('normal', 'adult'):
        p = f'{BASE}/out/translations/{s}.jsonl'
        if not os.path.exists(p): continue
        for line in open(p, encoding='utf-8'):
            e = json.loads(line)
            if e.get('ok') and e.get('zh'):
                trans[e['key']] = e['zh']
            elif e.get('zh'):
                bad.append(e['key'])
    print(f'译文单元 {len(trans)} | 未过质检（跳过） {len(bad)}', flush=True)

    man = json.load(open(f'{BASE}/corpus_manifest.json'))[220:]
    os.makedirs(f'{BASE}/build/eer_cn', exist_ok=True)
    tot_hit = tot_miss = tot_noen = 0; written = 0; miss_units = Counter(); badhdr = 0
    for idx, m in enumerate(man):
        d = ft.xor(open(f"{BASE}/corpus/eer/{idx:03d}.egpack", 'rb').read())  # corpus = 存储态（XOR），先解密
        toks, tail = ft.eg_tokens(d)
        recs = ft.eg_records(toks)
        n_hit = 0
        for rec in recs:
            jp = None; en_tok = None
            for t in rec:
                if t[1] == JP: jp = t[2].decode('utf-8', 'replace')
                elif t[1] == EN: en_tok = t
            if not jp: continue
            zh = trans.get(key_of(jp))
            if zh:
                if en_tok is None:
                    tot_noen += 1
                else:
                    en_tok[2] = zh.encode('utf-8'); n_hit += 1
            else:
                tot_miss += 1
                if len(miss_units) < 50 or key_of(jp) in miss_units:
                    miss_units[key_of(jp)] += 1
        if n_hit:
            out = bytearray(ft.eg_rebuild(d, toks, tail))
            struct.pack_into('<I', out, 8, len(out))
            open(f"{BASE}/build/eer_cn/{idx:03d}.egpack", 'wb').write(bytes(out))
            # 回读自检
            d2 = open(f"{BASE}/build/eer_cn/{idx:03d}.egpack", 'rb').read()
            L, = struct.unpack_from('<I', d2, 8)
            assert L == len(d2), f'{idx}: 0x08 长度 {L} != {len(d2)}'
            t2, _ = ft.eg_tokens(d2); r2 = ft.eg_records(t2)
            n2 = 0
            for rec in r2:
                jp = None; en = None
                for t in rec:
                    if t[1] == JP: jp = t[2].decode('utf-8', 'replace')
                    elif t[1] == EN: en = t[2].decode('utf-8', 'replace')
                if jp and trans.get(key_of(jp)) and en == trans[key_of(jp)]: n2 += 1
            assert n2 == n_hit, f'{idx}: 回读命中 {n2} != 写入 {n_hit}'
            written += 1
            tot_hit += n_hit
    print(f'写入文件 {written} | 命中记录 {tot_hit} | 未译记录 {tot_miss} | 无 en 槽 {tot_noen}')
    # 未译单元覆盖率
    all_units = [json.loads(l) for l in open(f'{BASE}/out/units.jsonl', encoding='utf-8')]
    done = sum(1 for u in all_units if u['key'] in trans)
    print(f'单元覆盖: {done}/{len(all_units)} = {done/len(all_units)*100:.1f}%')

if __name__ == '__main__':
    main()
```

> **新手注意**：生产版（v13 之后）我们已换用**结构式解析器**做回写（`eg_parse_struct`/`eg_build_struct`）——本脚本演示的是管线逻辑，**新项目请把 `eg_tokens`/`eg_rebuild` 替换为 §4.5 的结构式版本**（函数名一一对应）。出厂 v20 里那 41 个幽灵 token 就是这个正则版在早期构建阶段留下的（无害但脏）。

### 8.3 容器合并：`pipeline/merge_obb.py`

把「EER 官方 patch00m + 回写后的 232 个剧本 + 中文字体条目 + 中文 UI 串」合并成最终容器：

```python
#!/usr/bin/env python3
"""合并容器：EER 官方 patch00m.bin + 中文化剧本（build/eer_cn）+ 中文字体 + 中文 UI 串
输出: out/patch00m_cn.bin
用法: python3 pipeline/merge_obb.py
"""
import json, os, importlib.util, struct

BASE = '.'
spec = importlib.util.spec_from_file_location('ft', f'{BASE}/fpd_tool.py')
ft = importlib.util.module_from_spec(spec); spec.loader.exec_module(ft)

FONTS = [
    ('root/assets/data_spec/gui/font/FOT-UDKakugo_LargePro-DB.otf', f'{BASE}/fonts_cn/CN-mimic.otf'),
    ('root/assets/data_spec/gui/font/beatfont1.otf', f'{BASE}/fonts_cn/beatfont1-mimic.otf'),
    ('root/assets/data_spec/gui/font/FOT-SkipProN-D.otf', f'{BASE}/fonts_cn/Skip-mimic.otf'),
    ('root/assets/data_spec/gui/font/LINESeedJP_A_OTF_Bd.otf', f'{BASE}/fonts_cn/LINESeed-mimic.otf'),
    ('root/assets/data_spec/gui/font/BIZ-UDMinchoM.ttc', f'{BASE}/fonts_cn/BIZ-UDMinchoM-mimic.ttc'),
]
# UI 字符串表（en 槽 → 中文版，EPK 加密态；root/ck 保持原样）
UISTRING = [('root/assets/data/locale/en/epk/uistring.epk', f'{BASE}/ui_cn_build/uistring.epk_enc')]

def replace_or_add(ents, path, plain):
    for e in ents:
        if e['path'] == path:
            e['plain'] = plain; e.pop('stored', None); return False
    ents.append(dict(path=path, plain=plain, usz=0)); return True

def main():
    ver, ents = ft.read_fpd(f'{BASE}/eer_obb/patch00m.bin')
    print(f'EER 容器: ver={ver} 条目={len(ents)}', flush=True)
    man = json.load(open(f'{BASE}/corpus_manifest.json'))[220:]
    path2idx = {m['path']: i for i, m in enumerate(man)}

    n_scr = 0
    for e in ents:
        p = e['path']
        if p.endswith('.egpack') and p in path2idx:
            f = f"{BASE}/build/eer_cn/{path2idx[p]:03d}.egpack"
            if os.path.exists(f):
                e['plain'] = open(f, 'rb').read()
                e.pop('stored', None)
                n_scr += 1
    print(f'替换剧本条目 {n_scr}', flush=True)

    n_font = n_add = 0
    for fp, src in FONTS:
        if replace_or_add(ents, fp, open(src, 'rb').read()): n_add += 1
        n_font += 1
    print(f'字体条目 {n_font}（新增 {n_add}）', flush=True)

    n_ui = 0
    for fp, src in UISTRING:
        replace_or_add(ents, fp, open(src, 'rb').read()); n_ui += 1
    print(f'UI 字符串条目 {n_ui}', flush=True)

    out = ft.build_fpd(ents, ver=ver)
    dst = f'{BASE}/out/patch00m_cn.bin'
    open(dst, 'wb').write(out)
    print(f'写出 {dst} | {len(out)} 字节 ({len(out)/1e6:.1f} MB)', flush=True)

    # 回读自检
    ver2, ents2 = ft.read_fpd(dst)
    assert ver2 == ver and len(ents2) == len(ents), '回读条目数不符'
    names2 = {e['path']: e for e in ents2}
    for fp, src in FONTS:
        assert names2[fp]['plain'] == open(src, 'rb').read(), f'字体回读不符 {fp}'
    # 抽一个剧本核对中文
    chk = [p for p in path2idx if os.path.exists(f"{BASE}/build/eer_cn/{path2idx[p]:03d}.egpack")]
    if chk:
        p = chk[0]
        e = names2[p]
        toks, _ = ft.eg_tokens(e['plain']); recs = ft.eg_records(toks)
        print(f'回读样例 {p.split("/")[-1]}: 记录 {len(recs)}', flush=True)
        for rec in recs[:60]:
            vals = {t[1].hex(): t[2].decode('utf-8', 'replace') for t in rec}
            zh = vals.get('42c159f3', '')
            if any('\u4e00' <= c <= '\u9fff' for c in zh):
                print(f'   中文样例: {zh[:50]}', flush=True)
                break
    print('合并自检通过 ✓', flush=True)

if __name__ == '__main__':
    main()
```

> ⚠️ **`stored` 键陷阱**：替换条目时**必须先 `e.pop('stored', None)`**——否则 `build_fpd` 会优先用旧的存储字节，你的修改**静默失效**（文件没变，回读却"通过"）。这是全项目最阴险的坑之一（问题档案 §4.1）。

### 8.4 外科手术式重建（修复层专用，大容器策略）

**为什么需要**：完整重建一个 60MB+ 的容器要几十秒到几分钟、吃大量内存；而"修复 705 条译文"只涉及 93 个剧本文件。**外科手术 = 只重建受影响的文件，其余条目字节原样搬运**（`stored` 不动）。

```python
# -*- coding: utf-8 -*-
# v13 外科手术: v12 容器 + 705 条 normal 修复（复检修复层）
import importlib.util, hashlib, json, struct

spec = importlib.util.spec_from_file_location('ft', 'fpd_tool.py')
ft = importlib.util.module_from_spec(spec); spec.loader.exec_module(ft)

def key_of(jp):
    return hashlib.sha1(jp.encode()).hexdigest()[:12]

# 1. 本轮修复: 相对原始备份 (bak_20260927_112003) 的变更
orig = {}
for line in open('out/translations_final/normal.jsonl.bak_20260927_112003', encoding='utf-8'):
    t = json.loads(line); orig[t['key']] = t['zh']
cur = {}
for line in open('out/translations_final/normal.jsonl', encoding='utf-8'):
    t = json.loads(line); cur[t['key']] = t['zh']
fixed = {k: cur[k] for k in cur if k in orig and orig[k] != cur[k]}
print('本轮修复:', len(fixed), '条')

# 2. 受影响 egpack
units = {}
for line in open('out/units.jsonl', encoding='utf-8'):
    u = json.loads(line); units[u['key']] = u
man = json.load(open('corpus_manifest.json'))[220:]
affected = {}
for k in fixed:
    for r in units[k]['refs']:
        idx = int(r.split('-')[0])
        path = man[idx]['path']
        affected.setdefault(path, set()).add(k)
print('受影响 egpack:', len(affected))
for p, ks in sorted(affected.items()):
    print(f"  {p.split('/')[-1]}: {len(ks)} 条")

# 3. 读 v12 → 手术
nv, ents = ft.read_fpd('out/patch00m_v12.bin')
print('v12 条目:', len(ents), 'ver:', nv)

EN_HEX = bytes.fromhex('42c159f3')
JP_HEX = bytes.fromhex('eee0ce8e')
total_replaced = 0
touched_files = 0
zero_hit = []
for e in ents:
    p = e['path']
    if p not in affected:
        continue
    want = affected[p]
    d = e['plain']
    toks, tail = ft.eg_tokens(d)
    recs = ft.eg_records(toks)
    n_hit = 0
    for rec in recs:
        jp = None; en_tok = None
        for t in rec:
            if t[1] == JP_HEX:
                jp = t[2].decode('utf-8', 'replace')
            elif t[1] == EN_HEX:
                en_tok = t
        if not jp or en_tok is None:
            continue
        k = key_of(jp)
        if k in want:
            en_tok[2] = fixed[k].encode('utf-8')
            n_hit += 1
    if n_hit:
        out = bytearray(ft.eg_rebuild(d, toks, tail))
        struct.pack_into('<I', out, 8, len(out))
        e['plain'] = bytes(out)
        e.pop('stored', None)
        touched_files += 1
        total_replaced += n_hit
        print(f"  {p.split('/')[-1]}: 替换 {n_hit}/{len(want)}")
    else:
        zero_hit.append(p.split('/')[-1])
        print(f"  !! {p.split('/')[-1]}: 零命中 (检查)")

print('触及文件:', touched_files, '| 总替换记录:', total_replaced)
if zero_hit:
    print('零命中文件:', zero_hit)

# 4. 写出 v13
data = ft.build_fpd(ents, ver=nv)
open('out/patch00m_v13.bin', 'wb').write(data)
print('v13 大小:', len(data), 'B')
h = hashlib.sha256(data).hexdigest()
print('v13 sha256:', h)

# 5. 回读验证
nv2, ents2 = ft.read_fpd('out/patch00m_v13.bin')
m2 = {e['path']: e for e in ents2}
ok_cnt = 0
for p, ks in affected.items():
    d = m2[p]['plain']
    toks, _ = ft.eg_tokens(d)
    recs = ft.eg_records(toks)
    for rec in recs:
        jp = None; en = None
        for t in rec:
            if t[1] == JP_HEX: jp = t[2].decode('utf-8', 'replace')
            elif t[1] == EN_HEX: en = t[2].decode('utf-8', 'replace')
        if jp and key_of(jp) in ks and en == fixed[key_of(jp)]:
            ok_cnt += 1
print(f'回读验证: {ok_cnt}/{total_replaced} 条命中修复后文本')

# 6. 未触及条目抽样一致性
import random
random.seed(13)
others = [e for e in ents2 if e['path'] not in affected]
for e in random.sample(others, 3):
    o = next(x for x in ents if x['path'] == e['path'])
    same = hashlib.sha256(e['plain']).hexdigest() == hashlib.sha256(o['plain']).hexdigest()
    print(f"  抽样 {e['path'].split('/')[-1][:40]}: {'一致 OK' if same else '不一致 !!'}")```

**实测数据**：705 条修复 → 触及 93 个 egpack → 替换 934 条记录 → 回读 934/934 命中 ✓。

### 8.5 验收门槛（每次构建必过）

1. **回读断言**：替换 N 条 → 回读命中 N 条（脚本内置 assert，不过就崩）；
2. **未触及条目抽样**：随机抽 3 个未修改条目，与原容器逐字节比对一致；
3. **记录尺寸/哈希记录**：每版容器记录 size + sha256（版本链见附录 B）；
4. **装机前**：与上一版做 `diff`（如 v20 与原版全量 diff —— 4,328/4,964 记录一致，全部差异均为预期汉化）。

### 8.6 附加功能：回想全解锁模块（出厂版已含）

**背景**：游戏的存档是「压缩+加密流」（逆向成本高），但回想/画廊的解锁状态由游戏自己的 **flag 机制**（`<value_save>` 标志）控制——**不碰存档、改剧本即可全解锁**。

**原理**：
1. 扫描容器内全部 XML，收集所有 `<value_save>` 标志键（共 **67 种**：回想 open_key 40 + 结局 + 章节）；
2. 把「全部置 1」的注入块写进**每个剧情脚本（scr/*.xml，407 个）的根节点入口**——任意场景加载即全解锁。

**实测**：40 个回想 open_key 与存档标志 40/40 对应；出厂 v20 中 407 个 scr XML 全部含注入（原版仅 4 个）。

```python
#!/usr/bin/env python3
"""add_unlock.py — 把「全解锁模块」注入 CN 容器（可重复用于每次重建）
原理: 收集容器内全部 <value_save> 标志（67 种：回想 open_key 40 + 结局 + 章节），
      注入到每个剧情脚本 (scr/*.xml) 的根节点入口 → 任意场景加载即置位。
用法: python3 pipeline/add_unlock.py [--src out/patch00m_cn.bin] [--dst out/patch00m_cn_unlock.bin]
"""
import importlib.util, re, argparse, os

BASE = '.'
spec = importlib.util.spec_from_file_location('ft', f'{BASE}/fpd_tool.py')
ft = importlib.util.module_from_spec(spec); spec.loader.exec_module(ft)

ap = argparse.ArgumentParser()
ap.add_argument('--src', default=f'{BASE}/out/patch00m_cn.bin')
ap.add_argument('--dst', default=f'{BASE}/out/patch00m_cn_unlock.bin')
args = ap.parse_args()

ver, ents = ft.read_fpd(args.src)
flags = set()
for e in ents:
    if e['path'].endswith('.xml'):
        for m in re.finditer(rb'<value_save[^>]*key="([^"]+)"', e['plain']):
            flags.add(m.group(1).decode('utf-8', 'replace'))
inj = ''.join(f'\r\n  <value_save key="{k}" value="1"/>' for k in sorted(flags)).encode('utf-8')
n = 0
for e in ents:
    if e['path'].endswith('.xml') and '/scr/' in e['path'] and b'<value_save' not in e['plain'][:2000]:
        m = re.search(rb'(<node\b[^>]*>)', e['plain'])
        if m:
            e['plain'] = e['plain'][:m.end()] + inj + e['plain'][m.end():]
            e.pop('stored', None); n += 1
data = ft.build_fpd(ents, ver=ver)
open(args.dst, 'wb').write(data)
v2, e2 = ft.read_fpd(args.dst)
print(f'注入 {n} 个脚本 × {len(flags)} 标志 | 写出 {args.dst} ({len(data)}B) | 回读 {len(e2)} 条 ✓')
```

> 用法提示：在 merge 出 `patch00m_cn.bin` 之后、装机之前跑一次即可；该模块为**可选**（不影响汉化本体）。

---

## 9. 字体层 —— 中文渲染（Step ⑤，本项目最大难点）★★

> 游戏原始字体全部是**日文字体**（不含简体字、无回退机制）——不注入中文字体，中文直接显示为 `.notdef` 方框「□」。**字体层是汉化成败的关键。**

### 9.1 引擎的字体机制（先看懂再动手）

- 字体配置在 `.gut` 文件里（`<FontName>` 节点），引擎按**槽位**取字体：

| 槽（Font_en.cfg） | 对应字体文件 | 用在哪 |
|---|---|---|
| Common | `beatfont1.otf` | backlog / 菜单 / 按钮；**H 场景消息文本也走此槽** |
| Message | `FOT-UDKakugo_LargePro-DB.otf` | 对话正文 |
| Speaker | `FOT-SkipProN-D.otf` | 名字框 |
| Hud | `FOT-UDKakugo_LargePro-DB.otf`（与 Message 同一文件） | HUD 元素 |
| StaffCN | `BIZ-UDMinchoM.ttc` | 片尾 staff roll |

- 字体文件分布：**本体 pack.bin 一份（原版）** + 我们的补丁容器一份（替换版）；
- 替换机制：补丁容器里的同名条目覆盖原版；
- **无回退**：缺字 = 方框，不会自动找别的字体。

### 9.2 mimic 字体（做法）

「mimic」= **拿一套含全量简体的开源字体（Noto Sans SC），只保留需要的字形（子集化），再把名称表移植成原字体的**——让引擎以为它就是原字体：

```python
#!/usr/bin/env python3
"""为未汉化的三个字体构建 mimic：复用已验证的 Noto 字形集 + 移植原字体名称表。
生成：
  fonts_cn/Skip-mimic.otf     ← FOT-SkipProN-D （Speaker = 名字栏 ★关键）
  fonts_cn/BIZ-mimic.ttc      ← BIZ-UDMinchoM  （StaffCN = 制作名单）
  fonts_cn/LINESeed-mimic.otf ← LINESeedJP_A_OTF_Bd （JP 模式 Message）
"""
from fontTools.ttLib import TTFont, TTCollection
import os

BASE = '.'
PF = f'{BASE}/pack_fonts'
FC = f'{BASE}/fonts_cn'

# (原字体文件, 用哪个已验证字形集, 输出文件)
JOBS = [
    ('FOT-SkipProN-D.otf',   'CN-mimic.otf',        'Skip-mimic.otf'),
    ('LINESeedJP_A_OTF_Bd.otf', 'CN-mimic.otf',     'LINESeed-mimic.otf'),
    ('BIZ-UDMinchoM.ttc',    'beatfont1-mimic.otf', 'BIZ-mimic.ttf'),
]

for orig_f, glyph_src, out_f in JOBS:
    op = f'{PF}/{orig_f}'
    if orig_f.lower().endswith('.ttc'):
        o = TTCollection(op).fonts[0]
    else:
        o = TTFont(op, fontNumber=0)
    m = TTFont(f'{FC}/{glyph_src}')

    # 名称表全量移植（36/相关记录原样搬过去）
    m['name'].names = [r for r in o['name'].names]

    op2 = f'{FC}/{out_f}'
    m.save(op2)
    m.close(); o.close()
    print(f'{out_f:26} ← {orig_f:28} ({os.path.getsize(op2)/1e6:.1f} MB)')

# 校验
print('\n=== 校验 ===')
for orig_f, glyph_src, out_f in JOBS:
    o = (TTCollection(f'{PF}/{orig_f}').fonts[0] if orig_f.lower().endswith('.ttc') else TTFont(f'{PF}/{orig_f}', fontNumber=0))
    m = TTFont(f'{FC}/{out_f}')
    od = {(r.nameID, r.platformID): r.toUnicode() for r in o['name'].names}
    md = {(r.nameID, r.platformID): r.toUnicode() for r in m['name'].names}
    mismatch = [k for k in od if od[k] != md.get(k)]
    cm = m.getBestCmap()
    need = '速濑水月穗村爱凉宫遥茜鸣海孝之怎么样游泳部之星臂力如何'
    miss = [c for c in need if ord(c) not in cm]
    print(f'{out_f:26} 名称表移植={"✓ 完全一致" if not mismatch else f"✗ {mismatch[:3]}"}  字形数={len(cm)}  缺字={"无 ✓" if not miss else "".join(miss)}')
    # 名称表首条确认
    n1 = next((r.toUnicode() for r in m['name'].names if r.nameID == 1), '?')
    print(f'{"":26} nameID=1 = {n1!r}')```

**子集化**（CN-mimic / beatfont1-mimic 的制作，在 make_mimics 之前做）：用 fontTools 的 subset 功能，从 Noto Sans SC 全量里保留「游戏全部译文 + UI 串 + 人名」用到的字符集 + ASCII + 标点（我们的成品字形数 ≈ 21,765，每个 ≈ 6.5MB）。

### 9.3 fsSelection 类别对齐（出厂的第二条硬规则）

引擎对补丁字体有一条**静默校验**：**`OS/2` 表的 `fsSelection` 类别位必须与原版槽位字体一致**——不一致 = 静默拒绝 = 回退原版日文字体 = 方框。

| 槽位 | 原版类别 | 出厂 mimic 实测值 |
|---|---|---|
| Message（UDKakugo） | `0x0020`（粗体类） | `0x0020` ✓ |
| Common（beatfont1） | `0x0040`（常规类） | `0x0040` ✓ |
| Speaker / StaffCN 等 | 按原版 | 对齐 ✓ |

对齐方法（fontTools）：

```python
o = f['OS/2']
o.fsSelection = (o.fsSelection & ~0x40) | 0x20   # → 粗体类
o.fsSelection = (o.fsSelection & ~0x20) | 0x40   # → 常规类
```

### 9.4 「方框」根因：字体条目数（v19/v20 单变量实证）★

- 症状：**对话正常，但 backlog 里部分条目显示方框**；同一 backlog 里甚至逐条不同（H 场景条目方框、普通条目正常）；
- 排查过程（见问题档案 §3 全程）：从字体身份、字形、名称表一路查到「容器内字体条目数」；
- **单变量实验（从零开始，避免旧结论干扰）**：

| 版本 | 字体条目数 | 结果 |
|---|---|---|
| v18 | 5（全套：UDKakugo/beatfont1/Skip/LINESeed/BIZ） | ✗ backlog 方框 |
| v19 | 2（仅 UDKakugo + beatfont1，v5 的确切字节） | ✓ 方框消失 |
| **v20（出厂）** | **3（+BIZ 加回，片尾 staff roll 必需）** | **✓ 方框消失** |

- **根因**：容器内字体条目数过多 → **污染引擎字体池**（引擎对补丁字体池的处理有上限/干扰）；
- **出厂配置**（v20，实测值）：
  - `FOT-UDKakugo_LargePro-DB.otf` ← CN-mimic（NotoSansSC-Bold 字形集，fsSel `0x0020`）
  - `beatfont1.otf` ← beatfont1-mimic（NotoSansSC-Regular，fsSel `0x0040`）
  - `BIZ-UDMinchoM.ttc` ← BIZ-mimic（fsSel `0x0040`）
- 验收：装机实测 —— 对话 / backlog / 名字框 / staff roll 全部无方框（用户全程试玩确认）。

> **给复现者的建议**：字体层**从 3 个开始**（Message + Common + BIZ）；若你要替换 Speaker/LINESeed，**务必用单变量法验证**（一次只加一个字体条目，装进游戏看 backlog），不要一次上 5 个。

### 9.5 字体验收工具

```python
# PIL 渲染验证（比 cmap 检查可靠：有映射不代表字形非空）
from PIL import Image, ImageDraw, ImageFont
f = ImageFont.truetype('fonts_cn/CN-mimic.otf', 44)
img = Image.new('RGB', (800, 100), 'white')
ImageDraw.Draw(img).text((10, 20), '速濑水月 鸣海孝之 凉宫遥 方框测试', font=f, fill='black')
img.save('font_test.png')   # 目视检查
```

---

## 10. 与 EER 18+ 层整合（Step ⑥）

### 10.1 EER 官方补丁结构（三容器）

| 文件 | 大小 | 内容 | 处理 |
|---|---|---|---|
| `patch00d.bin` | 326,230,153 B | 图形层（916 webp / 3,712 fcd / pso·vso / png / gut） | **原样**（无文本） |
| `patch01m.bin` | 252,576,737 B | 视频层（3 个开场 ogv） | **原样**（无文本） |
| `patch00m.bin` | 31,566,760 B | 文本层（232 剧本 + 4,716 XML + 8 UI 串 + 8 JSON） | **汉化的 base**（我们的 patch00m 基于它） |

### 10.2 整合逻辑

- 我们的汉化容器 `patch00m.bin`（v20）**已经内嵌 EER 文本层**（EER 原文 + 中文翻译）；
- 装到游戏：三个 EER 文件都进 `kiminozs\obb\`；其中 `patch00m.bin` **放我们汉化版**（覆盖 EER 原版）；
- 用户若只想要 18+ 不要汉化：装 EER 原版三件套即可（我们安装器的「18+ 补丁」组件就是这个）。

### 10.3 打包与校验

```powershell
# _assemble_eer.ps1 —— 把 EER 三容器复制进交付包 + 全包清单
$src = 'F:\Rodion\Download\Kiminozo_EER_Patch_EN_Steam_1.0\obb'
$dst = 'F:\Rodion\Download\Kiminozo_EE_CN_Patch_v1.0\files\eer'
foreach ($f in @('patch00d.bin','patch01m.bin','patch00m.bin')) {
  Copy-Item (Join-Path $src $f) (Join-Path $dst $f) -Force
}
# …（清单输出略）
```

**EER 官方文件 MD5（便于与官方发布版比对）**：`patch00d` `2D9B5E84565752B024F8C56A37F0DD27` ／ `patch01m` `732E9B456FA3E30170D3E94098FF1FE8` ／ `patch00m` `3947D58877E77B58575FE0A3AD05D969`。

---

## 11. DLC 汉化（Step ⑦）

### 11.1 DLC 结构（实测）

DLC《Another Episode Collection+》是**独立目录**（`kiminoaz/`），两个容器：

| 容器 | 条目 | 内容 |
|---|---|---|
| `pack00m.bin`（原版 5.5MB） | 2,726 条 | **15 个 egpack 剧本** + 2,685 XML + 8 uistring.epk + 9 png + 9 json |
| `pack.bin`（原版 ~400MB） | 3,948 条 | 图形资源（png/webp/fcd/pso/vso/gut）+ **5 个字体** + cfg |

- DLC 剧本（15 个）：`024大空寺降臨`、`10前編：孝之編`、`11水月1`、`12茜1`、`13愛美1`、`14まゆ1`、`16蛍1`、`17慎二1`、`18遙2`、`19水月2`、`20後編：剛田編`、`30エンディング`、`30バカシナリオ`、`__speakers__`、`__staffroll__`；
- 数据量：**4,135 记录 → 3,688 唯一单元**（普通 2,302 + 成人 1,386），105,719 字符。

### 11.2 流程（与本体同构）

1. **提取**：`pack00m.bin` → 15 个 egpack → 单元库（`build_dlc_units.py`）；
2. **翻译**：agy + GPT-OSS 120B（normal 并发 6 / adult 并发 4，batch 20，约 126 条/分）；**主游戏译名预填充 444 条**（`prefill_from_main.py`——保证称呼与本体统一）；本地 qwen38-fast 兜底 ~850 条；
3. **质检（四层 + 复扫，见 11.3）**；
4. **回写**：`inject_dlc.py`（3,688/3,688 覆盖）→ **组装**：`merge_dlc.py` → `pack00m_dlc_cn.bin`；
5. **字体**：DLC 的 5 个字体**两层都替换**——补丁层（pack00m 加 5 字体条目）+ 资产层（pack.bin 内 5 字体替换）→ 无论引擎走哪条回退链都是中文字体；
6. **装机**：`kiminoaz\obb\`（备份原 pack00m.bin → 安装 → 哈希校验）。

### 11.3 质检战役（四层 + 复扫，实测数据）

```
① jp-zh 审校    37 批/3,688 条 → 310 发现（高114/中168/低28）
   ↓ 聚合 → 280 工单 → 14 批 → 264/279 应用（15 拒绝）
② 硬码验证（apply_fixes_dlc.py）
③ 复检          264 条 → 6 批 → 33 问题（12.5%）→ round2 终修 → 32/33 应用
④ EN-ZH 对照    36 批/3,504 条 → 830 问题 → 【整体回滚，见下】
⑤ qwen 段复扫   9 批/850 条（jp+en 双参照）→ 48 工单 → 48/48 应用
⑥ 人工精修      11 条（串位 3 + 词汇误读 + 主体错位 + 括号 + 术语）
```

**④ 的教训（最重要）**：DLC 官方 en 槽是**改编版**（非直译，如「ぶっとばすわよ」（揍飞你）→ "I will slice off your tongue!"）。基于 en 差异自动生成的 494 条"修复"**系统性劣化**（丢括号、词句破坏、跟随英译改错）→ **整体回滚**，改为「**JP 权威重判**」模式（明示英译改编、无误则原样输出）。

**沉淀规则**（复现者必读）：
1. 任何自动修复批**先抽样评审再应用**；
2. apply 器加**括号配对校验**（「」数量 + 顺序）；
3. **版本链字段**（prevN_zh）是回滚的前提，apply 前必写；
4. 手动修复放自动链**之后**（实测被 round2 覆盖过 1 例）。

### 11.4 DLC 产物与装机

- 终态：**3,688/3,688 = 100%**、硬码 0 不一致；
- 出厂文件：`DLC_CN_pack00m.bin` 37,256,314 B（sha `D5086639...`）→ `kiminoaz\obb\pack00m.bin`；`DLC_CN_pack.bin` 399,507,242 B（sha `92553BB2...`）→ `kiminoaz\obb\pack.bin`；
- 安装：`_install_dlc.ps1`（备份原 pack00m.bin → 安装 → 哈希校验）。

---

## 12. 装机与验收（Step ⑧）

### 12.1 装机路径总表

| 组件 | 文件 | 目标路径 |
|---|---|---|
| 18+ 补丁（EER 层） | `patch00d.bin` / `patch01m.bin` | `kiminozs\obb\` |
| 18+ 补丁 + 本体汉化 | `patch00m.bin`（汉化版） | `kiminozs\obb\`（同一位置，汉化版覆盖 EER 原版） |
| DLC 汉化 | `pack00m.bin` + `pack.bin` | `kiminoaz\obb\` |

### 12.2 装机纪律（血的教训）

1. **先备份**：目标文件存在就 `copy 原文件 原文件.bak_<日期>_v<版本>`（我们的装机脚本自动做）；
2. **装机脚本自带哈希校验**：装完立即对目标文件算 sha256 与预期比对；
3. **游戏必须先退出**（文件被占用时拷贝会失败）；
4. **装机前建议 Steam「校验文件完整性」**——保证是干净基底（也清掉历史补丁残留）；
5. **回滚 = 把 `.bak_` 备份复制回去**（保留每一版的备份，就能回退到任意版本）。

### 12.3 实机验收清单（逐项过）

- [ ] 启动游戏，**开场第一句就是中文**（实测锚点：第一章开场首句）；
- [ ] 对话正文无方框、无乱码；
- [ ] **backlog（历史回顾）逐条翻页检查**——特别是 H 场景条目（走 Common 槽，最容易方框）；
- [ ] 名字框（说话人名）中文正常；
- [ ] 菜单/按钮（UI 串）中文正常；
- [ ] 片尾 staff roll（走到结局或快进）无方框；
- [ ] 回想画廊可进入（theater 内容中文）；
- [ ] DLC（如装）进入任意章节抽查；
- [ ] 存档/读档正常（汉化不碰存档系统）。

### 12.4 常见装机问题

| 症状 | 原因 | 处理 |
|---|---|---|
| 进游戏还是英文 | 补丁没放对 / Steam 更新还原了文件 | 确认路径；重装补丁 |
| 中文显示方框 | 字体条目问题 | 查问题档案 §3 诊断树 |
| 游戏黑屏 | 容器重建细节错（如字符串表未 16 字节对齐 / 0x08 未同步） | 对拍容器字节（问题档案 §1.3） |
| 装完闪退 | 容器被破坏 | 回滚备份，重新构建 |

---

## 13. 打包发布（Step ⑨，概要）

### 13.1 单文件安装器（我们的做法）

- **技术**：C# 原生 exe（.NET Framework，Win10/11 自带运行时）+ **尾部附着全量载荷**（运行时流式读取自身，零临时解压）；
- **功能**：自动检测游戏目录（Steam 注册表 + libraryfolders.vdf + 常见路径兜底）；三组件独立勾选（DLC 未装则禁用并引导去 Steam）；**哈希级检测已装状态**（避免重复安装）；安装前自动备份 + 安装后哈希校验；**一键卸载（还原备份）**；
- **界面**：浅色主题；首屏许可（不同意禁止安装）；进度/结果如实显示。
- （安装器源码与构建流水线细节：见问题档案 §6；如需可直接复用我们的工程。）

### 13.2 GitHub 发布（要点）

- 仓库 = README（说明 + 技术记录 + 校验表 + 致谢）；
- **安装器 exe 与全部补丁文件都作为 Release 资产**（GitHub 单文件 >100MB 不能进 git 仓库，但可以作为 Release 资产上传）；
- 上传后**逐文件 SHA256 对拍**（字节级校验通过才算发布完成）。

---

## 附录 A：复现检查清单（照着勾）

**环境**
- [ ] 游戏本体安装（Steam）+ 运行一次确认能玩
- [ ] decryptKey.bin 拿到（64KB，首字节 `461f3c0876 0f128d`）并**复制进项目目录**
- [ ] Python 3 + fontTools
- [ ] EER 官方包下载（如需 18+ 层）

**提取与单元**
- [ ] corpus 提取：本体 220 + EER 232 = 452 ✓
- [ ] corpus_manifest.json 452 条（base 在前、eer 在后）
- [ ] units.jsonl 79,256 单元（75,379 + 3,877）

**翻译**
- [ ] SYS 提示词全文（术语表 + 控制符规则）✓
- [ ] 试跑 `--limit 100` 看硬码合规率（应 ≥80%）
- [ ] 全量跑完（断点续跑保护）
- [ ] fix_codes 修复 + fix_ws 归一化 → 硬码 100%
- [ ] 全库扫描：零假名 / 零繁体残留

**回写与构建**
- [ ] inject 回写（回读断言 N/N 通过）
- [ ] merge 容器（回读自检 ✓）
- [ ] 用**结构式解析器**做全部重建（双检验：往返逐字节 + 幂等）
- [ ] 容器版本记录（size + sha256）
- [ ] （可选）回想解锁模块注入 → 装机后回想画廊全开

**字体**
- [ ] 5 槽字体身份 + fsSel 检查
- [ ] mimic 生成（名称表移植 + 子集化）
- [ ] **单变量实验**：先 2-3 字体装机验证，再加
- [ ] backlog 逐条翻页检查（含 H 场景）

**整合与装机**
- [ ] EER 三容器就位（MD5 对拍）
- [ ] 装机（备份 + 哈希校验）
- [ ] 实机验收清单全过（§12.3）

**DLC（可选）**
- [ ] DLC 15 剧本提取 + 单元 3,688
- [ ] 翻译 + 四层质检（抽样评审！）
- [ ] 回写 + 组装 + 双容器字体
- [ ] 装机 + 抽查

**发布（可选）**
- [ ] 安装器打包（或手动安装脚本）
- [ ] Release 上传 + 逐文件哈希校验

---

## 附录 B：关键数据表（全部实测）

### B.1 出厂文件校验表（与 Release 一致）

| 文件 | 大小（字节） | SHA256 |
|---|---|---|
| `Kiminozo_EE_CN_Installer_v1.0.exe` | 1,100,177,375 | `3FC8E397B1834F65065B954045FDDC1567C60A14371C825FC3CD15BFE10FBF03` |
| `Kiminozo_CN_patch00m.bin`（本体汉化 v20） | 52,435,254 | `EAF2B9924482EE533722F4D0502C656A9424C0C5F6A7BE4840D66DBF5154F8A5` |
| `EER_patch00d.bin` | 326,230,153 | `DB9DC0168A159C88F677931FDFD4BEB9ABC03EB61DDFBCA9206FB0EB61D155CD` |
| `EER_patch01m.bin` | 252,576,737 | `9F279AB449F205AAEEDB6DB0D294677A82FE5111965C4FF6B08FBCD37708D1C0` |
| `EER_patch00m.bin`（原版） | 31,566,760 | `F8B294ACB7AB2A7D8CBFAF59C1D55A8BC7BE6DDC1D026B8DB8E2A4231A8A893F` |
| `DLC_CN_pack.bin` | 399,507,242 | `92553BB2E2023750899FB6F3B878A464BE763017F4B4875BEDCAC7FA6ABE4FB0` |
| `DLC_CN_pack00m.bin` | 37,256,314 | `D5086639409551B0CD8D2C334E60BA42F16CE11DD339001DA14BD7A521EC4452` |

### B.2 容器版本链（本体 patch00m）

| 版本 | 大小（B） | 字体数 | 说明 |
|---|---|---|---|
| v5（测试） | ~13.65MB | 2 | 首通实机（基础汉化） |
| v12 | 64,419,635 | 5 | 全量翻译完成版 |
| v13 | 64,422,806 | 5 | +705 条复检修复（sha `FBF3C186...`） |
| v18 | 64,785,026 | 5 | 字体实验版（方框） |
| v19 | 45,905,939 | 2 | 单变量验证：方框消失 |
| **v20（出厂）** | **52,435,254** | **3** | **+BIZ 加回（staff roll）；方框全消** |

### B.3 翻译数据

| 项 | 数值 |
|---|---|
| 记录 → 单元 | 114,329 → 79,256（去重 30.7%） |
| 普通 / 成人 | 75,379（223 万字符）/ 3,877（11.7 万字符） |
| 复检覆盖 | 26,700/75,379（35.4%）→ 671 发现（高 18 / 中 173 / 低 480） |
| 修复应用 | 705 key / 934 记录（93 个 egpack） |
| DLC | 4,135 记录 → 3,688 单元；四层质检；3,688/3,688 |

---

## 附录 C：术语表（完整，与提示词一致）

**人名**：鳴海孝之→鸣海孝之 ｜ 速瀬水月→速濑水月 ｜ 涼宮遙→凉宫遥（强制简体「遥」）｜ 涼宮茜→凉宫茜 ｜ 平慎二→平慎二 ｜ 穂村愛美→穗村爱美 ｜ 玉野まゆ→玉野真由 ｜ 大空寺あゆ→大空寺亚由 ｜ 天川蛍→天川萤 ｜ 星乃文緒→星乃文绪 ｜ 香月モトコ→香月素子（医生）｜ マナマナ→真奈真奈 ｜ 千鶴→千鹤 ｜ 勲→勋 ｜ 健さん→健哥（店长）｜ 真智子→真智子

**称呼**：さん→同学/女士·先生（按场景）｜ くん→君 ｜ ちゃん→小〜 ｜ 先輩→前辈 ｜ 先生→老师/医生 ｜ 様→小姐/大人 ｜ 呼び捨て→直呼其名

**专有名词（拍板）**：餐厅名=**天空神殿** ｜ 文绪昵称=**文绪酱** ｜ 绘本名=**玛雅乌尔** ｜ 大川→**天川**（错字修正）｜ 肥兽→**胖兽**

**称谓政策**：不做全局一刀切（同一角色不同场景称呼不同是合法的）；全局扫描只用于发现可疑偏差。

**18+ 术语**：中に出す→内射 ｜ 外に出す→外射 ｜ アナル→后庭 ｜ アソコ→私处（露骨处可「小穴」）｜ 飲ませる→吞精 ｜ おっぱい→胸部（露骨处可「奶子」）｜ イく→高潮 ｜ 膣→阴道 ｜ クリトリス→阴蒂 ｜ モノ等→肉棒/阴茎（**严禁**生造婉辞）

---

## 附录 D：脚本清单（管线全景）

| 脚本 | 作用 | 本文位置 |
|---|---|---|
| `fpd_tool.py` | 容器读写核心工具 | §4.4 |
| （结构式解析器） | 重建正确做法 | §4.5 |
| `build_units.py` | 单元库构建 | §6.3 |
| `translate.py` | 翻译引擎（4 通道） | §7.3 |
| `fix_codes.py` | 控制符修复 | §7.5 |
| `fix_ws.py` | 空白符归一化 | §7.5 |
| `inject.py` | 译文回写 | §8.2 |
| `merge_obb.py` | 容器合并 | §8.3 |
| `_build_v13_surgical.py` | 外科手术式重建 | §8.4 |
| `make_mimics.py` | 字体拟态 | §9.2 |
| DLC 管线（约 30 个脚本） | 同构流程 + 四层质检 | §11（清单见问题档案附录） |

---

> **下一步**：请通读配套的《君望EE_汉化问题与解决_全档案.md》——那里有本文档提到的每一个坑的完整现场（现象 → 根因 → 解法 → 教训），以及全部**已证伪路线**（帮你省下最多时间的就是它）。
>
> 祝顺利。 — 幻灭文学出版社 · 2026-09
