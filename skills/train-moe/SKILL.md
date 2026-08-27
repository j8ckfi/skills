---
name: train-moe
description: Pretrains, optimizes, and post-trains sparse Mixture of Experts (MoE) models using verified August 2026 SOTA methods. Covers DeepSeek-V4 CSA+HCA hybrid attention with mHC residuals, Kimi K3 Delta Attention with block AttnRes and Stable LatentMoE, Nemotron-3 Super latent dispatch compression, Muon² and KL-SOAP 2nd-order optimizers, Mixture-of-Kittens (MoK) Blackwell NVL72 megakernels, MXFP4 QAT, and SAPO/SAO/OPDVR post-training. Use when designing, debugging, or auditing an MoE pretraining cluster run, selecting expert routing topologies, resolving communication bottlenecks, or post-training sparse models.
license: MIT
metadata:
  version: "0.1.0"
  as_of: "2026-08-27"
  library: https://github.com/j8ckfi/library
  library_query: "python -m library sota 'pretrain MoE'"
---

# Train MoE

Operating procedure for designing, configuring, executing, and auditing frontier Mixture of Experts (MoE) pretraining and post-training runs.

Sparse pretraining succeeds when token dispatch, residual stream stability, second-moment orthogonalization, and communication topologies align with target cluster hardware.

---

## 1. Default Recipe

The standard operating configuration for frontier sparse pretraining as of August 2026.

```
+-----------------------------------------------------------------------------------------+
|                                    DEFAULT STACK                                        |
+-------------------+---------------------------------------------------------------------+
| Backbone (Hybrid) | DeepSeek-V4 (CSA+HCA + mHC n=4 + DeepSeekMoE + MTP) [2606.19348]     |
| Alternate (Gated) | Kimi K3 (Delta Attention + MLA + Block AttnRes + LatentMoE)         |
| Compression       | LatentMoE latent-space token projection before All-to-All [2604.12374]|
| Hidden Optimizer  | Muon² (2nd-moment preconditioned Newton-Schulz) [2604.09967]        |
| Head/Embed Opt    | AdamW (decoupled weight decay, standard learning rate schedule)     |
| NVL72 Hardware    | Mixture-of-Kittens (MoK) fused deterministic megakernel [SM100/103] |
| Non-NVL72 Cluster | DeepEP + Megatron-Core MoE communication pipeline                   |
| Expert Precision  | MXFP4 Quantization-Aware Training (QAT) / MXFP8 compute             |
| Data Mixture      | Dolma 3 (open 2512.13961) or Nemotron-3 (20-25T industrial mix)     |
| Post-Train (RL)   | SAPO (adaptive-clip MoE RL 2511.20347) / SAO (agentic 2607.07508)   |
| Distill / Merge   | OPD / MOPD [2604.13016]; OPDVR if reward verifier active [2608.24696] |
+-------------------+---------------------------------------------------------------------+
```

---

## 2. Symptom Router

Locate the observed operational failure or workload constraint, then execute the corresponding checklist check.

| Observed Symptom / Operational Constraint | Check | Action |
|---|---|---|
| Pretraining an MoE model from scratch | C1, C2, C4 | Select DeepSeek-V4 or Kimi K3 backbone; configure Muon²; wire LatentMoE dispatch. |
| Deploying on GB200 / GB300 NVL72 (Blackwell SM100/103) | C3 | Load Mixture-of-Kittens (`cursor/mixture-of-kittens`) megakernel; enforce MXFP8. |
| Deploying on non-Blackwell / non-NVL72 clusters (e.g. H100/H200) | C3 | Route to DeepEP + Megatron-Core MoE; disable MoK kernel hooks. |
| Gradient norm explosion or residual collapse at depth >60 | C2 | Enforce mHC ($n=4$) Sinkhorn doubly-stochastic transition matrices or block AttnRes. |
| Inter-node All-to-All communication saturates cluster network | C1 | Enable Nemotron-3 Super LatentMoE dispatch compression before All-to-All. |
| Optimizer state OOM under large-batch training regimes | C4 | Switch from KL-SOAP to Muon²; verify hidden 2D matrices vs AdamW embedding split. |
| Post-training RL causing expert collapse or policy divergence | C5 | Replace vanilla GRPO with SAPO; route asynchronous agent rollouts to SAO. |
| Compressing specialist checkpoints or distilling into MoE | C6 | Run MOPD multi-teacher distillation; attach verifier for OPDVR if reward exists. |
| Target runtime requires sub-8-bit serving / deployment | C1, C3 | Enable native MXFP4 QAT during pretraining; do not rely on post-hoc PTQ-to-1.58. |

---

## 3. Operational Audit

Execute each verification check. Evaluate against strict PASS / FAIL criteria.

### C1 -- Architecture & Dispatch Compression

**Measure.** Inspect layer definitions for attention hybridity, hyper-connections, and routing dispatch communication.

**Pass.** 
1. Attention is CSA+HCA hybrid (`2606.19348`) or Kimi Delta Attention interleaved with gated MLA (`2607.24653`).
2. Residual connections use manifold constraints (mHC $n=4$, Sinkhorn normalization `2512.24880`) or Block Attention Residuals (`2603.15031`).
3. Cross-device All-to-All transfers run in compressed latent space (LatentMoE `2604.12374`) rather than uncompressed full-hidden states.

**Why.** Vanilla uncompressed All-to-All causes EP network saturation. Standard residual streams suffer from representation collapse or gradient explosion at scale.

**Fix.** Refactor layer stack to DeepSeek-V4 or Kimi K3 specification. Insert latent projection prior to dispatch.

### C2 -- Residual Stability & Routing Diversity

**Measure.** Log per-expert token routing histograms and residual stream variance across layers $[0, L]$.

**Pass.** 
1. Coefficient of variation ($CV = \sigma / \mu$) across expert assignment counts stays $<0.15$ during warm-up and steady state.
2. Residual stream variance growth is bounded sub-linearly across depth $L$.

**Why.** Uneven routing causes straggler stalls and dead experts. Unbounded residual growth destabilizes deep attention layers.

**Fix.** Enforce Sinkhorn doubly-stochastic normalization on transition matrices. Verify auxiliary load-balancing loss coefficients.

### C3 -- Training Megakernels & Compute Topology

**Measure.** Check cluster hardware architecture, PyTorch version, CUDA version, and active MoE execution kernel.

**Pass.**
- **If NVL72 (Blackwell SM100/103, PyTorch 2.10+, CUDA 13):** Mixture-of-Kittens (MoK) fused megakernel is active with deterministic MXFP8 dispatch and reduction.
- **If non-NVL72 / Hopper (H100/H200):** DeepEP and Megatron-Core MoE pipelines are configured. MoK is explicitly disabled.

**Why.** MoK delivers up to $2.37\times$ MXFP8 forward throughput and $\sim 1.41\times$ end-to-end tokens/s on 512 Blackwell GPUs by co-designing shared and routed expert execution. It is hardware-locked to Blackwell NVLink topologies; running MoK on unsupported hardware produces fatal runtime errors.

**Fix.** Match kernel runtime to physical cluster fabric.

### C4 -- Optimizer Parameter Partitioning

**Measure.** Audit optimizer parameter groups. Trace assignment of 2D hidden tensors vs 1D/embedding tensors.

**Pass.**
1. All 2D internal weight matrices (expert projections, attention weights) are optimized via Muon² (`2604.09967`) or KL-SOAP (`2607.20548`).
2. Token embeddings, normalization scales, and final `lm_head` remain on AdamW with standard decoupled weight decay.

**Why.** First-order AdamW on large hidden weight matrices exhibits inferior loss trajectories per FLOP compared to second-moment preconditioned Newton-Schulz orthogonal updates. Applying Muon to 1D vectors or token embeddings degrades lexical representations.

**Fix.** Partition parameters by dimension and module type into `muon_params` and `adamw_params`.

### C5 -- Post-Training & Sparse Policy Optimization

**Measure.** Check RL policy update objectives, advantage calculation methods, and rollout synchronization.

**Pass.**
1. Sparse MoE and VL RL runs use SAPO (`2511.20347`) with dynamic per-expert clipping boundaries.
2. Asynchronous multi-turn agentic environments use SAO (`2607.07508`) with off-policy importance correction.
3. Vanilla GRPO is not used for sparse MoE backbones.

**Why.** Standard GRPO creates extreme gradient variance across sparse routing gates, inducing routing collapse and policy entropy collapse.

**Fix.** Replace RL step update with SAPO adaptive clipping loss formulation.

### C6 -- Distillation & Specialist Merging

**Measure.** Verify distillation loss formulation and verifier presence.

**Pass.**
1. Transferring reasoning from teacher models uses On-Policy Distillation (OPD) or multi-teacher MOPD (`2604.13016`).
2. When formal verifiers or verified unit tests exist, distillation uses OPDVR (`2608.24696`).

**Why.** Off-policy distillation causes distribution shift on multi-step reasoning trajectories. Unverified distillation propagates hallucinated intermediate tokens.

**Fix.** Wire student on-policy rollouts with verifier-regularized loss terms.

---

## 4. Hard Don'ts

Never permit these anti-patterns in configuration or code:

1. **Do not use AdamW on hidden weight matrices.** Hidden 2D projections must use second-moment orthogonal optimizers (Muon² or KL-SOAP).
2. **Do not copy vanilla DeepSeek-V3 as the modern template.** V3 lacks CSA+HCA hybrid attention, mHC manifold-constrained residuals, and LatentMoE dispatch compression.
3. **Do not use post-training PTQ-to-1.58 on unquantized checkpoints.** If low-bit weights are required for serving, train with native MXFP4 Quantization-Aware Training (QAT).
4. **Do not run vanilla GRPO on sparse MoE models.** Standard GRPO causes expert routing collapse; enforce SAPO for sparse RL and SAO for asynchronous agent rollouts.
5. **Do not invoke Mixture-of-Kittens (MoK) on non-Blackwell hardware.** MoK requires Blackwell SM100/SM103 NVL72 architecture. Use DeepEP / Megatron-Core for other clusters.

---

## 5. Provenance & Literature Map

Direct mappings from verified library graph nodes (`j8ckfi/library`) to operational primitives.

| Operational Component | Library Node ID | Reference / Identifier | Code Repository / Source |
|---|---|---|---|
| DeepSeek-V4 Architecture | `arch-deepseek-v4` | arXiv:`2606.19348` | Frontier sparse CSA+HCA specification |
| Hyper-Connections (mHC $n=4$) | `arch-mhc-residuals` | arXiv:`2512.24880` | Sinkhorn doubly-stochastic residuals |
| Kimi K3 Backbone & LatentMoE | `arch-kimi-k3` | arXiv:`2607.24653` | https://github.com/MoonshotAI/Kimi-K3 |
| Attention Residuals (AttnRes) | `arch-block-attnres` | arXiv:`2603.15031` | Block-level attention skip connections |
| LatentMoE Dispatch Compression | `arch-latentmoe-disp` | arXiv:`2604.12374` | Nemotron-3 Super latent All-to-All |
| Muon² 2nd-Moment Optimizer | `opt-muon2-ns` | arXiv:`2604.09967` | Second-moment Newton-Schulz iterations |
| KL-SOAP Optimizer | `opt-kl-soap` | arXiv:`2607.20548` | Kronecker-factored 2nd-order optimizer |
| Mixture-of-Kittens Megakernel | `sys-mok-nvl72` | https://cursor.com/blog/mixture-of-kittens | https://github.com/cursor/mixture-of-kittens |
| Open Data Mix (Dolma 3) | `data-dolma-3` | arXiv:`2512.13961` | OLMo 3 / Dolma 3 open pretraining recipe |
| Industrial Data Mix (Nemotron-3)| `data-nemotron-3` | arXiv:`2604.12374` | 20-25T token industrial open mix |
| SAPO Adaptive Policy RL | `post-sapo-rl` | arXiv:`2511.20347` | Adaptive clipping sparse MoE RL |
| SAO Asynchronous Agent RL | `post-sao-agent` | arXiv:`2607.07508` | Scalable asynchronous optimization |
| OPD / MOPD Distillation | `post-mopd-distill` | arXiv:`2604.13016` | On-policy specialist distillation |
| OPDVR Verifier Distillation | `post-opdvr-distill`| arXiv:`2608.24696` | Verifier-regularized policy distillation |

---

## 6. Audit Report Format

Emit one line per operational check executed:

```
C1 PASS  CSA+HCA attention hybrid + LatentMoE compression verified
C2 PASS  routing CV 0.082 < 0.15; mHC Sinkhorn normalization active
C3 PASS  NVL72 Blackwell cluster detected; MoK megakernel active (1.41x e2e tok/s)
C4 PASS  Muon2 assigned to hidden matrices; AdamW on embeddings and lm_head
C5 SKIP  pretraining phase; post-training checks not applicable
C6 SKIP  distillation inactive
```

---

## 7. When This Skill Is Stale

This compiled skill reflects the frontier MoE training landscape as of **August 27, 2026**.

If the current date exceeds `as_of + 30 days` or a refreshed query to `j8ckfi/library` indicates movement in the `pretrain MoE` frontier, re-query the library before finalizing cluster configurations:

```bash
python -m library sota 'pretrain MoE'
```

Inspect the returned graph delta for updates to attention hybrids, second-order optimizers, kernel megakernels, or post-training algorithms.
