# Compressing iPhone Screen Recordings for Discord with FFmpeg

A beginner-friendly guide for getting a large `.mov` screen recording (music video or otherwise) down to Discord's **8 MB free-tier limit** while keeping audio at CD quality.

---

## Prerequisites

### Install FFmpeg

**macOS (Homebrew):**
```bash
brew install ffmpeg
```

**Windows (winget):**
```powershell
winget install ffmpeg
```

**Ubuntu / Debian:**
```bash
sudo apt update && sudo apt install ffmpeg
```

Verify it worked:
```bash
ffmpeg -version
```

---

## Step 1 — Inspect Your File First

Before touching the file, learn what you're working with:

```bash
ffprobe -v quiet -print_format json -show_format -show_streams your_file.mov
```

The key numbers to note:
- **`duration`** — length in seconds (you need this for the math below)
- **`bit_rate`** — original bitrate in bits/second
- **`codec_name`** — usually `h264` or `hevc` for iPhone recordings
- **`width` / `height`** — resolution (likely 1920×1080 or 2532×1170 for newer iPhones)

---

## Step 2 — Understand the Math

Discord's free tier allows **8 MB** uploads. The total bitrate budget for your output file is:

```
total_bitrate_kbps = (target_size_MB × 8192) ÷ duration_seconds
video_bitrate_kbps = total_bitrate_kbps − audio_bitrate_kbps
```

**Example — 3-minute (180 s) clip, 128 kbps audio:**
```
total  = (8 × 8192) ÷ 180 = 364 kbps
video  = 364 − 128        = 236 kbps
```

**Example — 5-minute (300 s) clip:**
```
total  = (8 × 8192) ÷ 300 = 218 kbps
video  = 218 − 128        = 90 kbps   ← very low; drop to 480p
```

> **Rule of thumb:** if your video bitrate math comes out below ~150 kbps, drop the resolution to 480p. Below ~80 kbps, go to 360p.

---

## Step 3 — Choose Your Codec

The codec is the single biggest lever you have. Newer codecs compress more efficiently, so you get better quality at the same file size.

| Codec | Flag | Compression | Compatibility | Speed |
|-------|------|-------------|---------------|-------|
| H.264 | `libx264` | Baseline | Plays everywhere | Fast |
| H.265 / HEVC | `libx265` | ~40–50% better than H.264 | Most modern devices | Medium |
| AV1 | `libaom-av1` | ~30% better than H.265 | Newer browsers/Discord | Very slow |

**For Discord sharing, H.265 is the best trade-off** — Discord's desktop and web clients both support it, and you get significantly better quality at the same file size vs H.264.

---

## Step 4 — Audio Settings

For **CD-quality audio** (suitable for music), use AAC at 128 kbps. This is the sweet spot — indistinguishable from CD for most listeners on typical speakers/earbuds.

```bash
-c:a aac -b:a 128k
```

If the source audio is already AAC (common on iPhone), you can copy it without re-encoding to avoid any quality loss — **only do this if you are not changing the container format and quality is the top priority**:
```bash
-c:a copy
```

> Do **not** go below 96 kbps for music. Below that, compression artifacts (warbling, muddiness) become clearly audible.

---

## Step 5 — Encode

### Option A: Two-Pass Encoding (most accurate file size)

Two-pass is the most reliable way to hit a target size. Pass 1 analyzes the video; Pass 2 uses that analysis to hit your bitrate exactly.

Replace `VIDEO_BITRATE` with your calculated value from Step 2.

**macOS / Linux:**
```bash
# Pass 1 (analysis only — no output file)
ffmpeg -y -i input.mov \
  -c:v libx265 -b:v VIDEO_BITRATEk \
  -x265-params pass=1 \
  -an -f null /dev/null

# Pass 2 (actual encode)
ffmpeg -i input.mov \
  -c:v libx265 -b:v VIDEO_BITRATEk \
  -x265-params pass=2 \
  -vf scale=-2:720 \
  -c:a aac -b:a 128k \
  output.mp4
```

**Windows (PowerShell):**
```powershell
# Pass 1
ffmpeg -y -i input.mov `
  -c:v libx265 -b:v VIDEO_BITRATEk `
  -x265-params pass=1 `
  -an -f null NUL

# Pass 2
ffmpeg -i input.mov `
  -c:v libx265 -b:v VIDEO_BITRATEk `
  -x265-params pass=2 `
  -vf scale=-2:720 `
  -c:a aac -b:a 128k `
  output.mp4
```

---

### Option B: CRF Encoding (best quality, no size guarantee)

CRF (Constant Rate Factor) targets a quality level rather than a bitrate. Good for when you want the best possible quality and will check the size afterward.

- **H.265 CRF scale:** 0 (lossless) → 51 (worst). **18–28 is the practical range.** Start at 24 and adjust.
- Lower number = better quality = larger file.

```bash
ffmpeg -i input.mov \
  -c:v libx265 -crf 24 \
  -vf scale=-2:720 \
  -c:a aac -b:a 128k \
  output.mp4
```

Check output size with `ls -lh output.mp4`. If too big, raise CRF by 2–3 and re-run.

---

## Quick-Reference Recipes

All examples use H.265 and 128 kbps AAC. Swap `libx265` → `libx264` if you need maximum compatibility (older devices, slower machines).

### Short clip (< 2 min) → 8 MB, 720p
```bash
ffmpeg -i input.mov -c:v libx265 -b:v 400k -x265-params pass=1 -an -f null /dev/null && \
ffmpeg -i input.mov -c:v libx265 -b:v 400k -x265-params pass=2 -vf scale=-2:720 -c:a aac -b:a 128k output.mp4
```

### Medium clip (2–4 min) → 8 MB, 480p
```bash
ffmpeg -i input.mov -c:v libx265 -b:v 180k -x265-params pass=1 -an -f null /dev/null && \
ffmpeg -i input.mov -c:v libx265 -b:v 180k -x265-params pass=2 -vf scale=-2:480 -c:a aac -b:a 128k output.mp4
```

### Long clip (4–8 min) → 8 MB, 360p
```bash
ffmpeg -i input.mov -c:v libx265 -b:v 70k -x265-params pass=1 -an -f null /dev/null && \
ffmpeg -i input.mov -c:v libx265 -b:v 70k -x265-params pass=2 -vf scale=-2:360 -c:a aac -b:a 128k output.mp4
```

### Just shrink it, don't care about exact size (CRF, 720p)
```bash
ffmpeg -i input.mov -c:v libx265 -crf 26 -vf scale=-2:720 -c:a aac -b:a 128k output.mp4
```

---

## Tips for Music Video Content

- **Prioritize audio bitrate.** A blurry frame is forgiven; muffled audio ruins the experience. Always allocate 128 kbps to audio before calculating what's left for video.
- **Scale before bitrate.** Reducing from 1080p to 720p costs nothing in quality perception on a Discord embed (it plays in a small player). That resolution drop alone can halve your video bitrate requirement.
- **Use `-vf scale=-2:HEIGHT`** not `-vf scale=WIDTH:HEIGHT`. The `-2` keeps the aspect ratio correct and ensures dimensions are divisible by 2 (required by most codecs).
- **Trim dead air.** If the recording has silence before/after the music, cut it:
  ```bash
  ffmpeg -i input.mov -ss 00:00:03 -to 00:04:12 -c copy trimmed.mov
  ```
  Then compress `trimmed.mov`. Less duration = more bits per second for the same file size.
- **Check your output before sharing:**
  ```bash
  ffprobe -v quiet -show_format output.mp4 | grep size
  ```

---

## Realistic Expectations for a 600 MB Source

A 600 MB iPhone screen recording at standard quality is roughly **3–8 minutes** of footage. Going to 8 MB is a **75× compression ratio** — video quality *will* be noticeably reduced. Here's what to expect:

| Duration | Resolution | Video Quality |
|----------|------------|---------------|
| 2 min | 720p | Acceptable, some motion blur |
| 4 min | 480p | Watchable, soft |
| 6 min | 360p | Low quality, OK for reference |
| 8+ min | 360p | Poor video — consider trimming or hosting elsewhere |

For anything over 5 minutes, consider hosting on YouTube (unlisted) or Google Drive and dropping a link in Discord instead of attaching the file.

---

## Troubleshooting

**`libx265` not found:**
Your ffmpeg build doesn't include H.265. Install a full build:
- macOS: `brew install ffmpeg` (Homebrew's build includes it by default)
- Windows: download the "full" build from [ffmpeg.org/download.html](https://ffmpeg.org/download.html)

**Output is slightly over 8 MB:**
Two-pass doesn't always land exactly on target due to container overhead. Knock the video bitrate down by 5–10% and re-run.

**Audio sounds fine but video is unwatchable:**
Lower the resolution first (`scale=-2:480` or `scale=-2:360`) before lowering bitrate further — resolution reduction is much more efficient than just squeezing a high-res stream at low bitrate.

**`-vf scale=-2:720` gives an error about non-divisible dimensions:**
Use `-vf scale=trunc(iw/2)*2:720` as a fallback.
