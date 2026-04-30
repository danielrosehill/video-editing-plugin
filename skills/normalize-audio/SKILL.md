---
name: normalize-audio
description: Apply ffmpeg loudnorm (two-pass EBU R128) to a video or audio file, hitting the user's preferred LUFS / true-peak target (or one specified inline). Use when the user says "normalize this", "fix the levels", "make this -14 LUFS", "loudnorm this", "normalize for YouTube/podcast/broadcast". Pairs with audio-analysis (analyze recommends, this skill applies). Re-muxes video stream-copy when the input is video.
disable-model-invocation: false
allowed-tools: Bash(ffmpeg *), Bash(ffprobe *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(cat *), Bash(awk *), Bash(grep *), Bash(sed *), Bash(realpath *), Read, Write
---

# Normalize Audio

Two-pass `loudnorm` to hit a precise integrated LUFS + true-peak target. Reads the user's default target from `preferences.json` if present; otherwise accepts an inline target.

This is the apply step that pairs with the read-only `audio-analysis` skill.

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Source | required (video or audio) |
| Target | from `preferences.json` (`loudness.target`); else ask. Choices: `youtube` (-14/-1), `spotify` (-14/-1), `apple-music` (-16/-1), `podcast` (-16/-1), `broadcast` (-23/-2) |
| Output | sibling `<basename>.normalized.<ext>` (or project's `working/` if inside an index project) |

Resolve target → `I` (integrated LUFS), `TP` (true peak), `LRA` (default 11).

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
PREFS="$DATA_DIR/preferences.json"
if [ -f "$PREFS" ] && [ -z "$TARGET" ]; then
  TARGET=$(jq -r '.loudness.target // empty' "$PREFS")
fi

case "$TARGET" in
  youtube|spotify)        I=-14; TP=-1; LRA=11 ;;
  apple-music|podcast)    I=-16; TP=-1; LRA=11 ;;
  broadcast)              I=-23; TP=-2; LRA=11 ;;
  custom)                 ;; # caller sets I/TP/LRA
  *) echo "Unknown target: $TARGET"; exit 1 ;;
esac
```

### 2. Pass 1 — measure

```bash
PASS1=$(ffmpeg -hide_banner -nostats -i "$SRC" \
  -af "loudnorm=I=$I:TP=$TP:LRA=$LRA:print_format=json" \
  -f null - 2>&1 | sed -n '/^{/,/^}/p')

MI=$(echo "$PASS1"  | jq -r '.input_i')
MTP=$(echo "$PASS1" | jq -r '.input_tp')
MLRA=$(echo "$PASS1"| jq -r '.input_lra')
MTHRESH=$(echo "$PASS1" | jq -r '.input_thresh')
OFFSET=$(echo "$PASS1"  | jq -r '.target_offset')
```

Sanity-check: if any of these are `"-inf"` or null, the source has silence-only or non-finite measurements. Fall back to single-pass `loudnorm` and warn.

### 3. Pass 2 — apply with measured values

```bash
LOUDNORM="loudnorm=I=$I:TP=$TP:LRA=$LRA:measured_I=$MI:measured_TP=$MTP:measured_LRA=$MLRA:measured_thresh=$MTHRESH:offset=$OFFSET:linear=true:print_format=summary"
```

#### If source is video

Stream-copy video, re-encode audio only:

```bash
ffmpeg -hide_banner -y -i "$SRC" \
  -map 0:v -map 0:a \
  -c:v copy \
  -af "$LOUDNORM" \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  "$OUT"
```

If the source has multiple audio tracks, normalize only `a:0` and stream-copy the rest unless the user asked otherwise.

#### If source is audio-only

Pick the output codec from the input extension; default to wav for lossless intermediates, mp3/aac for delivery:

```bash
ffmpeg -hide_banner -y -i "$SRC" -af "$LOUDNORM" "$OUT"
```

### 4. Verify

Re-run a quick `ebur128` pass on the output and confirm integrated is within ±0.5 LU of target and true-peak is at or below the ceiling:

```bash
ffmpeg -hide_banner -nostats -i "$OUT" -af 'ebur128=peak=true:framelog=quiet' -f null - 2>&1 | tail -25
```

### 5. Report

Print a one-table summary:

```
Source : path/to/in.mp4
Target : YouTube (-14 LUFS / -1 dBTP)
Before : I=-19.2 LUFS, TP=-2.1 dBTP, LRA=8.3 LU
After  : I=-14.0 LUFS, TP=-1.0 dBTP, LRA=7.9 LU
Output : path/to/in.normalized.mp4
```

## Notes

- `linear=true` keeps dynamics intact when the source already fits within the target LRA. Drop it (or omit) if you want loudnorm to compress to fit.
- For batch normalization (folder), iterate this same two-pass over each file. Don't try to share Pass-1 measurements across files — each clip is measured independently.
- This skill never overwrites the source. Output goes to a sibling file or the project's `working/` directory.
