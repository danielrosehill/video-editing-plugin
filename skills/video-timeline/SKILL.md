---
name: video-timeline
description: Generate a folder of low-resolution frame snapshots at evenly spaced timestamps so Claude (or you) can "see" a video without ingesting the full file. Use when the user says "give me a visual timeline", "let me see this video", "show me frames from this", "white balance check", "what does this video look like", or any request where Claude needs visual context about a video. Output is a folder of small JPEGs plus a timeline.md index.
disable-model-invocation: false
allowed-tools: Bash(ffmpeg *), Bash(ffprobe *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(basename *), Bash(dirname *), Bash(awk *), Bash(printf *), Read, Write
---

# Video Timeline

Drop a folder of evenly-spaced, low-res frames from a video so they can be opened (by Claude or by the user) without processing the whole file. Cheap visual context for white-balance checks, scene scrubbing, editorial review.

## Procedure

### 1. Inputs

Ask if not provided:

- **Video** — path to a video file (required).
- **Frame count** — default `12`.
- **Width** — default `480` px.
- **Output dir** — default:
  - if invoked inside an index project, `<project>/video_timeline/<video-basename>/`
  - else, sibling to the source: `<dir>/video_timeline/<basename>/`

### 2. Get duration

```bash
DURATION=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$VIDEO")
```

Compute even-spaced timestamps. To keep the first and last frames meaningful (not the very edge), pad 5% in from each end:

```bash
N=12
PAD=$(awk -v d="$DURATION" 'BEGIN { printf "%.3f", d*0.05 }')
USABLE=$(awk -v d="$DURATION" -v p="$PAD" 'BEGIN { printf "%.3f", d - 2*p }')
STEP=$(awk -v u="$USABLE" -v n="$N" 'BEGIN { printf "%.3f", u/(n-1) }')
```

### 3. Loop seek-and-grab

`-ss` *before* `-i` is fast (keyframe seek) but can land on the nearest keyframe; `-ss` *after* `-i` is accurate but slower. Default to fast seek, since we only want a representative frame, not a precise one.

```bash
mkdir -p "$OUT_DIR"
for ((i=0; i<N; i++)); do
  T=$(awk -v p="$PAD" -v s="$STEP" -v i="$i" 'BEGIN { printf "%.3f", p + i*s }')
  HMS=$(awk -v t="$T" 'BEGIN { h=int(t/3600); m=int((t%3600)/60); s=t-h*3600-m*60; printf "%02d-%02d-%06.3f", h, m, s }')
  IDX=$(printf "%04d" $((i+1)))
  ffmpeg -hide_banner -loglevel error -y -ss "$T" -i "$VIDEO" \
    -frames:v 1 -vf "scale=$WIDTH:-1" -q:v 5 \
    "$OUT_DIR/frame_${IDX}_${HMS}.jpg"
done
```

### 4. Write `timeline.md`

A small index Claude can read in a single tool call:

```markdown
# Video Timeline: <basename>

- Source: `<absolute path>`
- Duration: <H:MM:SS>
- Frames: 12 @ 480px wide

| # | Timestamp | File |
|---|-----------|------|
| 1 | 00:00:01.234 | frame_0001_00-00-01.234.jpg |
| 2 | 00:00:11.567 | frame_0002_00-00-11.567.jpg |
| ... |
```

### 5. Report + offer follow-ups

Print the output dir path and frame count. Offer:

- Open the folder in a file manager.
- A vision-pass critique — Claude reads the frames and comments on white balance, exposure, framing consistency.
- Pair with `wb-preview` if a white-balance issue is spotted.
