# Architecture & Residual Notes (Dense SOTA as of August 2026)

Dense technical notes on backbone design, residual stream experiments, and native low-bit quantization.

---

## 1. Backbone: Olmo 3 7B (`2512.13961`)

### Architectural Overview
- **Shape & Geometry**: Standard dense decoder-only transformer shape (submitted 2025-12-15, v2 2026-04-14).
- **Open Stack**: Standardized on open-source weights, configs, and training harnesses via OLMo-core.
- **Repository**: `https://github.com/allenai/OLMo-core`
- **Honest Gap**: No 2026 dense-backbone paper replaced Llama-like dense geometry the way Kimi K3 replaced DeepSeek-V3 in sparse MoE. Olmo 3 7B is the verified default.
- **Legacy Distinction**: OLMo-2 (`2501.00656`) is superseded and should not be used as the base design.

---

## 2. Residual Stream Dynamics & Experiments

### Standard Dense Residual Stream
- Default baseline: standard identity residual connection $x_{l+1} = x_l + f(x_l)$.
- Stable for ~7B parameter class up to typical 32–40 layer depths without requiring dynamic routing or multi-stream manifolds.

### Optional Experiment: Manifold-Constrained Hyper-Connections (mHC `2512.24880`)
- Explores multi-channel hyper-connections ($n=4$) with Sinkhorn-normalized doubly-stochastic transition matrices.
- Primarily validated and scaled on large MoE models; can be evaluated as an experimental stability ablation on ultra-deep dense models.

### Pointers (Not Defaults)
- **Extreme Hyper-Connections (xHC `2607.14530`)**: Scaled on MoE (18B/28B with $N=16, k=4$). Pointer only; not a dense recipe default.
- **Attention Residuals (AttnRes `2603.15031`)**: Dynamic attention-weighted skip connections across layer blocks. Scaled on MoE; not a dense recipe default.

---

## 3. Native Low-Bit Training

### Sparse-BitNet (`2603.05168`)
- **Repository**: `https://github.com/AAzdi/Sparse-BitNet`
- **Mechanism**: Native 1.58-bit ternary weight pretraining from scratch with sparse activation patterns.
- **Scope**: Deployed only if building a native 1.58-bit model from scratch. Not a substitute for standard BF16 / FP4 dense runs.
