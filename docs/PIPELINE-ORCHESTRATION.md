# Video Pipeline Orchestration

`open-sora-lite` treats video generation as a pipeline of discrete, composable stages. This document describes the orchestration layer that chains those stages.

## Pipeline stages

| Stage | Input | Output | Cancellable |
|---|---|---|---|
| `text_encode` | prompt string | CLIP/T5 text features | Yes |
| `image_preprocess` | conditioning image | normalized tensor | Yes |
| `diffusion_sample` | text features + noise schedule | latent video | Yes (checkpoint-backed) |
| `vae_decode` | latent video | frame buffer | Yes |
| `encode_video` | frame buffer | MP4 file | No (fast) |

## Composition

Stages are composed via a directed acyclic graph (DAG). The default T2V pipeline:

```
text_encode ─▶ diffusion_sample ─▶ vae_decode ─▶ encode_video
```

I2V adds an image preprocessing stage that feeds into the diffusion sampler as conditioning:

```
text_encode ──┐
              ├─▶ diffusion_sample ─▶ vae_decode ─▶ encode_video
image_preprocess ─┘
```

## Cancellation

Each stage checks a cancellation token between steps. The diffusion sampler is the longest-running stage; it checks the token every N steps (default N=5). Cancellation triggers a graceful shutdown:

1. Stop the current stage.
2. Discard intermediate results.
3. Release VRAM.
4. Return a `Cancelled` error to the caller.

## Checkpointing

The diffusion sampler can save intermediate latents to disk at configurable intervals. This enables:

- **Resume after crash.** Restart from the last checkpoint instead of from scratch.
- **Branching.** Save a latent at step 25, then run two different continuations (e.g., different prompts for the second half).

```toml
[checkpoint]
enabled = true
interval_steps = 10
dir = "/tmp/osl-checkpoints"
max_keep = 3
```

## Parallel pipelines

Multiple pipelines can run concurrently, subject to VRAM constraints. The orchestrator uses a semaphore to limit concurrent diffusion sampling (default: 1 per GPU). Other stages (text encode, VAE decode) can overlap with diffusion on a different CUDA stream.

## Error handling

Stage failures propagate as typed errors:

```rust
pub enum PipelineError {
    StageFailed { stage: StageKind, source: anyhow::Error },
    Cancelled,
    CheckpointCorrupt(PathBuf),
    VramExhausted { requested: usize, available: usize },
}
```

The orchestrator retries idempotent stages (text encode, image preprocess) up to 2 times. Non-idempotent stages (diffusion sampling) are not retried automatically; the caller must decide whether to restart from a checkpoint.
