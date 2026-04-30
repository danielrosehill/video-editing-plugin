---
name: dedupe-clips
description: Find duplicate or near-duplicate clips in a folder. Two modes — `exact` (sha256 of file bytes, fastest, catches true duplicates) and `perceptual` (sample frames + dHash, catches re-encodes / format converts of the same source). Use when the user says "find duplicate clips", "dedupe this folder", "remove duplicate videos", "flatten and dedupe". Quarantines duplicates to `_duplicates/` by default; never deletes without explicit confirmation.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(mv *), Bash(rm *), Bash(find *), Bash(stat *), Bash(sha256sum *), Bash(sort *), Bash(uniq *), Bash(awk *), Bash(ffmpeg *), Bash(ffprobe *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(printf *), Bash(python3 *), Read, Write
---

# Dedupe Clips

Identify duplicate or near-duplicate video clips in a folder. Always quarantine, never auto-delete.

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Source folder | required |
| Mode | `exact` (sha256) — `perceptual` (frame-hash) for finding re-encodes |
| Recursive | `no` |
| Hamming threshold (perceptual) | `8` (out of 64 bits) |
| Frames to sample (perceptual) | `5` |
| Action | `quarantine` (move dups to `_duplicates/`) — `delete` requires explicit confirmation |
| Keeper | `largest` (keep biggest file) — alternatives: `oldest`, `newest`, `first` (alpha) |

### 2. Enumerate

```bash
SRC="$(realpath "$INPUT")"
test -d "$SRC" || { echo "Not a folder: $SRC"; exit 1; }
if [ "$RECURSIVE" = "yes" ]; then
  mapfile -t FILES < <(find "$SRC" -type f \( -iname '*.mp4' -o -iname '*.mov' -o -iname '*.mkv' -o -iname '*.avi' -o -iname '*.m4v' -o -iname '*.webm' \) -not -path '*/_duplicates/*')
else
  mapfile -t FILES < <(find "$SRC" -maxdepth 1 -type f \( -iname '*.mp4' -o -iname '*.mov' -o -iname '*.mkv' -o -iname '*.avi' -o -iname '*.m4v' -o -iname '*.webm' \))
fi
```

### 3a. Mode: exact

```bash
sha256sum "${FILES[@]}" | sort > /tmp/hashes.txt
awk '{print $1}' /tmp/hashes.txt | sort | uniq -d > /tmp/dup_hashes.txt
```

For each duplicate hash, group the files. Apply the keeper rule and mark the rest for quarantine.

### 3b. Mode: perceptual

For each file, sample `N` frames at evenly-spaced timestamps and compute a dHash per frame. Concatenate the per-frame dHashes; two clips are near-duplicates if **all** frame-pair Hamming distances are `≤ THRESHOLD`.

Use a Python helper (one-shot, inline) — only needs `Pillow` (already in many envs; if missing, instruct to `uv pip install Pillow` into the plugin venv).

```python
# /tmp/dhash_clip.py
import sys, subprocess, json, tempfile, os
from PIL import Image

def dhash(img, hash_size=8):
    img = img.convert("L").resize((hash_size+1, hash_size))
    bits = 0
    for y in range(hash_size):
        for x in range(hash_size):
            left = img.getpixel((x, y))
            right = img.getpixel((x+1, y))
            bits = (bits << 1) | (1 if left > right else 0)
    return bits

def hashes_for(path, n=5):
    dur = float(subprocess.check_output(
        ["ffprobe","-v","error","-show_entries","format=duration","-of","default=nw=1:nk=1",path]
    ).strip() or 0)
    if dur <= 0: return []
    out = []
    with tempfile.TemporaryDirectory() as td:
        for i in range(n):
            t = dur * (i+1) / (n+1)
            jpg = os.path.join(td, f"{i}.jpg")
            subprocess.run(["ffmpeg","-v","error","-ss",str(t),"-i",path,
                            "-frames:v","1","-vf","scale=64:64","-y",jpg], check=False)
            if os.path.exists(jpg):
                out.append(dhash(Image.open(jpg)))
    return out

if __name__ == "__main__":
    print(json.dumps({p: hashes_for(p) for p in sys.argv[1:]}))
```

Hamming compare:

```python
def hamming(a, b): return bin(a ^ b).count("1")
def near(hs1, hs2, thr):
    if len(hs1) != len(hs2) or not hs1: return False
    return all(hamming(a,b) <= thr for a,b in zip(hs1,hs2))
```

Run pairwise comparison; group transitively (union-find) so chains of near-dups become one cluster.

### 4. Preview

Print clusters:

```
Cluster 1 — keeping: holiday_v2.mp4 (412 MB, h264 1080p)
  duplicate: holiday_v2_copy.mp4   (412 MB)        [exact match]
  duplicate: holiday_export.mp4    (138 MB, hevc)  [perceptual, hamming=2]

Cluster 2 — keeping: GX010044.MP4 (3.2 GB)
  duplicate: GX010044 (1).MP4      (3.2 GB)        [exact match]
```

Show total clusters, total dup files, total reclaimable bytes. Ask the user to **confirm**. For `delete` mode, require an extra confirmation.

### 5. Apply

For `quarantine`:

```bash
mkdir -p "$SRC/_duplicates"
for f in "${DUPS[@]}"; do
  mv -n "$f" "$SRC/_duplicates/"
done
```

For `delete` (only after explicit second confirmation): `rm -f --` each file.

### 6. Optional: flatten + dedupe

If the source has nested subfolders and the user asks for "flatten and dedupe" (the legacy `videos-here` workflow):

1. Move every video file to the top level (renaming on collision: `name.mp4` → `name__1.mp4`).
2. Remove now-empty subfolders.
3. Run dedupe in `exact` mode on the flattened folder.

Always preview the move plan before executing. This supersedes the legacy `commands/videos-here.md`.

## Notes

- `exact` mode is fast and safe; `perceptual` mode is slower and can have false positives — always preview.
- Quarantine is the default; the `_duplicates/` folder is preserved for the user to spot-check before manual deletion.
- The keeper rule is applied per cluster, so the user always retains exactly one file per duplicate group.
- Hidden files and existing `_duplicates/` contents are excluded.
