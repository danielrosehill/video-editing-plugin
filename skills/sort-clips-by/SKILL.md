---
name: sort-clips-by
description: Sort a folder of media files into subfolders by a chosen attribute — resolution (e.g. 4K vs 1080p), aspect ratio (16:9, 9:16, 1:1), framerate, codec, or media kind (photo vs video). Use when the user says "separate the 4K clips", "split photos and videos", "sort by aspect ratio", "organize this folder by resolution". Moves files; never modifies content. Dry-run preview by default.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(mv *), Bash(find *), Bash(file *), Bash(ffprobe *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(printf *), Read, Write
---

# Sort Clips By

Bucket a flat folder of media into subfolders by a single attribute. Operates by `mv` only — files are not transcoded or modified.

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Source folder | required |
| Attribute | required: `resolution` \| `aspect` \| `fps` \| `codec` \| `kind` |
| Recursive | `no` (only files at top level of source) |
| Mode | `preview` (dry-run) — flip to `commit` once user confirms |

### 2. Enumerate files

```bash
SRC="$(realpath "$INPUT")"
test -d "$SRC" || { echo "Not a folder: $SRC"; exit 1; }
if [ "$RECURSIVE" = "yes" ]; then
  mapfile -t FILES < <(find "$SRC" -type f -not -path '*/.*')
else
  mapfile -t FILES < <(find "$SRC" -maxdepth 1 -type f -not -name '.*')
fi
```

### 3. Classify each file

Use `ffprobe` for video/image streams; fall back to `file --mime-type` for non-media (skip those unless attribute is `kind`).

```bash
probe() {
  ffprobe -v error -select_streams v:0 \
    -show_entries stream=width,height,r_frame_rate,codec_name,codec_type \
    -of json "$1" 2>/dev/null
}
```

Bucket rules:

| Attribute | Bucket name |
|-----------|-------------|
| `resolution` | `4K` (≥3840w), `1440p` (≥2560w), `1080p` (≥1920w), `720p` (≥1280w), `SD` (else) |
| `aspect` | `16x9`, `9x16`, `1x1`, `4x3`, `21x9`, `other` (round to nearest standard within 2%) |
| `fps` | `24fps`, `25fps`, `30fps`, `50fps`, `60fps`, `120fps`, `other` (snap to nearest standard within 1 fps) |
| `codec` | the codec_name verbatim (`h264`, `hevc`, `prores`, etc.) |
| `kind` | `videos`, `photos`, `other` (mime-type prefix `video/` / `image/`) |

For `kind`, photos use `ffprobe` for image dimensions if needed but the bucket is decided by mime-type.

### 4. Preview

Print a table:

```
bucket          count   sample file
--------------  ------  --------------------------
4K              12      DJI_0042.MP4
1080p           48      GX010234.MP4
SD              3       handycam_old.AVI
```

Ask the user: **commit?** (default no).

### 5. Commit

```bash
for f in "${FILES[@]}"; do
  bucket=$(classify "$f")
  mkdir -p "$SRC/$bucket"
  mv -n "$f" "$SRC/$bucket/"  # -n: never overwrite
done
```

Skip (don't move) if `mv -n` would clobber. Report any skips.

### 6. Confirm

Print final counts per bucket and the source folder path.

## Notes

- Empty buckets are not created.
- Hidden files (`.DS_Store`, `.gitkeep`) are ignored.
- Files that fail `ffprobe` (when attribute requires it) land in a `_unprobed/` bucket and are listed in the report.
- This supersedes the legacy `commands/separate-4k.md` and `commands/separate-photos-and-video.md` scratch notes.
