# Skills

A collection of skills for AI coding agents. Skills are packaged instructions that extend agent capabilities.

Skills follow the [Agent Skills](https://agentskills.io/) format.

[![skills.sh](https://skills.sh/b/j8ckfi/skills)](https://skills.sh/j8ckfi/skills)

## Available Skills

### harness-design

Audits, redesigns, and debugs LM harnesses and agent scaffolds so that every individual model call stays in-distribution even when the overall task is not. Built around the **locally in-distribution (LID)** invariant: the global trajectory may be wildly out-of-distribution, but each constituent call must not be.

The skill is an operating procedure, not an explainer. It gives the agent a symptom router, a ten-check audit with pass criteria and measurements, a default-deny admission list for the root context, a worked before/after refactor, and a debug playbook.

**Use when:**

- Building or reviewing an agent scaffold, orchestrator loop, or sub-agent / multi-agent system
- Deciding what may enter the root/main context, or doing context-window management
- Planning task decomposition across sub-agents
- Designing a long-context pipeline or RAG orchestration
- Someone reports "my agent degrades over long runs", context rot, quality falling off after N turns, or a scaffold that works on small inputs and fails on large ones
- A harness trains or tunes well on one domain and transfers to none

**What it checks:**

- `A1` Root-context growth — does root context size track task size?
- `A2` Verbatim leak grep — is input text reaching the root at all?
- `A3` Tool and sub-agent returns bind, not print
- `A4` Growing prefix history — the ReAct/CodeAct failure
- `A5` Prefix stuffing
- `A6` Half-fix detection — offloading without programmatic sub-calling
- `A7` Degenerate decomposition — the harness that silently collapses into its own baseline
- `A8` Isomorphism spot-check
- `A9` Beyond-context capability
- `A10` Recursion and plan size

**Source:** the skill distills Alex L. Zhang and Omar Khattab, ["Language model harnesses are compositional generalizers"](https://alexzhang13.github.io/blog/2026/harness/) (July 2026). It carries the post's own caveats — the technique is not guaranteed to work, training runtime is 1.5–3x, and the authors explicitly warn against over-engineering harnesses into hand-authored pipelines. Its **Provenance** section separates the post's measured findings from the operational thresholds supplied to make them checkable.

### ontology

Derives ground truth into a durable on-disk network of nodes and edges. Repo docs, papers, comments, and unaudited inference stay hypotheses until a claim is reconstructed and the reconstruction holds.

The artifact is `ontology/` at the workspace root: `INDEX.md`, `graph.jsonl`, and `nodes/<id>.md`. Chat is a delta for the run, not the ontology.

**Use when:**

- `/ontology`, stale or scattered docs, or greenfield literature
- A claim must be formalized before it is believed
- README, comments, or a paper disagree with the code or the proof
- Names for the same object are scattered and need `same-as` or a split

**What it does:**

- Frame questions evidence can answer, then inventory sources by trust
- Derive from the highest-trust source that can produce the answer
- Admit only what survived; mark gaps `unknown` or `rejected`
- Diff admitted nodes against docs and papers; record `contradicts`

### train-moe

Operating procedure for pretraining, optimizing, and post-training frontier Mixture of Experts (MoE) models using verified August 2026 SOTA methods. Evaluates architecture, residual stability, dispatch compression, second-order optimizers, training megakernels, and sparse policy RL.

The skill provides a one-screen default stack, a symptom router for cluster and training failure modes, a six-check operational audit with pass/fail criteria, hard anti-patterns, and direct provenance links to library nodes.

**Use when:**

- Designing, configuring, or auditing an MoE pretraining cluster run
- Selecting sparse routing topologies (DeepSeek-V4 CSA+HCA with mHC residuals vs Kimi K3 Delta Attention with block AttnRes)
- Resolving inter-node All-to-All communication saturation with LatentMoE dispatch compression
- Configuring second-order optimizers (Muon² for 2D hidden matrices vs AdamW for embeddings/heads)
- Deploying Mixture-of-Kittens (MoK) fused megakernels on GB200/GB300 NVL72 Blackwell systems
- Post-training sparse MoE backbones with SAPO or asynchronous agentic RL with SAO
- Distilling specialist checkpoints into MoE using OPD/MOPD and OPDVR

**What it checks:**

- `C1` Architecture & Dispatch Compression — CSA+HCA hybrid attention, manifold-constrained hyper-connections (mHC $n=4$), LatentMoE dispatch compression
- `C2` Residual Stability & Routing Diversity — expert routing coefficient of variation and sub-linear residual growth
- `C3` Training Megakernels & Compute Topology — Mixture-of-Kittens (MoK) on Blackwell NVL72 vs DeepEP/Megatron-Core on non-NVL72
- `C4` Optimizer Parameter Partitioning — Muon² / KL-SOAP on 2D hidden matrices; AdamW on embeddings and lm_head
- `C5` Post-Training & Sparse Policy Optimization — SAPO for MoE/VL RL, SAO for asynchronous agent rollouts (no vanilla GRPO)
- `C6` Distillation & Specialist Merging — OPD/MOPD on-policy distillation and OPDVR verifier regularization

**Source:** compiled view from `j8ckfi/library` as of August 27, 2026. Dense paper notes and mathematical formulations live in `skills/train-moe/references/`.

## Installation

```bash
npx skills add j8ckfi/skills
```

Install a single skill:

```bash
npx skills add j8ckfi/skills --skill harness-design
npx skills add j8ckfi/skills --skill ontology
npx skills add j8ckfi/skills --skill train-moe
```

Use it once without installing:

```bash
npx skills use j8ckfi/skills@harness-design | claude
npx skills use j8ckfi/skills@ontology | claude
npx skills use j8ckfi/skills@train-moe | claude
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

**Examples:**

```
Review this agent scaffold — it gets noticeably worse after about 20 turns.
```

```
I'm adding a sub-agent to this pipeline. What should it be allowed to return?
```

```
This orchestrator works fine on a 30k-token doc and falls apart on a 2M-token one.
```

```
The README and the code disagree about how auth works. Build an ontology.
```

```
Formalize the claims in this paper before we treat them as true.
```

```
Audit our Blackwell cluster pretraining config for an 800B MoE run.
```

```
We're seeing routing collapse and expert imbalance during GRPO post-training on our MoE model.
```

## Skill Structure

Each skill contains:

- `SKILL.md` - Instructions for the agent
- `scripts/` - Helper scripts for automation (optional)
- `references/` - Supporting documentation (optional)

## License

MIT
