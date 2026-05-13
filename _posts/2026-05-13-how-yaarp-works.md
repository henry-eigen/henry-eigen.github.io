---
layout: post
title: "How YAARP works"
date: 2026-05-13
description: "How an automated research loop is built so agent self-evaluation doesn't corrupt the verdict: the design checked before any code is written, the controlled conditions structurally unable to vary, the verdict a function of the design and the raw numbers only."
---

## The structural argument

Auto-research systems (agents which design an experiment, run it, and evaluate the results [3]) show a recurring set of failure modes when nothing structural constrains them: unfair comparisons, train/test leakage, metric inconsistency, selection on the test set, an agent grading its own work [1][2]. The shape across all of them is the same. An agent makes a methodological choice (which baseline counts, which metric, which threshold), and the same agent later judges whether the results satisfy that choice. There is no party in the loop without a stake in the outcome.

At the front of YAARP's loop sits an *experiment definition*: a structured specification of what is being compared, what is held fixed across every condition, and what would count as the technique holding up. Producing that definition is the first stage of the loop, and because it is structured rather than free-form prose, two specific properties become deterministically checkable. First, the soundness of the definition before any code is written: whether the comparison set is well-formed, whether the metric is named and present in the spec, whether the directional claim is coherent with the comparison structure. Second, the verdict at the back of the loop: given the locked metric and threshold from the definition and the raw numbers from the run, whether the result satisfies the criterion. Both checks are run by deterministic programs rather than by agents.

In between sit implementation and execution. Implementation is where an agent writes the code that operationalizes the definition. Unlike the spec's internal consistency or the verdict's arithmetic, faithful implementation has no formal test, so this step cannot be reduced to a deterministic check. Implementation is therefore made trustworthy a different way: by being structurally constrained up front, so the comparison cannot be broken by the choices an agent makes while filling in the code. Execution then runs the constrained code against the controlled conditions the definition specifies.

**design** → implementation → execution → **verdict**

In outline: a technique is submitted; an experiment is designed to test it; agents build it and its variants; the variants run; the results are evaluated mechanically; a report is returned. Two human checkpoints sit at the cost boundaries. We begin with the build, then come back to the design and the verdict at each end.

---

## Implementing the design

![How the agents set up a controlled experiment: Design produces the experiment definition; Define splits it into the shared harness and the technique slot; Build has the harness builder and one technique implementor per entry; Review has the code reviewer inspecting all implementations together](/assets/images/how-yaarp-works-build.png)

The experiment has a fixed design before any code is written, and implementation has to realize it in a way the agents writing the code cannot subsequently disturb.

The *harness* is a complete training pipeline (data loading, model, trainer, evaluator, configuration) that operationalizes the controlled conditions and is shared across every entry. The *technique slot* is the single point at which anything is permitted to vary; it is filled in turn by the new technique, each comparison technique, and the baseline.

A harness builder writes the harness body and exposes exactly one typed extension point; the wiring between harness and module is a fixed template the agents never touch. Each technique implementor writes only its own module against the fixed interface and proves it runs via a smoke test before handing it back. The experimental factor takes two shapes, both in the diagram: in the *additive* case the baseline is the bare harness with the slot defaulting to a no-op; in the *substitutive* case the varying component is itself routed through the technique interface and even the baseline implements it.

A technique implementor cannot subsample the evaluation set or retune a fixed learning rate because it does not own the evaluator or write the harness body. The set of files an entry must produce, and the symbols it can and must export, are determined before the implementor sees the comparison group; the pipeline reconciles each entry's output against that set before letting it advance. Comparability is enforced by the architecture, not asked of the agents [4].

A code reviewer then makes a single pass over all the implementations together, checking that they are mutually consistent and faithful to what was specified. This part is agentic and best-effort: the reviewer covers the parts of "faithful" that resist formal definition, like whether the code reflects what the paper actually claims, not just what the slot's signature requires. A critical finding bounces one entry back for a single re-implementation; a second failure drops it and the comparison proceeds with the remaining entries, since the design's full entry set was declared up front and downstream stages reconcile against that declaration.

---

## Designing the experiment

![The Experiment Planner turns a technique description plus a survey of related work into three artifacts: comparison techniques, the experiment definition, and the mechanical-evaluator targets](/assets/images/how-yaarp-works-planner.png)

Every notion of "correct" in the build above is defined relative to the design.

A design agent takes the hypothesis, surveys the related work, and produces three artifacts. The first is the *comparison set*: the techniques the new method will be measured against, plus a baseline, with a classification of the *experimental factor* as either *additive* (baseline is the factor's absence) or *substitutive* (baseline is a reference method). The second is the *experiment definition*: the conditions held fixed across every entry (e.g., dataset, model, training regime, evaluation protocol). The third is a set of directional assertions the mechanical evaluator will later check: statements like "technique X should outperform baseline Y on top-1 accuracy", committed before any experiment code exists.

Together these are a compiled specification, against which the first of the pipeline's deterministic checks runs: the factor classification matches the comparison set, each assertion names a real entry and a real metric, the metric is well-formed against the evaluation protocol.

Past that gate the design hits the first human checkpoint, where a reviewer makes the calls the structural check cannot: is this the right comparison set, is the metric meaningful, is the hypothesis worth the model budget. On approval, the specification is materialized as a comparison group plus one entry per condition (the baseline among them; in the additive case, carrying no technique at all), and downstream stages read and reconcile against it.

---

## Execution and evaluation

![The experiment plan as a schema: it compiles into a fixed set of typed records — one comparison group, k techniques, N entries, A directional assertions, N×S runs to come — whose known row counts add up to an "expected manifest" that every downstream stage is reconciled against](/assets/images/how-yaarp-works-records.png)

Every stage past design is reconciliation against the registered counts: review cannot advance until every entry is accounted for, the run phase cannot hand off until every seed is finished or failed, evaluation aggregates each entry over its `S` seeds and resolves all `A` assertions. An entry that fails review persists as an empty row, and assertions that referenced it return unevaluable rather than being silently dropped. The reconciliations are bookkeeping, not judgement.

This rigidity has a cost: a comparison the designer realizes is missing partway through must be re-run as a separate experiment. Forgoing the ability to adapt in response to intermediate results is what makes the verdict reflect the experiment committed to before any code ran.

The second human checkpoint approves the compute spend. Execution and the verdict that follows it involve no model.

![The evaluation dataflow: raw per-seed metrics, aggregate across seeds, mechanical check against the frozen assertions, group verdict, all inside a no-LLM box; the Analyst agent sits downstream of the finished verdict with a one-way arrow and cannot change it](/assets/images/how-yaarp-works-evaluation.png)

Each entry runs once per seed on an ephemeral sandbox that holds no platform credentials: workspace in, metrics out. A seed that fails does not count toward the aggregate. Evaluation runs in two layers. The first is the second of the pipeline's deterministic checks: aggregate metrics across seeds, resolve each assertion against the tolerance, roll the resolutions into a group verdict. The second layer is a single model call that writes the narrative around the finished verdict and cannot modify it.

The verdict is a function of the design and the raw numbers only, not of an LLM judge against a reference [4], an automated reviewer [3], or the implementing agent.

---

## Orchestration

![The YAARP pipeline, end to end: lifecycle states, the gates between them, and the worker behind each state](/assets/images/how-yaarp-works-pipeline.png)

A stateless, crash-safe orchestrator evaluates the gate on each lifecycle transition, polls for finished work, and dispatches the next item. Two of the five gates are the human checkpoints already mentioned (model budget, compute spend); three are automatic: review passed, runs complete, verdict computed.

The pipeline shape is conventional; the properties that make it trustworthy do not come from the shape but from what sits at the gates. The orchestrator dispatches agents into the work between gates and stops them advancing until a gate permits it. It is not itself trusted to know whether the work is sound; the gates do that, deterministically, on artifacts the agents have produced.

No agent processes the results of its own work. This is the main divergence from related approaches: Curie wraps a rigor engine of LLM validators around a free-form Architect/Technician pair [4], with the validators themselves agents and conclusions still scored by an LLM judge; Ara structures the research artifact after the fact [5], on the consumption side rather than the production side. YAARP places deterministic checks at the load-bearing transitions and lets agents own the parts of the work that are not formalizable.

---

## What's next

### Routing work to the right tier

![The lifecycle forking at the `implementing` step into tiered lanes — a strong model for complicated techniques, a smaller model for simple ones — and again at `running` — heavier runs on GPU, lightweight ones on CPU — rejoining at the `implemented` and `evaluating` barriers](/assets/images/how-yaarp-works-tiers.png)

Every unit of work reads its inputs from object storage, runs, and writes outputs back; the orchestrator is the only thing that sequences them. Where a unit runs is therefore a routing choice. Today's routing is coarse, tied to an agent's role rather than to the specific job, so the pipeline does not distinguish between a complex technique that warrants a strong model and a one-line library call that does not, or between a heavy training run and a trivial one. The missing piece is small: annotate each entry and each run with a tier (in the design agent or a separate router) and dispatch off that. The fan-out is already in the pipeline shape; the lanes are currently identical only by default.

### A reusable corpus of code and traces

Everything an experiment produces lands in object storage as the experiment runs. For a five-condition comparison group, this is on the order of a hundred files:

```text
comparison-groups/01KN8S9F…/                 # one comparison group = one experiment
│
├── harness/                                 # the harness builder's workspace
│   ├── experiment.py                        #   fixed template — the entry point (not agent-written)
│   ├── harness/
│   │   ├── config.yaml                      #   the controlled conditions, frozen at design time
│   │   ├── data_loader.py
│   │   ├── model.py
│   │   ├── trainer.py                       #   ← the agent writes the harness body
│   │   └── evaluator.py
│   ├── techniques/
│   │   ├── __init__.py                      #   fixed template — wires the one extension point
│   │   └── baseline.py
│   ├── validation_results.json              #   the smoke run proving the harness actually runs
│   ├── status.json                          #   progress, written periodically by the agent
│   └── conversation-trace.json              #   the full agent transcript
│
├── entries/                                 # one workspace per condition — same harness, different slot
│   ├── 01KN8S9F…/code/                       #   technique A
│   │   ├── harness/…                         #     the harness, copied in unchanged
│   │   ├── techniques/
│   │   │   ├── baseline.py
│   │   │   └── technique_a.py                #     ← the one file this implementor wrote
│   │   ├── validation_results.json
│   │   ├── metrics.json
│   │   └── conversation-trace.json
│   ├── 01KMY6B4…/code/techniques/technique_b.py
│   ├── 01KMY69Z…/code/techniques/technique_c.py
│   ├── 01KMY6EE…/code/techniques/technique_d.py   #   failed review — entry kept, no metrics
│   └── …                                     #   the remaining conditions, incl. the baseline
│
├── code-review-attempt-1.json               # first pass — bounced an entry
├── code-review-attempt-2.json               # second pass — clean
├── code-review.json                         #   (latest)
│
└── runs/                                    # one directory per (entry, seed)
    ├── 01KN8S9FDR…/seed-0/
    │   ├── metrics.json                     #   what the GPU sandbox wrote out
    │   ├── run_status.json
    │   └── modal_execution.json
    └── …
```

This artifact trail is a foundation for two things. First, the code: every technique implementation the platform has produced is stored, validated, and paired with the harness it ran against. A queryable store would let a future implementor agent start from a working version of a related technique, not a blank file. Each implementation has already passed the conformance checks for its experiment, so it is a known-good module against a specific harness rather than a snippet pulled from the open internet. Second, the traces: conversation transcripts, code-review findings, run statuses, and verdicts form a labeled record of where the pipeline ran smoothly and where it did not, useful raw material for tuning the pipeline as a whole (prompts, gates, scaffolding) rather than each agent in isolation.

### The open version, and why a DSL

YAARP is wired tightly to its own infrastructure and is not something a third party can clone and run. A from-scratch rebuild ([SMAI](https://github.com/yaarp-org/smai/tree/main)) is in progress: same pipeline shape, modular and infrastructure-agnostic, with the substantive change in how the inter-stage contracts are produced. Today they are derived by the design agent and validated by the structural check at the entry to build; SMAI moves the derivation into a compiler. The experiment is expressed in a small declarative language and the compiler emits the contracts as typed artifacts. What the harness must expose, what a technique implementation may modify, which metric the verdict will read and what threshold it must clear: checkable properties of a typed object, not conventions the code is expected to honor.

Declarative experiment languages have been doing a version of it for years in adjacent fields: the design is expressed in a small language, the system compiles it into the artifact that runs (statistical tests, a model specification, an assignment procedure), and a checker rejects structurally invalid designs before any data is collected [6]. The consequence in SMAI is that the DSL determines what can be checked downstream: every property pushed from prose into the language is one more thing the pipeline can verify mechanically instead of asking an agent to confirm.

The disclaimer these languages carry applies here too: this is a compiler, not a proof engine. It cannot tell whether the hypothesis is worth testing, whether the metric is meaningful, or whether the directional assertions were the right ones to commit to. The parts of research where that judgement lives (idea generation, novelty, the writeup) are deliberately outside YAARP's scope.

---

## Appendix: the operator's view

<div class="blog-figure-row" markdown="1">

![A comparison group's design page — the frozen controlled conditions, evaluation mode, seeds, tolerance, budget, and the directional assertions in plain English](/assets/images/how-yaarp-works-shot-overview.png)
![The same comparison group mid-build — harness done, entries being implemented, several agent jobs running](/assets/images/how-yaarp-works-shot-build.png)

</div>

<div class="blog-figure-row" markdown="1">

![The experimental-configuration tab — the full harness specification alongside the directional assertions, with the pipeline bar showing code review in progress](/assets/images/how-yaarp-works-shot-config.png)
![A comparison group's runs partway through — some training runs complete, one in flight, results landing per entry](/assets/images/how-yaarp-works-shot-running.png)

</div>

![A completed Evaluation tab — directional-assertion results and an overall verdict](/assets/images/how-yaarp-works-shot-results.png)

---

## Prior and related work

- [1] **Auditing automated research:** Luo, Kasirzadeh & Shah (2025), *The More You Automate, the Less You See: Hidden Pitfalls of AI Scientist Systems* — controlled probes showing benchmark cherry-picking, data leakage, metric misuse, and post-hoc selection bias in two open-source AI-scientist systems, all largely invisible from the final paper; recommends that venues require, and audit, the trace logs and code.
- [2] **Evaluating an autonomous-research system:** Beel, Kan & Baumgart (2025), *Evaluating Sakana's AI Scientist* — a hands-on assessment: ~42% of proposed experiments fail on code errors, ~57% of manuscripts carry hallucinated numbers, and the system "cannot critically assess its own results."
- [3] **End-to-end research agents:** Lu et al. (2024) and Yamada et al. (2025), *The AI Scientist* (v1 and v2) — a single agentic pipeline that generates ideas, writes and runs code, drafts the paper, and runs its own peer review; the canonical "one agent does the whole loop" system.
- [4] **Rigor as a wrapper around agents:** Kon et al. (2025), *Curie: Toward Rigorous and Automated Scientific Experimentation with AI Agents* — an "Experimental Rigor Engine" of LLM validators, a control-flow state machine, and a role-scoped knowledge store, around an Architect/Technician agent pair; the closest published system in spirit, with conclusions still scored by an LLM judge against ground truth.
- [5] **Structuring the artifact rather than the run:** Liu et al. (2026), *The Last Human-Written Paper: Agent-Native Research Artifacts* — replaces the narrative paper with a machine-executable package (logic / code / trace / evidence) joined by traceable bindings, plus a tiered verification credential.
- [6] **Experiment DSLs:** Jun et al. (2019), *Tea*; Jun et al. (2022), *Tisane*; Bielicke et al. (2025), *PLanet* — declarative languages for study and experiment design that compile a spec into the artifact you run (a set of statistical tests, a model, an assignment procedure) and reject structurally invalid designs before any data is collected. Earlier: Soldatova & King (2006), *EXPO* — formalizing published experiments and catching real structural flaws in peer-reviewed papers.
