# Post-Training & Reasoning Notes (Dense SOTA as of August 2026)

Dense technical notes on instruction alignment, reinforcement learning, and distillation methods.

---

## 1. Instruction Alignment

### Olmo 3 Dolci Recipe (`2512.13961`)
- **Three-Stage Pipeline**: Supervised Fine-Tuning (SFT) $\to$ Delta-DPO (Direct Preference Optimization with reference margin) $\to$ Reinforcement Learning with Verifiable Rewards (RLVR).

### Nemotron-Cascade 2 (`2603.19220`)
- **Industrial Alternative**: High-throughput multi-turn alignment and cascade filtering avoiding explicit DPO stages.

---

## 2. Math & Code Reinforcement Learning

### CISPO Baseline (`2506.13585`, `2510.13786`)
- **Status**: Default baseline for dense math and code RL (MiniMax-M1 `2506.13585`, ScaleRL `2510.13786`). Note: verified 2025 algorithm serving as standard foundation.
- **Mechanism**: Clipped importance-sampling policy optimization mitigating policy entropy collapse under multi-turn reasoning steps.

### 2026 RL Algorithms
- **CPPO (`2606.10968`)**: Constrained Proximal Policy Optimization enforcing trust-region divergence bounds.
- **MinPRO (`2601.22718`)**: Minimal Policy Ratio Optimization stabilizing gradient variance across long reasoning trajectories.
- **SSPO (`2602.19327`)**: Step-size regularized policy optimization for verifiable reasoning tasks.
- **BPCO (`2608.23566`)**: Best-of-Policy Critic Optimization using single sample per prompt (`https://github.com/QPHutu/golden_critic`).

---

## 3. On-Policy Distillation

### OPD (`2604.13016`)
- **Repository**: `https://github.com/thunlp/OPD`.
- **Mechanism**: On-policy policy distillation minimizing distribution shift by optimizing student rollouts against teacher token probabilities.

### OPDVR (`2608.24696`)
- **Repository**: `https://github.com/LeapLabTHU/OPDVR`.
- **Mechanism**: Verifier-regularized on-policy distillation incorporating ground-truth outcome rewards to suppress teacher hallucination transfer.
