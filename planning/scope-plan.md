# Video Editing Plugin — Scope Plan

Captured 2026-04-30. The scaffolding (workspace pattern + data store) has been built. This document captures the rest, to be executed in a fresh session opened inside this repo.

## Assumptions

- **Linux only.** Skills assume `lspci`, `nvidia-smi`, `vainfo`, `ffmpeg`, `mlt`, `kdenlive`, `rsync`, `pactl`. Windows/macOS users fork.
- **Per-user data store** at `${XDG_DATA_HOME:-$HOME/.local/share}/claude-media-plugins/video-editing/` — never under `~/.claude/`.

## Already built (foundation — committed)

- `skills/_data-store/SKILL.md` — reference doc for the data store
- `skills/setup-index/SKILL.md` — register/create the video index (base dir holding all projects)
- `skills/open-index/SKILL.md` — open the index in a terminal
- `skills/new-project/SKILL.md` — scaffold a project workspace inside the index, with raw/proxies/working/renders/exports/assets/subtitles/graphics layout

Two-tier concept is in place: **index** (base dir) → **project** (per-video workspace).

## To build

### Data store files (consumed by skills below)

| File                  | Written by                  | Read by                                   |
|-----------------------|-----------------------------|-------------------------------------------|
| `index.json`          | `setup-index`               | `open-index`, `new-project`, all renderers |
| `system-profile.json` | `profile-system`            | every render skill                         |
| `render-profiles.json`| `save-render-profile`       | `render-clips`, `render-from-library`     |
| `nas.json`            | `setup-nas`                 | `pull-from-nas`, `push-to-nas`            |

### System / capability

- **`profile-system`** — detect GPU (`lspci`, `nvidia-smi`, `vainfo`), available ffmpeg encoders (`ffmpeg -encoders | grep -iE 'nvenc|vaapi|qsv|amf'`), preferred encoder per codec, fallback to `libx264`. Write to `system-profile.json`.
- Existing `commands/check-codecs.md` — keep, optionally fold into profile-system.

### Render profiles

- **`save-render-profile`** — name + codec + resolution + bitrate + container → `render-profiles.json`.
- **`list-render-profiles`** — list saved profiles.
- **`delete-render-profile`** — remove by name.

### Render / edit operations

- **`render-clips`** — "here are clips, render at 1080p/4K". Inputs: clip paths + target profile. Uses encoder from `system-profile.json`.
- **`render-from-library`** — assemble a single video from a library directory (concat, simple ordering).
- **`burn-subtitles`** — generate (whisper.cpp / faster-whisper) + burn into video. Default output: timestamped SRT alongside the rendered MP4.
- **`burn-graphics`** — overlay lower thirds, watermarks, image graphics onto a video at given timestamps.
- **`mlt-render`** — render an MLT XML timeline to a deliverable.
- **`open-in-kdenlive`** — open a project's `working/` dir in Kdenlive.
- **`generate-deliverables`** — for a rendered video, produce: thumbnail (poster frame, ffmpeg), description (LLM summary from transcript), transcription (timestamped SRT, default). Each can be requested individually.
- **`clean-transcription`** — heuristic + LLM cleanup of a raw transcript: filler words, obvious mistranscriptions, common-error patterns (e.g., "ya" → "yeah", proper-noun fixes from a per-project glossary).

### Media / clip management

- **`sort-clips-by`** — sort a clip folder into subfolders by aspect ratio, framerate, or resolution. (Existing `separate-4k.md`, `separate-photos-and-video.md` are precursors.)
- **`scrub-takes`** — remove accidental short takes (configurable minimum duration).
- **`dedupe-clips`** — find and remove duplicate / near-duplicate clips (hash + perceptual).

### NAS lifecycle

- **`setup-nas`** — register NAS path/mount + raw + render destination paths to `nas.json`.
- **`pull-from-nas`** — `rsync` raw clips from NAS into the active project's `raw/`.
- **`push-to-nas`** — `rsync` rendered output up to the NAS render destination.

## Suggested build order

1. `profile-system` (other skills depend on it)
2. Render profile CRUD (`save-render-profile`, `list-render-profiles`)
3. `render-clips` (centerpiece — proves the data-store + profile + encoder pipeline)
4. `generate-deliverables` (high value, mostly ffmpeg + transcription)
5. `clean-transcription`
6. `burn-subtitles`, `burn-graphics`
7. NAS skills
8. Clip management (`sort-clips-by`, `scrub-takes`, `dedupe-clips`)
9. `mlt-render`, `open-in-kdenlive`, `render-from-library`

## Naming pattern across media plugins

The audio and image plugins use a single-tier workspace (no index/project split — just `workspace-setup` / `open-workspace` / `new-project`). Video is the only one with the two-tier `index` + `project` split, because video projects are heavy and benefit from a richer subfolder layout.
