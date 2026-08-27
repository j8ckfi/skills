# Post-Training & Alignment Notes (MoE SOTA as of August 2026)

Dense notes on reinforcement learning, asynchronous agentic optimization, and policy distillation for sparse MoE models.

---

## 1. Reinforcement Learning on Sparse Models

### SAPO: Self-Adaptive Policy Optimization (`2511.20347`)
- **Failure of Vanilla GRPO**: Standard Group Relative Policy Optimization suffers from severe routing collapse and entropy drift when applied to sparse MoE networks due to uneven gradient allocation across dormant vs active experts.
- **Adaptive Clipping**: SAPO introduces dynamic advantage-dependent clipping boundaries that adjust per-expert activation frequency and variance.
- **Modalities**: Default RL algorithm for sparse MoE and Vision-Language (VL) post-training.
- **GSPO Variant**: Group Shift Policy Optimization is reserved specifically for omni-modal audio-speech backbones (e.g., Qwen3.5-Omni Talker).

### SAO: Scalable Asynchronous Optimization (`2607.07508`)
- **Target Setting**: Agentic workflows with multi-turn tool calling, environment stepping, and variable trajectory lengths.
- **Asynchronous Decoupling**: Decouples policy rollouts from gradient computation across distributed actors, applying off-policy importance weight corrections to prevent expert routing lag.

---

## 2. Distillation & Specialist Merging

### OPD & MOPD (`2604.13016`)
- **On-Policy Distillation (OPD)**: Policy optimization transferring reasoning capabilities from frontier teacher models to student MoE architectures using student-generated rollouts.
- **Mixture On-Policy Distillation (MOPD)**: Multi-teacher distillation framework designed to compress heterogeneous specialist models into a single unified sparse MoE backbone.

### OPDVR (`2608.24696`)
- **Verifier-Regularized OPD**: When an external ground-truth verifier / reward model is present, OPDVR integrates dense reward verification into on-policy distillation gradients to penalize hallucinated reasoning steps.
