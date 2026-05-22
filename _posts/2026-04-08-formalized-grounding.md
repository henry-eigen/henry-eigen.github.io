---
layout: post
title: "SMAI: Scientific Method as Infrastructure"
date: 2026-04-08
description: "Auto-research agents can explore solution spaces faster than humans. But purely agent-driven approaches lose the structural rigor that pre-defined environments provide: methodology drift, post-hoc selection, inferential breakdowns. Structural verification provides the scaffolding that lets scaled systems be trustworthy."
---

*Update, 2026-05-12. Since this post first went up (on 2026-04-08) two items have been added to the [prior and related work](#prior-and-related-work) section. Neither one informed the original post.*

## The verifiability tradeoff

Auto-research systems (Karpathy's autoresearch, Google's AlphaEvolve) allow agents to iteratively modify code, run experiments, and evaluate results, in order to search for optimizations. Given a baseline implementation to optimize, and a scoring function with which to evaluate modifications, agents are remarkably capable at hill-climbing in code space.

This requirement for existing evaluation mechanisms, though, restricts research agents to working on problems for which this "experiment infrastructure" has already been defined. What if, in order to remove this limitation, the research agent were additionally tasked with building the infrastructure needed to evaluate its own work?

![Diagram 1](/assets/images/formalized-grounding-diagram-1.png)

Several recent works have demonstrated the viability of this bootstrapped approach. A notable example, AI Scientist v2 (Sakana AI, 2025), tasks agents with designing experiments, choosing datasets, selecting metrics, and evaluating the results end-to-end, substantially expanding the breadth of problems to which auto-research can be applied.

But subsequent evaluations of purely agent-driven systems have revealed significant shortcomings [5]: methodological flaws can go undetected, results can be hallucinated, and metrics used inconsistently. These are the kinds of issues that pre-defined environments guarded against. How then can we allow agents to build experiment infrastructure, while keeping the same level of rigor that pre-defined environments provide?

The failure modes span multiple reproducibility types: methods, results, and inferential [1]. Some are catchable programmatically (hallucinated results live in code/data space); the harder ones, like methodology drift, post-hoc selection, and inferential breakdowns, live in reasoning space, where the apparent option is to delegate verification to the same agent whose work is being evaluated. The question is whether agent-driven research loops can be structurally constrained to catch these failure classes, without losing the exploratory breadth that bootstrapping enables.

## Have your cake and eat it too

To bridge this gap, we propose SMAI (Scientific Method as Infrastructure), a structural framework against which agents can ground the design, implementation, execution, and evaluation of their experiments.

![Diagram 2](/assets/images/formalized-grounding-diagram-2.png)

A careful reader, upon inspection of Diagram 2, might reasonably ask: isn't this just another end-to-end agent workflow? Doesn't this leave unaddressed the same inferential gaps SMAI sets out to patch? The reader would be correct to flag this, because the structural verification doesn't happen in the agent layer shown in the diagram; it happens in the orchestration layer upon which the agent layer is built. SMAI's value lives in the validation contracts which serve as deterministic checkpoints between stages in the workflow.

These contracts take several forms throughout the workflow, but they originate in a DSL that SMAI provides for expressing experiment definitions. This DSL, drawing on prior work in scientific ontology and experiment design formalization [2][3], allows structural validity to be enforced independently of the agent's reasoning at the most foundational part of the workflow, experiment design. Concretely, by forcing the agent to use the DSL, we get:

- A compiler that catches structural issues in experiment definitions before anything downstream runs.
- Contracts that flow forward through the workflow: the compiled definition yields a harness contract (extension points, fixed variables, required metrics), and the harness yields a technique contract constraining what each method implementation can modify. Each stage narrows what downstream agents can get wrong (see appendix for concrete examples).
- A shared data model across all experiments, enabling structured cross-experiment collaboration.

![Diagram 3](/assets/images/formalized-grounding-diagram-3.png)

Once techniques pass code review, the remainder of the workflow is fully mechanical (no agents). The orchestrator picks up the code artifacts, executes them, and aggregates the raw results.

The final link in the chain is evaluation. A validation config, compiled from the experiment definition and enriched by the method implementation, specifies what to compare, which metric to use, and the minimum threshold. A fixed evaluation runner takes this config and the raw results, and returns a boolean: did the results meet the validation criteria?

This boolean, the ultimate output of the workflow, answers the question of whether the experiment validated the hypothesis.

![Diagram 4](/assets/images/formalized-grounding-diagram-6.png)

With this, the full inferential chain is structurally grounded: from hypothesis through experiment design, implementation, execution, to evaluation.

## Render unto agents the things that are reasoning

The careful reader will have noticed that, even with contracts that structurally verify each link in the inferential chain, we fall short of any claims to formal verification. This is because SMAI doesn't verify experiments the way a proof engine verifies theorems, but rather in the way a code compiler verifies code. A compiler can't tell you whether your code does what you intended, but it can guarantee syntactic and structural correctness. Similarly, SMAI's experiment compiler can't tell you whether the hypothesis is trivial, but it can ensure that the scaffolding is sound.

One reasonable objection: isn't structural verification anti-Bitter-Lesson, imposing hand-engineered scaffolding where general methods plus scale should win? The argument doesn't transfer. The Bitter Lesson concerns the modeling layer, where hand-engineered priors compete with scale-plus-search inside the learning algorithm. Verification infrastructure lives at a different layer. We don't argue that compilers, type checkers, or test frameworks should be replaced by larger models; they're the scaffolding that lets scaled systems be trustworthy. A larger agent that's still self-evaluating is more sophisticated at fooling itself, not less.

This level of verification would have caught many of the specific failures documented in existing agent-driven systems [5]. Metric substitution is impossible when the validation config locks the metric at compile time. Hallucinated results don't survive non-agentic execution. A parameter that should vary can't be held fixed when the compiler enforces the experimental factor. The structural contracts may not provide formal guarantees for every failure mode, but they cover the ones that have emerged in practice.

The divergent paths in auto-research suggest that the tradeoff between exploratory freedom and experimental reliability is inherent. SMAI seeks to demonstrate that it is not zero-sum.

## Appendix: Contract examples

The following pseudocode illustrates what these contracts look like in practice, using a simple augmentation experiment as an example.

![Diagram 5](/assets/images/formalized-grounding-diagram-5.png)

![Diagram 6](/assets/images/formalized-grounding-diagram-4.png)

## Prior and related work

- [1] **Inferential reproducibility:** Goodman, Fanelli & Ioannidis (2016) defined the three types of reproducibility — methods, results, and inferential — and identified inferential reproducibility as the most neglected.
- [2] **Scientific ontology:** EXPO (Soldatova & King, 2006) formalized published experiments in Prolog, catching real structural flaws in peer-reviewed papers.
- [3] **Experiment DSLs:** Tea (Jun et al., 2019), Tisane (Jun et al., 2022), and PLanet (Jun et al., 2025) demonstrated compilation-based approaches to experimental design — deriving statistical tests, model specifications, and assignment procedures from declarative specifications.
- [4] **Auto-research:** AlphaEvolve (Google, 2025), AI Scientist (Sakana AI, 2024/2025), and Karpathy's autoresearch have shown the power and limitations of agent-driven experimentation at both ends of the verifiability tradeoff.
- [5] **Agent-driven system evaluations:** Beel et al. (2025) found that the AI Scientist agent "cannot critically assess its own results" and "fails to detect methodological flaws or logical inconsistencies," with 4 of 7 successful manuscripts containing hallucinated results. "Hidden Pitfalls of AI Scientist Systems" (2025) documented post-hoc selection bias, benchmark shopping, metric substitution, and data fabrication. EXP-Bench (Kon et al., 2025) benchmarked agents on 461 paper-derived experimental tasks and found end-to-end success near 0.5%, with conjunctive failures concentrated in experiment design and conclusion validity.
- [6] **Rigor as infrastructure** (added 2026-05-12; not drawn on for the original post, see the note at the top): Curie (Kon, Liu et al., 2025) embeds intra- and inter-agent rigor modules directly into the agent loop (procedural validators and a control-flow state machine), making rigor a runtime concern rather than prompt-engineering. The closest published thread to SMAI's framing; the architectural distinction is that Curie's rigor is procedural and lives inside the agent surface, while SMAI's is structural and lives in the orchestration layer below it.
- [7] **Agent-native research artifacts** (added 2026-05-12; published after this post): ARA (Liu et al., 2026) restructures research papers from narrative PDFs into machine-executable knowledge packages, separating logic, source, exploration, and evidence into typed layers. ARA targets the consumption side of the gap (turning finished research into agent-readable artifacts); SMAI targets the production side (constraining how new research is run).
