# Architecture

`open-sora-lite` is a layered system: a Rust inference core, a data pipeline, and an optional HTTP server. Python bindings wrap the core for integration into existing ML stacks.

## Workspace layout

```
open-sora-lite/
├── crates/
│   ├── osl-core/          # Inference core: model loading, sampling, VAE decode
│   ├── osl-pipeline/      # Data pipeline: tokenize, image preprocess, video postprocess
│   └── osl-server/        # HTTP server with SSE progress streaming
├── python/                # PyO3 bindings, importable as `open_sora_lite`
├── examples/              # CLI examples for each supported model
└── docs/                  # User + contributor docs
```

## Core loop

```
prompt/image ─▶ [osl-pipeline] tokenize + preprocess
                          │
                          ▼
              [osl-core] diffusion sampling loop
                          │
                          ▼
              [osl-core] VAE decode latent → frames
                          │
                          ▼
              [osl-pipeline] encode frames → MP4
```

## Module responsibilities

### `osl-core`

The inference core. Responsibilities:
- Load model checkpoints (safetensors) via `candle-core`
- Manage device placement (CUDA / CPU / Metal)
- Run the diffusion sampling loop (DPM-Solver, Euler)
- Decode latent representations to RGB frames via VAE

`osl-core` is model-agnostic: each supported model implements the `VideoModel` trait.

```rust
pub trait VideoModel: Send + Sync {
    fn load(checkpoint: &Path, device: &Device) -> Result<Self, Error>;
    fn text_to_video(&self, prompt: &str, opts: &GenOpts) -> Result<LatentVideo, Error>;
    fn image_to_video(&self, image: &Image, prompt: &str, opts: &GenOpts) -> Result<LatentVideo, Error>;
    fn vae_decode(&self, latent: &LatentVideo) -> Result<FrameBuffer, Error>;
}
```

### `osl-pipeline`

The data pipeline. Responsibilities:
- Tokenize text prompts via CLIP/T5
- Preprocess conditioning images (resize, normalize)
- Encode decoded frames to MP4 via ffmpeg
- Apply post-processing (interpolation, upscaling)

### `osl-server`

An optional HTTP server. Responsibilities:
- Expose `POST /generate` with JSON params
- Stream progress via SSE (`event: progress`, `event: complete`)
- Manage a request queue with configurable concurrency
- Health check endpoint for load balancers

## Threading model

- **Inference thread.** One dedicated tokio task per active generation, pinned to a CUDA stream.
- **Pipeline threads.** Preprocessing and postprocessing run on a separate thread pool to avoid blocking inference.
- **Server threads.** Axum's async runtime handles HTTP; long-running generations are offloaded to the inference task pool.

## Memory management

Video generation is memory-intensive. `open-sora-lite` employs:
- **Chunked latent processing.** Long videos are split into chunks; VAE decode processes one chunk at a time.
- **Checkpoint memory mapping.** Large checkpoints are mmap'd rather than loaded into RAM.
- **Gradient-free inference.** No autograd tape; we use `candle`'s inference-only mode.

See [docs/COLD-START-OPTIMIZATION.md](docs/COLD-START-OPTIMIZATION.md) for the warm-start strategy that cuts model load from 14s to 3.2s.
