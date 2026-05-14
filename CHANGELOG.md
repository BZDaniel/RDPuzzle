# Changelog

## v0.1.0 - 2026-05-14

### Initial Release

- Neural EdgeMatch scoring (self-trained ONNX model, runs in-browser via WebGPU/WASM)
- Auto-stitching with configurable thresholds
- OCR (Tesseract + PaddleOCR)
- Bundled HSV metric (70% histogram overlap + 30% per-pixel edge continuity)
- Dynamic Pearson down-weighting on low-detail tiles
- Near-duplicate tile skipping on import (99% visual similarity)
- Low-information tile filtering
- Multi-tab workspaces
- Session save/load
- Loads BMC and BIN cache fragments
- Export reconstructed grids as images
