# Systems, Optimizers & Data Notes (Dense SOTA as of August 2026)

Dense technical notes on second-moment optimizers, orthogonalization kernels, attention acceleration, numeric precision, and data mixtures.

---

## 1. Optimizers

### Muon² (`2604.09967`)
- **Mechanism**: Evaluates running second-moment gradient statistics before applying orthogonalization polar decomposition:
  $$\text{polar}(G \odot V^{-1/2})$$
  computed via truncated Newton–Schulz series iterations.
- **Scope**: Applied to all 2D internal hidden weight matrices (attention projections, MLP linear layers).
- **Embeddings & 1D**: Token embeddings, normalization parameters (RMSNorm scales), and final `lm_head` remain on AdamW with standard decoupled weight decay.
- **Learning Rate Matching**: Muon learning rates must be scaled to match AdamW update root-mean-square ($\text{RMS}$):
  $$\eta_{\text{Muon}} = \eta_{\text{AdamW}} \cdot \frac{\text{RMS}(U_{\text{AdamW}})}{\text{RMS}(U_{\text{Muon}})}$$
  Never copy AdamW scalar learning rates directly to Muon.

### KL-SOAP (`2607.20548`)
- **Mechanism**: Layerwise-adaptive orthogonal preconditioning with Kronecker factorization under Kullback-Leibler curvature bounds.
- **Reference Implementation**: `https://github.com/NVIDIA-NeMo/Emerging-Optimizers` (and Megatron-LM).
- **Selection Rule**: Co-default alongside Muon² when GPU memory accommodates Kronecker state tensors and batch size is large ($\sim 100\text{M}$ tokens).

### Muon Variants (Literature & Provenance)
- **MONA (`2605.26842`)**: Momentum-orthogonalized Newton acceleration.
- **HTMuon (`2603.10067`)**: Heavy-tailed adaptive Muon (`https://github.com/TDCSZ327/HTmuon`).
- **Variance-Adaptive Muon (`2601.14603`)**: Adaptive variance scaling for NS iterations.
- **SF-NorMuon (`2605.23061`)**: Scale-free normalized Muon variant.
- **Newton-Muon (`2604.01472`)**: Higher-order curvature integration.

---

## 2. Kernels & Substrates

### FlashAttention-4 (`2603.05451`)
- **Repository**: `https://github.com/Dao-AILab/flash-attention` (FA4 CuTeDSL path).
- **Key Features**: Blackwell-first CuTe-DSL implementation, 2-CTA MMA tiling, software-emulated exponentiation, deterministic backward pass for RL pipelines.
- **Performance**: Up to $1.3\times$ speedup vs cuDNN 9.13 BF16 on NVIDIA B200.
- **Hardware Policy**: Blackwell/Hopper optimized; do not claim FA4 speedup metrics on A100.

### Gram Newton-Schulz (Dao Lab 2026 Blog)
- **Source**: Dao Lab Blog (`https://dao-lab.ai/blog/2026/gram-newton-schulz/`).
- **Repository**: `https://github.com/Dao-AILab/gram-newton-schulz`.
- **Mechanism**: Symmetric CuTe-DSL GEMMs in QuACK substrate; delivers $\sim 2\times$ speedup over standard NS series, serving as a drop-in polar decomposition replacement.

### Hierarchical Muon (HiMuon `2606.27216`)
- **Mechanism**: Tiled Triton Newton-Schulz kernel with SRAM-resident execution path for tile dimensions $T \le 128$.
- **Selection**: Verified paper reference; prefer Gram-NS on Hopper/Blackwell when available.

---

## 3. Numeric Precision & Quantization

### Standard Precision
- Default: BF16 computation with optional MXFP8 linear layer representations.

### Quartet II NVFP4 Training (`2601.22813`)
- **Repository**: `https://github.com/IST-DASLab/Quartet-II`.
- **Mechanism**: NVFP4 pretraining using Multi-Scale Error-Decoupled Estimator with Noise shaping (MS-EDEN).

### MI355X MXFP4 (`2605.09825`)
- **Mechanism**: Hardware-accelerated microscopic scaling FP4 (MXFP4) training validated on AMD MI355X hardware.

---

## 4. Pretraining Data Curricula

### Dolma 3 Dataset & Curriculum (`2512.13961`)
- **Repository**: `https://github.com/allenai/dolma3` (Dataset: `allenai/dolma3_mix-6T`).
- **Curriculum**: Broad pretraining mixture across diverse domains followed by targeted high-quality decay/annealing (Dolmino-style anneal).

### Read-Next Experimental Mixtures
- **DeMix (`2602.00747`)**: Deduplicated and domain-rebalanced pretraining mix (`https://github.com/Lucius-lsr/DeMix`).
- **CausalMix (`2607.01104`)**: Causal structure-guided token filtering.
- **OP-Mix (`2605.15220`)**: Optimization-profiled pretraining dataset allocation.
