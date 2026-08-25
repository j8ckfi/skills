---
name: ontology
description: Derive ground truth into a durable on-disk network of nodes and edges. Do not trust repo docs, papers, or unaudited inference. Use for /ontology, stale or scattered docs, greenfield literature, or when a claim must be formalized before it is believed.
disable-model-invocation: true
---

# Ontology

Ground truth is a fact you derived and checked, with evidence. Everything else is a hypothesis: READMEs, comments, surveys, and published papers. This skill may have been called because those sources are out of date, scattered, or wrong.

The artifact is the ontology network on disk. Chat is a delta for this run, not the ontology. A large investigation lives in files you grow, not in a recap that disappears.

A source does not enter the network as admitted because you found it. It enters after you reconstruct the claim and the reconstruction holds. For a mathematics paper that means formalize it first. Locate the passage with `citations`. Then do this skill's work. Location is not admission.

Do not answer from a doc, a paper's abstract, or your own earlier summary. Do not skip derivation because the answer looks obvious. If you can derive a fact, derive it.

This is a full investigation, not a skim. Open a todolist with one entry per phase before you start.

1. Open
2. Frame
3. Inventory
4. Derive
5. Diff
6. Report

## Phase A: Open

Default path: `ontology/` at the workspace root. Use a path the user gives if they give one. Create the layout if it is missing. If it exists, read `INDEX.md` and search aliases before you create a node. Merge. Do not rebuild. Do not load every node. Grep the index and the graph for the objects in the query.

```
ontology/
├── INDEX.md
├── graph.jsonl
└── nodes/
    └── <id>.md
```

`INDEX.md` is the catalog. `graph.jsonl` is the edge list, one JSON object per line. `nodes/<id>.md` is the body of each object. Those three are the network. Keep them in sync as you go, not at the end.

## Phase B: Frame

Restate the query as questions that evidence can answer. Name the objects: symbols, files, commands, theorems, definitions, papers, datasets, behaviors.

Map each name to an existing node id when the index already has it. New names stay new until Derive.

State the done predicate: which questions must have admitted nodes, and that the network on disk must reflect them.

If the query is vague, pick the smallest set of questions that would let a tired reader act. Do not encyclopedize the repo or the field in one run. Grow the network across runs.

## Phase C: Inventory

List candidate sources. Rank them. Do not read docs or papers first and treat derivation as confirmation.

Trust, high to low:

1. Independent derivation. You proved it, ran it, counted it, or built it.
2. Checked reconstruction. A proof you formalized, a code path you traced, an experiment you can restate as a falsifiable claim whose reported evidence you inspected.
3. The primary artifact after you opened it. The function that runs. The paper's proof or data tables, not the title.
4. Tests, lemmas, or corollaries you have not yet checked.
5. Comments, docs, surveys, related-work paragraphs, blog posts.
6. Abstracts, titles, and memory. Hypothesis only.

Record every name each object goes by. Scattered names are data. Write them as aliases on a node or as `same-as` edges once Derive shows they are one object. "Auth service", "identity gateway", and "login worker" may be one thing or three. Two papers may state the same theorem under different hypotheses. You do not know until you derive it.

Create `candidate` nodes for objects you have named but not yet checked. A candidate is in the network as unfinished work, not as ground truth.

## Phase D: Derive

For each framed question, produce the answer from the highest-trust source that can produce it. Update the node as soon as you have the result. Nothing is admitted until this phase succeeds.

### Software

- Trace from an entry point. Main, CLI, route, job, exported API. Follow calls. Do not start from a comment and look for code that matches it.
- Read the function that runs, not a similarly named one.
- Count with a command. Do not quote a number from a doc.
- Read the config file that is loaded. Check how it is selected: flags, env, defaults.
- If the question is behavior, run the smallest command that shows it.
- If two implementations exist, find which one is wired. Dead code is not ground truth.

### Literature

Pull the source with `citations`. Then reconstruct. Do not admit a claim because the paper is famous, peer reviewed, or convenient.

For mathematics, formalize before admission:

1. Write definition nodes for every notion the argument uses, including the implicit ones.
2. Restate the theorem on a claim node with every hypothesis visible.
3. Reconstruct the proof, step by step, in that node's body. A proof assistant is the high bar when one is in reach. A checked reconstruction is the minimum.
4. Draw `depends-on` edges to the definitions and lemmas you used. Record gaps, hidden axioms, quantifier mistakes, and steps that do not follow.
5. Admit only what survived. A named lemma with a hole stays `unknown` or becomes `rejected`. It is not ground truth.

For empirical work, restate the claim so it can be false. Check that the method, sample, and reported numbers support that claim and not a larger one. The abstract's story is not the result.

Write each derived fact as one sentence on the node, plus a locator: path, symbol, command, paper identifier, or the reconstructed statement.

If you cannot derive a fact, mark the node `unknown` and say what you tried. Do not fill the gap from a doc or a paper you have not formalized.

## Phase E: Diff

Compare admitted nodes to every doc and paper that talks about the same objects.

For each mismatch, add a `contradicts` edge and a one-line note on both nodes:

- What the source says
- What you derived
- Where both live
- Whether the source is stale, about a different object, overclaiming, or still right

Also record scatter. Several names for one object become `same-as` once you have derived that. The same theorem with different hypotheses stays two claim nodes, with edges that say how they relate. Copied facts that now disagree get `contradicts`.

Sources that match still get a `derived-from` edge. Matching is a finding, not a reason to have skipped derivation.

## Phase F: Report

The network on disk is the result. The chat is a delta. Do not paste the graph, raw file dumps, or unreworked papers.

### This run

Path to `ontology/`. Counts: nodes created, nodes updated, edges added, by status.

### Ground truth

The admitted answers to the framed questions. Each line points at a node id. `unknown` stays `unknown`. `rejected` names the source node and why the reconstruction failed.

### Evidence

Compact table of this run only:

| Question | Node | Derived fact | Locator | Trust |
| --- | --- | --- | --- | --- |
| ... | id | ... | path, symbol, command, or reconstructed statement | independent, reconstructed, primary |

### Drift

One line per mismatch or name collision. Source node, what it claims, what is true instead, edge added.

### Unify

Suggestions only. Do not rewrite the docs or the notes unless the user asks. Do write the network: canonical names, `same-as`, `contradicts`, and which source nodes to treat as stale.

- One canonical node for each object, named with the name the code uses or the statement you admitted.
- Which files or papers to redirect, merge, split, or drop.
- Which claims to update, with the admitted sentence that should replace them.
- Follow `technical-writing` if you are asked to apply doc edits: one Diátaxis mode per file, real symbols, no invented synonyms.

## Shape

Four node types. Do not add more.

| Type | What it is |
| --- | --- |
| `entity` | A thing: module, dataset, construction, person, system |
| `definition` | A term with a precise meaning you wrote |
| `claim` | A theorem, result, or behavioral fact |
| `source` | A paper, doc, file, or other hypothesis container |

Five relations. Do not add more.

| Rel | Meaning |
| --- | --- |
| `depends-on` | This node needs that node |
| `derived-from` | This node was checked against that source or artifact |
| `same-as` | Two ids are one object |
| `part-of` | This node is a piece of that node |
| `contradicts` | These two cannot both be admitted as written |

Node file, `ontology/nodes/<id>.md`:

```markdown
---
id: budget-mjs
name: budget.mjs
type: entity
status: admitted
aliases:
  - ratchet
  - proto import budget
---

# budget.mjs

Admitted: `budget.mjs` reads `budget.json` and fails CI when the count exceeds the budget.

## Evidence

- locator: `scripts/budget.mjs`
- trust: reconstructed
- derived-from: `budget-mjs-source`

## Notes

Reconstruction, gaps, and why a status is `unknown` or `rejected`.
```

Status is one of `candidate`, `admitted`, `unknown`, `rejected`. Only `admitted` is ground truth. The others stay in the network so the next run does not start from zero.

Edge line, appended to `ontology/graph.jsonl`:

```json
{"from":"thm-import-bound","rel":"depends-on","to":"def-import-count"}
```

Ids are stable kebab-case slugs of the canonical name. Look up aliases before minting a new id. If new evidence contradicts an `admitted` node, do not silently overwrite it. Add `contradicts`, set the node to `unknown` or `rejected` only when the new derivation is stronger, and say so in the delta.

`INDEX.md` is a table, one row per node: id, name, type, status. Rebuild it from `nodes/` when you add or change a node. Do not let it drift.

## Gotchas

- Leaving the ontology in chat.
- Rebuilding `ontology/` instead of merging.
- Minting a second node for an alias already in the index.
- Believing the README or the paper, then skimming for support.
- Admitting a theorem from the author's statement without formalizing the proof.
- Treating a survey or related-work paragraph as a primary result.
- Believing comments, types, or names that describe an old design.
- Believing your own recap from earlier in the chat.
- Counting a skipped test, a mock, dead code, or an un-checked lemma as behavior.
- Treating default config in-repo as the config that runs in prod.
- Answering "how it should work" or "what they meant" instead of what holds.
- Unifying by writing a new overview and leaving the stale copies in place.
- Dumping every node into context. Grep the index and the graph.
