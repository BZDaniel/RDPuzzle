# EdgeMatch Neural Model

## Overview

EdgeMatch is a self-trained convolutional neural network that predicts whether two 64x64 tiles are likely to be adjacent in a reconstructed RDP session screenshot. It turns tiles into direction-aware embeddings and compares them by cosine similarity.

Tiles that are likely to be actual neighbors end up with similar embeddings. The model was trained on real RDP cache captures with hard-negative mining and cross-frame contrastive loss.

## What the Model Predicts

Given a tile and a direction (left, right, top, bottom), the model produces a fixed-length embedding vector. Two embeddings from opposite sides of the same shared edge should have high cosine similarity. Non-neighbor pairs should have low similarity.

This is a ranking signal, not a binary classifier. Higher similarity means the tile is *more likely* to be the correct neighbor, relative to other candidates.

## Inputs

| Input | Shape | Type | Description |
|---|---|---|---|
| Tile | `(batch, 3, 64, 64)` | float32 (0–1) | RGB tile pixels |
| Direction | `(batch,)` | int64 | 0=left, 1=right, 2=top, 3=bottom |

Before the CNN sees the tile, it is rotated or flipped so the edge being matched is always on the right side (canonicalization). This lets the network focus on learning right-edge matching while the direction is handled separately through learned embeddings and FiLM modulation.

## Output

| Output | Shape | Type |
|---|---|---|
| Embedding | `(batch, 256)` or `(batch, 512)` | float32, L2-normalized |

## Why Edge Matching Helps

Simple pixel comparisons (HSV, Pearson) work for obvious boundaries but fail on:
- Photos and gradients
- Anti-aliased UI elements
- Text-heavy tiles
- Noisy regions
- Repeated UI patterns where color alone is ambiguous

EdgeMatch learns which visual features actually predict tile adjacency, making it robust in cases where pixel-level metrics are unreliable.

## ONNX Models

Two ONNX models are included in the repository and release assets:

### EdgeMatch Small (width=1, embed_dim=256)

- **File:** `EdgeMatch_Small.onnx` (~40 MB)
- **Parameters:** ~11.5M
- **Performance:** Fast; runs well on integrated graphics and via WASM fallback
- **Channel progression:** 3 → 32 → 64 → 128 → 256 (via four ResNet stages, GroupNorm)

### EdgeMatch Big (width=2, embed_dim=512)

- **File:** `EdgeMatch_Big.onnx` (~116 MB)
- **Parameters:** ~44.5M
- **Performance:** More accurate, roughly 4× slower; benefits from a dedicated GPU
- **Channel progression:** 3 → 64 → 128 → 256 → 512

### Architecture (both models)

- ResNet-style backbone with GroupNorm (not BatchNorm, which behaves poorly with small per-GPU batches)
- Per-direction FiLM (Feature-wise Linear Modulation) applied after the backbone, scaling features differently per direction
- Dedicated edge branch: a small conv net that processes only the right-edge strip (16px wide) of the canonicalized tile
- Learned 32-dim side embedding for each direction
- Two hidden layers (1024 → 512 → embed_dim) with LayerNorm, GELU, and dropout

## How It Runs

- **Inference:** ONNX Runtime Web via `onnxruntime-web` library
- **Backend:** WebGPU when available, WASM fallback
- **Fully local:** No data leaves the browser. The model is downloaded once and cached. All inference happens client-side.
- **Model loading:** Models are loaded from `media.githubusercontent.com` (Git LFS) on first use and cached by the browser.

## Training

- **Training data:** Real RDP bitmap cache captures, split into 64x64 tiles
- **Loss:** Information Noise Contrastive Estimation (InfoNCE) with positive pairs from true adjacent edges and hard-negative pairs mined from visually similar but incorrect edges
- **Hard negatives:** Materialized during preprocessing by HSV similarity search — for each true pair, the script finds the visually closest non-neighbor edges and uses them as negatives
- **Augmentations:** Light color jitter, no geometric transforms (orientation is meaningful)
- **Validation:** Cross-frame accuracy — trains on one set of RDP session frames and validates on held-out frames from different sessions, ensuring the model generalizes rather than memorizing

## Limitations

- **False positives:** The model can confidently match tiles that look similar but are not actually neighbors. Repeated UI patterns (identical buttons, same-colored regions) are a common source of false positives.
- **Not proof:** EdgeMatch similarity is a *ranking signal* to prioritize candidates, not proof of adjacency. Always verify matches visually.
- **Solid tiles:** Flat, single-color tiles carry no edge information and the model performs poorly on them. The application's low-information tile filter handles this.
- **Model scale:** The Small model is fast but may miss subtle match signals. The Big model is more accurate but slower — pick based on your GPU and patience.
- **Training distribution:** The model was trained on a specific set of RDP cache captures. It may generalize poorly to very different display content (e.g., 3D applications, video playback).
