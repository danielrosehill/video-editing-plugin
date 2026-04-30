---
name: editly-render
description: Render a video from a declarative editly JSON spec (clips, transitions, titles, audio mix). Use when the user says "render this editly spec", "compose this slideshow", "make a video from this JSON", "editly render". Validates the spec, runs editly headlessly, and reports the deliverable. Editly must be installed via `deps-setup` (npm-global).
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(command *), Bash(editly *), Bash(node *), Bash(ffprobe *), Read, Write
---

# Editly Render

Run [`editly`](https://github.com/mifi/editly) on a JSON spec to compose a video. Editly is a declarative slideshow / montage tool — clips, titles, transitions, audio mix all described as JSON.

## Procedure

### 1. Verify editly

```bash
command -v editly >/dev/null || { echo "editly not installed. Run deps-setup and select editly (npm-global, requires Cairo + ffmpeg)."; exit 1; }
editly --version
```

### 2. Inputs

| Field | Default |
|-------|---------|
| Spec | required: path to `.json` or `.json5` editly config |
| Output | default: derived from spec filename, `.mp4` |
| Fast mode | `no` (full quality). `yes` adds `--fast` for quick previews |
| Audio normalize | `yes` (editly's `--audio-norm`) |

### 3. Validate the spec

```bash
SPEC="$(realpath "$INPUT")"
test -f "$SPEC" || { echo "Spec not found: $SPEC"; exit 1; }
jq -e . "$SPEC" >/dev/null 2>&1 || {
  # editly accepts json5; only fail if the file is also not valid json5
  node -e "require('json5').parse(require('fs').readFileSync('$SPEC','utf8'))" 2>/dev/null \
    || { echo "Spec is neither valid JSON nor JSON5."; exit 1; }
}
```

Sanity-check required top-level fields:

```bash
jq -e '.outPath // .clips' "$SPEC" >/dev/null || echo "Warning: spec has no .clips — editly will likely error."
```

Resolve the output path. If `outPath` is set in the spec, use it; otherwise pass `--out`.

### 4. Pre-flight: source files exist

For each clip/audio reference inside `clips[].layers[]` and the top-level `audioFilePath`, verify the file exists relative to the spec's directory. List any missing files and abort before invoking editly.

```bash
SPEC_DIR="$(dirname "$SPEC")"
MISSING=$(jq -r '
  [(.clips // [])[] | (.layers // [])[] | (.path // empty)] +
  [(.audioFilePath // empty)] | .[]
' "$SPEC" | while read -r p; do
  [ -z "$p" ] && continue
  case "$p" in
    /*) f="$p" ;;
    *)  f="$SPEC_DIR/$p" ;;
  esac
  [ -f "$f" ] || echo "$p"
done)
[ -n "$MISSING" ] && { echo "Missing source files:"; echo "$MISSING"; exit 1; }
```

### 5. Run editly

```bash
ARGS=( "$SPEC" )
[ -n "$OUT" ] && ARGS+=( --out "$OUT" )
[ "$FAST" = "yes" ] && ARGS+=( --fast )
[ "$AUDIO_NORM" = "yes" ] && ARGS+=( --audio-norm )

editly "${ARGS[@]}"
```

Stream editly's progress output live.

### 6. Verify

```bash
ffprobe -v error -show_entries format=duration,size -of default=nw=1 "$OUT"
```

Print: output path, duration, size, container.

### 7. Confirm

Suggest follow-ups: `audio-analysis` (loudness check), `transcode` (re-encode to a final delivery profile — editly's default output is software h264), `burn-graphics` (add overlays editly can't compose).

## Notes

- Editly's defaults produce software-encoded h264 — for hardware-encoded final deliverables, run `transcode` or `render-with-profile` on the editly output.
- The `--fast` flag drops resolution and quality; useful for iterating on a spec, never for final delivery.
- Cairo is a hard dependency on Linux for editly's text rendering — if rendering fails with a Cairo error, the user needs `libcairo2-dev libpango1.0-dev libjpeg-dev libgif-dev librsvg2-dev` (Debian/Ubuntu) before reinstalling editly.
- For non-declarative ("describe what to keep") cuts, use `agentic-edit` instead.
