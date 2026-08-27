---
name: train-dense
description: Pretrains dense (non-MoE) LMs, ~7B-class, using verified Aug 2026 SOTA: Olmo-3-shaped backbone, Dolma 3 curriculum, Muon² (KL-SOAP if memory), FlashAttention-4, Gram Newton-Schulz / Hierarchical Muon kernels, optional NVFP4/MXFP4. Use when training a dense 7B from scratch, auditing a dense pretrain run, choosing Muon vs AdamW vs SOAP, picking attention/optimizer kernels, or deciding FP4 vs BF16. Not for MoE (use train-moe) or 24GB LoRA (use peft-24gb when it exists).
license: MIT
metadata:
  version: "0.1.0"
  as_of: "2026-08-27"
  library: https://github.com/j8ckfi/library
  library_query: "python -m library sota 'pretrain dense 7B'"
---

# Train Dense

Operating procedure for designing, configuring, executing, and auditing dense (non-MoE) LM pretraining runs (~7B parameter class) and post-training branches.

Dense pretraining succeeds when parameter-partitioned second-moment optimization, hardware-matched attention and orthogonalization kernels, multi-stage staged curricula, and numeric precisions are strictly aligned.

---

## 1. Honest Gap

There is no 2026 dense-backbone paper that replaced Llama-like architectures the way Kimi K3 replaced DeepSeek-V3 in the sparse ecosystem. The default pretraining architecture remains the standard dense transformer shape instantiated by Olmo 3 7B (`2512.13961`).

Sparse architectures, routing topologies, and sparse kernels (DeepSeek-V4, Kimi K3, LatentMoE, Mixture-of-Kittens) belong exclusively in `train-moe`.

---

## 2. Default Recipe

The standard operating configuration for dense ~7B pretraining from scratch as of August 2026.

```
+-----------------------------------------------------------------------------------------+
|                                    DEFAULT STACK                                        |
+-------------------+---------------------------------------------------------------------+
| Backbone          | Dense Transformer, Olmo 3 7B shape [2512.13961]                     |
| Hidden Optimizer  | Muon² (2nd-moment before Newton-Schulz) [2604.09967] on 2D matrices |
| Embed/Norm/Head   | AdamW (decoupled weight decay, matched LR via update-RMS)           |
| Large-Batch Co-Def| KL-SOAP [2607.20548] (if GPU memory allows SOAP state & batch ~100M)|
| Attention Kernel  | FlashAttention-4 [2603.05451] (CuTe-DSL, Blackwell/Hopper)          |
| Muon Kernel       | Gram Newton-Schulz [dao-lab-blog-2026] / Hierarchical Muon [2606.27216]|
| Numeric Precision | BF16 / MXFP8 default; Quartet II NVFP4 [2601.22813] on Blackwell    |
| Data Mixture      | Dolma 3 staged curriculum (mix-6T -> Dolmino-style anneal)          |
| Residual Stream   | Standard residual (mHC n=4 optional experiment; xHC pointer only)   |
| Post-Train Branch | Dolci SFT -> Delta-DPO -> RLVR [2512.13961] or Nemotron-Cascade 2   |
+-------------------+---------------------------------------------------------------------+
```

---

## 3. Symptom Router

Locate the observed operational constraint or failure mode, then execute the corresponding checklist check.

| Observed Symptom / Operational Constraint | Check | Action |
|---|---|---|
| Pretrain a dense ~7B model from scratch | C1, C2, C4 | Select Olmo 3 7B shape; partition Muon² / AdamW; stage Dolma 3 data. |
| Newton-Schulz (NS) is the step-time bottleneck | C3 | Load Gram Newton-Schulz symmetric CuTe-DSL kernel or Hierarchical Muon. |
| Attention backward pass dominates step time | C3 | Switch to FlashAttention-4 CuTe-DSL path (deterministic bwd enabled). |
| OOM on SOAP optimizer state under large batches | C4 | Switch 2D hidden matrices from KL-SOAP to Muon²; keep AdamW on 1D/heads. |
| Blackwell hardware (NVFP4 / MXFP4) available | C5 | Enable Quartet II MS-EDEN NVFP4 training; evaluate validation perplexity. |
| Residual stream instability at depth >40 | C1 | Standard residual default; test optional mHC ($n=4$) Sinkhorn projection. |
| Post-train reasoner (math / code RL) | C6 | Apply CISPO (2025 default), CPPO, MinPRO, SSPO, or OPD/OPDVR distillation. |
| Attempt to paste DeepSeek-V4 / MoK into dense run | C1, C3 | Hard reject: route MoE backbones and MoK megakernels to `train-moe`. |

---

## 4. Operational Audit

Execute each verification check. Evaluate against strict PASS / FAIL criteria.

### C1 -- Architecture & Residual Stream

**Measure.** Inspect model definition, parameter count, layer dimensions, and residual topology.

**Pass.**
1. Backbone matches dense transformer Olmo 3 7B shape (`2512.13961`). No MoE routing layers present.
2. Residual connections use standard identity residual stream.
3. If experimental residual is enabled, it is restricted to mHC ($n=4$, Sinkhorn doubly-stochastic normalization `2512.24880`). xHC (`2607.14530`) is noted for reference only and not deployed as dense default.
4. OLMo-2 architecture (`2501.00656`) is not used.

**Why.** Olmo 3 7B provides the verified open dense foundation. MoE topologies (DeepSeek-V4, Kimi K3, LatentMoE) belong in sparse workflows.

**Fix.** Align layer configs with Olmo 3 7B dense transformer specifications. Remove sparse routing gates.

### C2 -- Data Curriculum & Annealing

**Measure.** Check token dataset mixtures, token counts, and training stage schedules.

**Pass.**
1. Primary dataset uses Dolma 3 (`allenai/dolma3_mix-6T` or staged equivalent).
2. Training follows a staged curriculum: broad pretraining mixture followed by targeted high-quality decay/annealing (Dolmino-style anneal).
3. Specialized research mixtures (DeMix `2602.00747`, CausalMix `2607.01104`, OP-Mix `2605.15220`) are evaluated as read-next mix experiments, not unverified base replacements.
4. OLMo-2 dataset mixtures are not used.

**Why.** Staged curricula with high-reasoning annealing phases produce superior downstream benchmark scores compared to flat static mixtures.

**Fix.** Configure training data loader with Dolma 3 staged mixture and schedule learning rate decay during the final annealing phase.

### C3 -- Attention & Optimizer Kernel Acceleration

**Measure.** Inspect kernel bindings for attention operations and Newton-Schulz orthogonalization iterations.

**Pass.**
1. **Attention:** FlashAttention-4 (`2603.05451`, CuTe-DSL, 2-CTA MMA, software-emulated exp) active on Blackwell / Hopper with deterministic backward pass. FA4 speedup claims (up to $1.3\times$ vs cuDNN 9.13 BF16 on B200) are not claimed on A100.
2. **Newton-Schulz:** Muon² orthogonalization uses Gram Newton-Schulz (Dao lab 2026 CuTe-DSL symmetric GEMMs in QuACK, $\sim 2\times$ NS speedup) on Hopper/Blackwell, or Hierarchical Muon (`2606.27216`, tiled Triton NS, SRAM-resident for $T \le 128$).
3. Mixture-of-Kittens (MoK) is not loaded in dense pipelines.

**Why.** Unfused PyTorch Newton-Schulz or legacy attention kernels bottleneck compute utilization on modern GPU clusters.

**Fix.** Bind FlashAttention-4 CuTe-DSL and Gram-NS / Hierarchical Muon kernel implementations in the training loop.

### C4 -- Optimizer Parameter Partitioning & LR Matching

**Measure.** Trace parameter group assignment across model tensors and verify learning rate scaling rules.

**Pass.**
1. All 2D internal hidden weight matrices (attention projections, MLP projections) are optimized via Muon² (`2604.09967`) or KL-SOAP (`2607.20548`).
2. Embeddings, 1D normalization scales/biases, and final `lm_head` remain on AdamW with decoupled weight decay.
3. Learning rates across Muon² and AdamW groups are matched by update-RMS, never by directly copying AdamW learning rates to Muon.
4. If batch size is $\sim 100\text{M}$ tokens and GPU memory accommodates Kronecker state tensors, KL-SOAP is co-default; otherwise Muon² is active.

**Why.** First-order AdamW on hidden weights produces suboptimal loss per FLOP. Applying Muon to 1D vectors or token embeddings degrades lexical embeddings. Direct LR copying causes divergence due to disparate update magnitude geometries.

**Fix.** Partition parameters into `muon_params` and `adamw_params`. Compute RMS scaling factor: $\eta_{\text{Muon}} = \eta_{\text{AdamW}} \cdot \frac{\text{RMS}(U_{\text{AdamW}})}{\text{RMS}(U_{\text{Muon}})}$.

### C5 -- Numeric Format & Quantization Gating

**Measure.** Audit execution data types across forward pass, backward pass, and optimizer state.

**Pass.**
1. Default precision is BF16 compute with optional MXFP8 linear layers.
2. NVFP4 / MXFP4 training is enabled only when hardware supports it and gated via Quartet II MS-EDEN (`2601.22813`) or MI355X MXFP4 (`2605.09825`).
3. Native 1.58-bit training (Sparse-BitNet `2603.05168`) is used only if building a native ternary model from scratch, not as the default dense pretrain path.
4. Post-hoc PTQ-to-1.58 is not substituted for native quantization.

**Why.** Low-precision FP4 requires scale-compensated gradient estimators (MS-EDEN) to prevent catastrophic underflow during backward propagation.

**Fix.** Enforce standard BF16 / MXFP8 baseline. Gate NVFP4 behind Quartet II QAT wrappers.

### C6 -- Post-Training Alignment & Math/Code Reasoning

**Measure.** Inspect post-training pipeline stages, loss formulations, and reward verifier integration.

**Pass.**
1. Instruction alignment uses Olmo 3 Dolci three-stage recipe (SFT $\to$ Delta-DPO $\to$ RLVR) or Nemotron-Cascade 2 (`2603.19220`).
2. Dense math/code RL defaults to CISPO (MiniMax-M1 `2506.13585`, ScaleRL `2510.13786`), acknowledging CISPO as a verified 2025 baseline.
3. 2026 exploration algorithms in the post-train branch use CPPO (`2606.10968`), MinPRO (`2601.22718`), SSPO (`2602.19327`), OPD (`2604.13016`), OPDVR (`2608.24696`), or BPCO single-sample critic (`2608.23566`).
4. LoRA / PEFT 24GB workloads are directed to `peft-24gb` (forthcoming).

**Why.** Unregularized on-policy RL causes policy entropy collapse. CISPO and modern clipped objectives preserve exploration bounds across multi-step reasoning rollouts.

**Fix.** Configure post-training pipeline with Dolci SFT $\to$ Delta-DPO $\to$ RLVR or wire CISPO / OPDVR for verifiable math and code domains.

---

## 5. Hard Don'ts

Never permit these anti-patterns in configuration or code:

1. **Do not use AdamW on 2D hidden matrices.** Internal linear projections must use Muon² or KL-SOAP.
2. **Do not copy AdamW learning rates to Muon.** Match learning rates strictly by update-RMS.
3. **Do not use OLMo-2 as the architectural or data recipe.** Use Olmo 3 7B shape and Dolma 3 curricula.
4. **Do not paste MoE backbones, routing gates, or Mixture-of-Kittens into dense runs.** Route all MoE components (DeepSeek-V4, Kimi K3, LatentMoE, MoK, SAPO, SAO) to `train-moe`.
5. **Do not route PEFT / LoRA workflows to this skill.** Route QLoRA, AQLoRA, DoRA, and 24GB fine-tuning to `peft-24gb` (note: `peft-24gb` does not exist yet; state so explicitly).
6. **Do not claim FlashAttention-4 speedups on unsupported hardware.** FA4 is optimized for Blackwell/Hopper architectures; do not claim FA4 benchmark figures on A100.
7. **Do not use post-training PTQ-to-1.58 as a substitute for native BitNet from scratch or NVFP4 QAT.**

---

## 6. Provenance & Literature Map

Direct mappings from verified library graph nodes (`j8ckfi/library`) and arXiv identifiers to operational primitives.

| Operational Component | Library Node ID / Key | Reference / Identifier | Code Repository / Source |
|---|---|---|---|
| Olmo 3 7B Backbone & Dolma 3 | `method:olmo-3` | arXiv:`2512.13961` | https://github.com/allenai/OLMo-core |
| Dolma 3 Dataset Mixture | `dataset:dolma-3` | arXiv:`2512.13961` | https://github.com/allenai/dolma3 |
| Muon² 2nd-Moment Optimizer | `method:muon2` | arXiv:`2604.09967` | Second-moment Newton-Schulz (no verified abs repo) |
| KL-SOAP Optimizer | `paper:soap-muon-scale` | arXiv:`2607.20548` | https://github.com/NVIDIA-NeMo/Emerging-Optimizers |
| FlashAttention-4 Kernel | `method:flashattention-4` | arXiv:`2603.05451` | https://github.com/Dao-AILab/flash-attention |
| Gram Newton-Schulz Kernel | `blog:gram-newton-schulz` | Blog (no arXiv ID) | https://github.com/Dao-AILab/gram-newton-schulz |
| Hierarchical Muon (HiMuon) | `method:himuon` | arXiv:`2606.27216` | Tiled Triton NS kernel |
| Quartet II NVFP4 Training | `method:quartet-ii` | arXiv:`2601.22813` | https://github.com/IST-DASLab/Quartet-II |
| MI355X MXFP4 Training | `paper:mxfp4-mi355x` | arXiv:`2605.09825` | AMD MI355X MXFP4 pretraining |
| Sparse-BitNet (Native 1.58b) | `method:sparse-bitnet` | arXiv:`2603.05168` | https://github.com/AAzdi/Sparse-BitNet |
| DeMix Read-Next Mix | `dataset:demix` | arXiv:`2602.00747` | https://github.com/Lucius-lsr/DeMix |
| CausalMix Read-Next Mix | `dataset:causalmix` | arXiv:`2607.01104` | Causal pretraining mixture |
| OP-Mix Read-Next Mix | `dataset:op-mix` | arXiv:`2605.15220` | Optimization-guided pretraining mix |
| Manifold Hyper-Connections | `arch:mhc-residuals` | arXiv:`2512.24880` | Optional dense experiment ($n=4$) |
| Extreme Hyper-Connections | `arch:xhc-residuals` | arXiv:`2607.14530` | Pointer only (MoE 18B/28B scaled) |
| Attention Residuals | `arch:attnres` | arXiv:`2603.15031` | Pointer only (MoE scaled) |
| Nemotron-Cascade 2 Post-Train | `method:nemotron-cascade-2` | arXiv:`2603.19220` | SFT and cascade alignment |
| CISPO Math/Code RL Baseline | `method:cispo` | arXiv:`2506.13585` | MiniMax-M1 / ScaleRL `2510.13786` (2025 baseline) |
| CPPO Policy Optimization | `method:cppo` | arXiv:`2606.10968` | Constrained PPO exploration |
| MinPRO Policy Optimization | `method:minpro` | arXiv:`2601.22718` | Minimal policy ratio optimization |
| SSPO Policy Optimization | `method:sspo` | arXiv:`2602.19327` | Step-size regularized policy optimization |
| OPD Policy Distillation | `method:opd` | arXiv:`2604.13016` | https://github.com/thunlp/OPD |
| OPDVR Verifier Distillation | `method:opdvr` | arXiv:`2608.24696` | https://github.com/LeapLabTHU/OPDVR |
| BPCO Single-Sample Critic | `method:bpco` | arXiv:`2608.23566` | https://github.com/QPHutu/golden_critic |
| MONA (Muon Variant) | `method:mona` | arXiv:`2605.26842` | Provenance only, not default |
| HTMuon (Muon Variant) | `method:htmuon` | arXiv:`2603.10067` | https://github.com/TDCSZ327/HTmuon (provenance only) |
| Variance-Adaptive Muon | `method:va-muon` | arXiv:`2601.14603` | Provenance only, not default |
| SF-NorMuon (Muon Variant) | `method:sf-normuon` | arXiv:`2605.23061` | Provenance only, not default |
| Newton-Muon (Muon Variant) | `method:newton-muon` | arXiv:`2604.01472` | Provenance only, not default |

*Note on substrates:* QuACK and ThunderKittens serve as underlying kernel substrates in provenance, not standalone operational directives.

---

## 7. Audit Report Format

Emit one line per operational check executed:

```
C1 PASS  Olmo 3 7B dense transformer shape verified; identity residuals active
C2 PASS  Dolma 3 mix-6T configured with staged annealing schedule
C3 PASS  FlashAttention-4 (CuTe-DSL) + Gram-NS Newton-Schulz kernels loaded
C4 PASS  Muon2 on 2D hidden weights; AdamW on embed/lm_head; LR matched by update-RMS
C5 PASS  BF16/MXFP8 compute verified; Quartet II NVFP4 ready for Blackwell stage
C6 SKIP  pretraining phase; post-training reasoning checks not active
```

---

## 8. When This Skill Is Stale

This compiled skill reflects the verified dense pretraining landscape as of **August 27, 2026**.

If the current date exceeds `as_of + 30 days` or a refreshed query to `j8ckfi/library` indicates movement in the `pretrain dense 7B` frontier, re-query the library before finalizing pretraining cluster configurations:

```bash
python -m library sota 'pretrain dense 7B'
```

Inspect the returned graph delta for updates to dense transformer backbones, second-moment optimizers, attention/orthogonalization kernels, or post-training reasoning algorithms.
