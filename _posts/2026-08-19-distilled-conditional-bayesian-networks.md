---
layout: post
title: "Distilled-Conditional Bayesian Networks"
date: 2026-08-19
mathjax: true
citation: true
citation_key: eigen2026disco
description: "LLMs supply conditional structure but make poor coherent generators. We distill their conditionals, node by node, into an offline Bayesian network that generates arbitrarily large coherent synthetic populations after a one-time distillation cost."
---

In recent years, LLMs have increasingly been explored as a viable source of behavioral priors where attributes of interest are either not jointly observed, or are unobserved altogether in microdata. The hope is that LLMs capture enough of the latent structure of human behavior and preferences that, when queried with sufficient context on an individual, the LLM can, to a widely debated extent, reason about the individual's beliefs and intentions, and correspondingly make predictions about the individual's behaviors. 

It's generally been observed that they perform much more reliably as conditionals than as joint distributions. That is, they perform strongest when asked to simulate the responses for a single specified question, given conditioning context. And correspondingly, they struggle greatly with generating sequences of responses, both in terms of coherence within the response and variance across responses. In order to create rich synthetic populations, we need a way to assemble those conditionals to produce a joint model, without which we have no choice but to treat any simulated data as mutually independent. This reduces, in a sense, the LM to being a response model rather than a supplier of the structure needed to build synthetic populations. 

### Distilled-Conditional Bayesian Network

In this post, we introduce a method for constructing an offline Distilled-Conditional Bayesian Network (DisCo-BN) by distilling the LLM-supplied conditional structure into a series of log-linear models. The resulting network can generate arbitrarily large amounts of coherent synthetic microdata after a one-time, upfront LLM distillation cost. Our distilled model allows us to generate individuals whose attributes are coherent across all LLM-provided conditionals. It can also be calibrated to known marginals in order to ensure our generated populations are representative.


## From conditionals to a joint

How do we obtain a joint that was never jointly observed? We may attempt to do so the same way we got our conditionals via the LLM. Just as we elicited a verbalized univariate conditional from the LLM, we might try to elicit a verbalized joint conditional. Because joint conditional distributions are Cartesian products of the attribute levels, though, this quickly becomes infeasible as parameter counts grow exponentially with more attributes. To get around this, we can use the chain rule of probability to factorize a joint conditional distribution into a series of univariate conditional distributions, avoiding the combinatorial explosion 

$$P(A_1, A_2, \dots, A_n \mid A_c) = \prod_{i=1}^n P(A_i \mid A_1, \dots, A_{i-1}, A_c)$$

We can represent this decomposition as a Bayesian network, which is just to say a directed graph where each node corresponds to an attribute conditioned on the nodes with edges into it. We fix an ordering of the attributes, and each node $$t$$ keeps as its parent set $$\operatorname{Pa}(t)$$ whichever earlier attributes, along with the context $$A_c$$, it conditions on. Each factor then reduces to $$P(A_t \mid \operatorname{Pa}(t))$$, and the network's joint distribution is the product of these local conditionals. Omitting an edge asserts a conditional independence between two attributes.

Given some initial conditioning values $$A_c$$, we can perform forward sampling through the graph to generate full sequences of values. This procedure involves traversing the nodes in order, and instantiating a value at each node along the way by sampling the conditional probability distribution for that node's attribute given the instantiated values of its parents. A full traversal through the graph then accumulates a value for each attribute.

At step t, we define the sampling state $$s_t$$ as the sequence of values instantiated so far, beginning with the initial context. Generating a synthetic record is therefore a sequential rollout where, at each step $$t$$

$$
s_t = (a_c, a_1, a_2, \ldots, a_{t-1}) 
\quad \rightarrow \quad a_t \sim P(A_t \mid \operatorname{Pa}(t)) 
\quad \rightarrow \quad s_{t+1} = (s_t, a_t)
$$

<img src="/assets/images/distilled-conditional-bayesian-networks-inference-plain.svg" width="720" style="display:block;margin:1.5rem auto;" alt="Forward sampling through the network, each node drawing its value from its local conditional given the instantiated values of its parents">

Drawing each attribute sequentially from its local conditional produces a completed sequence that is mathematically equivalent to a direct draw from the network's joint conditional distribution

$$(a_1, \dots, a_n) \sim P(A_1, \dots, A_n \mid A_c)$$

## Conditional probability parameterization

For each node $$t$$, we define a local policy $$\pi_t(s_t)$$: our offline model representing the conditional distribution over values of $$A_t$$ given the values of its parents in the state:

$$\pi_t(s_t) \approx P(A_t \mid \operatorname{Pa}(t))$$

We parameterize each policy with additive energies. Each response option $$c$$ of $$A_t$$ carries a baseline energy $$E_0(c)$$, and each level of each parent attribute contributes an energy $$E_c(a_j)$$. The total energy of an option given the state is their sum:

$$E(c) = E_0(c) + \sum_{j \in \operatorname{Pa}(t)} E_c(a_j)$$

A low total energy marks a combination of parent values compatible with option $$c$$, and a high total energy marks a mismatch. The node's response distribution normalizes the exponentiated negative energies over the options

$$\pi_t(A_t = c \mid s_t) = \frac{e^{-E(c)}}{\sum_{c'} e^{-E(c')}}$$

Learning these is equivalent to fitting a multinomial logistic regression. It maps arbitrary combinations of parent values to a normalized response distribution, can be fitted with a convex objective, and shares information across states rather than maintaining a separate table for every parent configuration. For each parent, one reference level's contribution is fixed at zero to make the remaining energies identifiable.

Each node normalizes its own distribution, so the energies are local to a node. The joint distribution arises from composing the nodes through forward sampling, and no single global energy over complete records is defined. An edge in the graph is a table of these contributions, one entry per parent level and response option. We display tables in score form (negated energies), so a higher entry favors its option. A sparser graph carries fewer tables.

<img src="/assets/images/distilled-conditional-bayesian-networks-node-table.svg" width="720" style="display:block;margin:1.5rem auto;" alt="An edge of the network as a table of score contributions, one entry per parent level and response option">

## Conditional distillation

We treat the LLM as an oracle $$\pi_{LLM}(\cdot \mid A_t, s_t)$$. Unlike our local student policies, which are each dedicated to a single attribute, the oracle is an all-purpose queryable interface that returns a probability distribution over the categories of any attribute given any state prompt. To fit each node $$\pi_t$$, we collect a training set of $$N$$ states $$s_t^{(n)}$$, query the oracle for the target response distribution at each state, and minimize the empirical forward KL divergence:

$$\pi_t = \arg\min_\pi \frac{1}{N} \sum_{i=1}^N KL\big(\pi_{LLM}(s_t^{(i)}) \,\|\, \pi(s_t^{(i)})\big)$$

Minimizing forward KL against soft target distributions is equivalent to minimizing soft-label cross-entropy (since the teacher's entropy is constant with respect to the student's parameters). This objective is convex in the node's energies, so each fit reaches a global optimum with no sensitivity to initialization. When we distilled the single conditional $$P(A_1 \mid A_c)$$, the context attributes $$A_c$$ came from a microdata-sourced joint, so it was trivial to generate representative query states.

Once we move to a multi-node network, however, each subsequent conditional $$P(A_t \mid \operatorname{Pa}(t))$$ must condition not just on $$A_c$$, but on parent attributes that are themselves generated. This raises an immediate question: where do the training states for downstream nodes come from?

## Why we can't learn the nodes independently

Typically, training a sequential generative model is a matter of fitting unknown parameters to known full samples (microdata rows). We have the opposite aim, using independently fit conditionals (distilled from the LLM) to learn a generator for samples that never existed. In order to fit some $$\pi_t$$, we require sequences of values up to $$s_t$$, which we use to query the oracle to fit $$KL(\pi_{LLM} \,\|\, \pi_t)$$. We can't rely on the teacher model (the LLM) to supply sample sequences. After all, one of the fundamental motivations for our method is that the oracle struggles with generating trajectories. Nor should we create randomly generated states.

The combinatorial state space $$\mathcal{A}_c \times \mathcal{A}_1 \times \cdots \times \mathcal{A}_{t-1}$$ grows exponentially with each additional attribute. The vast majority of this product space consists of nonsensical or vanishingly rare combinations. Sampling arbitrary combinations across this space exposes us to the classic off-policy covariate shift problem in imitation learning. When a student policy is trained on an off-policy distribution of states, rather than the distribution generated by its own sequential rollout, small estimation errors compound quadratically along the rollout trajectory.

Compounding this issue is the nature of the oracle itself. While an LLM will obligingly return a probability distribution for any prompt we hand it, we cannot expect grounded or coherent conditionals on nonsensical, off-manifold profiles that have no basis in human reality. Fitting our student models on these phantom states distorts the learned parameters in the realistic regions where the model actually needs to operate.

## Sequential bootstrapping

We therefore use on-policy sequential bootstrapping, which directly mirrors the Forward Training algorithm in imitation learning. To construct the network, we seed a training population by sampling context attributes $$A_c \sim P(A_c)$$. For each subsequent attribute node $$t$$, we treat the learned prefix $$(\pi_1, \dots, \pi_{t-1})$$ as an active, self-contained Bayesian network and use closed-loop forward sampling to generate our training states:

$$s_t^{(i)} \sim P_{\pi_{1:t-1}}(s_t)$$

We query the LLM oracle exclusively on these student-generated states, fit $$\pi_t$$ by minimizing the empirical forward KL, and sample $$a_t \sim \pi_t$$ to form the histories for node $$t+1$$. Every node is thus trained strictly on-policy with respect to the prefix that precedes it, completely eliminating rollout distribution mismatch.

<img src="/assets/images/distilled-conditional-bayesian-networks-training-loop.svg" width="720" style="display:block;margin:1.5rem auto;" alt="The sequential bootstrapping loop, sampling states from the learned prefix, querying the LLM oracle, fitting the next node, and appending its samples">

Crucially, unlike standard imitation learning where an expert provides only a single discrete demonstration action (or scalar reward), our oracle returns the full categorical probability distribution over all response options. Minimizing forward KL divergence against this soft target forces the student to match the full variance, entropy, and relative probabilities of the teacher's conditional distribution, rather than collapsing to a single mode.

Once the full chain of nodes is distilled, this construction-time population is discarded; we can then seed fresh populations of any size from $$P(A_c)$$ and simulate complete profiles offline at zero marginal LLM cost.

## Calibrating the levels

A well-documented failure mode of LLMs in social science is prior bias. As we showed in [the previous post](/2026/08/03/build-your-simulator.html), the oracle often has the right slope and relative ordering, but its absolute baseline can be shifted. If a trusted marginal $$M_t$$ is known for $$A_t$$, after fitting $$\pi_t$$ we shift its baseline energies while holding every parent contribution fixed, until the mean predicted distribution across the training population matches that marginal. Writing $$E_0^\star$$ for the shifted baselines, calibration solves

$$\frac{1}{N}\sum_{i=1}^{N} \pi_t\big(\cdot \mid s_t^{(i)};\, E_0^\star\big) = M_t$$

This is the same intercept tuning we solved in the previous post, now averaged over the on-policy training population rather than an external frame. Like the distillation objective, the problem is convex, with a unique solution up to the reference-level normalization. The shift changes the node's overall level while preserving the learned between-state contrasts. The realized shares after hard sampling then differ only by ordinary sampling noise. We make this adjustment before sampling $$A_t$$ and appending it to the state. Because each sampled answer becomes part of the history for later nodes, calibrating the level at this point prevents a baseline error from propagating through the rest of the chain. If no trusted marginal is available, the fitted policy enters the prefix unchanged.

Nothing in the operation requires the target marginal $$M_t$$ to be a measured one. Setting a node's level to a hypothetical value and resampling the attributes downstream shows what the fitted network implies for the rest of the chain, at no additional distillation cost. In this post we use the lever only for calibration against trusted statistics.

## Query cost and model size

For a node with $$K_t$$ possible values, its parameter count grows linearly with the encoded size of its parent set:

$$\text{Parameters}(\pi_t) = (K_t - 1)\left[1 + d_c + \sum_{j \in \operatorname{Pa}(t)}(K_j - 1)\right],$$

where $$d_c$$ is the encoded dimension of the context attributes. Across the full network, parameter growth follows the number of edges: quadratic in the worst-case complete DAG, but linear for a sparse graph with bounded in-degree. Let $$p = 1 + d_c + \sum_{j \in \operatorname{Pa}(t)}(K_j - 1)$$ be the conditional's design dimension. Because each oracle query returns the complete response distribution, one query supplies $$K_t - 1$$ independent soft targets. In our experiments, projection loss typically stabilized once the number of queried states reached a small multiple of $$p$$, roughly $$2p$$. This is an empirical planning heuristic rather than a universal guarantee.

## Conclusion

A queryable LLM gives us conditionals, but population simulation requires a reusable joint. A DisCo-BN builds that joint one factor at a time, using an on-policy forward training approach to distill the oracle LLM. When a trusted marginal is available, we can shift the new policy's baseline energies before it enters the prefix. After the final node is learned, the network can simulate arbitrarily large populations of coherent people through ordinary forward sampling, with no further LLM calls.
