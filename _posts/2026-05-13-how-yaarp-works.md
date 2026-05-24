---
layout: post
title: "The Architecture of Mechanized Research"
date: 2026-05-13
description: "SMAI separates agent reasoning from scientific verdicts by turning experiment designs into mechanically checkable commitments before any results are observed"
---

_This post began as a single piece and has since split into two. The motivating argument now lives in its own post, [found here](/2026/05/18/where-agents-belong.html). That post should be read before this one._

## Picking up the chain

The motivating post took the inferential chain of an experiment (hypothesis, design, experiment, results, verdict) as an object of design, and at each link asked what could be established by a mechanical check and what could only be left to an agent's reasoning. It closed by proposing a system built to take advantage of whatever guarantees of validity could be established mechanically.

This post describes that system. SMAI takes a submitted technique, designs an experiment to test it, implements and runs that experiment, and returns a grounded verdict. What follows walks the same chain a second time, now as a description of working parts, and shows that the mechanical surfaces fall where the earlier argument said they should.

---

## Designing the experiment

![A research agent submits a hypothesis to an Experiment Designer agent which draws on a prior-experiments database and a literature-review pass to produce an Experiment Plan in a small declarative language; the plan is consumed by a pure-function Compiler that either bounces back a structured error to the designer or emits three typed contract artifacts: an Env Harness Contract, a per-entry Factor (technique) Contract, and an Evaluation Contract](/assets/images/how-yaarp-works-design.png)

Most of SMAI's mechanical grounding lies downstream of the experiment design. Beyond a check on its own well-formedness, the design stage produces the artifacts that anchor every check that follows.

The agent writes its experiment design in a DSL (the language itself is the subject of a later post), and the design stage concludes when that design successfully compiles. Compilation does three things:

1. It checks the experiment for well-formedness, confirming that it compares treatments of a single factor along a common metric and ruling out the structural errors (confounded conditions, missing or incoherent baselines, a directional claim at odds with the comparison set) that would allow explanations apart from the technique itself.
2. It emits implementation contracts, one for the experiment harness and one for each technique.
3. It emits a mechanical evaluator, a function that consumes the metrics the design named (the same ones the implementation contracts require the runs to emit) and applies the design's directional and threshold criteria to return a boolean verdict on the hypothesis.

Compilation itself gives an assurance about the design's internal validity, and the implementation contracts it produces extend a verification surface across every stage that follows. Beyond what these two establish, the evaluator freezes the mapping from output to verdict before any output exists, guarding the eventual verdict against both miscalculation downstream and revision of the criteria once results are in hand. Taken together, these three artifacts put SMAI's mechanical grounding exactly at the boundary the companion post located.

Everything upstream of the compiler stays in reasoning space. Generating the hypothesis is an agent's work, as is reasoning about appropriate baselines and the comparisons to make. Only once an agent has settled what it wants to show, and how, does it meet a mechanical surface at all.

---

## Implementing the design

![How the agents set up a controlled experiment: Design produces the experiment definition; Define splits it into the shared harness and the technique slot; Build has the harness builder and one technique implementor per entry; Review has the code reviewer inspecting all implementations together](/assets/images/how-yaarp-works-build.png)

Implementation offers guarantees only insofar as it leans on the contracts the design emitted; everywhere else it is back in reasoning space.

One guarantee we encode structurally, in the split between the harness (the controlled conditions held fixed across every treatment of the factor) and the treatments (the part that varies). The harness contract's natural-language naming of those conditions guides the agent building the harness but offers us no surface to check the resulting work against. It also specifies the single extension point into which a treatment plugs, and the shapes the metrics must take, and these specifications we can check directly, confirming both that the harness exposes the extension point in the shape the contract fixed and that it emits its metrics in the shape the evaluator will later require.

Each treatment is implemented separately against its own technique contract, which has the same two-part structure as the harness contract. Its natural-language part tells the agent which method to build and offers no checkable surface, while its statically checkable part confirms that the implementation fills the extension point and overrides none of the harness's methods or variables.

This divide, between the natural-language parts of a contract and the statically checkable parts, falls along the verification-versus-validation distinction drawn in the companion post. Verification the contracts give us directly. Validation, the question of whether an implementation is faithful to the method it is meant to be, remains a reasoning-space judgment that falls to a code-review agent.

Where the verification surface lands depends on the grammar of the DSL. The more the grammar encodes, the more of each implementation we can check, but every concept added to the grammar narrows the language's domain. Were we to make "dataset" or "model architecture" first-class grammatical concepts, we would gain verification and, in the same move, restrict the language to experiments that train a model on a dataset. The system described here aims at the broader domain of valid scientific experiments, and so makes no such additions; a field like `dataset` is carried as an opaque keyed value, a string routed to an agent inside a natural-language specification, not a construct the grammar understands. Whether it is worth maintaining a family of narrower DSLs with richer grammars, each buying more verification within a smaller domain, is a question for a later post.

---

## Execution and evaluation

![The experiment plan as a schema: it compiles into a fixed set of typed records (one comparison group, k techniques, N entries, A directional assertions, N×S runs to come) whose known row counts add up to an "expected manifest" that every downstream stage is reconciled against](/assets/images/how-yaarp-works-records.png)

Past implementation, the rest of the pipeline runs mechanically. Implementation yields a set of executables which, together with a few values carried over from the design (the randomization seeds, the seed count), can be routed blindly to the execution environment and run.

![The evaluation dataflow: raw per-seed metrics, aggregate across seeds, mechanical check against the frozen assertions, group verdict, all inside a no-LLM box; the Analyst agent sits downstream of the finished verdict with a one-way arrow and cannot change it](/assets/images/how-yaarp-works-evaluation.png)

Because the implementation contracts fixed the shape of every run's output, the metrics that come back go straight to the evaluator the compiler emitted, which applies the design's thresholds and directional criteria and returns the verdict. No agent reads the results of its own work, and no model sits anywhere on the path from raw metrics to the verdict. The analyst agent that sits downstream of the evaluator writes the narrative around a verdict it cannot alter.

---

## Orchestration

![The SMAI pipeline, end to end: lifecycle states, the gates between them, and the worker behind each state](/assets/images/how-yaarp-works-pipeline.png)

Holding the stages together is a stateless orchestrator that evaluates a gate on each lifecycle transition, polls for finished work, and dispatches the next unit. The shape of the pipeline is conventional. What makes it trustworthy is not the shape but what sits at the gates (the compiler, the conformance checks, the evaluator). The orchestrator dispatches agents into the work between gates and holds them there until a gate, deterministic and operating on the artifacts the agents produced, permits the advance. It is not itself trusted to judge whether the work is sound.

Two of the gates are human checkpoints, placed where cost is committed (the spend on agent model calls before implementation, and the spend on compute before the runs). The rest are automatic. This is the same division the companion post argued for, now drawn as a control structure in which agents own the links that are not formalizable and a deterministic check stands at every load-bearing transition between them.

---

## Code

Note, the system described here has been developed and tested in a closed source environment. It is in the process of being ported over to an open source version, but that version is very much in-progress. If you are curious, you can find it at [github.com/yaarp-org/smai](https://github.com/yaarp-org/smai/tree/main). But it does not yet make any guarantees of stability.

---

<!--
=============================================================================
NOTES
- Second half of a post split in two; the motivating argument is now its own
  post, "Where agents belong in automated research". This half is a pure
  system description, in that post's voice ("we"/"our", conceptual prose, no
  em-dashes).
- Trimmed pass: the body now mainly argues that SMAI's design is consistent
  with the system imagined in the companion post; the diagrams carry the
  descriptive detail the prose used to spell out.
- Structure: §1 sequel handoff → §2 design [DSL + compiler, three compile
  functions, autoformalization analogy] → §3 implementation [harness/treatment
  split, verification vs validation, grammar/domain tradeoff] → §4 execution +
  evaluation → §5 orchestration → §6 what's next (tiers / corpus) →
  §7 scope (caveat + SMAI repo link) → appendix gallery → related-work pointer.
- DSL specifics, the size of the verifiable surface, and the narrower-DSL
  question are all deferred to separate posts.
- Artifact tree dropped in this pass (too specific for the trimmed body); the
  corpus point is made in prose instead.
- Screenshots: parked decoratively in the appendix; nothing in the body leans
  on them. Decide later whether to keep, caption, or cut.
=============================================================================
-->
