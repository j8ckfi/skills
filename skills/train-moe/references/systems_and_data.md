# Systems & Optimization Notes (MoE SOTA as of August 2026)

Dense technical notes on optimizers, execution megakernels, communication primitives, and training datasets.

---

## 1. Optimizers

### Muon² (`2604.09967`)
- **Core Mechanism**: Second-moment preconditioned orthogonalized Newton–Schulz iterations.
- **Update Rule**: Evaluates running second-moment gradient statistics before applying the orthogonalization polar decomposition step $\text{polar}(G \odot V^{-1/2})$ via truncated Newton–Schulz series.
- **Application Scope**: Assigned to all 2D hidden weight matrices (expert projections, attention projection layers, QK/VO matrices).
- **Embeddings & Heads**: Token embeddings and final `lm_head` remain on standard AdamW with decoupled weight decay.

### KL-SOAP (`2607.20548`)
- **Core Mechanism**: Kronecker-factored second-order optimization with Layerwise-Adaptive Orthogonal Preconditioning under Kullback-Leibler curvature bounds.
- **Trade-off Profile**: Maximizes sample efficiency under large-batch regimes; introduces higher optimizer memory overhead relative to Muon².
- **Selection Rule**: Deploy when GPU memory headroom accommodates large-batch state tensors; fall back to Muon² when scaling activation batch size or expert capacity.

---

## 2. Megakernels & Hardware Systems

### Mixture-of-Kittens (MoK) on GB200 / GB300 NVL72
- **Repository**: `https://github.com/cursor/mixture-of-kittens`
- **Technical Report**: `https://cursor.com/blog/mixture-of-kittens`
- **Target Hardware**: NVIDIA Blackwell architecture (SM100 / SM103), PyTorch 2.10+, CUDA 13.
- **Kernel Topology**:
  - Fused deterministic MoE megakernel explicitly optimized for DeepSeek-V3 / V4 style layer topologies (shared expert + hundreds of fine-grained routed experts).
  - Co-designs token dispatch, MXFP8 matrix multiplication, expert GEMM scheduling, and token reduction in a single unified warp group pipeline.
- **Measured Performance**:
  - Up to $2.37\times$ forward pass throughput in MXFP8 compared to public DeepEP / Megatron stacks.
  - $\sim 1.41\times$ end-to-end training throughput (tokens per second) across 512-GPU Blackwell clusters.
- **Scope Restriction**: Hardware-locked to NVL72 / Blackwell NVLink fabric. For Hopper (H100/H200) or non-NVL configurations, route to standard DeepEP + Megatron-LM / Megatron-Core MoE pipelines.

---

## 3. Pretraining Data Curricula

### Fully-Open Pipeline: OLMo 3 / Dolma 3 (`2512.13961`)
- Curated open web, synthetic reasoning pipelines, math, and code mixtures with verifiable provenance and deduplication.

### Industrial Scale Mix: Nemotron-3 (`2604.12374`)
- 20T to 25T token industrial-scale open recipe balancing dense coding, math derivations, multilingual corpora, and instruction-filtered pretraining tokens.
