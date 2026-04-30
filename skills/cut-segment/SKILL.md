---
name: cut-segment
description: Extract one or more time-range segments from a video. Use when the user says "cut from 1:30 to 3:45", "trim the first 30 seconds", "extract these segments", "split this clip at 5:00". Stream-copies losslessly by default; offers a frame-accurate re-encode mode when the cut must be exact. Multiple segments are written as separate files (or optionally stitched in one pass).
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(printf *), Bash(awk *), Bash(ffmpeg *), Bash(ffprobe *), Read, Write
---

# Cut Segment

Extract precise time ranges from a video. Replaces the legacy `cut-video-segment` command.

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Source | required |
| Segments | required: one or more `start..end` ranges (e.g. `00:01:30..00:03:45`) or a JSON array `[{start, end, label?}, ...]` |
| Output dir | default: same dir as source (or `<project>/exports/` when run inside a project) |
| Mode | `copy` (lossless, stream-copy) — `precise` (re-encode for frame-accurate cuts) |
| Stitch | `no` (one file per segment) — `yes` (concatenate all segments into a single output) |

### 2. Validate ranges

```bash
SRC="$(realpath "$INPUT")"
DUR=$(ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 "$SRC")

to_seconds() {
  awk -F: '{
    n = NF
    if (n == 3)      print $1*3600 + $2*60 + $3
    else if (n == 2) print $1*60 + $2
    else             print $1
  }' <<<"$1"
}
```

For each segment: convert start/end to seconds; reject if `start >= end`, if `start < 0`, or if `end > DUR`. Report which segments were rejected and abort.

### 3. Mode: `copy` (default)

Stream-copy each segment. Fast, lossless, but cuts land on the nearest keyframe (typically within 1–2s of the requested mark).

```bash
i=0
for seg in "${SEGMENTS[@]}"; do
  read s e <<<"$seg"
  OUT="$OUTDIR/$(basename "${SRC%.*}")_seg_$(printf '%02d' $i)${EXT}"
  ffmpeg -y -ss "$s" -to "$e" -i "$SRC" -c copy -avoid_negative_ts make_zero "$OUT"
  i=$((i+1))
done
```

`-ss` *before* `-i` enables fast seek; combined with `-c copy` this is the lossless path.

### 4. Mode: `precise`

Frame-accurate cuts via re-encode. `-ss` goes *after* `-i` to force decode-side seek.

```bash
ffmpeg -y -i "$SRC" -ss "$s" -to "$e" \
  -c:v libx264 -crf 18 -preset medium \
  -c:a aac -b:a 192k \
  "$OUT"
```

If a render profile name was supplied (from `render-profiles.json`), build the encoder args from that profile instead.

### 5. Stitch (optional)

If `STITCH=yes`:

```bash
LIST=$(mktemp)
for f in "${SEGMENT_OUTPUTS[@]}"; do printf "file '%s'\n" "$f" >> "$LIST"; done
ffmpeg -y -f concat -safe 0 -i "$LIST" -c copy "$OUTDIR/$(basename "${SRC%.*}")_stitched${EXT}"
```

If the segments were produced via `precise` mode they share encoder params and concat-copy works. For `copy` mode + stitch, fall back to a re-encode through `render-from-library` if stream-copy concat fails.

### 6. Verify

For each output:

```bash
ffprobe -v error -show_entries format=duration,size -of default=nw=1 "$OUT"
```

Compare actual duration to requested `(end - start)`. In `copy` mode, mention the keyframe-snap drift if it's >0.5s.

### 7. Confirm

Print a table of segment outputs with their durations and sizes. Offer follow-ups (`burn-graphics` to overlay a label on each, `render-from-library` to stitch in a richer way, `audio-analysis` per segment).

## Notes

- Default output extension preserves the source's container.
- Multiple segments with the same numeric index won't clobber each other (they get `_seg_00`, `_seg_01`, …); supply a `label` per segment in the JSON form to get human-readable names instead.
- For "trim the first/last N seconds" shorthands, translate to a single segment: `00:00:00..-00:00:N` (use ffprobe duration) before validation.
- For multi-file concatenation (the legacy `merge-videos` command's job), use `render-from-library` instead.
- For watermarks and overlays (the legacy `add-watermark` command's job), use `burn-graphics` instead.
