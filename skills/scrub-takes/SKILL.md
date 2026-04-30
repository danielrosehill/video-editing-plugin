---
name: scrub-takes
description: Remove accidental short takes from a clip folder — clips below a minimum duration (default 3s), or below a minimum filesize, that are typically the result of a fumbled record button. Use when the user says "scrub the short takes", "remove accidental clips", "clean up the takes folder". Moves clips to a `_rejected/` subfolder by default; only deletes on explicit confirmation.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(mv *), Bash(rm *), Bash(find *), Bash(stat *), Bash(ffprobe *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(awk *), Bash(printf *), Read, Write
---

# Scrub Takes

Identify and quarantine clips that look like fumbled or accidental recordings — too short to be useful.

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Source folder | required |
| Min duration (sec) | `3` |
| Min filesize (MB) | `0` (off — duration is the primary signal) |
| Recursive | `no` |
| Action | `quarantine` (move to `_rejected/`) — `delete` requires explicit confirmation |

### 2. Enumerate

```bash
SRC="$(realpath "$INPUT")"
test -d "$SRC" || { echo "Not a folder: $SRC"; exit 1; }
if [ "$RECURSIVE" = "yes" ]; then
  mapfile -t FILES < <(find "$SRC" -type f \( -iname '*.mp4' -o -iname '*.mov' -o -iname '*.mkv' -o -iname '*.avi' -o -iname '*.m4v' \))
else
  mapfile -t FILES < <(find "$SRC" -maxdepth 1 -type f \( -iname '*.mp4' -o -iname '*.mov' -o -iname '*.mkv' -o -iname '*.avi' -o -iname '*.m4v' \))
fi
```

### 3. Probe each clip

```bash
duration_of() {
  ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 "$1" 2>/dev/null
}
```

Skip files that fail to probe — list them but never move/delete them.

### 4. Build the reject list

Mark a clip for rejection if **either**:

- duration is `< MIN_DURATION` seconds, OR
- (when `MIN_SIZE_MB > 0`) filesize is `< MIN_SIZE_MB` MB.

### 5. Preview

Print a table of rejects sorted by duration ascending:

```
file                          duration   size
----------------------------  --------   -------
GX010199.MP4                    0.4s      8.2 MB
GX010202.MP4                    1.1s     21.0 MB
DJI_0011.MP4                    2.7s     45.3 MB
```

Print total kept vs rejected. Ask the user to **confirm**. For `delete` mode, require an extra "yes, delete N files" confirmation.

### 6. Apply

For `quarantine`:

```bash
mkdir -p "$SRC/_rejected"
for f in "${REJECTS[@]}"; do
  mv -n "$f" "$SRC/_rejected/"
done
```

For `delete` (only after explicit second confirmation):

```bash
for f in "${REJECTS[@]}"; do
  rm -f -- "$f"
done
```

### 7. Confirm

Print the count moved/deleted and the rejected folder path (for `quarantine`).

## Notes

- Default action is `quarantine`, never `delete`. The `_rejected/` folder is a safety net the user can clear manually after spot-checking.
- Audio-only files are not in scope. Use the audio-production plugin for audio.
- Hidden files and existing `_rejected/` contents are ignored.
