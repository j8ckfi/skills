---
name: harness-design
description: Audits, redesigns, and debugs LM harnesses and agent scaffolds so every individual model call stays in-distribution even when the overall task is not — context offloaded into symbolic variables, tools and sub-agents as REPL functions, root-context size independent of task size. Use when building or reviewing an agent scaffold, orchestrator loop, sub-agent or multi-agent system, tool-call flow, long-context pipeline, or RAG orchestration; when deciding what enters the root/main context or doing context-window management; when planning task decomposition for an agent; or when someone reports "my agent degrades over long runs", context rot, quality falling off after N turns, blown context windows, or a scaffold that works on small inputs and fails on large ones. Use it even when the request sounds smaller — "add a tool to my agent", "add a sub-agent", "make my pipeline handle bigger inputs", "why does my agent get dumber later" are harness design decisions.
license: MIT
metadata:
  version: "1.0.0"
  source: https://alexzhang13.github.io/blog/2026/harness/
---

# Harness Design

A harness is the program between the environment and the model: it decides how the current state becomes one or more model inputs, and how outputs become the next action. Its job is not tool invocation — that is just part of the agent. Its job is to reduce an arbitrarily large state into observations each individual call can actually handle.

**The invariant to enforce:** every single LM call sees a prompt that is in-distribution with respect to that model's training data, *even when the overall task is not*. This is **locally in-distribution (LID)**. The global trajectory may be wildly OOD; each constituent call must not be.

**The operational proxy:** root-context token count stays roughly constant as task size grows. If the root context grows with input length, corpus size, or turn count, LID is already broken, and everything downstream — length generalization, cross-domain transfer, late-run quality — degrades with it.

**Why it generalizes:** a harness induces an equivalence relation over task states. Tasks whose *root* trajectories collapse to the same shape are isomorphic under that harness, and competence on one should transitively reach the other. Design for that quotient set, not for the task.

**When not to use:** a single call comfortably handles the whole state. A harness adds real overhead — the post measured RLM *training* runtime at 1.5–3x the base Transformer on similarly-sized tasks, plus extra memory, from multiple steps per sample and waiting on sub-calls; inference overhead was not measured — and it buys nothing if there is no length or domain axis to generalize along.

Run the audit below on any existing scaffold before proposing changes.

---

## Symptom router

| Reported symptom | Run |
|---|---|
| "Degrades over long runs", "gets dumber after a while", context rot | A1, A4 |
| Works on small inputs, fails on large ones | A1, A2, A9 |
| Blows the context window; needs mid-run truncation, summarization, or compaction to survive | A1, A3 (compaction is the symptom, not the fix) |
| Added sub-agents, nothing improved | A3, A6, A7 |
| Tuned or trained for one domain, transfers to none | A5, A8 |
| Re-reads the same file or re-runs the same search repeatedly | A3 |
| Picks a different strategy every run; ignores its own plan | A10 |
| Train reward climbing, held-out eval flat | "If you are training the harness" |

---

## The audit

Each check: what to measure, what passes, why it matters, what to do when it fails.

### A1 — Root-context growth (the decisive one)

**Measure.** Run the harness on the same task family at three input sizes spanning ≥16x (e.g. 8k / 128k / 2M of task input). For each run, count tokens in the **root context only** — the message list actually sent to the top-level model. Plot root tokens vs. input tokens.

**Pass.** Slope ≈ 0; root context within ~1.5x across the full sweep.

**Why.** A root context whose size tracks the task is by construction leaving the region the model was trained on. A rising slope predicts context rot and length-generalization failure before you ever observe them.

**Fix.** A2–A4. If the slope is positive, no prompt engineering at the root will save it.

### A2 — Verbatim leak grep

**Measure.** Grep the root transcript for any contiguous span of ~40+ tokens appearing verbatim in the task input (document text, retrieved passages, file contents, tool stdout, sub-agent prose).

**Pass.** Zero hits.

**Why.** Input-specific context is supposed to reach the root as a *symbolic variable the root can name but not read* — context offloading. That is what makes two structurally different tasks look alike at step one.

**Fix.** Bind the input to a variable in an execution environment. Give the root the handle, the type, the length, the schema. Never the payload.

### A3 — Tool and sub-agent returns bind, not print

**Measure.** For every tool call site and every sub-agent call site in the transcript, check what immediately follows: an assignment, or an inlined result blob?

**Pass.** Every call site is `x = tool(...)` inside a REPL, with results living in REPL memory and reachable by later sub-calls. The root sees `len(x)`, `type(x)`, `x[0].keys()` — never `x`.

**Why.** Programmatic sub-calling is stated to be *equally as important* as context offloading. Standard tool calling returns information directly into the main context; that is the leak that reopens after you have offloaded the input.

**Fix.** Put tools and sub-agents behind a code REPL so they are ordinary functions. Route data between them through variables. If the root must branch on a result, branch on a derived scalar computed in the REPL — a count, a boolean, a label, a top-k id list — not on the payload.

### A4 — Growing prefix history

**Measure.** Find the message list. Does the code append observations to it and re-send the whole thing each turn?

**Pass.** Each turn appends at most an O(1)-in-task-size observation — a clipped print, a scalar, a handle. The list grows with plan steps, never with data volume; nothing appended is a function of input size.

**Why.** This is the ReAct / CodeAct / Claude Code / Codex pattern the post names directly: they "fundamentally rely on flooding the context window of the Transformer with interleaved task-specific information, tool call outputs, and reasoning that get continuously appended", and "these bloated histories quickly fall out of the training distribution, manifesting in the 'context rot' phenomenon".

**Fix.** Same as A3. Treat any summarize-or-compact-to-survive step as evidence of this failure, not a remedy for it.

### A5 — Prefix stuffing

**Measure.** Is the query and/or the context concatenated as a prefix into the system or first user message?

**Pass.** The root prompt is fixed scaffolding plus handles. Two different tasks in the family produce near-identical first messages.

**Why.** "Between two different tasks, this prefix significantly changes the output distribution of the model, even if the tasks are solved the same way." Isomorphism dies at step one, before any decomposition happens.

**Fix.** Make the query a handle too when you want two task families to share a root trajectory; defer task-specific phrasing to sub-calls.

### A6 — Half-fix detection

**Measure.** Is context offloading implemented *without* programmatic sub-calling — input hidden behind a variable, but tools and sub-agents still returning prose into the main context?

**Pass.** Both components present.

**Why.** Stated explicitly: on its own, context offloading does not prevent environment feedback or sub-agent information from returning to the main context, so over long horizons the main context still goes OOD and LID still breaks.

**Fix.** Add the REPL layer. Offloading alone buys you the first turn and nothing after it.

### A7 — Degenerate decomposition

**Measure.** Per run, log (a) number of sub-calls, (b) `largest sub-call input / total task input`.

**Pass.** Sub-call count > 1 and the largest-share ratio well below 1.0.

**Why.** This is how a correct-looking harness silently collapses: the root offloads the entire problem to one sub-call and returns its answer, "effectively becoming equivalent to the long-context Transformer baseline". On short tasks this is a perfectly viable strategy, which is exactly why it gets learned and why it dies at length.

**Fix.** Alert or penalize on this metric. If it persists, add a minimal nudge — a message asking the model to *state the decomposition it intends to perform* — rather than a hand-coded plan. That nudge is what the post used on MRCRv2, where the RLM otherwise failed to learn the generalizable strategy.

### A8 — Isomorphism spot-check

**Measure.** Pick the two most superficially different tasks your harness is meant to cover. Run both. Diff the **root** transcripts only.

**Pass.** Near token-for-token identical, modulo variable values and small scalars.

**Why.** Structurally similar tasks should fall into the same harness-induced equivalence class and produce the same root trajectory. Solve X, transitively solve Y. See "Measuring isomorphism" for the quantitative version.

**Fix.** Wherever the two diverge, that divergence is domain-specific information that belongs in a sub-call.

### A9 — Beyond-context capability

**Measure.** Feed an input larger than the base model's context window, with no context-extension tricks.

**Pass.** It runs and produces an answer.

**Why.** A well-designed harness generalizes to trajectories "including those that do not fit in the base model's context window". In the post the long-context baseline had to be dropped from MRCRv2 entirely, because 2M tokens do not fit even with context extension; the RLM ran fine. That is the clearest statement of what a harness buys.

**Fix.** If it cannot run, A1 has already failed; fix that first.

### A10 — Recursion and plan size

**Measure.** Apply A1–A5 to each sub-agent, treating its context as its own root. Separately, measure the length of the root's stated decomposition.

**Pass.** Sub-agents obey the same admission rules, and the root's stated decomposition fits in under ~10 lines / ~150 tokens.

**Why.** The argument holds recursively — each offloaded sub-agent is its own instance with its own main context. A sub-agent implemented as a growing-prefix ReAct loop re-introduces the failure one level down. And per the authors' Mismanaged Geniuses framing, the useful property is that "the decomposition itself is short and simple"; a sprawling plan is not the kind of plan that stays in-distribution.

**Fix.** Split fat sub-agents. Simplify plans that do not fit in a few lines. (Recursion is the intended RLM design but "is not strictly necessary" for LID — a flat root plus small, individually in-distribution sub-tasks also satisfies the invariant.)

---

## Report format

Emit one line per check actually run, then the fixes ranked by blast radius:

    A1 FAIL  root tokens 4.1k / 61k / 940k over 8k/128k/2M — slope 0.46
    A2 FAIL  3 spans >40 tok verbatim from corpus in root turns 4, 7, 9
    A3 PASS
    A7 SKIP  single-run trace, need >=3 runs

Use PASS / FAIL / SKIP only. Cite the measured number, never a judgement.
Checks you did not run are SKIP with the reason — never omitted.

---

## Root-context admission list

Default-deny. The root context is an allow-list, not a buffer.

**Admitted:** the harness's own fixed scaffolding and system prompt; an abstract, domain-free statement of the task *shape*; symbolic handles and variable names; small structural metadata — lengths, counts, indices, ids, booleans, enum labels, exit codes, type and schema names; small derived scalars the root branches on.

**Denied by default:** raw documents and corpora, raw tool output, raw sub-agent prose, retrieved passages, file contents, task-specific strings, full stack traces (clip to exception class plus a bounded head), anything whose size is a function of the input.

Make the boundary an observable event rather than an accident:

```python
def run(code):
    out = env.exec(code)
    if leak_score(out, env["corpus"]) > 0:      # verbatim n-gram overlap with the input
        log.warn("root-context leak", n=leak_score(out, env["corpus"]))
    return clip(out, 512)
```

Expect leakage even in a correct implementation — the post observes the RLM "will still choose to print out task-specific information and pass it back to the main context", calls it undesirable for generalization, and says it can *likely* be trained out (asserted, not demonstrated). So enforce it in the harness: clip, warn, and if you are training, penalize. Treat every new leak as a regression.

---

## The two mechanisms

Both are required. An RLM — "a harness in which the model offloads its context and defers execution to programmatic decomposition and recursive sub-calls" — is exactly these two pieces.

**Broken: prefix-stuffed, appended history.**

```python
msgs = [system, user(f"Question: {q}\n\n<corpus>\n{corpus}\n</corpus>")]   # A5: prefix-stuffed
while True:
    step = llm(msgs)
    if step.tool_call:
        out = TOOLS[step.tool_call.name](**step.tool_call.args)
        msgs += [assistant(step), tool_result(str(out))]   # A3: payload into root; A4: re-sent every turn
    else:
        return step.text
# root tokens = f(input size, turn count)  ->  LID broken within a few turns
```

**Working: context offloading + programmatic sub-calling.**

```python
env = Repl()
env["corpus"], env["query"] = corpus, q     # bytes live in REPL memory, never in a prompt

root = [sys_prompt, user("In scope: corpus (str, 2_100_337 chars), query (str, 61 chars). "
                         "Write Python. Tools and sub-agents are functions. Call final(answer).")]
while not env.finished:
    code  = llm(root).code
    root += [assistant(code), user(clip(env.run(code), 512))]   # +O(1) per step: only what the root chose to print
```

Surface exposed to the root: `chunk`, `grep`, `search`, `ask` (one sub-agent call), `fanout` (many, in parallel), `final` (out-of-band return, so the answer never transits the root). Swapping a 4 KB corpus for a 4 GB one changes the root prompt by three characters. That is the test for whether offloading is real.

---

## Worked example: two tasks, one root trajectory

Constructed illustration, not a result from the post — the post trained these two environments independently and never tested transfer between them.

Task A: find the *i*-th occurrence of a needle sentence answering `query` in a 2M-token conversation log. Task B: list graph nodes satisfying a constraint over a 1M-token edge list. Nothing in common on the surface. Under a well-built harness the root emits the same program for both:

```python
chunks = chunk(corpus, 50_000)
print(len(chunks))                              # A: 42   B: 21
cand = [c for c in chunks if grep(query, c)]
print(len(cand))                                # A: 7    B: 9
notes = fanout(ask, [(query, c) for c in cand])  # sub-calls read the text; the root never does
found = [n for n in notes if n.found]
print(len(found))                               # A: 3    B: 5
final(ask("reconcile these candidates", found))
```

The two root trajectories differ only in three integers. Nothing in the root reveals which task it is — which is precisely why competence on one should transfer to the other. Note what the root never sees: `corpus`, `chunks`, `cand`, `notes`, `found`. It sees `42`, `7`, `3`. The query is a handle too, which is what lets an entirely different task family reuse this trajectory.

This is the shape to aim at when restructuring. The post's own illustration is BrowseComp-Plus and OOLONG, which "can (in theory) see the exact same trajectory on the root LM's context window by deferring task-specific queries to its sub-calls and moving information through REPL variables" — a demonstration of the mechanism, not a measured result.

---

## Designing one from scratch

1. **Name the task family, not the task.** Write down the two most different members you want covered. *Done when both fit in one line each and share no surface vocabulary.*
2. **Hand-write the intended root trajectory for both, before writing code.** *Done when they are near-identical modulo variable values.* If they are not, stop — the decomposition is wrong, and no prompt tuning fixes it later.
3. **Fix the root's alphabet.** The small set of operations the root may emit: chunk, fan out sub-calls, regex, filter/search, sort, aggregate. Prefer domain-agnostic primitives — those are what transferred across domains in the post. Offer them; do not hardcode them as a required plan.
4. **Make the environment a REPL, not a message list.** *Done when no code path appends a tool result to a growing message array.*
5. **Bind every input as a handle**, including the query when you want cross-family isomorphism. *Done when A2 returns zero hits.*
6. **Define the metadata surface** — exactly what the root may learn about a variable without reading it (`len`, `type`, count, schema, `.found`, top-k ids). *Done when that list is finite and written down.*
7. **Add an out-of-band return channel** (`final()`), so the answer leaves without transiting the root.
8. **Recurse.** Re-run steps 4–6 for each sub-agent.
9. **Instrument A1, A2 and A7 from day one.** They are cheap, and they are the three that fail silently. Make the A1 plot before you make any accuracy plot.

---

## Measuring isomorphism

Use when you need evidence rather than a spot-check. Capture **only the root LM trajectory**. For a training run at step *t*, compute the average nearest-neighbour distance from each eval trajectory to its closest previously-seen training trajectory:

```
E_{e~E_t}[ min_{τ∈T_≤t} d(x_τ, x_e) ]   ≈   (1/|E_t|) Σ_e min_τ d(x_τ, x_e)
```

Report similarity = 1 − d under all five proxies:

| Metric | d(train, eval) |
|---|---|
| Edit | token-level Levenshtein / max(‖a‖, ‖b‖) |
| Contain (asymmetric) | 1 − \|N₃(eval) ∩ N₃(train)\| / \|N₃(eval)\| |
| Jaccard | 1 − \|N₃(eval) ∩ N₃(train)\| / \|N₃(eval) ∪ N₃(train)\| |
| Weighted Jaccard | 1 − Σ min(c_train, c_eval) / Σ max(c_train, c_eval) |
| Length (content-blind) | 1 − min(‖train‖, ‖eval‖) / max(‖train‖, ‖eval‖) |

N₃ = multiset of word 3-grams; c_x(t) = count of type t in x.

- Beat an appended-context baseline on **every** metric, not on average.
- Keep the content-blind **Length** metric. Similar content at very different lengths is the A1 failure showing up from the trajectory side.
- **Do not stop at the proxies.** The authors are explicit that these "don't fully capture the semantic similarities between these trajectories (e.g. it doesn't say anything about if the RLM generally chooses the same decomposition strategy as its train trajectories)". Add a strategy-level check: extract the sequence of programmatic operations (chunk, fan-out, regex, filter, aggregate, sort) from train and eval runs and compare those *sequences*, not just tokens.

---

## If you are training the harness

Only the root LM is trained in the post's setup; sub-agents are not.

1. **Train short, evaluate long.** Train on a short split, hold out a split 8–32x longer. This only pays off if the harness is length-agnostic (A1), which is what makes it a sharp test.
2. **Never select checkpoints on train reward.** In the post's runs the base Transformer's train reward "generally exceeds that of the RLM despite clear gaps in performance on the eval", while its eval stays flat. Gate on held-out long / cross-domain eval, at a fixed cadence.
3. **Discount early eval bumps.** The baseline's initial gains on OBLIQ-Bench "generally come from learning to follow the proper answer format, which quickly diminishes."
4. **Treat eval lift exceeding train lift as signal.** In some of the post's length-generalization runs it meant the policy started with a non-generalizable short-task solution and later discovered a generalizable decomposition. Find that switch and reinforce it.
5. **Build eval splits that are structurally isomorphic but token-distributionally disjoint.** The post's transfer setups: train on Twitter stance → eval on Wildchat errors; train on essay-authorship search → eval on math-reasoning-analogy search. Shared surface tokens between train and eval mean you are not measuring compositional generalization.
6. **Baseline correctly.** Compare against (a) the same base model as a plain long-context Transformer with context extension, and (b) a frontier model inside *your* harness as a ceiling. If a small trained model in your harness approaches the frontier model in the same harness, the harness is doing the work.

---

## Gotchas

- **Compaction and mid-run summarization are diagnoses, not treatments.** If a scaffold needs them to survive, A1 has already failed.
- **Small structural leaks are fine; unbounded ones are not.** Printing 17 ids is admitted. Printing 17 summaries is a leak. The test is whether the printed thing is O(1) in task size.
- **A passing A2 with a failing A1 means the leak is in tool returns, not the input.** Check A3 next, not A2 harder.
- **Prefix-stuffing survives refactors.** The corpus moves into a variable but the query, the file list, or the system prompt still carries task-specific strings. Grep the whole root prompt, not just the user turn.
- **Errors are payload.** Stack traces and tool stderr are the sneakiest LID break — uninvited and unbounded in size.
- **Sub-agent count is not a quality metric.** Many sub-agents that all pipe prose back to the root is worse than two that pipe handles.
- **"It works on my 30k-token test case" proves nothing.** Every failure mode here is invisible below the point where the naive approach also works. Always sweep ≥16x.
- **High Levenshtein/Jaccard similarity is not proof of isomorphism.** Confirm the decomposition strategy matched too.

---

## What this does not license

- **The structure does not guarantee the behavior.** "Length generalization occurs because the RLM learns a generalizable strategy, but this is not always guaranteed." A correctly-built harness can still collapse into a single sub-call and become the long-context baseline it was supposed to beat. A7 exists for this.
- **Cost is real.** Training runtime is 1.5–3x longer than the base Transformer counterparts on similarly-sized tasks, plus extra memory, "due to multiple steps per sample and waiting on sub-calls". The mitigation is comparative, not absolute: the ratio improves as task complexity grows, since training a Transformer on longer-context tasks is itself much more expensive.
- **Do not over-engineer the program.** The post's own warning: it is easy to walk away from these results thinking we should all be "imposing our problem-specific intuitions around overly structured programmatic strategies such as MapReduce or dynamic programming. But make no mistake, doing that we will inevitably run afoul of the bitter lesson and fall by the wayside within months." The defensible position is a *general, learnable* inductive bias — offloading plus programmatic sub-calls, trained end to end — not a hand-authored task-specific pipeline. And: "Scaling data will remain the biggest driver of progress" — the harness sets the coefficients, not the primary term.
- **How much supervision is needed is unresolved.** MRCRv2 required a decomposition nudge to find the generalizable strategy. The authors' stated *intuition* is that at scale no supervision is necessary, with some hint helpful for sample-efficient learning — explicitly a conjecture, not a result.
- **The formalism is loose by the authors' own account.** Judging whether a prompt is in-distribution is "rather difficult", and benchmark performance is used as a proxy; for trajectories it is "even harder". The equivalence-class construction needs an epsilon-ball framing to preserve transitivity, and "a more rigorous treatment of this topic is likely necessary." Treat LID as a design heuristic with measurable consequences, not a theorem.
- **The negative premise is hedged too.** "It seems very plausible that compositional generalization may emerge at a vanilla neural level." The claim is about today's inductive biases and the coefficients of scaling, not an impossibility result.

---

## Evidence

What was actually measured. Do not overstate beyond this.

- Model: Qwen3-30B-A3B-Instruct-2507, trained two ways — as an RLM (**only the root LM is trained**) and as a base Transformer on its own, with YaRN for longer-context settings. RL via prime-rl, decoupled PPO with GRPO-like advantages plus KL loss. 150 steps for length generalization, 500 for domain transfer.
- Headline, verbatim: "Training on only short tasks generalizes to held-out tasks 8-32x longer, with roughly 10x the eval lift with the same train lift over training the underlying Transformer directly."
- Six length-generalization benchmarks (MRCRv2 64k→2M, GraphWalks <128k→>1M, LongBenchPro 32k→256k, OOLONG 32k→256k, OOLONG-Pairs on output length, Ada-LEval 8k→128k). Across all six, the RLM yielded significantly better long-eval results while training exclusively on short tasks, sometimes from a lower step-0 score, while the base Transformer's eval was generally flat. On MRCRv2, GraphWalks, OOLONG and OOLONG-Pairs the trained RLM approached or exceeded an RLM running GPT-5.5.
- The MRCRv2 Transformer baseline could not be run at all — 2M tokens do not fit in context, even with context extension.
- Three domain-transfer setups with completely different token distributions between train and eval; the RLM showed clear transfer, the base Transformer struggled to meaningfully improve, and its train reward generally exceeded the RLM's.
- Trajectory-distance analysis — across the 5 of 6 length experiments that have a base Transformer baseline plus the 3 domain-transfer experiments, comparing the best RLM checkpoint against the best base Transformer checkpoint — found the root LM's eval trajectories much closer to its training trajectories than an appended-context baseline, attributed primarily to context offloading.
- Strategies reused at eval were the general ones: chunking, fanning out sub-calls, regex.

## Provenance

Principles, failure modes, results and caveats above come from the source post. The operational thresholds — the ≥16x sweep, slope ≈ 0, the ~40-token leak grep, the largest-sub-call-share ratio, the A-numbering, and the worked two-task example are engineering conventions and illustrations supplied here to make the principles checkable. They are not figures from the post; do not attribute them to the authors.

Alex L. Zhang and Omar Khattab, "Language model harnesses are compositional generalizers", July 2026. https://alexzhang13.github.io/blog/2026/harness/

```bibtex
@article{zhang2026harnesses,
  title   = "Language model harnesses are compositional generalizers",
  author  = "Zhang, Alex and Khattab, Omar",
  year    = "2026",
  month   = "July",
  url     = "https://alexzhang13.github.io/blog/2026/harness/"
}
```
