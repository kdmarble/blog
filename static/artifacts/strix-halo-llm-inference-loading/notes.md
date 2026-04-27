---
type: notes
status: ready
area: video-vis
created: 2026-04-26
---

# LLM Inference: Loading & Quantization

**Video:** Kilbright's Code — LLM Inference Engine Loading & Quantization
**URL:** https://youtu.be/B18zBnjZKmc
**Generated:** 2026-04-26
**Screencast:** `strix-halo-llm-inference-screencast-2026-04-26.webm`
**Poster:** `strix-halo-llm-inference-screencast-poster.jpg`

## Concepts Extracted

1. **Memory Hierarchy** (0:50-5:10) — SSD → RAM → GPU model loading, memory mapping, lazy vs eager loading. Visualization: layered memory stack with data flow animation.

2. **Quantization as Compression** (6:40-9:50) — BF16 → int4, symmetric vs asymmetric quantization, range mapping. Visualization: precision levels showing value distribution.

3. **K-Quants Hierarchical Scaling** (10:13-11:44) — 256 weights → 8 groups of 32, mixed precision per layer. Visualization: tree structure showing hierarchical quantization.

4. **Salient Weight Detection** (11:51-13:43) — AWQ/EXL2 finding important weights via activation magnitude, Hessian matrix sensitivity. Visualization: weight heatmap showing salient vs normal weights.

5. **Quantization Method Comparison** (13:30-end) — RTN, AWQ, EXL2, GGUF, FP8, MVF4 tradeoffs. Visualization: comparison chart with speed vs quality axes.

## Best Concepts for Visualization

1. Memory hierarchy with lazy loading (concept 1)
2. Symmetric vs asymmetric quantization (concept 2)
3. K-quants hierarchical tree (concept 3)
4. Salient weight heatmap (concept 4)
