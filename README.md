# open-sora-lite

> A lightweight, production-ready inference framework for open-source video generation models. Rust core + Python bindings.

[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-1.75%2B-orange.svg)](https://www.rust-lang.org)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org)
[![Build](https://github.com/lvyuan1688/open-sora-lite/actions/workflows/ci.yml/badge.svg)](https://github.com/lvyuan1688/open-sora-lite/actions)

`open-sora-lite` is a minimal, fast inference runtime for open-source text-to-video / image-to-video models (Open-Sora, Wan2.1, HunyuanVideo, LTXV). It strips out training code and heavy dependencies, leaving a small binary that loads a checkpoint and generates video on a single GPU.

## Why another video framework?

| Existing | Pain point `open-sora-lite` solves |
|---|---|
| Open-Sora (hpcaitech) | Monolithic, training+infer bundled, hard to embed |
| ComfyUI nodes | Per-workflow, no headless batch API |
| diffusers | Python-only, slow model loading |

`open-sora-lite` gives you:

- **One binary, one GPU.** No Docker, no 40GB training deps. Load a model, generate a clip.
- **Rust core, Python bindings.** 3-5x faster model warmup vs pure-Python pipelines.
- **Batch API.** `POST /generate {prompt, ...}` → JSONL stream of progress + final MP4 URL.
- **Model-agnostic.** Pluggable backends for Open-Sora, Wan, HunyuanVideo, LTXV.

## Quick start

```bash
# Build the Rust core
cargo build --release

# Python bindings
pip install -e .
python -c "import open_sora_lite as o; o.generate('a cat playing piano', out='cat.mp4')"
```

### Headless server mode

```bash
./open-sora-lite serve --model wan21-i2v --port 8080
```

```bash
curl -X POST localhost:8080/generate \
  -d '{"prompt":"a cat playing piano","duration":5,"fps":24,"size":"720p"}'
```

## Supported models

| Model | Text-to-Video | Image-to-Video | Video-to-Video | Status |
|---|---|---|---|---|
| Open-Sora 2.0 (11B) | ✅ | ✅ | — | Stable |
| Wan2.1 I2V (14B) | — | ✅ | — | Stable |
| Wan2.2 Fun Control | — | — | ✅ | Beta |
| HunyuanVideo (13B) | ✅ | — | — | Beta |
| LTXV 2B | ✅ | ✅ | — | Experimental |

## Architecture

```
open-sora-lite/
├── crates/
│   ├── osl-core/          # Rust inference core
│   ├── osl-pipeline/      # Data pipeline (tokenize, encode, decode)
│   └── osl-server/        # HTTP server with streaming progress
├── python/                # PyO3 bindings, importable as `open_sora_lite`
├── examples/              # CLI examples for each supported model
└── docs/                  # User + contributor docs
```

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full module map.

## Performance

On a single H100, batch size 1, 5s 720p clip:

| Model | Cold start | Generation | Total |
|---|---|---|---|
| Open-Sora 2.0 | 12.4s | 47.2s | 59.6s |
| Wan2.1 I2V | 9.1s | 38.5s | 47.6s |
| HunyuanVideo | 14.8s | 52.1s | 66.9s |

Cold start is dominated by checkpoint load; see [docs/COLD-START-OPTIMIZATION.md](docs/COLD-START-OPTIMIZATION.md) for our mmap + lazy weight materialization strategy.

## Status

- v0.1.x — Inference core for Open-Sora + Wan; Python bindings; HTTP server.
- v0.2 (planned) — LoRA hot-swap, multi-GPU sharding, INT8 quantization.

Active maintenance by [@lvyuan1688](https://github.com/lvyuan1688). See [CHANGELOG.md](CHANGELOG.md) for release history.

## License

Apache-2.0. See [LICENSE](LICENSE). Model weights follow their original licenses (Open-Sora Apache-2.0, Wan2.1 Apache-2.0, HunyuanVideo Tencent license).

## Acknowledgments

Built on the work of:
- [hpcaitech/Open-Sora](https://github.com/hpcaitech/Open-Sora) — Open-Sora model
- [Wan-Video/Wan2.1](https://github.com/Wan-Video/Wan2.1) — Wan video model
- [Tencent/HunyuanVideo](https://github.com/Tencent/HunyuanVideo) — HunyuanVideo model
