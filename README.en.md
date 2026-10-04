# video-watch

[中文](README.md) | **English**

Turns video, GIFs, and screen recordings into an **accountable sequence of contact sheets**, so a model with
single-frame vision can still deliver timestamped conclusions about what happened.

## The problem it solves

Letting a model "watch" a video usually comes down to two cheap approaches, each with its own flaw:

- **Sample a handful of frames evenly** — fast motion is missed entirely, the frame count is a guess,
  and nobody can say whether the coverage has blind spots;
- **Feed in every frame** — token costs explode, and most frames carry no new information.

video-watch splits the work instead:

> **ffmpeg looks at every frame (zero tokens). The model looks at a few dozen (bounded budget). The two are stitched together by timestamps.**

The tool handles measurement and forensics: scene cuts, freeze segments, silence, per-second motion,
pixel deltas, frame-by-frame diffs, connected components. The model handles looking and naming.
They meet at a **manifest**: row and column → second → frame number.

The one-line criterion: **measurement nominates candidates, eyes assign names.**

## Install

### Required

| Dependency | Version / requirement | Notes |
|---|---|---|
| Python | 3.8+ | the core pipeline uses the standard library only |
| ffmpeg + ffprobe | **full build, ≥ 5.1** | must include `scdet` `freezedetect` `silencedetect` `tblend` `signalstats` `tile` `drawtext`; `vwtools` also needs `-fps_mode` (available since 5.1 — `-vsync` was removed in 9.0) |
| ImageMagick | 6.x or 7.x | tiling contact sheets, diff compositing |
| TrueType font | | `drawtext` burns in frame indices; usually present already |

**`drawtext` requires an ffmpeg compiled with libfreetype**, and some distributions' builds leave it out.
Without it, contact sheets come out with no burned-in indices, and the failure surfaces at the frame-extraction
step rather than at the real cause — so the tool validates it at startup and says so plainly.

**Pick the group matching your OS — these are alternatives, not a sequence:**

```bash
# Windows
winget install Gyan.FFmpeg
winget install ImageMagick.ImageMagick

# macOS
brew install ffmpeg imagemagick

# Debian / Ubuntu
sudo apt install ffmpeg imagemagick fonts-dejavu-core
```

### Optional

```bash
pip install -r requirements.txt        # pillow (gif subcommand) + numpy (vwtools helper scripts)
pip install -r requirements-asr.txt    # faster-whisper, asr subcommand
pip install -r requirements-ocr.txt    # rapidocr-onnxruntime, ocr subcommand
```

Optional dependencies are imported lazily inside functions: **skipping them does not affect the core pipeline.**

### Initialize

```bash
python vw.py doctor     # per-item self-check: status of each program, what is missing, how to fix it
python vw.py init       # write the detected paths to vw.config.json (refuses to overwrite an existing file)
python vw.py doctor     # run again; every required item should now be reported as ready
```

> Note: the CLI's own messages are currently in Chinese. Command names, options, and the JSON artifacts are
> language-neutral, and `SKILL.md` is the full Chinese-language manual.

## Quick start

```bash
python vw.py probe clip.mp4 --out work/           # measure the whole clip -> timeline.json
python vw.py plan work/timeline.json              # sampling plan -> shotlist.json
python vw.py grid --media clip.mp4 --frames 25 --cols 5 --out work/   # one image, whole clip
python vw.py read --media clip.mp4 --times "2,5" --region "100,100,600,400" --out work/read/   # read text (adjust times/region to your clip)
python vw.py report work/timeline.json work/shotlist.json --manifest work/manifest.json --out report.md
```

`report` produces a delivery skeleton (measurement table, position mapping, boundary statement).
The "full description" section is filled in by the model after looking at the sheets.

## Subcommands

| Subcommand | What it does | Output |
|---|---|---|
| `probe` | measurement: cuts, freezes, motion, silence, plus a content-type verdict and signal guidance | `timeline.json` |
| `plan` | skeleton-first sampling plan (with a guaranteed bound on blind spots) | `shotlist.json` |
| `sheet` | contact sheets; supports `--diff` and `--strip` region strips | sheets + `manifest.json` |
| `gif` | normalizes frame delays into a real timeline | sheets + `manifest.json` |
| `grid` | N frames tiled into one image (3×3 / 5×5 / 6×6) — the workhorse for overview and motion | `grid.png` + `manifest.json` |
| `seq` | region sequence ordered by frame number — the main channel for reading motion | strips + `manifest.json` |
| `read` | cuts panels at a readable resolution — the main channel for reading text | panels + `manifest.json` |
| `ocr` | local OCR, as an index of *where* text is (not recommended as a way to read it) | `ocr.json` |
| `asr` | speech transcription (faster-whisper), timestamped and attached to sheet positions | `asr.json` |
| `report` | builds the viewing report skeleton | Markdown |
| `doctor` | dependency self-check | status report |
| `init` | writes detected dependency paths to a config file | `vw.config.json` |

## Configuration

The first file that exists, in this order, is used:

1. `--config <path>`
2. the `VW_CONFIG` environment variable
3. `./vw.config.json` (project-level)
4. `~/.config/video-watch/vw.config.json` (user-level)

The config only supplies **defaults**. Precedence:
**command line > environment variables > config file > auto-detection > built-in defaults**.

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

| Key | Purpose |
|---|---|
| `ffmpeg` / `ffprobe` / `magick` / `font` | external program and font paths — absolute paths, or **leave empty to auto-detect (recommended)** |
| `outdir` | root directory for all output; when empty most subcommands write to a `vw_*` folder under the current directory (`probe` → `vw_out/`, `sheet` → `vw_sheets/`), but **`plan` writes next to the timeline** and **`asr` next to the media** |
| `defaults.*` | per-subcommand default arguments, so you stop typing them |

**In practice you can leave all four path keys empty** — they auto-detect from the environment and built-in
candidates, and `vw.py init` writes the detected paths in for you. Fill them by hand only when auto-detection
fails (for example, when the ffmpeg on your PATH is a stripped build), using an absolute path:
`/usr/bin/ffmpeg` (Linux), `/opt/homebrew/bin/ffmpeg` (macOS), `D:\ffmpeg\bin\ffmpeg.exe` (Windows).

For one-off overrides use the environment: `VW_FFMPEG`, `VW_FFPROBE`, `VW_MAGICK`, `VW_FONT`, `VW_CONFIG`.

## Design notes

**The vision budget is the first constraint.** A 756×756 image costs roughly 346 tokens (384 is the per-image
ceiling) and buys 324 grid cells. "More tiles for coverage" and "crop for resolution" therefore trade off along
a single axis — you can see everything, or see it clearly, but not both. For the same whole-clip overview,
one 5×5 sheet costs 2/3 less than three separate images — which is why it is the default entry point.

**Resolution and temporal density are independent axes.** Density decides *how it changed*; resolution decides
*what it is*. Thirty-six consecutive frames tiled 6×6 (126×70 per cell) still reveal an action chain, while a
3×3 sampled every 0.25s flattens the same motion into a vague "swipe".

**Indices are burned into the image, but the authoritative table lives in the manifest.** The image carries only
large frame numbers, for human cross-checking. "Row and column → second" comes from the manifest alone;
a visual impression never decides ordering.

**Two hard problems are solved in the tool rather than in the documentation:** skeleton sampling guarantees a
bound on blind spots instead of leaving coverage to intuition, and the sampling regime switches automatically
between low-motion and high-motion content (scene-cut detection is simply the wrong tool for a static document).

## Known limits

- Motion is read from **frame sequences ordered by frame number**, not from impressions. The real limit is
  **temporal aliasing**: a 4-frame gap resolves roughly 0.1–0.5s actions.
- Small text must be cropped and enlarged to be read. **Text "read" straight off a full frame or a contact sheet
  may be fabricated** — at insufficient resolution, vision does not report "cannot read"; it renders something plausible.
- OCR and ASR are machine output with frequent homophone errors. When there is an image, the image is authoritative.
- "Cover the whole clip" and "read every character" cannot both be had.

The full criteria, tier selection, and pitfall log live in `SKILL.md` (Chinese).

## License

MIT, see `LICENSE`.
