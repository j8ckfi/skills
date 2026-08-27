# Architecture Notes (MoE SOTA as of August 2026)

Dense notes from literature on frontier sparse architectures, attention hybrids, hyper-connections, and routing.

---

## 1. DeepSeek-V4 (`2606.19348`)

### Architecture Overview
- **Hybrid Attention**: CSA (Compressed Self-Attention) + HCA (Hierarchical Cross-Attention) hybrid attention mechanism.
- **mHC (Manifold-Constrained Hyper-Connections)**: Extended from `2512.24880` with depth $n=4$.
  - Constrains residual stream updates via Sinkhorn-normalized doubly-stochastic transition matrices.
  - Mitigates representation collapse and stabilizes gradient propagation across ultra-deep networks without exploding norm.
- **Sparse Routing**: Retains DeepSeekMoE fine-grained expert routing topology + MTP (Multi-Token Prediction) training objective.
  - Distinct from vanilla DeepSeek-V3: replaces pure MLA / standard residual backbones with CSA+HCA and mHC.

---

## 2. Kimi K3 (`2607.24653`)

### Architecture Overview
- **Interleaved Attention**: Kimi Delta Attention interleaved with gated MLA (Multi-Head Latent Attention).
- **Block Attention Residuals (AttnRes)**: Employs Attention Residuals (`2603.15031`) operating over block-level boundaries.
  - Allows shallow representation bypass directly to deep layers via learned attention-weighted skip connections.
- **Routing & Capacity**: Stable LatentMoE with top-16 routed out of 896 fine-grained experts (16/896 active/total ratio).
- **Quantization-Aware Training**: Native MXFP4 QAT applied across expert feed-forward weights during pretraining.
- **Reference Implementation**: `https://github.com/MoonshotAI/Kimi-K3`

---

## 3. LatentMoE & Dispatch Compression: Nemotron-3 Super (`2604.12374`)

### Dispatch Topology
- **Latent Space Routing**: Input hidden states are projected down to a low-dimensional latent bottleneck before the cross-device All-to-All dispatch communication step.
- **Expert Input Reconstruction**: Hidden activations are reconstructed / expanded at the destination device before expert computation.
- **Communication Savings**: Drastically reduces all-to-all communication volume across high-degree expert parallelism (EP), eliminating the primary network bottleneck in large-scale cluster training.

---

## 4. Hyper-Connections (`2512.24880`)

### Mathematical Foundation
- Expands residual stream width via multi-channel hyper-connections ($n$-stream residuals).
- Uses iterative Sinkhorn operations to project inter-channel transition matrices onto the Birkhoff polytope (doubly stochastic matrices $\sum_j M_{ij} = 1, \sum_i M_{ij} = 1$).
- Prevents cross-channel signal explosion and ensures variance preservation over 100+ layer depths.

---

## 5. Block Attention Residuals (`2603.15031`)

### Structural Dynamics
- Replaces fixed identity residual streams $x_{l+1} = x_l + f(x_l)$ with dynamic, input-conditioned attention across preceding layer representations.
- Block AttnRes computes softmax weights over block-aggregated historical representations, allowing selective retrieval of low-level syntactic features in late-stage layers.
