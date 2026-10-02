# AGENTS.md · 安装指引（写给智能体）

本文件讲两件事：**把这个技能包装进你的技能目录**，以及**把它的依赖装到能用**。
子命令用法见 `SKILL.md`，给人看的说明见 `README.md`，此处不重复。

## 一、把技能包装上

入口是带 frontmatter 的 `SKILL.md`。把**整个目录**放进宿主的技能目录：

| 宿主 | 放哪 |
|---|---|
| DSH | `.dsh/skills/video-watch/` |
| Claude Code | `~/.claude/skills/video-watch/` |
| 其他 | 查该宿主的技能目录约定；只要它能读到 `SKILL.md` 就成立 |

目录名用 `video-watch`，与 `SKILL.md` 里的 `name:` 字段一致。

验证：宿主能在技能列表里列出它，并且 `description` 显示得出来。

## 二、把依赖装到能用

**技能装上了不等于能用** —— 依赖没齐时它照样被加载，但一跑就报错。
按 `SKILL.md` 的「安装」一节补齐依赖，这里只给自检流程。

```bash
python vw.py doctor     # 逐项报告缺什么
# 按提示补齐，直到必需项全绿
python vw.py init       # 把探测到的路径写成配置文件
python vw.py doctor     # 复查
```

手边没有合适素材时，**用 ffmpeg 现造一个**（它本来就是必需依赖，不用另找）：

```bash
ffmpeg -f lavfi -i "testsrc2=size=1280x720:rate=30:duration=9" -f lavfi -i "sine=frequency=440:duration=9" -c:v libx264 -pix_fmt yuv420p -c:a aac -shortest -y smoke.mp4
```

9 秒 / 30fps / **270 帧**，带音频，足够跑通 `probe` / `plan` / `grid` / `seq` / `read` 与 `sheet --strip`。
拿它当基准可以对预期：`probe` 应报「帧 270  实际帧率 30.00」；
`grid --frames 9 --cols 3` 的产物里应看得见烧上去的帧号索引 —— 看不到就是字体没生效（见 FAQ 3）。
若该构建没有 `testsrc2`，把第一个 `-i` 换成 `-f lavfi -i testsrc`。

## FAQ

### 1. PATH 是「谁」的 PATH

如果宿主进程**已经在运行**，它继承的是启动时的旧环境块。你把 ffmpeg 装好、把路径写进环境变量，
**它当场看不到** —— 于是 `doctor` 报找不到 ffmpeg，而你在自己的 shell 里试又是好的。

对策：让它读**每次现读的配置文件**（`python vw.py init` 写出的 `vw.config.json`），
或先用 `VW_FFMPEG` 指向绝对路径。别只写环境变量就以为完事。

### 2. 缺 `drawtext` 的报错**出现在抽帧那一步**

ffmpeg 需要这七个滤镜：`scdet` `freezedetect` `silencedetect` `tblend` `signalstats`
`tile` `drawtext`。

其中 `drawtext` 依赖编译时的 libfreetype，部分构建不带。缺它时，`probe` / `plan` 一切正常，
直到 `grid` / `sheet` / `seq` 才失败，而报错落在**抽帧**那一步，看起来与字体毫无关系。

对策：`doctor` 会逐个校验，以它为准；别顺着抽帧的报错去查抽帧。

### 3. 字体可能**静默失效**

`drawtext` 的字体路径格式不对时，ffmpeg **不一定报错** —— 它会退回默认字体。
表现是：退出码 0、图版照常生成、索引清晰可读，**但那不是你指定的字体**。
配置里写了 `font` 却看不出效果时，先怀疑这一条。

判据：**直接渲染两次**（一次用目标字体、一次用一个保证不存在的路径），产物相同即说明
`fontfile` 没被读。跑 `doctor` 更省事，它的字体项内部就是这么比的。

注意**别用 `VW_FONT` 去设那个坏路径**来复现：`find_font()` 有存在性检查，会跳过它回落到
下一个候选，两次渲染其实是同一个字体，判据会给**假阳性**。

### 4. ImageMagick 官方只发 `.7z`

官方 Release 的 `.7z` 用了 BCJ2 过滤器，`py7zr` 解不开，需要官方独立版 `7zr.exe`。

## 装可选依赖 `asr` 时的三处坑

- 直连 HuggingFace 超时，需要镜像；
- 走镜像仍可能 401，是 Xet 传输域名不可达，需设 `HF_HUB_DISABLE_XET=1`；
- `av` 必须 `<19` —— faster-whisper 1.2.1 用了 PyAV 19 已移除的 `metadata_errors`。

> 上面第 4 条与本节三条来自上游 issue #3 的部署记录，**开发者尚未复现验证**，放置于此供参考。

## 完成标准

- 宿主技能列表里能看到它
- `doctor` 报「必需项就绪」
- 真实素材跑通 `probe` 与 `grid`，图版里能看见烧上去的帧号索引
