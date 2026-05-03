---
name: mkv-extract-track
description: Extract a single track (audio, subtitle, video) from a Matroska container to its own file using mkvextract. Use when the user says "pull out this audio track", "extract the subtitles", "save this audio as a separate file", "get the SRT from this MKV". No re-encode.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(which *), Bash(mkvextract *), Bash(mkvmerge *), Bash(mkdir *), Bash(ls *), Bash(dirname *), Bash(basename *), Read, Write
---

# MKV Extract Track

Pull a single audio, subtitle, or video track out of a Matroska container and save it as a standalone file. Uses mkvextract for speed and fidelity; no re-encoding.

## When to use

- User wants to extract an audio track and save as standalone WAV/AAC/FLAC
- Extract subtitle track as SRT or ASS/SSA
- Pull an embedded font or attachment from an MKV
- Get a video stream without audio/subs as separate file

## Inputs to gather

- **File** — path to source MKV (required)
- **Track index** — the track to extract (required; from `mkv-track-list`)
- **Output path** — where to save the extracted track (required; extension matters: `.srt`, `.mka`, `.aac`, `.wav`, `.ass`, etc.)

## Procedure

### 1. Check mkvtoolnix is installed

```bash
if ! command -v mkvextract >/dev/null; then
  echo "mkvtoolnix not installed. Install: apt install mkvtoolnix"
  exit 1
fi
```

### 2. Determine track type from source

Query the source file:

```bash
mkvmerge -i "$FILE" | grep "^Track ID $TRACK_INDEX:" | head -1
```

Output will indicate the type: video, audio, subtitles, attachments, etc.

### 3. Validate output extension

mkvextract infers the codec from the source and writes appropriate format. Guide the user:

| Source Codec | Suggested Extension |
|--------------|---------------------|
| UTF-8 text (subs) | `.srt` |
| ASS/SSA (subs) | `.ass` |
| AAC audio | `.aac` or `.mka` (Matroska audio) |
| FLAC audio | `.flac` |
| PCM / WAV | `.wav` |
| H.264 video | `.h264` |
| H.265 video | `.h265` or `.hevc` |

If the user provides a mismatched extension, warn but allow (mkvextract will write the correct format regardless).

### 4. Run mkvextract

```bash
mkvextract "tracks" "$FILE" "$TRACK_INDEX:$OUTPUT"
```

Example:
```bash
# Extract subtitle track 2 as SRT
mkvextract tracks in.mkv 2:out.srt

# Extract audio track 1 as AAC
mkvextract tracks in.mkv 1:out.aac

# Extract audio track 3 as Matroska audio container
mkvextract tracks in.mkv 3:out.mka
```

### 5. Report

```
Source  : path/to/in.mkv
Track   : 1 (audio, AAC stereo)
Output  : path/to/out.aac
Size    : 12.4 MB
Status  : ✓ OK
```

## Output / side effects

- Creates a new file at the specified output path containing only the extracted track.
- Original MKV is untouched.
- No re-encoding; the track is copied bit-for-bit (lossless).
- For subtitle tracks, mkvextract transcodes to the target format (UTF-8 text → SRT or ASS depending on codec) — this is safe and standard.

## Notes

- mkvextract is the fastest way to pull tracks out of an MKV compared to ffmpeg, which would require a full demux-re-encode pipeline.
- If the user wants to extract multiple tracks, run this skill once per track. A batch skill is a future enhancement.
- Subtitle tracks: UTF-8 text subtitles are extracted as SRT by default. ASS/SSA subtitles preserve styling (colors, positioning, fonts) if extracted as `.ass`.
- Audio tracks: Extracted as the codec found in the MKV (AAC, FLAC, Opus, etc.). No normalization or filtering applied.
- Attachments and fonts can also be extracted with `mkvextract attachments` and `mkvextract fonts` — future enhancements.
