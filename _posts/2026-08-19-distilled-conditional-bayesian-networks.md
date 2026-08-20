---
layout: post
title: "Distilled-Conditional Bayesian Networks"
date: 2026-08-19
mathjax: true
citation: true
citation_key: eigen2026disco
description: "LLMs supply conditional structure but make poor coherent generators. We distill their conditionals, node by node, into an offline Bayesian network that generates arbitrarily large coherent synthetic populations after a one-time distillation cost."
---

In recent years, LLMs have been explored as a promising augmentation to, or outright stand-in for, human-sourced microdata. The hope is that LLMs capture enough of the latent structure of human behavior and preferences such that, when provided sufficient context on an individual, the LLM can, in turn, provide a reasonable estimate of how the individual might respond in a specified situation.

In [the previous post](/2026/08/03/build-your-simulator.html), we evaluated the effectiveness of distilling and calibrating an LLM's estimate of the conditional distribution, for a single variable, into an offline, linear model. Doing so allowed us to generate representative samples of respondents with no LLM inference beyond the initial queries needed for distillation.

In general, we are interested in studying clusters of related attributes, and the relationships between them. In order to generate synthetic samples which accurately model how, for instance, beliefs can mutually inform opinions on multiple related issues, or how preferences drive behavior, we need to be able to generate samples which are coherent across multiple attributes. Doing so requires our model to support sampling from the joint distribution of our modeled attributes.

### Population Generation

We could try eliciting the joint conditional probabilities from the LLM the way we did for conditionals. But joint conditional distributions are Cartesian products of the attribute levels, so parameter count grows exponentially as we model more attributes. We should also not attempt a sample & count approach, where we elicit samples from the LLM. It has been widely observed that LLMs perform poorly as coherent generators, as they are vulnerable to mode collapse.

In this post, we introduce a method for constructing an offline **Distilled-Conditional Bayesian Network (DisCo-BN)** from a queryable LLM. The LLM supplies otherwise unavailable conditional structure, while trusted aggregate marginals, when available, can correct the baseline levels of individual nodes. The resulting network can generate arbitrarily large amounts of coherent synthetic microdata after a one-time, upfront LLM distillation cost.


## From conditionals to a joint

The chain rule of probability allows us to factorize our desired joint conditional distributions into a series of univariate conditional distributions:

$$P(A_1, A_2, \dots, A_n \mid A_c) = \prod_{i=1}^n P(A_i \mid A_1, \dots, A_{i-1}, A_c)$$

We can represent this decomposition as a Bayesian network, which is just to say a graph where each node corresponds to an attribute conditioned on the incoming nodes. For simplicity, in this post, we assume a complete graph, where every node conditions on all preceding nodes. While the method naturally generalizes to sparse graphs as well, fixing an arbitrary attribute ordering is sufficient for the purposes of demonstration. What is important is that the lack of conditional independence assumptions means our network is autoregressive.

Given some initial conditioning values $$A_c$$, we can perform forward sampling through the graph to generate full sequences of values. This procedure involves traversing the nodes in order, and instantiating a value at each node along the way by sampling the conditional probability distribution for that node's attribute conditioned on previously instantiated values. A full traversal through the graph then accumulates a value for each attribute.

At step t, we define the sampling state $$s_t$$ as the sequence of values instantiated so far, beginning with the initial context

$$s_t = (a_c, a_1, a_2, \ldots, a_{t-1})$$

Generating a synthetic record is therefore a sequential rollout where each step $$t$$ samples $$a_t \sim P(A_t \mid s_t)$$ and transitions to $$s_{t+1} = (s_t, a_t)$$

By the chain rule, drawing each attribute sequentially from its local conditional produces a completed sequence that is mathematically equivalent to a direct draw from the joint conditional distribution

$$(a_1, \dots, a_n) \sim P(A_1, \dots, A_n \mid A_c)$$

## Conditional distillation

We want to learn a parameterization of these conditionals. For each node $$t$$, we define a local policy $$\pi_t(s_t)$$: our offline model representing the conditional distribution over values of $$A_t$$ given the state:

$$\pi_t(s_t) \approx P(A_t \mid A_c = a_c, A_1 = a_1, \ldots, A_{t-1} = a_{t-1})$$

We treat the LLM as an oracle $$\pi_{LLM}(\cdot \mid A_t, s_t)$$. Unlike our local student policies, which are each dedicated to a single attribute, the oracle is an all-purpose queryable interface that returns a probability distribution over the categories of any attribute given any state prompt.

To fit each node $$\pi_t$$, we collect a training set of $$N$$ states $$s_t^{(n)}$$, query the oracle for the target response distribution at each state, and minimize the empirical forward KL divergence:

$$\pi_t = \arg\min_\pi \frac{1}{N} \sum_{i=1}^N KL\big(\pi_{LLM}(s_t^{(i)}) \,\|\, \pi(s_t^{(i)})\big)$$

Minimizing forward KL against soft target distributions is equivalent to minimizing soft-label cross-entropy (since the teacher's entropy is constant with respect to the student's parameters). The DisCo-BN framework does not prescribe the student's architecture. Later in this post, we specify the additive multinomial-logit model we use.

In our previous post, we only distilled a single conditional distribution $$P(A_1 \mid A_c)$$. Because the context attributes $$A_c$$ came directly from observed external microdata, it was trivial to generate representative query states.

Once we move to a multi-node network, however, each subsequent conditional $$P(A_t \mid s_t)$$ must condition not just on $$A_c$$, but on the accumulated history of generated attributes $$(A_1, \dots, A_{t-1})$$. This raises an immediate question: where do the training states for downstream nodes come from?

## Why we can't learn the nodes independently

Typically, training autoregressive generative models is a matter of fitting unknown parameters to known full samples (microdata rows). We have the opposite aim, using independently fit conditionals (distilled from the LLM) to learn a generator for microdata samples.

In order to fit some $$\pi_t$$, we require sequences of values up to $$s_t$$, which we use to query the oracle to fit $$KL(\pi_{LLM} \,\|\, \pi_t)$$. We can't rely on the teacher model (the LLM) to supply sample sequences. After all, one of the fundamental motivations for our method is that the oracle struggles with generating trajectories. Nor should we create randomly generated states.

The combinatorial state space $$\mathcal{A}_c \times \mathcal{A}_1 \times \cdots \times \mathcal{A}_{t-1}$$ grows exponentially with each additional attribute. The vast majority of this product space consists of nonsensical or vanishingly rare combinations.

Sampling arbitrary combinations across this space exposes us to the classic off-policy covariate shift problem in imitation learning. When a student policy is trained on an off-policy distribution of states, rather than the distribution generated by its own sequential rollout, small estimation errors compound quadratically along the rollout trajectory.

Compounding this issue is the nature of the oracle itself. While an LLM will obligingly return a probability distribution for any prompt we hand it, we cannot expect grounded or coherent conditionals on nonsensical, off-manifold profiles that have no basis in human reality. Fitting our student models on these phantom states distorts the learned parameters in the realistic regions where the model actually needs to operate.

## The resolution: sequential bootstrapping

We therefore use on-policy sequential bootstrapping, which directly mirrors the Forward Training algorithm in imitation learning. For each conditional node $$t$$, we only need the distribution over histories generated by the preceding nodes. We can treat the learned prefix $$(\pi_1, \dots, \pi_{t-1})$$ as an active, self-contained Bayesian network and use closed-loop ancestral sampling from it to generate our training states:

$$s_t^{(i)} \sim P_{\pi_{1:t-1}}(s_t)$$

We then query the LLM oracle exclusively on these student-generated states. As a result, every node is trained strictly on-policy with respect to the prefix that precedes it, eliminating rollout distribution mismatch.

Crucially, unlike standard imitation learning where an expert provides only a single discrete demonstration action (or scalar reward), our oracle returns the full categorical probability distribution over all response options. Minimizing forward KL divergence against this soft target forces the student to match the full variance, entropy, and relative probabilities of the teacher's conditional distribution, rather than collapsing to a single mode.

## The training loop

For each attribute we want to model, we assign an ordering: $$A_1, \ldots, A_n$$.

We seed a training population by sampling context attributes $$A_c$$ from some external $$P(A_c)$$. Initially, each state contains only these context attributes.

Then, at each time t:

1. Elicit a training distribution from $$\pi_{LLM}$$ for each current state
2. Fit $$\pi_t = \arg\min_\pi \frac{1}{N}\sum_n KL(\pi_{LLM} \| \pi)$$
3. If a trusted marginal for $$A_t$$ is available, tune the policy's intercepts to match it
4. For every state, sample a value of $$A_t$$ from $$\pi_t$$ and append it to that state

The updated states become the training population for the next node, so the same population is carried forward throughout distillation. This is only the construction-time population. Once every node has been learned, we can seed fresh populations of any size from $$P(A_c)$$ and generate complete rows through the same ancestral-sampling process.

This is the general form of our proposed method. To implement it, we now describe the architecture used for the distilled policies and the optional intercept-tuning step.

## Parameterizing the conditionals

The DisCo-BN procedure is agnostic to the class of model used for each local policy, so long as the model can represent a normalized categorical conditional distribution. For practical purposes, there are reasons to prefer simpler, sample-efficient models (remember that each training sample costs one oracle query).

For each node, we use an additive multinomial-logit conditional. This is a natural fit for categorical soft targets, as it maps arbitrary combinations of parent values to a normalized response distribution, can be fitted with a convex objective, and shares information across states rather than maintaining a separate table for every parent configuration.

$$\log \left( \frac{\pi_t(A_t = c \mid s_t)}{\pi_t(A_t = c_0 \mid s_t)} \right) = \alpha_{t,c} + \sum_{j} \sum_{v \in \text{vals}(A_j) \setminus \{v_{j,0}\}} \beta_{j,v,c}^{(t)} \cdot \mathbf{1}[A_j = v], \qquad c \neq c_0$$

where $$j$$ ranges over all modeled attributes in the state, in addition to the context attributes $$A_c, A_1, \ldots, A_{t-1}$$.

This formulation enforces an additive main-effects structure where each non-reference value of each earlier attribute contributes an explicit log-odds score, relative to a baseline response category $$c_0$$, for the current attribute's response options. For each predictor $$A_j$$, one reference value $$v_{j,0}$$ is omitted to make the coefficients identifiable.

For a node with $$K_t$$ possible values, its parameter count grows linearly with the encoded size of its parent set:

$$\text{Parameters}(\pi_t) = (K_t - 1)\left[1 + d_c + \sum_{j \in \operatorname{Pa}(t)}(K_j - 1)\right],$$

where $$d_c$$ is the encoded dimension of the context attributes. Across the full network, parameter growth follows the number of edges: quadratic in the worst-case complete DAG, but linear for a sparse graph with bounded in-degree.

Let $$p = 1 + d_c + \sum_{j \in \operatorname{Pa}(t)}(K_j - 1)$$ be the conditional's design dimension. Because each oracle query returns the complete response distribution, one query supplies $$K_t - 1$$ independent soft targets. In our experiments, projection loss typically stabilized once the number of queried states reached a small multiple of $$p$$, roughly $$2p$$. This is an empirical planning heuristic rather than a universal guarantee.

## Calibrating the levels

A well-documented failure mode of LLMs in social science is prior bias. As we showed in the previous post, the oracle often has the right slope and relative ordering, but its absolute baseline can be shifted. If a trusted marginal is known for $$A_t$$, after fitting $$\pi_t$$ we tune its option intercepts while holding the other coefficients fixed, until its mean predicted distribution across the training population matches that marginal. This changes the node's overall level while preserving the learned between-state log-odds contrasts. The realized shares after hard sampling then differ only by ordinary sampling noise.

We make this adjustment before sampling $$A_t$$ and appending it to the state. Because each sampled answer becomes part of the history for later nodes, calibrating the level at this point prevents a baseline error from propagating through the rest of the chain. If no trusted marginal is available, the fitted policy enters the prefix unchanged.

## Conclusion

A queryable LLM gives us conditionals, but population generation requires a reusable joint. A DisCo-BN builds that joint one factor at a time, using an on-policy forward training approach to distill the oracle LLM. When a trusted marginal is available, we can tune the new policy's intercepts before it enters the prefix.

After the final node is learned, the network can generate arbitrarily large populations of coherent synthetic rows through ordinary ancestral sampling, with no further LLM calls.
