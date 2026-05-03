---
name: mkv-strip-tracks
description: Remove unwanted tracks (audio, subtitle, video) from a Matroska container and save a new, smaller file. Use mkvmerge syntax (-a, -s, -d to keep tracks; -A, -S, -D to drop all of a type). Use when the user says "remove this audio track", "drop all subtitles except English", "keep only this video and audio", "make this file smaller". No re-encode.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(which *), Bash(mkvmerge *), Bash(mkdir *), Bash(ls *), Bash(dirname *), Bash(basename *), Bash(stat *), Read, Write
---

# MKV Strip Tracks

Selectively keep or discard audio, subtitle, and video tracks in a Matroska container using mkvmerge. Produces a new MKV with only the requested tracks; original is preserved. No re-encoding.

## When to use

- User wants to drop unwanted audio or subtitle tracks from a multi-language file
- Reduce file size by removing all subtitles, or all but one audio track
- Clean up a master file before distribution (keep video + one audio + one subtitle)

## Inputs to gather

- **File** — path to source MKV (required)
- **Keep audio tracks** — comma-separated list of track indices (e.g., `1,2`) or language codes; OR `-A` to drop all audio (optional)
- **Keep subtitle tracks** — comma-separated list (e.g., `3,4`) or `-S` to drop all subs (optional)
- **Keep video tracks** — track index (usually `0`) or `-D` to drop all video (rare) (optional, default: keep track 0)
- **Output path** — where to write the new MKV (optional, default: `<original-name>.stripped.mkv` in same dir)

## Procedure

### 1. Check mkvtoolnix is installed

```bash
if ! command -v mkvmerge >/dev/null; then
  echo "mkvtoolnix not installed. Install: apt install mkvtoolnix"
  exit 1
fi
```

### 2. Build the mkvmerge command

mkvmerge uses track-selection flags:

| Flag | Effect |
|------|--------|
| `-d 0` | keep video track 0 |
| `-a 1,2` | keep audio tracks 1 and 2 |
| `-s 3` | keep subtitle track 3 |
| `-A` | drop all audio |
| `-S` | drop all subtitles |
| `-D` | drop all video |

Example: keep video 0, audio 1, subs 3–4:

```bash
mkvmerge \
  -d 0 -a 1 -s 3,4 \
  -o "$OUTPUT" "$FILE"
```

### 3. Validate tracks exist

Before running mkvmerge, query the source with `mkvmerge -i "$FILE"` to verify all requested track indices exist. Warn if a track is missing.

### 4. Show the user the command

Print the full mkvmerge invocation and ask for confirmation (especially if many tracks will be dropped).

### 5. Run and report

```bash
mkvmerge -d 0 -a "$AUDIO_TRACKS" -s "$SUB_TRACKS" -o "$OUTPUT" "$FILE"
```

After completion, show:
```
Source    : path/to/in.mkv (12.5 MB)
Output    : path/to/in.stripped.mkv (3.2 MB)
Removed   : audio track 2, all other subtitles
Reduction : 74%
```

## Output / side effects

- Creates a new MKV file at the specified output path.
- Original file is untouched.
- No re-encoding; mkvmerge copies streams as-is.
- The output file is **always a new file** — mkvmerge cannot edit in-place.

## Notes

- mkvmerge is fast and non-destructive compared to ffmpeg, which would require re-encoding if stream-copy alone fails.
- Track indices refer to the **physical track order in the file**, shown by `mkv-track-list`. Not all indices are sequential (e.g., if you dropped a track earlier, indices don't shift).
- Language-based selection (e.g., `-a eng`) is more portable than track indices if the MKV has proper language tags. mkvmerge supports ISO 639-2 codes.
- If the user wants to keep multiple tracks but doesn't know the indices, run `mkv-track-list` first.
- Chapters, attachments, and global metadata are preserved in the output by default; mkvmerge can also strip these with additional flags if needed (e.g., `--no-chapters`).
