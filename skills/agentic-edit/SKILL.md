---
name: agentic-edit
description: Run an agentic / LLM-driven cut-down on a long video — "make a 60-second highlight reel of this hour-long stream", "cut the boring parts from this lecture", "find the best moments in this footage". Wraps `video-use` (browser-use) by default; falls back to `VideoAgent` (HKU) for research-grade understanding. Both come from the shared uv venv set up by `deps-setup`. Outputs a cut-list (timestamps + reasons) and an optional rendered preview.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(command *), Bash(ffmpeg *), Bash(ffprobe *), Bash(*/python *), Bash(python3 *), Read, Write
---

# Agentic Edit

Drive an LLM-based agent over a long video to produce a shorter cut. The agent watches the video (or samples it), proposes timestamped segments worth keeping, and emits a cut-list. The skill optionally renders the cut.

## Procedure

### 1. Resolve backend

Two backends, picked by user preference (or by what's installed):

| Backend | Best for | Source |
|---|---|---|
| `video-use` | conversational queries over a video, fast iteration | `pip install git+https://github.com/browser-use/video-use` |
| `VideoAgent` | research-grade multi-agent reasoning over long video | `git clone https://github.com/HKUDS/VideoAgent` |

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
PREFS="$DATA_DIR/preferences.json"
VENV=$(jq -r '.python_venv // empty' "$PREFS" 2>/dev/null)
PYBIN="$VENV/bin/python"
test -x "$PYBIN" || { echo "No plugin venv. Run deps-setup first."; exit 1; }

# default backend = video-use if installed
if "$PYBIN" -c 'import video_use' 2>/dev/null; then
  BACKEND="video-use"
elif [ -d "$DATA_DIR/tools/VideoAgent" ]; then
  BACKEND="VideoAgent"
else
  echo "Neither video-use nor VideoAgent is installed. Run deps-setup."; exit 1
fi
```

### 2. Inputs

| Field | Default |
|-------|---------|
| Source video | required |
| Goal / prompt | required ("60-second highlight reel", "remove the audience-Q&A sections", etc.) |
| Target duration | optional — only used as a hint to the agent |
| Mode | `cut-list-only` (default) or `cut-list-and-render` |
| Backend | auto (preferences.preferred_video_agent_backend) |

### 3. Drive the backend

**3a. video-use** — uses its own conversational loop. Send the prompt + video path; capture the structured cut list.

```bash
"$PYBIN" - <<PY
from video_use import VideoAgent  # API may differ — check installed version
agent = VideoAgent(video_path="$SRC")
result = agent.ask("$GOAL — return JSON list of {start, end, reason}")
import json; print(json.dumps(result, indent=2))
PY
```

> **Version drift warning.** `video-use` is young and its API is not yet stable. Before assuming the snippet above runs, the skill should `"$PYBIN" -c 'import video_use; print(video_use.__version__, dir(video_use))'` and adapt. If the import path or method signature differs, surface that to the user and ask them to upgrade or pin a version.

**3b. VideoAgent** — invoked from its cloned repo:

```bash
cd "$DATA_DIR/tools/VideoAgent"
"$PYBIN" -m videoagent.cli --video "$SRC" --task "$GOAL" --output /tmp/cutlist.json
```

Exact entrypoint depends on the upstream layout — read the repo's README at first use and write the resolved command back to `preferences.tools.VideoAgent.entrypoint` so it's cached.

### 4. Validate the cut list

Expect JSON: `[{ "start": "00:01:23.4", "end": "00:01:55.0", "reason": "..." }, ...]`. Reject and re-prompt if:

- timestamps overlap or run backwards
- any cut extends past the source duration (compare against `ffprobe -show_entries format=duration`)
- total kept duration exceeds the requested target by >20% (warn but allow)

### 5. Save the cut-list

```bash
mkdir -p "$DATA_DIR/cut-lists"
OUT_JSON="$DATA_DIR/cut-lists/$(basename "$SRC")_$(date +%s).json"
cp /tmp/cutlist.json "$OUT_JSON"
```

### 6. (Optional) Render the cut

If mode = `cut-list-and-render`:

```bash
# Build a concat list of trimmed segments
TMPDIR=$(mktemp -d)
i=0
for row in $(jq -c '.[]' "$OUT_JSON"); do
  s=$(jq -r .start <<<"$row"); e=$(jq -r .end <<<"$row")
  ffmpeg -y -ss "$s" -to "$e" -i "$SRC" -c copy "$TMPDIR/seg_$i.mp4"
  echo "file '$TMPDIR/seg_$i.mp4'" >> "$TMPDIR/list.txt"
  i=$((i+1))
done
ffmpeg -f concat -safe 0 -i "$TMPDIR/list.txt" -c copy "$OUT_VIDEO"
```

Stream-copy by default; if the segments fail to concat losslessly (timestamp gaps, codec quirks), fall back to a re-encode through `render-from-library`.

### 7. Confirm

Print:
- backend used + version
- cut-list path + segment count + total kept duration
- (if rendered) output video path, duration, size
- token / time cost reported by the backend, if available

Suggest follow-ups: review the cut-list manually, refine the goal and re-run, or run `generate-deliverables` on the rendered cut.

## Notes

- This skill is a **driver**, not the agent itself. All semantic decisions come from the backend.
- Treat all proposed cuts as a draft. Always show the cut-list to the user before rendering.
- For repeatable / declarative cuts (no LLM), use `editly-render` instead.
