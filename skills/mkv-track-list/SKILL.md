---
name: mkv-track-list
description: Inspect tracks in a Matroska (MKV) container and display a formatted table showing track number, type (video/audio/subs), codec, language, default flag, and track name. Use when the user asks "what's in this MKV", "list the tracks", "show audio/subtitle languages", "inspect this file before editing".
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(which *), Bash(mkvinfo *), Bash(mkvmerge *), Bash(jq *), Read
---

# MKV Track List

Read the track structure of a Matroska container without modifying it. Produces a clean, human-readable table of all tracks, their codecs, languages, and metadata.

## When to use

- User wants to see what audio, subtitle, and video tracks are in an MKV
- Before running `mkv-strip-tracks`, `mkv-set-default-track`, or `mkv-extract-track`, inspect the source
- Verify language tags and default-track flags on multi-language masters

## Inputs to gather

- **File** — path to MKV file (required)

## Procedure

### 1. Check mkvtoolnix is installed

```bash
if ! command -v mkvmerge >/dev/null; then
  echo "mkvtoolnix not installed. Install: apt install mkvtoolnix"
  exit 1
fi
```

### 2. Run mkvmerge -i and parse output

```bash
mkvmerge -i "$FILE" > /tmp/mkv_info.txt
```

The output includes lines like:
```
Track ID 0: video (V_UNCOMPRESSED, 1920x1080, 25.000 fps)
Track ID 1: audio (A_AAC, 2 channels, 48000 Hz, 'English')
Track ID 2: subtitles (S_TEXT/UTF8, 'English', default)
```

### 3. Parse and tabulate

Extract and format as a table with columns: **Track #** | **Type** | **Codec** | **Language** | **Default?** | **Title**

Example:
```
Track # │ Type     │ Codec              │ Language  │ Default │ Title
────────┼──────────┼────────────────────┼───────────┼─────────┼─────────
0       │ Video    │ H.265 (HEVC)       │ —         │ yes     │
1       │ Audio    │ AAC Stereo         │ English   │ yes     │ Commentary
2       │ Audio    │ AAC 5.1            │ Hebrew    │ no      │
3       │ Subs     │ UTF-8 text         │ English   │ no      │
4       │ Subs     │ UTF-8 text         │ Hebrew    │ yes     │
```

### 4. Show summary

Count total tracks by type and note any atypical flags (multiple default tracks in one type, etc.).

## Output / side effects

Prints a formatted table to stdout. No files created or modified.

## Notes

- mkvmerge is fast and non-destructive — safe to run on any MKV.
- Language codes are ISO 639-2 (three letters: `eng`, `heb`, `fra`, etc.). If a track has no language tag, show `—`.
- The "Default" column indicates the default_track flag for that track within its type (one default audio, one default subtitle, etc.).
- For detailed JSON output, use `mkvinfo --json $FILE` (if mkvtoolnix ≥ 75) — this skill aims for human readability instead.
