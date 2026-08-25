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

## Installation

```bash
npx skills add j8ckfi/skills
```

Install a single skill:

```bash
npx skills add j8ckfi/skills --skill harness-design
npx skills add j8ckfi/skills --skill ontology
```

Use it once without installing:

```bash
npx skills use j8ckfi/skills@harness-design | claude
npx skills use j8ckfi/skills@ontology | claude
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

## Skill Structure

Each skill contains:

- `SKILL.md` - Instructions for the agent
- `scripts/` - Helper scripts for automation (optional)
- `references/` - Supporting documentation (optional)

## License

MIT
