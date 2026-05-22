---
layout: post
title: "Drawing the Line on Automated Research"
subtitle: "Where mechanism ends and agents begin"
date: 2026-05-18
description: "Automated research systems are built by intuition and mimicry, not from the theory of experimental validity. Walking the inferential chain of an experiment link by link, this post locates where mechanical checking ends and agent reasoning must begin."
---

## What is auto-research

It is by now a familiar observation that agents perform most effectively in domains with a large verifiability surface, that is, domains in which much of the work can be mechanically checked for correctness. Mathematics is the standard example of a fully verifiable domain where, through autoformalization, an entire proof can be rendered in a formal system and machine-checked. Code is another where, although semantics lie outside the verifiability surface, syntax and form provide enough checkability that agents remain effective. Our interest is primarily in automated research agents, and so this post sets out to explore the verifiability surface of the scientific process itself.

Automated research systems task agents with formulating hypotheses, designing experiments to test them, running the experiments, and interpreting the results. These systems tend to fall into one of two categories, distinguished by how much of that process is fixed mechanically and how much is left to the agents' reasoning. 

In the narrower formulations, a specified baseline and a pre-built grader are provided, removing experiment design and results analysis from the agent's responsibilities. The agent's task then is only to mutate the baseline in order to optimize the score it receives from the grader. Because the grader fixes much of the process mechanically, and in advance, it can act as a strong verifier of the agent’s output. That same fixedness, however, also limits the system, rendering anything the grader was not built to score outside the scope of the agent's exploration.

The more general systems, those we are primarily interested in, delegate the entire process to agents. The agent loop is responsible for selecting baselines, designing experiments, and evaluating the results. A verifiability surface is available to them as well, but it is not so clearly defined as a pre-built grader. More on this below.

Among these general systems, none we found tasked a single agent with the end-to-end responsibilities [1]. All decompose the process into a series of phases or tasks, to which specific prompts or specialized agents are assigned. What distinguishes one system from another, their novel contribution, largely concerns how tasks are split amongst agents, if/how agents communicate, how procedural or how open-ended their instructions are, and whether domain knowledge is hardcoded or ingested.


## What is it missing

Setting aside prompt context, there is little in the machinery of these systems which is specific to the shape of scientific research, as opposed to agent orchestration work in general [2].

This is not for want of a body of theory to build on. A long tradition of work on experimental design and validity methodology offers a strong foundation upon which to build such machinery. These bodies of work seem largely neglected by the prominent auto-research systems however; a scan of the works cited sections of several uncovered nearly no references to works in the philosophy of science concerning statistical validity, experiment design, causal inference.

This would be a pedantic complaint if the performance of these systems suggested they had little to benefit from such grounding. But common failure points identified by audits [3, 4] of these systems suggest otherwise. Issues surfaced by these audits include unvaried factor levels, post-hoc selection bias, failure to isolate control factors, etc. These are exactly the types of issues which structural machinery can catch mechanically.

There is a deeper pattern beneath these failures. While these systems typically structure their pipelines into a set of phases roughly mapping to the stages in the scientific process, they seem to do so more as a means of organizing the agent tasks. The stages are treated as steps in a workflow, rather than links in the inferential chain, by which experimental results are able to render a verdict on the hypothesis.


<figure class="blog-figure">
<svg width="100%" viewBox="0 0 900 108" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="The full inferential chain: hypothesis, design, experiment, results, verdict" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif" style="display:block;">
<defs>
<marker id="k-neutral" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#8b8b8b"/></marker>
<marker id="k-gap" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#b5822f"/></marker>
<marker id="k-mech" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#3f7d6a"/></marker>
</defs>
<line x1="153" y1="54" x2="207" y2="54" stroke="#8b8b8b" stroke-width="3.1" marker-end="url(#k-neutral)"/>
<line x1="333" y1="54" x2="387" y2="54" stroke="#b5822f" stroke-width="3.1" marker-end="url(#k-gap)"/>
<line x1="513" y1="54" x2="567" y2="54" stroke="#3f7d6a" stroke-width="3.1" marker-end="url(#k-mech)"/>
<line x1="693" y1="54" x2="747" y2="54" stroke="#3f7d6a" stroke-width="3.1" marker-end="url(#k-mech)"/>
<g><rect x="30" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="90" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Hypothesis</text></g>
<g><rect x="210" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="270" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Design</text></g>
<g><rect x="390" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="450" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Experiment</text></g>
<g><rect x="570" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="630" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Results</text></g>
<g><rect x="750" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="810" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Verdict</text></g>
</svg>
<figcaption>Various surveys decompose the research process according to slightly different phase taxonomies. Some group more broadly (e.g. a four-stage pipeline which treats experiment planning and execution as one). Some introduce additional phases (e.g. a six-stage pipeline which distinguishes literature review from idea generation). But none otherwise deviate significantly from each other, nor from our five-stage chain.</figcaption>
</figure>


Each link in that chain holds only under certain conditions: a comparison must be well-formed, an implementation must be faithful to its design, a verdict must follow from the metrics it rests on. The experimental-design and validity methodology invoked earlier exists precisely to characterize these conditions. The degree to which the satisfaction of these conditions can be checked mechanically defines the verifiability surface of the scientific process in general. Unlike the hill-climber's grader, which is fixed to one experiment in advance, this surface is intrinsic to the structure every valid experiment shares. A system built on it can therefore offer mechanical guarantees over whatever experiment its agents design.

In the following sections, we take that inferential chain as the object of design, and try to understand what each link requires to be valid. The system we propose is built on the result.

## Design

<svg width="100%" viewBox="0 0 900 108" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="The inferential chain, with the design link highlighted" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif" style="display:block;margin:1.75rem 0;">
<defs>
<marker id="d1-neutral" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#8b8b8b"/></marker>
<marker id="d1-dim" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#cdd0d2"/></marker>
</defs>
<line x1="153" y1="54" x2="207" y2="54" stroke="#8b8b8b" stroke-width="3.1" marker-end="url(#d1-neutral)"/>
<line x1="333" y1="54" x2="387" y2="54" stroke="#cdd0d2" stroke-width="2.4" marker-end="url(#d1-dim)"/>
<line x1="513" y1="54" x2="567" y2="54" stroke="#cdd0d2" stroke-width="2.4" marker-end="url(#d1-dim)"/>
<line x1="693" y1="54" x2="747" y2="54" stroke="#cdd0d2" stroke-width="2.4" marker-end="url(#d1-dim)"/>
<g><rect x="30" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="90" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Hypothesis</text></g>
<g><rect x="210" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="270" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Design</text></g>
<g opacity="0.85"><rect x="390" y="28" width="120" height="52" rx="10" ry="10" fill="#f2f3f3" stroke="#cdd0d2" stroke-width="1.5"/><text x="450" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#b4b7b9">Experiment</text></g>
<g opacity="0.85"><rect x="570" y="28" width="120" height="52" rx="10" ry="10" fill="#f2f3f3" stroke="#cdd0d2" stroke-width="1.5"/><text x="630" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#b4b7b9">Results</text></g>
<g opacity="0.85"><rect x="750" y="28" width="120" height="52" rx="10" ry="10" fill="#f2f3f3" stroke="#cdd0d2" stroke-width="1.5"/><text x="810" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#b4b7b9">Verdict</text></g>
</svg>


The first link in the chain sees the design of an experiment to be used to test the hypothesis. This design defines the experimental factor, controlled conditions, variable treatments, etc. Crucially, it also defines the verdict function, that is, the metrics, thresholds, and directional assertions which map the eventual output of the experiment to a verdict on the correctness of the hypothesis. (The decisions on what outputs are being measured, and how they are interpreted have to be defined up front; it can't be decided after results have been seen)

The validity of this link hinges on whether the experiment, as defined, will actually yield a confirmation / disconfirmation of the hypothesis. We can decompose that into two separate, more precise questions, roughly matching the distinction Cook & Campbell (1979) [5] drew between internal validity and construct validity:

1. Is it a well-formed experiment comparing different treatments of the same factor along a common metric, and without confounding conditions (i.e. the experiment is structured so as to rule out any explanations for a difference in the metric apart from the techniques themselves)?
2. Do the factor, treatments, and metrics operationalize the constructs named in the hypothesis (i.e. is the experiment relevant to the hypothesis)?

The first question can be mechanically checked to a certain extent. Prior works have built DSLs in which to specify experiment designs such that they can be statically analyzed for problems like treatment-assignment errors and gaps in causal sufficiency (PlanAlyzer [6]; PLanet [7]).

The latter question, however, exists in "reasoning space," and the responsibility for validation therefore falls to an agent.

## Implementation

<svg width="100%" viewBox="0 0 900 108" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="The inferential chain, with the implementation link highlighted" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif" style="display:block;margin:1.75rem 0;">
<defs>
<marker id="d2-gap" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#b5822f"/></marker>
<marker id="d2-dim" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#cdd0d2"/></marker>
</defs>
<line x1="153" y1="54" x2="207" y2="54" stroke="#cdd0d2" stroke-width="2.4" marker-end="url(#d2-dim)"/>
<line x1="333" y1="54" x2="387" y2="54" stroke="#b5822f" stroke-width="3.1" marker-end="url(#d2-gap)"/>
<line x1="513" y1="54" x2="567" y2="54" stroke="#cdd0d2" stroke-width="2.4" marker-end="url(#d2-dim)"/>
<line x1="693" y1="54" x2="747" y2="54" stroke="#cdd0d2" stroke-width="2.4" marker-end="url(#d2-dim)"/>
<g opacity="0.85"><rect x="30" y="28" width="120" height="52" rx="10" ry="10" fill="#f2f3f3" stroke="#cdd0d2" stroke-width="1.5"/><text x="90" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#b4b7b9">Hypothesis</text></g>
<g><rect x="210" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="270" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Design</text></g>
<g><rect x="390" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="450" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Experiment</text></g>
<g opacity="0.85"><rect x="570" y="28" width="120" height="52" rx="10" ry="10" fill="#f2f3f3" stroke="#cdd0d2" stroke-width="1.5"/><text x="630" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#b4b7b9">Results</text></g>
<g opacity="0.85"><rect x="750" y="28" width="120" height="52" rx="10" ry="10" fill="#f2f3f3" stroke="#cdd0d2" stroke-width="1.5"/><text x="810" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#b4b7b9">Verdict</text></g>
</svg>

Next, the experiment specified by the design is implemented. The validity criterion here is whether or not the implementation is faithful to the design.

As we did before, we'll consider the two separate interpretations of "faithful," roughly mapping onto the concept of verification vs validation in software engineering:

- **Verification** is concerned with the implementation's adherence to the explicit specifications of the design. That is, whether there is an implementation for each technique, whether each of these emits correctly labeled and shaped metrics, whether each plugs into a shared harness which purportedly implements the controlled conditions, etc.
- **Validation** asks whether the implementations are faithful to the methods which the design intends them to be.

In terms of verification, the formulation of the DSL determines the degree to which structural questions are explicitly answered in the design, and therefore the surface area for mechanical checkability. Typically though, even the lighter-weight DSLs surveyed can compile into an interface against which type definitions, data models, and AST-level parsing can be checked.

Validation, on the other hand, asks whether the implementations are faithful to the design's intent. Even generally speaking, in terms of SWE tasks, this validation is relegated to a code reviewer agent, since "intent" is very much a reasoning-space concept.

Our particular problem leans on agent reasoning for validating intent even more heavily than a typical engineering task does. Tests normally carry part of the load by checking code against known-correct outputs, but our implementations have no such output to check against. Behavior tests we can still write, but only for behaviors we specify in advance. Choosing what to specify is itself the interpretive work, and passing the tests is never the same as faithfulness to the method as a whole.

The check that would settle faithfulness, holding the implementation against the experiment's own expected results, we can never run. Were those results known, the experiment would be pointless, its verdict already in hand. Ours is, in Weyuker's sense, a "non-testable program" (Weyuker 1982) [8], written to reveal an answer unknowable beforehand. Validating its faithfulness is therefore solely the province of reasoning-space actors.

> **Footnote.** Since the experiment set-up (implementation) is, for our purposes, a matter of writing code, we appealed to Weyuker's argument about this tautological validation (i.e. the implementation is correct because it yields expected results, and the expected results are correct because they were yielded by a correct implementation) in the context of software engineering testability.
>
> However, this same "experimenter's regress" (Collins 1985) [9], in which an apparatus can be shown to work only by producing the correct result while the correct result is knowable only from an apparatus already shown to work, risks manifesting itself in experimental science more generally.

## Execution and verdict

<svg width="100%" viewBox="0 0 900 108" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="The inferential chain, with the execution and verdict links highlighted" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif" style="display:block;margin:1.75rem 0;">
<defs>
<marker id="d3-mech" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#3f7d6a"/></marker>
<marker id="d3-dim" markerWidth="8" markerHeight="7" refX="6.4" refY="3.5" orient="auto" markerUnits="userSpaceOnUse"><path d="M0,0 L7,3.5 L0,7 Z" fill="#cdd0d2"/></marker>
</defs>
<line x1="153" y1="54" x2="207" y2="54" stroke="#cdd0d2" stroke-width="2.4" marker-end="url(#d3-dim)"/>
<line x1="333" y1="54" x2="387" y2="54" stroke="#cdd0d2" stroke-width="2.4" marker-end="url(#d3-dim)"/>
<line x1="513" y1="54" x2="567" y2="54" stroke="#3f7d6a" stroke-width="3.1" marker-end="url(#d3-mech)"/>
<line x1="693" y1="54" x2="747" y2="54" stroke="#3f7d6a" stroke-width="3.1" marker-end="url(#d3-mech)"/>
<g opacity="0.85"><rect x="30" y="28" width="120" height="52" rx="10" ry="10" fill="#f2f3f3" stroke="#cdd0d2" stroke-width="1.5"/><text x="90" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#b4b7b9">Hypothesis</text></g>
<g opacity="0.85"><rect x="210" y="28" width="120" height="52" rx="10" ry="10" fill="#f2f3f3" stroke="#cdd0d2" stroke-width="1.5"/><text x="270" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#b4b7b9">Design</text></g>
<g><rect x="390" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="450" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Experiment</text></g>
<g><rect x="570" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="630" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Results</text></g>
<g><rect x="750" y="28" width="120" height="52" rx="10" ry="10" fill="#fafbfb" stroke="#54585c" stroke-width="1.5"/><text x="810" y="54" text-anchor="middle" dominant-baseline="central" font-size="13.5" font-weight="500" fill="#2b2b2b">Verdict</text></g>
</svg>

Finally, the implemented experiment is executed, and its output used to render a verdict. Execution is the one purely mechanical link in the chain, calling for no agent reasoning of its own. The final question, then, is whether the verdict is correct. There are two interpretations of correctness here:

1. Was the verdict correctly calculated using the proper metrics (this question largely overlaps with a third form of validity given by the aforementioned Cook & Campbell, statistical conclusion validity)?
2. Is the verdict truly confirming or disconfirming the hypothesis?

In regard to the first, this isn't so much mechanically verified as it is mechanically guaranteed: the design specifies the verdict calculation, and verification has already established that the implementation emits the metrics that calculation requires.

The second isn't really specific to this link, but rather concerns every link up to, and including, this final one. It gets into a sort of Quinean idea that we're rendering a verdict not only on the correctness of the hypothesis in isolation, but also on the aptness of our experiment design, and the faithfulness of our implementation.

> **Footnote.** Various questions throughout have mapped to three of Cook & Campbell's forms of validity: internal, construct, and statistical conclusion, but the framework from which these were drawn defined a fourth as well, external validity, the question of whether the experiment's findings are generalizable beyond the particular experiment's limited scope. Currently, we place this question in the wider research workflow, beyond the chain examined here. That is to say, it is the research agent's job to take this question into account when generating hypotheses. An argument could be made, though, that this question belongs to the chain itself, seeing as the chain is where experiments are designed, and follow-on experiments are what is required to satisfy external validity.

## The machinery

Taken link by link, the chain partitions cleanly. The design of an experiment can be checked for well-formedness, but whether it captures the constructs the hypothesis names is left to an agent's judgment. An implementation can be verified against the design's explicit specifications, yet its faithfulness to the method behind them again relies on an agent to judge. The verdict itself, once the design has fixed how it is computed, follows by construction, though it is sound only insofar as every link before it held.

That division between what is settled mechanically, and what is left to agents, is the verifiability surface this post set out to find. The surface is intrinsic to the chain, but it is latent. A mechanism can only check what has been made explicit, so how much of the surface a system reaches depends on the declarative language its experiments are written in. A language that forces more structure into the open exposes more of the surface to checking; a looser one leaves more to agents. All languages though, however well designed, will fall short of reaching the interpretive judgments this post has traced. That is irreducibly the domain of agents.

Early on we observed that many of the existing auto-research systems lacked machinery specific to scientific research. The companion post, [How SMAI works](/2026/05/13/how-yaarp-works.html), gives one example of how the concepts discussed in this post could be used to construct such machinery.

## References

1. [From AI for Science to Agentic Science: A Survey of Autonomous Scientific Discovery](https://arxiv.org/abs/2508.14111)
2. [A Survey of AI Scientists: Surveying the Architectural Landscape of Autonomous Research](https://arxiv.org/abs/2510.23045)
3. [Evaluating Sakana's AI Scientist for Autonomous Research: Wishful Thinking or an Emerging Reality?](https://arxiv.org/abs/2502.14297)
4. [The More You Automate, the Less You See: Hidden Pitfalls of AI Scientist Systems](https://arxiv.org/abs/2509.08713)
5. [Quasi-Experimentation: Design and Analysis Issues for Field Settings](https://www.guilford.com/excerpts/reichardt_introduction.pdf?t=1)
6. [PlanAlyzer: Assessing Threats to the Validity of Online Experiments](https://arxiv.org/abs/1909.13649)
7. [PLanet: Formalizing Assignment Procedures in the Design of Experiments](https://arxiv.org/abs/2505.09094)
8. [On Testing Non-testable Programs](https://ics.uci.edu/~dfredmil/ics221-FQ03/papers/Wey82.pdf)
9. [Changing Order: Replication and Induction in Scientific Practice](https://press.uchicago.edu/ucp/books/book/chicago/C/bo3623576.html)
