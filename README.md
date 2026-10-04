# video-watch

**中文** | [English](README.en.md)

把视频、GIF、录屏翻译成**可核算的图版序列**，让只有单帧视觉能力的模型也能给出带时间戳的观看结论。

## 它解决什么

让模型"看视频"通常有两种便宜做法，各有毛病：

- **均匀抽几帧** —— 快速动作整段漏掉，抽几帧全凭手感，覆盖有没有盲区说不清；
- **逐帧喂进去** —— token 爆炸，且绝大多数帧是重复信息。

video-watch 的做法是分工：

> **ffmpeg 看全部帧（零 token），模型只看几十帧（预算受控），两者靠时间戳缝合。**

工具负责量测与取证：切点、冻结段、静音段、逐秒运动量、像素突变、逐帧差分、连通域。
模型负责看图与命名。中间靠一份 **manifest** 对接：第几行第几列 = 第几秒 = 第几帧。

一句话判据：**量测给候选，眼睛给命名。**

## 安装

### 必需

| 依赖 | 版本/要求 | 说明 |
|---|---|---|
| Python | 3.8+ | 主链路只用标准库 |
| ffmpeg + ffprobe | **完整构建，≥ 5.1** | 必须含 `scdet` `freezedetect` `silencedetect` `tblend` `signalstats` `tile` `drawtext`；`vwtools` 还需要 `-fps_mode`（5.1 起，`-vsync` 已被 9.0 移除） |
| ImageMagick | 6.x 或 7.x | 拼贴图版与差分合成 |
| TrueType 字体 | | `drawtext` 烧帧号索引；各平台通常自带 |

其中 **`drawtext` 需要 ffmpeg 编译时带 libfreetype**，部分发行版与精简构建不带 —— 缺它时图版烧不上索引，所以工具在启动时就会校验并明确报错。

**下面三组命令按你的系统选一组执行，不是依次全跑**：

```bash
# Windows
winget install Gyan.FFmpeg
winget install ImageMagick.ImageMagick

# macOS
brew install ffmpeg imagemagick

# Debian / Ubuntu
sudo apt install ffmpeg imagemagick fonts-dejavu-core
```

### 可选

```bash
pip install -r requirements.txt        # pillow（gif 子命令）+ numpy（vwtools 专用脚本）
pip install -r requirements-asr.txt    # faster-whisper，asr 子命令
pip install -r requirements-ocr.txt    # rapidocr-onnxruntime，ocr 子命令
```

可选依赖都是函数内延迟导入：**不装不影响主链路**。

### 初始化

```bash
python vw.py doctor     # 逐项自检：报出每个程序的状态，缺什么、怎么补
python vw.py init       # 把探测到的路径写成 vw.config.json（已存在则拒绝覆盖）
python vw.py doctor     # 再跑一次，应报"必需项就绪"
```

## 快速开始

```bash
python vw.py probe clip.mp4 --out work/           # 量测全片，产出 timeline.json
python vw.py plan work/timeline.json              # 采样计划 → shotlist.json
python vw.py grid --media clip.mp4 --frames 25 --cols 5 --out work/   # 一张图粗看全片
python vw.py read --media clip.mp4 --times "2,5" --region "100,100,600,400" --out work/read/  # 读字（时刻与区域按你的素材调）
python vw.py report work/timeline.json work/shotlist.json --manifest work/manifest.json --out 报告.md
```

`report` 产出交付骨架（量测段落表、位置映射、边界声明），"完整描述"一节由模型看图后填写。

## 子命令

| 子命令 | 作用 | 产出 |
|---|---|---|
| `probe` | 量测：切点/冻结/运动/静音，并给出内容判型与口径提示 | `timeline.json` |
| `plan` | 骨架优先的采样计划（保证覆盖盲区上界） | `shotlist.json` |
| `sheet` | 联络表图版；支持 `--diff` 差分与 `--strip` 定窗条带 | 图版 + `manifest.json` |
| `gif` | 动图按帧延迟归一化成真实时间轴 | 图版 + `manifest.json` |
| `grid` | N 帧铺一张图（3×3 / 5×5 / 6×6），粗看与读动作的主力档位 | `grid.png` + `manifest.json` |
| `seq` | 按帧号排序的区域序列，读运动的主通道 | 条带 + `manifest.json` |
| `read` | 按可读分辨率切面板，读字的主通道 | 面板图 + `manifest.json` |
| `ocr` | 本地 OCR，仅作"哪里有字"的索引（不推荐当读字手段） | `ocr.json` |
| `asr` | 语音转写（faster-whisper），带时间戳并挂到图版位置 | `asr.json` |
| `report` | 生成观看报告骨架 | Markdown |
| `doctor` | 依赖自检 | 状态报告 |
| `init` | 把探测到的依赖路径写成配置文件 | `vw.config.json` |

## 配置文件

按以下顺序取第一个存在的：

1. `--config <路径>`
2. `VW_CONFIG` 环境变量
3. 当前目录 `vw.config.json`
4. `~/.config/video-watch/vw.config.json`

配置只提供**默认值**，优先级为：
**命令行参数 > 环境变量 > 配置文件 > 自动探测 > 内置默认**。

```json
{
  "ffmpeg": "/path/to/ffmpeg",
  "ffprobe": "/path/to/ffprobe",
  "magick": "/path/to/magick",
  "font": "/path/to/font.ttf",
  "outdir": "out",
  "defaults": { "skeleton": 16, "max": 36, "cols": 5, "panel": "medium" }
}
```

| 键 | 作用 |
|---|---|
| `ffmpeg` / `ffprobe` / `magick` / `font` | 外部程序与字体路径，填绝对路径；**留空则自动探测（推荐）** |
| `outdir` | 所有输出的根目录；留空时多数子命令落到当前目录下的 `vw_*\` 子目录（`probe` → `vw_out\`，`sheet` → `vw_sheets\`），但 **`plan` 落在时间轴同目录**、**`asr` 落在素材旁** |
| `defaults.*` | 子命令默认参数，省得每次手敲 |

**四个路径键通常留空即可** —— 留空就按环境变量与内置候选自动探测，`vw.py init` 还会把探测到的路径直接填好。
只有自动探测失败时（比如 PATH 里的 ffmpeg 是精简构建）才手填绝对路径：
`/usr/bin/ffmpeg`（Linux）、`/opt/homebrew/bin/ffmpeg`（macOS）、`D:\ffmpeg\bin\ffmpeg.exe`（Windows）。

临时覆盖用环境变量：`VW_FFMPEG`、`VW_FFPROBE`、`VW_MAGICK`、`VW_FONT`、`VW_CONFIG`。

## 设计要点

**视觉预算是第一约束。** 一张 756×756 的图计费约 346 token（单图上限 384），买 324 个网格格。
所以"拼版换覆盖率"和"裁切换分辨率"是单向兑换关系 —— 要么看全，要么看清，物理上不可兼得。
粗看全片用一张 5×5 比用三张图省 2/3 成本，为什么不省。

**分辨率与时序密度是两个独立维度。** 密度决定"怎么变的"，分辨率只决定"是什么"。
连续 36 帧铺 6×6（每格仅 126×70）照样读得出动作链，而 0.25s 稀疏采样的 3×3 会把同一段动作压扁。

**索引烧进图里，但权威表在 manifest。** 图内只烧大号帧号供人眼核对；
"第几行第几列 → 第几秒"只信 manifest，绝不让视觉印象决定先后顺序。

**两块硬骨头写进了工具而不是文档：** 骨架采样保证盲区有上界（不靠手感）；
低运动/高运动自动切换采样体制（文档类内容用切点检测是答非所问）。

## 已知边界

- 运动靠**按帧号排序的帧序列**读，不是靠感觉；真限制是**时间混叠**（间隔 4 帧 ≈ 看清 0.1~0.5 秒级动作）
- 小字必须裁切放大才能读；**整帧或图版里直接"读出"的文字可能是编的**（分辨率不足时视觉不报"读不到"，它渲染一个合理内容）
- OCR / ASR 都是机器输出，同音错字常见；有画面时以画面为准
- "全片覆盖"与"看清每个字"不可兼得

更完整的判据、档位选择与踩坑记录见 `SKILL.md`。

## 许可

MIT，见 `LICENSE`。
