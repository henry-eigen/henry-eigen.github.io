---
layout: post
title: "Use the LLM to Build Your Simulator, Not to Be Your Simulator"
date: 2026-08-03
mathjax: true
citation: true
citation_key: eigen2026simulator
description: "LLM-based population simulation usually means sampling personas at inference time. We distill the LLM's response surface into a small log-linear model once, calibrate its absolute rates against observed toplines, and answer every later subgroup question offline."
---

In recent years, LLMs have been explored as a promising augmentation to, or outright stand-in for, human-provided survey data. The hope is that they learn enough of the latent structure of human behavior and preferences to simulate the responses of given profiles, which can then be aggregated to model the behavior of a group.

Such approaches have several drawbacks. Simulating a group requires sampling many profiles, and every new group analyzed demands a distinct sample, each adding to the inference cost. Additionally, while LLMs demonstrate a rich relative geometry of human behavior, they struggle to quantify absolute probabilities.

In this post, we factor group modeling into population composition and a behavioral response field, distill the LLM's response surface into a reusable log-linear model, and calibrate its probability estimates with minimal ground-truth data. This makes LLM inference a one-time, up-front cost, subsequently allowing us to inspect arbitrary slices of the population offline.

Separating these tasks lets us use the LLM for its most reusable contribution, the structure of how behavior varies between profiles. Observed outcomes anchor its absolute probabilities, while the population model supplies the composition of each target group.

## 1. What Is Population Simulation?

We are interested in answering questions like *How will retired men in Texas vote?* or *What will unmarried women aged 21&ndash;35 buy?*

To do so, we decompose the modeling of a target subpopulation $$\mathcal S$$ into two underlying questions:

1. How do people with attributes $$x$$ respond? For a mutually exclusive response option $$r$$, define the cell-level response field as $$B_r(x) := P(Y{=}r \mid X{=}x)$$.
2. Who makes up the subpopulation? Let $$\mathcal S$$ denote the set of population cells matching the subgroup definition, and let $$P_{\mathcal S}(x) := P(X{=}x \mid X \in \mathcal S)$$ denote their distribution within it.

Combining the two, the expected share of a subpopulation choosing response $$r$$ is the population-weighted average of its cell-level response rates:

$$\mu_r(\mathcal S) := \sum_x B_r(x)\,P_{\mathcal S}(x) = \mathbb E_{X \sim P_{\mathcal S}}[B_r(X)].$$

## 2. What's So Hard About That?

In practice, both $$B(x)$$ and $$P_{\mathcal S}(x)$$ are difficult to obtain. Behavioral preferences vary sharply across demographic profiles, while the attributes which define those profiles are rarely observed jointly and completely.

One shortcut is to ask the LLM for $$\mu(\mathcal S)$$ directly. The model must then infer cell-level behavior, reconstruct the group's composition, and marginalize correctly over that composition in a single pass. Its aggregate answer gives us no way to inspect which of those operations succeeded.

Persona sampling makes the behavioral step explicit. We generate individual profiles, elicit a response from each, and average the results. But variety is not representativeness. The generated profiles must follow the group's actual composition $$P_{\mathcal S}$$, every weighted estimate still needs a sufficiently large effective sample, and every new subpopulation requires a new round of elicitation.

<img src="/assets/images/build-your-simulator-f1-two-routes.png" width="720" style="display:block;margin:1.5rem auto;" alt="Two routes to the expected response shares: sampling the model's aggregate answer versus composing the response model with the population model">

Both routes target the same estimate. The direct route hides the subgroup's composition and the marginalization inside one model response, while the composed route obtains $$B(x)$$ and $$P_{\mathcal S}(x)$$ separately and takes their weighted sum outside the model.

To make the composed route reusable, we need to query the LLM once and turn its behavioral responses into a model we can evaluate offline.

## 3. Distilling a Behavior Model

For a given question, we want a behavioral model mapping an individual's attributes $$x = (a_1, a_2, a_3, \dots)$$ to a probability distribution over mutually exclusive response choices, where $$a_j$$ is the individual's level of attribute $$j$$. For this post, we choose a simple, interpretable structure which is standard for modeling how individuals decide among discrete options.

### 3.1 Additive effect scores

We treat person-level attributes as categorical, or discretize them into categorical levels. For each response $$r$$, we assign a baseline score $$b_r$$ and a contribution $$w_r(a_j)$$ from the persona's level of each attribute $$j$$. We then approximate the persona's total score for response $$r$$ as their sum:

$$s_r(x) = b_r + \sum_j w_r(a_j).$$

This main-effects approximation replaces a separate parameter for every possible persona with a finite collection of attribute-level effects. Section 4 tests what we lose by omitting interactions.

The scores are not themselves probabilities. Normalizing their exponentials over the response options gives the response field:

$$B_r(x) = \frac{\exp s_r(x)}{\sum_{r'} \exp s_{r'}(x)}.$$

We use $$B(x)$$ for the full distribution over responses, with $$B_r(x)$$ denoting the probability assigned to response $$r$$.

### 3.2 Effects table and multi-hot encoding

Because the attributes are categorical, there is a finite number of levels across all attributes, which we call $$D$$. We flatten those levels into a single list and write the multi-hot encoding of persona $$x$$ as $$\mathbf{x} \in \{0,1\}^D$$, with one active entry for the persona's level of each attribute.

<img src="/assets/images/build-your-simulator-f12a-encode-persona.png" width="566" style="display:block;margin:1.5rem auto;" alt="Encoding a persona: the quoted persona becomes one multi-hot row vector on the effects table's columns">

With $$R$$ possible response options, the attribute-level contributions can be stored in a lookup table $$W=[w_{rd}]$$ with dimensions $$R \times D$$. Its rows are response options, its columns are attribute levels, and each entry $$w_{rd}$$ holds the contribution for response $$r$$ and flattened attribute-level column $$d$$. The table has the same response-by-attribute structure as a contingency table, but its cells contain log-linear effect scores rather than counts.

After flattening, the same score sum for a single response can be written as

$$s_r(x) = b_r + \sum_d w_{rd}\,\mathbf{x}_d.$$

Multiplying the full multi-hot persona vector by the table performs that sum for every response at once:

$$s(x) = b + \mathbf{x}W^\top.$$

<img src="/assets/images/build-your-simulator-f12b-score-persona.png" width="720" style="display:block;margin:1.5rem auto;" alt="Scoring a persona: the multi-hot vector times the transposed effects table, drawn as silhouettes and as the sum of the selected rows">

### 3.3 Learning the effects

Combining the table with the normalization above gives

$$B(x) = \operatorname{softmax}(\mathbf{x}W^\top + b).$$

This form is multinomial logistic regression. We elicit response distributions for a designed set of persona cells from the LLM oracle and use those reads as training targets to fit $$B(x)$$. How many elicited cells the fit needs is the subject of Section 4.

<img src="/assets/images/build-your-simulator-f6-elicit-distill.png" width="720" style="display:block;margin:1.5rem auto;" alt="How the response model is built: eliciting a distribution for one designed cell, then fitting one additive effects table to all the reads">

### 3.4 One fit, any target subgroup

Once fitted, the same response model can be applied to any target subgroup defined by attributes in its schema. A subgroup $$\mathcal S$$ identifies the set of complete persona cells consistent with its specified attributes. For women aged 65 and older, for example, age and gender are fixed while education, income, race, and the other modeled attributes vary across the compatible cells.

We poststratify by averaging the response model's predictions over those cells, weighted by their prevalence within the subgroup:

$$\mu_r(\mathcal S) = \sum_{x \in \mathcal S} B_r(x)\,P(X{=}x \mid X \in \mathcal S).$$

The behavior model $$B$$ remains fixed, since changing the target subgroup changes only the population weights.

## 4. Evaluating the Elicitation Budget

Because we elicit a distribution over all responses for every persona, learning $$B(x)$$ is a designed curve-fitting problem. We need enough well-placed reads to identify the effects and average over the parts of the oracle's surface which are not perfectly additive.

### 4.1 The theoretical floor

For a main-effects model, the number of distinct persona cells needed to identify the design is

$$p = 1 + \sum_j (L_j - 1),$$

where $$L_j$$ is the number of levels of attribute $$j$$. This minimum can be constructed from one reference cell and $$p-1$$ additional cells which change one attribute at a time.

In our primary binary experiment, the schema contains 15 attributes and 74 levels, giving $$p=60$$. Each elicitation supplies a distribution over every response option, so under a full-rank design and fixed identification constraints, these cells identify the response-specific effects in parallel. Additional cells then force the table toward the best additive description of the broader response surface rather than the particular reads in the minimum design.

### 4.2 The practical requirement

To test how many additional cells we needed, we fit tables using training sample counts from $$p$$ to $$4p$$ and scored each table against the LLM's elicitations for held-out persona cells. Most of the improvement from adding training cells appeared by $$2p$$, or 120 LLM elicitations in the primary experiment.

A companion evaluation repeated the budget curve on a second survey with multiclass response options and tables distilled from seven different LLMs. Although absolute performance varied across settings and source LLMs, the curves had remarkably similar shapes, with most of the attainable fidelity arriving at a relatively small elicitation budget.

<div class="blog-figure-row">
<img src="/assets/images/build-your-simulator-f10-oracle-fidelity.png" alt="Held-out fidelity to the oracle as the elicitation budget grows from one to four times the identification minimum, on CES and GSS">
<img src="/assets/images/build-your-simulator-f11-model-invariance.png" alt="The same budget curve for seven models: held-out total variation against elicitation budget, one line per model">
</div>

### 4.3 The convergence gap

The held-out error nevertheless flattened above zero. Since the main-effects fit is convex, optimization was not the problem. Taking the full span from a persona-free guess to the oracle as 100%, the additive table closed about 80%, while better cell placement could recover another 2%, read noise accounted for 1%, and the limitations of the additive model accounted for the remaining 17%.

<img src="/assets/images/build-your-simulator-f13-gap-decomposition.png" width="540" style="display:block;margin:1.5rem auto;" alt="Decomposition of the span from a naive guess to oracle behavior: 80% captured by the distillation, 2% cell selection, 1% read noise, 17% model limitation">

Pairwise interactions recovered about a sixth of the remaining gap, while higher-order terms bought essentially nothing further. We therefore retained the simpler main-effects model in the post's other experiments.

## 5. The Oracle Is Not Ground Truth

While we refer to the LLM as our oracle, a high-fidelity distillation of an inaccurate model is still only as good as the model it copies.

We therefore distinguish fidelity and accuracy with two separate measures:

- **Oracle fidelity** measures how much distance the table closes between a persona-free baseline and the LLM's held-out answers.
- **Ground-truth accuracy** measures how much of the progress from the same persona-free baseline to the best additive fit of held-out human data the distilled table achieves.

Both are normalized gap-closed measures in which zero represents the persona-free baseline, one represents the corresponding target, and higher values are better. The seven-model comparison below comes from the companion evaluation introduced in Section 4.2.

| model | oracle fidelity | ground-truth accuracy |
|---|---:|---:|
| gemini-3.5-flash | 0.530 | 0.511 |
| grok-4.5 | 0.595 | 0.489 |
| glm-5 | 0.597 | 0.484 |
| deepseek-v3.2 | 0.652 | 0.463 |
| qwen3-next-80b | 0.630 | 0.454 |
| gpt-5.4 | 0.660 | 0.435 |
| kimi-k2.5 | 0.556 | 0.422 |

Among these seven models, fidelity to the oracle did not predict accuracy against people. Gemini produced the least faithful distillation in the comparison, yet the most accurate one against ground truth. The table can copy its oracle well while inheriting the oracle's systematic errors.

That leaves the more useful diagnostic question of whether the oracle is wrong about how groups differ from one another, or instead captures those differences while placing them on the wrong absolute scale.

## 6. Can the Student Surpass the Teacher?

Recall from Section 3 that our log-linear effects model is composed of a global baseline $$b$$, shared by every group, and attribute-effect scores $$W$$, which describe how groups deviate from that baseline. We can evaluate the two separately to diagnose whether our model's errors reflect a misunderstanding of how attributes drive behavior, or a useful relative structure anchored to the wrong prior.

### 6.1 Scoring our scores

Returning to the primary binary experiment, we evaluate how well $$W$$ captures the divergence between groups by measuring Spearman rank correlation between the model's predictions and the ground-truth response rates. This gives us a proxy for the model's relative structure, separately from the accuracy of its absolute predictions.

<img src="/assets/images/build-your-simulator-f14-relative-ordering.png" width="344" style="display:block;margin:1.5rem auto;" alt="Relative ordering: the raw table's Spearman rank correlation against held-out truth at one, two, and three attributes">

A Spearman correlation of 0.82 indicates strong agreement between the model's ranking of groups and the ranking observed in the ground truth.

For the binary evaluation reported here, the predicted probability is monotone in the log odds between the two responses. Taking $$r_0$$ as the reference response,

$$\log \frac{B_r(x)}{B_{r_0}(x)} = (b_r - b_{r_0}) + \sum_j \left[\,w_r(a_j) - w_{r_0}(a_j)\,\right].$$

The baseline contrast is constant across groups, so their ordering is determined by the attribute effects in $$W$$. The correlation therefore suggests that the model (and, by extension, the LLM from which it was distilled) has learned a substantial part of the relative geometry of the group-behavior space, narrowing the source of the remaining error.

### 6.2 A baseline diagnostic

While $$W$$ determines how groups differ from one another, $$b$$ determines the baseline from which those differences are measured. Having found that the model captures the relative ordering of groups well, we next ask whether its poor absolute predictions can be explained in part by an incorrect baseline.

To test this, we hold the learned attribute scores $$W$$ fixed and optimize only a replacement baseline $$\tilde b$$ against the held-out ground-truth data. This is a diagnostic rather than a deployable procedure because it uses the same cell-level human responses against which the model is evaluated.

<img src="/assets/images/build-your-simulator-f15-hero-scatter.png" width="368" style="display:block;margin:1.5rem auto;" alt="One constant per question: held-out truth against the raw table's predictions before and after the question's single constant shift">

As the figure shows, replacing the original baseline with $$\tilde b$$, while leaving every attribute score intact, reduces mean absolute error by up to 65% across the evaluated source models, and by an average of 31%. Because this adjustment is shared across groups, it improves the model's absolute position without relearning the score differences encoded by $$W$$. This suggests that a large part of the error lies in the overall distribution to which the group structure is anchored.

Here, the training targets were elicited from an LLM rather than sampled from a population. One plausible interpretation is that the LLM has systematic response-level preferences which cause it to overstate some answers and understate others, even while representing the differences between demographic groups reasonably well.

Re-estimating the baseline corrects a global shift but cannot address every distortion, including any tendency to overstate or understate differences between groups. The calibration below therefore addresses only the baseline error. Because the diagnostic also uses more outcome information than would be available in practice, we next ask whether an observable population topline can anchor the same response field.

## 7. Calibrate Once, Then Compose

Composing the raw response table over the global population distribution $$P_G(x)$$ gives us the national response distribution implied by the model. Holding the learned attribute effects $$W$$ fixed, we solve for a new baseline $$b^\star$$ which shifts that implied distribution to match the observed topline $$T_G$$:

$$\sum_x \operatorname{softmax}\!\left(\mathbf{x}W^\top + b^\star\right) P_G(x) = T_G.$$

Whereas $$\tilde b$$ uses cell-level human responses to diagnose the error, $$b^\star$$ uses the topline as its only outcome information. Although the topline cannot answer any subgroup question by itself, it anchors every persona prediction produced by the same response field. Later subgroup estimates can then be obtained by composing the calibrated table with $$P_{\mathcal S}(x)$$, with no further LLM calls.

For a fair comparison, we repeat the direct ask with the same topline supplied in its prompt, separating the value of the anchor from the value of explicit composition.

<div class="table-pair">
<table>
<caption>CES 2022</caption>
<thead><tr><th>estimator</th><th style="text-align:right">no anchor</th><th style="text-align:right">topline supplied</th></tr></thead>
<tbody>
<tr><td>Direct group ask</td><td style="text-align:right"><strong>0.0449</strong></td><td style="text-align:right">0.0326</td></tr>
<tr><td>Explicit composition</td><td style="text-align:right">0.0511</td><td style="text-align:right"><strong>0.0239</strong></td></tr>
</tbody>
</table>
<table>
<caption>GSS 2022</caption>
<thead><tr><th>estimator</th><th style="text-align:right">no anchor</th><th style="text-align:right">topline supplied</th></tr></thead>
<tbody>
<tr><td>Direct group ask</td><td style="text-align:right"><strong>0.0548</strong></td><td style="text-align:right"><strong>0.0414</strong></td></tr>
<tr><td>Explicit composition</td><td style="text-align:right">0.0702</td><td style="text-align:right"><strong>0.0417</strong></td></tr>
</tbody>
</table>
</div>

<p class="table-note"><em>Mean absolute error, lower is better. Bold marks the better method within a column, and both anchored GSS values are bold because their difference is statistically indistinguishable.</em></p>

Without an outcome anchor, the direct ask beats the raw composed table on both datasets, showing that distillation makes the response field reusable without making its absolute rates accurate. Supplying the topline improves both routes, after which explicit composition moves ahead of the anchored direct ask on CES and reaches parity with it on GSS.

Across these two evaluations, the consistent result is that the anchor turns the raw composed table from worse than direct asking into a competitive or better subgroup estimator. Whether explicit composition supplies additional accuracy beyond the anchored direct ask depends on the setting.

Because CES supplies one rate for a binary outcome while GSS supplies a distribution over several response options, the difference between their results should not be interpreted as a clean dataset or domain effect.

The LLM's reusable contribution is the relative structure encoded in $$W$$, not a population of sampled respondents or a trustworthy absolute baseline. We elicit that structure once, anchor it with observed outcomes, and combine it with an explicit population frame. Every later subgroup estimate is then deterministic arithmetic outside the LLM.

In these experiments, the population frame is borrowed from real microdata. Building that frame when no complete population joint exists is the remaining half of the simulator.
