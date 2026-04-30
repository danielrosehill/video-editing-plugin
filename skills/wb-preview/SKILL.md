---
name: wb-preview
description: Render a labeled before/after preview clip showing a proposed white-balance, color, or exposure correction. Use when the user (or Claude) wants to evaluate a color tweak before committing to a full re-render — e.g. "show me what +500K would look like", "preview this WB fix", "before/after on the color correction". Output is a short side-by-side or sequential MP4 with on-screen labels.
disable-model-invocation: false
allowed-tools: Bash(ffmpeg *), Bash(ffprobe *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(basename *), Bash(dirname *), Bash(awk *), Read, Write
---

# White-Balance / Color Preview

Render a tiny labeled clip — original on the left, corrected on the right (or sequential before → after) — so the user can eyeball a color correction before re-rendering the whole video.

Pairs naturally with `video-timeline`: timeline → diagnose, wb-preview → propose.

## Procedure

### 1. Inputs

| Field | Example | Default |
|-------|---------|---------|
| Source video | `clip.mp4` | required |
| Timestamp | `00:01:23` (start of preview window) | midpoint of video |
| Window | seconds around timestamp | 10 |
| Layout | `sidebyside` / `sequential` | `sidebyside` |
| Correction | filter spec (see below) | required |

### 2. Correction filter spec

Accept any of these forms, build the filter chain:

| User intent | ffmpeg filter |
|-------------|---------------|
| "warmer +500K" | `colortemperature=temperature=6000` (or apply `colorbalance=rs=0.05:bs=-0.05`) |
| "cooler -300K" | `colortemperature=temperature=5000` |
| "+0.3 EV" | `eq=brightness=0.06` (rough; 0.06 ≈ 0.3 EV) |
| "less green" | `colorbalance=gm=-0.1` |
| "tint magenta" | `colorbalance=gm=0.1` |
| Custom | accept a raw filter string verbatim |

Always also offer an "auto" option: `eq=auto`-ish, or a measured neutralize via `curves=preset=increase_contrast` + a manual nudge — but only if the user asks.

### 3. Build the preview

**Sequential** (simpler, fewer ffmpeg gotchas):

```bash
ffmpeg -hide_banner -y \
  -ss "$START" -t "$WINDOW" -i "$VIDEO" \
  -ss "$START" -t "$WINDOW" -i "$VIDEO" \
  -filter_complex "
    [0:v]drawtext=text='BEFORE':fontcolor=white:box=1:boxcolor=black@0.6:boxborderw=8:x=20:y=20:fontsize=36[a];
    [1:v]<CORRECTION>,drawtext=text='AFTER\\: <label>':fontcolor=white:box=1:boxcolor=black@0.6:boxborderw=8:x=20:y=20:fontsize=36[b];
    [a][b]concat=n=2:v=1:a=0[v];
    [0:a][1:a]concat=n=2:v=0:a=1[a]
  " \
  -map '[v]' -map '[a]' \
  -c:v libx264 -crf 22 -preset veryfast \
  -c:a aac -b:a 128k \
  "$OUT"
```

**Side-by-side** (`hstack`):

```bash
... -filter_complex "
  [0:v]drawtext=text='BEFORE'...[a];
  [1:v]<CORRECTION>,drawtext=text='AFTER'...[b];
  [a][b]hstack=inputs=2[v]
" -map '[v]' -map '0:a' ...
```

Use libx264 + CRF 22 here — preview clips don't need GPU encode, and software keeps it portable across systems even if `profile-system` hasn't run.

### 4. Output

Write to:
- `<project>/renders/wb_preview_<timestamp>_<label>.mp4` if inside an index project, OR
- sibling to the source: `<dir>/wb_preview/<basename>_<label>.mp4`

### 5. Report

Print the output path, total length, file size. Offer:
- Open in default player (`xdg-open`).
- Adjust the correction (re-run with new params).
- Apply the corrected filter to the full video via `transcode` once the user is happy.
