---
name: mkv-set-default-track
description: Set which audio or subtitle track plays by default in a Matroska container using mkvpropedit. Use when the user says "set the default audio", "make this subtitle track default", "this audio language should play first", "change the default track". Idempotent, no re-encode.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(which *), Bash(mkvpropedit *), Bash(mkvmerge *), Bash(cp *), Bash(rm *), Bash(ls *), Read, Write
---

# MKV Set Default Track

Change which audio or subtitle track is marked as default in a Matroska container. Does not re-encode or mux; only updates metadata flags.

## When to use

- User wants a different audio or subtitle track to play by default
- After acquiring a multi-language rip, set the preferred language as default
- Unset defaults from other tracks in the same type (e.g., only one audio track should be default)

## Inputs to gather

- **File** — path to MKV file (required)
- **Track index** — the track number to set as default (required; from `mkv-track-list`)
- **Unset others** — boolean flag; if true, unset default flags on all other tracks of the same type (optional, default: false)

## Procedure

### 1. Check mkvtoolnix is installed

```bash
if ! command -v mkvpropedit >/dev/null; then
  echo "mkvtoolnix not installed. Install: apt install mkvtoolnix"
  exit 1
fi
```

### 2. Validate track index

Query the file to confirm the track exists:

```bash
mkvmerge -i "$FILE" | grep "^Track ID $TRACK_INDEX:" || {
  echo "Track $TRACK_INDEX not found in $FILE"
  exit 1
}
```

### 3. Set default flag on target track

```bash
mkvpropedit "$FILE" --edit "track:$TRACK_INDEX" --set "flag-default=1"
```

### 4. Optionally unset default on other tracks of the same type

If `--unset-others` was requested:

1. Detect the type of the target track (video/audio/subs) from `mkvmerge -i` output.
2. For each other track of the same type, run:

```bash
mkvpropedit "$FILE" --edit "track:$OTHER_TRACK_INDEX" --set "flag-default=0"
```

### 5. Verify

Re-run `mkvmerge -i` and show the user the updated flags for tracks of the target type.

## Output / side effects

- Modifies the target MKV file in-place (metadata only, no re-encode).
- Prints the updated track flags before/after.
- Original is unrecoverable; no backup created (this is an explicit design choice: the operation is idempotent and non-destructive).

## Notes

- mkvpropedit operates in-place. For extra safety, offer the user a one-line backup suggestion before committing: `cp file.mkv file.mkv.bak`.
- Setting a track as default is idempotent — running the command twice has the same effect as running it once.
- Only one audio and one subtitle track can be default within an MKV. If the user's player ignores the flag, check that no other track of the same type is also flagged.
- Some players (VLC, mpv) respect default flags; others (older hardware players) may ignore them. No way to enforce player behavior from this skill.
