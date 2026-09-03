# Data Pipeline

open-sora-lite requires text-video pairs in a specific format. The pipeline handles ingestion,
filtering, and sharding.

## Stages

```
raw videos → decode → caption → filter → normalize → shard
```

### 1. Decode

Videos are decoded with FFmpeg at the target FPS (default 24). Frames are stored as
lossless PNG sequences in a temp directory. Audio is discarded (text-video only).

### 2. Caption

Each video gets a text caption via one of:
- **BLIP-2** — automatic captioning, good general quality.
- **Manual** — human-written captions for curated datasets.
- **LLM-refined** — BLIP-2 caption → LLM rewrite for better detail.

### 3. Quality filter

| Filter | Threshold | Purpose |
|--------|-----------|---------|
| Aesthetic score | > 4.5 | Remove ugly/blurry clips |
| Motion score | 0.1 – 0.9 | Remove static or chaotic clips |
| Text overlap | < 0.3 | Remove text-heavy frames (subtitles, watermarks) |
| Resolution | >= 480p | Remove low-res source material |

### 4. Normalize

- Resize shortest side to target (480p/720p).
- Center-crop to target aspect ratio (16:9 or 9:16).
- Re-encode to H.264 CRF 18 for consistent quality.

### 5. Shard

Output is sharded into TFRecord files, each containing 1000 video-caption pairs. Metadata
(duration, resolution, aesthetic score) is stored in a sidecar JSONL.
