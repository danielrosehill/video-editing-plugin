---
name: auto-cut-silences
description: Remove silent / dead-air segments from a video or audio file using auto-editor. Use when the user says "cut the silences", "remove dead air", "tighten this up", "auto-edit this", "trim the gaps". Requires auto-editor installed via deps-setup (lives in the plugin's shared uv venv). Outputs a tightened copy alongside the source; original is preserved.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(ls *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(*/auto-editor *), Bash(auto-editor *), Read, Write
---

# Auto-Cut Silences

Wrapper around [`auto-editor`](https://github.com/WyattBlue/auto-editor) — detects silent segments based on an audio threshold and rebuilds the file with those segments removed. Re-encodes only the affected ranges; the rest is stream-copied where possible.

## Procedure

### 1. Resolve auto-editor

Auto-editor is installed by `deps-setup` into the plugin's shared uv venv. Locate it via the venv path recorded in `preferences.json`:

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
PREFS="$DATA_DIR/preferences.json"
VENV=$(jq -r '.python_venv // empty' "$PREFS" 2>/dev/null)

if [ -n "$VENV" ] && [ -x "$VENV/bin/auto-editor" ]; then
  AUTO_EDITOR="$VENV/bin/auto-editor"
elif command -v auto-editor >/dev/null; then
  AUTO_EDITOR="$(command -v auto-editor)"
else
  echo "auto-editor not found. Run deps-setup and select auto-editor."
  exit 1
fi
```

### 2. Inputs

| Field | Default |
|-------|---------|
| Source | required (video or audio) |
| Threshold | `4%` (auto-editor default — silence below 4% of peak) |
| Margin | `0.2sec` (keep 0.2s of padding around speech) |
| Min cut | `0.2sec` (don't cut a silence shorter than this) |
| Output | sibling `<basename>.tight.<ext>` (or project's `working/` inside an index project) |

Auto-editor accepts these as `--silent-threshold`, `--margin`, and `--min-clip-length` / `--min-cut-length`.

### 3. Run

```bash
"$AUTO_EDITOR" "$SRC" \
  --silent-threshold "$THRESHOLD" \
  --margin "$MARGIN" \
  --output "$OUT"
```

For more aggressive trimming (e.g. talking-head where pauses are abundant), suggest `--margin 0.1sec --silent-threshold 3%`.

For a preview-only run (don't write a new file, just report what would be cut):

```bash
"$AUTO_EDITOR" "$SRC" --export timeline -o /tmp/auto-cut-timeline.json
```

The exported timeline is a JSON list of kept ranges — useful for showing the user what would be removed before committing.

### 4. Report

Print before/after duration and the cut ratio:

```bash
DUR_IN=$(ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 "$SRC")
DUR_OUT=$(ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 "$OUT")
RATIO=$(awk "BEGIN{printf \"%.1f\", ($DUR_IN-$DUR_OUT)/$DUR_IN*100}")

echo "In : ${DUR_IN}s"
echo "Out: ${DUR_OUT}s"
echo "Cut: ${RATIO}% removed"
echo "File: $OUT"
```

If the cut ratio is <2% or >90%, warn — the threshold is probably wrong.

### 5. Iterate

Auto-editor is fast on clip-length input but slow on long lectures. For tuning, suggest the user:
1. Run on a 60-second extract first to dial in `--silent-threshold` and `--margin`.
2. Then run on the full file.

## Notes

- Auto-editor handles both video and pure audio. For video, it preserves video stream timing while cutting matching ranges from both streams.
- This skill does not normalize loudness. Pair with `normalize-audio` afterward if levels are uneven.
- For complex agentic editing ("make a 60-second cut down of this hour-long stream"), use a future `agentic-edit` skill that wraps `video-use` / `VideoAgent`. This skill is for the simple "remove silences" case only.
