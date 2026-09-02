# Training & Fine-tuning

open-sora-lite ships a frozen base model; customization is done with low-rank adapters so a
single 24GB GPU is enough.

## LoRA on the DiT backbone

- Adapter rank r=32, alpha=64, applied to attention QKV + FFN projections.
- Base weights stay frozen; only ~0.4% of params train.
- Checkpoints are delta-only (~400MB) and can be stacked (weight-averaged) for ensembles.

## Dataset spec

A manifest (JSONL) of text–video pairs:
```
{"video":"clips/0001.mp4","caption":"a slow pan across a misty lake at dawn","duration":4.0}
```
Captions should describe motion, not just the frame. A 1–2k pair set is enough to shift style.

## Mixed-precision schedule

- AdamW, bf16 compute, fp32 master weights for the LoRA.
- LR 1e-4 cosine to 1e-5 over the run, 500-step warmup.
- Gradient checkpointing on every transformer block to halve activation memory.

## Single-GPU budget

On 24GB: latent batch 1, 480p, 49 frames, grad-accum 8 → ~21GB peak. For 12GB, drop to 240p.

## Evaluation

Run the fixed 20-prompt eval set after every 500 steps; log FVD + CLIP-text score. Stop when
FVD plateaus for 3 consecutive windows.
