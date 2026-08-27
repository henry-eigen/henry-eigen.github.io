---
layout: post
title: "Distilled-Conditional Bayesian Networks"
date: 2026-08-19
mathjax: true
citation: true
citation_key: eigen2026disco
description: "LLMs supply conditional structure but make poor coherent generators. We distill their conditionals, node by node, into an offline Bayesian network that generates arbitrarily large coherent synthetic populations after a one-time distillation cost."
---

## Summary

In [the previous post](/2026/08/03/build-your-simulator.html), we demonstrated that much of the structure of an LLM's simulated responses for individuals can be captured by a linear, main-effects model, enabling sample-efficient distillation of LLM-estimated conditional probability distributions into lightweight, offline models. In this post, we show how this technique can be applied to build complex simulators of coherent, multi-attribute behaviors. While LLMs cannot reliably generate multi-attribute synthetic individuals in one shot, frequently suffering from mode collapse and failing to capture joint dependencies, they serve as effective suppliers of univariate conditional distributions. We propose the **Distilled-Conditional Bayesian Network (DisCo-BN)**, a framework for building offline simulators from LLM-provided priors by factorizing multi-attribute profiles into a Bayesian network, whose nodes we distill from the LLM elicited priors. After a one-time, upfront distillation cost, the resulting network can generate arbitrarily large volumes of coherent synthetic microdata entirely offline. Furthermore, because each node retains an explicit parametric form, the simulator can be calibrated to known ground-truth marginals for empirical grounding, or steered toward target marginals for counterfactual simulation.

---

## 1. Response Model vs Simulator

In recent years, LLMs have increasingly been explored as a viable source of behavioral priors where attributes of interest are either not jointly observed, or are unobserved altogether in microdata. The hope is that LLMs capture enough of the latent structure of human behavior and preferences that, when queried with sufficient context on an individual, the LLM can, to a widely debated extent, reason about the individual's beliefs and intentions, and correspondingly make predictions about the individual's behaviors. 

It is generally observed that they perform much more reliably as suppliers of conditional distributions than they do as suppliers of samples from a joint distribution. That is, they can effectively predict an individual's response to a single question given conditioning context on that individual. But when tasked with generating a sequence of responses for an individual, LLMs struggle with supplying coherence within sequences, as well as supplying variance between sequences.

This means that the LLM alone can be used as a general response model to independent questions. But it cannot, by itself, be used as a simulator of multiple responses for an individual, where the dependencies between responses are non-trivial or even the primary object of interest. For example, if we predict each response separately: 

$$
a_1 \sim P(\text{universal healthcare view} \mid \text{individual}),
\quad a_2 \sim P(\text{wealth inequality view} \mid \text{individual}),
\quad a_3 \sim P(\text{voting behavior} \mid \text{individual}), 
$$

we implicitly assume that an individual's views on wealth inequality and universal healthcare are completely independent of how they vote. Such an assumption undermines the utility of a simulator, both for generating realistic individual profiles and for conducting population-level analysis. In analysis, we want to understand how beliefs and behaviors cluster together, or how a shift in sentiment on one topic would affect support for another. To capture these relationships, we must be able to simulate sequences of attributes that are coherent over the joint conditional distribution:

$$(a_1, a_2, a_3) \sim P(\text{healthcare view}, \text{inequality view}, \text{voting} \mid \text{individual})$$

### The Simulator as a Bayesian Network

As we noted earlier, LLMs do a poor job of supplying samples from a joint distribution. This leaves us with a conundrum, since everything we know about these attributes lives in the LLM, yet the LLM cannot hand us the joint we need. We resolve this by using the chain rule of probability to factorize the desired joint conditional (which the LLM cannot supply) into a series of univariate conditionals (which it can supply). By conditioning each response not only on the individual, but on their previously generated answers as well, we can sample attributes autoregressively, meaning each realized answer shifts the probability distributions of subsequent answers.

$$
\begin{aligned}
a_1 \sim P(\text{universal healthcare} &\mid \text{individual}),
\newline a_2 \sim P(\text{wealth inequality} &\mid \text{individual}, \text{healthcare}), 
\newline a_3 \sim P(\text{voting behavior} &\mid \text{individual}, \text{healthcare}, \text{inequality})
\end{aligned}
$$

This factorization is the basis of a Bayesian network, where each attribute gets a node, defined as the conditional distribution of that attribute given the preceding attributes before it. Since this autoregressive generation approximates sampling from the joint conditional, we get coherent sequences of responses. And since we sample probabilistically at each node, we preserve the full variance of the population, and avoid the mode collapse faced when eliciting sequences directly from the LLM.

### Building a Bayesian Network out of Distilled Conditionals

Querying a live LLM sequentially for each generated individual would render large-scale simulation prohibitively expensive. This is a problem we partially addressed in the previous post, where we showed that we can distill an LLM's conditional distributions into offline log-linear models. In this post, we demonstrate how this same distillation approach can be used to construct the nodes of our Bayesian network. We query the LLM during an initial setup phase to estimate each node, and then run the simulation entirely offline. Having explicit parametric models at each node also gives us capabilities that an LLM alone cannot provide, including better calibration, increased interpretability, and the ability to construct counterfactuals which diverge from the LLM's world model.

<img src="/assets/images/distilled-conditional-bayesian-networks-bayesian-simulator.svg" width="720" style="display:block;margin:1.5rem auto;" alt="The DisCo-BN pipeline, distilling LLM-elicited conditional distributions into the nodes of a Bayesian network that then generates coherent synthetic populations offline">

---

## 2. Factorization and Generation

In order to simulate a coherent collection of attributes for an individual (an individual's response to a question being an example of an attribute), we factorize the joint distribution of attributes as a Bayesian network. That is, we define a directed graph where each node corresponds to a single attribute $$A_t$$, and directed edges indicate conditioning relationships between nodes. Writing $$\operatorname{Pa}(t)$$ for node $$t$$'s parent set (the nodes from which it has incoming edges), the network factorizes the joint conditional distribution given the initial seed attributes $$A_c$$ as:

$$P(A_1, A_2, \ldots, A_n \mid A_c) = \prod_{t=1}^n P(A_t \mid \operatorname{Pa}(t))$$

This graphical representation does require assigning a topological ordering to the nodes in the graph, and therefore to their corresponding attributes. Unlike text sequences or time-series data, our various demographic and behavioral attributes have no intrinsic order. It is therefore our responsibility to specify one. Fortunately, mathematically speaking, any ordering is an equally valid factorization under the chain rule. The ordering is really only a practical consideration then, since it determines which conditionals we elicit from the LLM, and in theory some orderings may yield prompts which elicit more stable responses from the LLM. We will consider this choice, as well as the choice of parent-set relations, to be implementation details beyond the scope of this post.

The generative nature of Bayesian networks is our primary motivation for using them. This generation is carried out by forward sampling, which is the process of traversing the graph and instantiating a value at each node by sampling its conditional distribution, using the previously instantiated values of its parents for conditioning. This process is initiated by specifying some seeded context values $$A_c$$ for the root nodes. At step $$t$$, we define the sampling state $$s_t$$ as the sequence of values instantiated so far during the traversal of nodes up to $$t$$. Generating a full sequence of attributes is a sequential rollout where, at each step:

$$
s_t = (a_c, a_1, \ldots, a_{t-1}) 
\quad \rightarrow \quad a_t \sim P(A_t \mid \operatorname{Pa}(t)=s_t)
\quad \rightarrow \quad s_{t+1} = (a_c, a_1, \ldots, a_{t-1}, a_t)
$$

<img src="/assets/images/distilled-conditional-bayesian-networks-inference-plain.svg" width="720" style="display:block;margin:1.5rem auto;" alt="Forward sampling through the network, each node drawing its value from its local conditional given the instantiated values of its parents">

Drawing each attribute sequentially from its local conditional produces a completed sequence that is mathematically equivalent to a direct draw from the joint conditional distribution

$$s_n \sim P(A_1, \dots, A_n \mid A_c)$$

---

## 3. Learning the Network

When modeling Bayesian networks, it is standard practice to express each node's conditional probability distribution (CPD) with a parameterized function, and to jointly fit all functions (one per node) using a shared corpus of fully observed data. We will adopt the former approach, defining a conditional probability function $$\pi_t$$ for each node $$t$$, such that

$$\pi_t(a_c, a_1, \ldots, a_{t-1}) \approx P(A_t \mid A_c=a_c, A_1=a_1, \ldots, A_{t-1}=a_{t-1})$$

As to the latter however, the non-existence of such fully observed data is the entire premise of this post's method. What is available instead to us is the LLM, which can provide estimates of conditional probabilities given specified conditioning attributes. We therefore treat parameter estimation as a knowledge distillation problem, using the LLM as a teacher model, rather than an empirical data-fitting problem.

Crucially, unlike standard estimation, where the same dataset of joint observations serves the entire network, our LLM queries produce targets specific to one conditional distribution at a time. Because we are providing the LLM values for a specific set of attributes as context, and requesting distributions for a specific attribute in response, data elicited to train one node cannot be recycled to then train another. Our distillation loop therefore involves separately eliciting data for, and fitting each node's function.

For each node $$t$$, we generate a batch of training inputs $$s_t^{(i)}$$, each comprising one value for each of the node's conditioning attributes. For each input, we then elicit from the LLM a probability distribution over possible values for the node's attribute, and treat this as the training target $$y^{(i)}$$. We then use these training inputs and targets to learn our function parameters $$\theta$$ by minimizing the forward KL divergence from the elicited distributions to our model's distributions:

$$
\theta
=
\arg\min_{\theta}
\frac{1}{N}\sum_{i=1}^{N}
D_{\mathrm{KL}}\!\left(
y^{(i)}
\,\Vert\,
\pi_t(s_t^{(i)} ; \theta)
\right)
$$

<img src="/assets/images/distilled-conditional-bayesian-networks-fitting-node.svg" width="720" style="display:block;margin:1.5rem auto;" alt="Fitting a single node, eliciting a target probability distribution from the LLM at each training state and minimizing forward KL divergence to the node's model">

### Considerations

At a high level, building our full Bayesian network appears to be a simple matter of repeating this distillation process for every node in the graph. In practice, however, translating this abstract single-node loss into an end-to-end generative simulator is not quite so straightforward. The objective above takes several critical ingredients for granted: it assumes we know which training inputs to sample, how to parameterize the function $$\pi_t$$, and how to trust the resulting numbers. In the sections that follow, we focus on three practical considerations that make this problem interesting:

* **1. Where do the training contexts come from?**  
  Because the LLM is an unconstrained oracle, we could theoretically prompt it with *any* arbitrary combination of conditioning attributes. But with a finite query budget and an imperfectly fitted model, we want to train $$\pi_t$$ on the specific parent configurations $$P(A_c, A_1, \ldots, A_{t-1})$$ the simulator will actually encounter during generation. This creates an apparent chicken-and-egg problem: we need samples from the joint distribution to train the conditional functions, but we are learning the conditional functions precisely to define that joint distribution. In **On-Policy Forward Training**, we show how building the network sequentially resolves this circularity.

* **2. What functional form should $$\pi_t$$ take?**  
  In theory, any expressive function approximator capable of fitting the training data could serve as a node. However, our end goal is not a black-box predictor, but an interactive simulator whose conditional relationships remain transparent, inspectable, and steerable. In **The Node as a Simulation Control Surface**, we discuss how to parameterize $$\pi_t$$ so that its fitted parameters double as intuitive simulation levers.

* **3. How do we ensure the simulation reflects reality?**  
  Our goal is not simply to replicate the LLM. While we rely on the model's conditional judgments as provisional *association structure* where observational records are missing, LLMs can be poorly calibrated on absolute base rates. In **Grounding the Levels**, we explore how to separate the LLM's relational patterns from real-world aggregate data, anchoring the simulator's population-level responses to measured evidence.

The targets themselves are straightforward to obtain: the LLM will return a probability distribution for any context we supply. The most immediate open question is how to generate those contexts in the first place.

---

## 4. Why we can't learn the nodes independently

The combinatorial state space $$\mathcal{A}_c \times \mathcal{A}_1 \times \cdots \times \mathcal{A}_{t-1}$$ grows exponentially with each additional attribute. The vast majority of this product space consists of nonsensical or vanishingly rare combinations. Sampling arbitrary combinations across this space exposes us to the classic off-policy covariate shift problem in imitation learning. When a student policy is trained on an off-policy distribution of states, rather than the distribution generated by its own sequential rollout, small estimation errors compound quadratically along the rollout trajectory.

Compounding this issue is the nature of the oracle itself. While an LLM will obligingly return a probability distribution for any prompt we hand it, we cannot expect grounded or coherent conditionals on nonsensical, off-manifold profiles that have no basis in human reality. Fitting our student models on these phantom states distorts the learned parameters in the realistic regions where the model actually needs to operate.

### Sequential bootstrapping

We therefore use on-policy sequential bootstrapping, which directly mirrors the Forward Training algorithm in imitation learning. To construct the network, we seed a training population by sampling context attributes $$A_c \sim P(A_c)$$. For each subsequent attribute node $$t$$, we treat the learned prefix $$(\pi_1, \dots, \pi_{t-1})$$ as an active, self-contained Bayesian network and use closed-loop forward sampling to generate our training states:

$$s_t^{(i)} \sim P_{\pi_{1:t-1}}(s_t)$$

We query the LLM oracle exclusively on these student-generated states, fit $$\pi_t$$ by minimizing the empirical forward KL, and sample $$a_t \sim \pi_t$$ to form the histories for node $$t+1$$. Every node is thus trained strictly on-policy with respect to the prefix that precedes it, completely eliminating rollout distribution mismatch.

<img src="/assets/images/distilled-conditional-bayesian-networks-training-loop-v2.svg" width="720" style="display:block;margin:1.5rem auto;" alt="The sequential bootstrapping loop, sampling states from the learned prefix, querying the LLM oracle, fitting the next node, and appending its samples">

Crucially, unlike standard imitation learning where an expert provides only a single discrete demonstration action (or scalar reward), our oracle returns the full categorical probability distribution over all response options. Minimizing forward KL divergence against this soft target forces the student to match the full variance, entropy, and relative probabilities of the teacher's conditional distribution, rather than collapsing to a single mode.

Once the full chain of nodes is distilled, this construction-time population is discarded; we can then seed fresh populations of any size from $$P(A_c)$$ and simulate complete profiles offline at zero marginal LLM cost.

---

## 5. Conditional probability parameterization

For each node $$t$$, we define a local policy $$\pi_t(s_t)$$: our offline model representing the conditional distribution over values of $$A_t$$ given the values of its parents in the state:

$$\pi_t(s_t) \approx P(A_t \mid \operatorname{Pa}(t))$$

We parameterize each policy with additive energies. Each response option $$c$$ of $$A_t$$ carries a baseline energy $$E_0(c)$$, and each level of each parent attribute contributes an energy $$E_c(a_j)$$. The total energy of an option given the state is their sum:

$$E(c) = E_0(c) + \sum_{j \in \operatorname{Pa}(t)} E_c(a_j)$$

A low total energy marks a combination of parent values compatible with option $$c$$, and a high total energy marks a mismatch. The node's response distribution normalizes the exponentiated negative energies over the options

$$\pi_t(A_t = c \mid s_t) = \frac{e^{-E(c)}}{\sum_{c'} e^{-E(c')}}$$

Learning these is equivalent to fitting a multinomial logistic regression. It maps arbitrary combinations of parent values to a normalized response distribution, can be fitted with a convex objective, and shares information across states rather than maintaining a separate table for every parent configuration. For each parent, one reference level's contribution is fixed at zero to make the remaining energies identifiable.

Each node normalizes its own distribution, so the energies are local to a node. The joint distribution arises from composing the nodes through forward sampling, and no single global energy over complete records is defined. An edge in the graph is a table of these contributions, one entry per parent level and response option. We display tables in score form (negated energies), so a higher entry favors its option. A sparser graph carries fewer tables.

<img src="/assets/images/distilled-conditional-bayesian-networks-node-table.svg" width="720" style="display:block;margin:1.5rem auto;" alt="An edge of the network as a table of score contributions, one entry per parent level and response option">

## Calibrating the levels

A well-documented failure mode of LLMs in social science is prior bias. As we showed in the previous post, the oracle often has the right slope and relative ordering, but its absolute baseline can be shifted. If a trusted marginal $$M_t$$ is known for $$A_t$$, after fitting $$\pi_t$$ we shift its baseline energies while holding every parent contribution fixed, until the mean predicted distribution across the training population matches that marginal. Writing $$E_0^\star$$ for the shifted baselines, calibration solves

$$\frac{1}{N}\sum_{i=1}^{N} \pi_t\big(\cdot \mid s_t^{(i)};\, E_0^\star\big) = M_t$$

This is the same intercept tuning we solved in the previous post, now averaged over the on-policy training population rather than an external frame. Like the distillation objective, the problem is convex, with a unique solution up to the reference-level normalization. The shift changes the node's overall level while preserving the learned between-state contrasts. The realized shares after hard sampling then differ only by ordinary sampling noise. We make this adjustment before sampling $$A_t$$ and appending it to the state. Because each sampled answer becomes part of the history for later nodes, calibrating the level at this point prevents a baseline error from propagating through the rest of the chain. If no trusted marginal is available, the fitted policy enters the prefix unchanged.

Nothing in the operation requires the target marginal $$M_t$$ to be a measured one. Setting a node's level to a hypothetical value and resampling the attributes downstream shows what the fitted network implies for the rest of the chain, at no additional distillation cost. In this post we use the lever only for calibration against trusted statistics.

## Query cost and model size

For a node with $$K_t$$ possible values, its parameter count grows linearly with the encoded size of its parent set:

$$\text{Parameters}(\pi_t) = (K_t - 1)\left[1 + d_c + \sum_{j \in \operatorname{Pa}(t)}(K_j - 1)\right],$$

where $$d_c$$ is the encoded dimension of the context attributes. Across the full network, parameter growth follows the number of edges: quadratic in the worst-case complete DAG, but linear for a sparse graph with bounded in-degree. Let $$p = 1 + d_c + \sum_{j \in \operatorname{Pa}(t)}(K_j - 1)$$ be the conditional's design dimension. Because each oracle query returns the complete response distribution, one query supplies $$K_t - 1$$ independent soft targets. In our experiments, projection loss typically stabilized once the number of queried states reached a small multiple of $$p$$, roughly $$2p$$. This is an empirical planning heuristic rather than a universal guarantee.

## Conclusion

A queryable LLM gives us conditionals, but population simulation requires a reusable joint. A DisCo-BN builds that joint one factor at a time, using an on-policy forward training approach to distill the oracle LLM. When a trusted marginal is available, we can shift the new policy's baseline energies before it enters the prefix. After the final node is learned, the network can simulate arbitrarily large populations of coherent people through ordinary forward sampling, with no further LLM calls.
