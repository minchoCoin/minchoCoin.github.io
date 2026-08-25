---
title: "Parameter-Efficient Attractor Tuning for Improving Reasoning in Pretrained Small Language Models"
last_modified_at: 2026-08-25T12:55:13+09:00
categories:
    - project-page
tags:
    - project-page

toc: true
toc_label: "My Table of Contents"
author_profile: true
---
# Introduction
Recent Large Language Models(LLMs) have strong performance across various domains that require complex reasoning
These models require substantial memory and computational resources

Small Language Models(SLMs) have emerged as a promising alternative for resource constrained on-device environments

However, due to their limited model capacity, SLMs often underperform LLMs on complex reasoning tasks

Loop Transformer is the approach to improving the reasoning capability of SLMs
- Loop Transformer repeatedly applies the same Transformer block
- Increasing the effective computational depth without increasing the number of parameters
- Enables iterative hidden-state refinement and latent reasoning
- However, increasing the number of iterations also increase inference latency

Attractor[2] addresses the limitation by treating iterative latent refinement as a process of finding a stable equilibrium representation
- Attractor exhibits equilibrium internalization during training, where the initial output embedding gradually moves closer to the equilibrium
- Reducing inference latency while preserving the benefits of latent refinement

However, original Attractor[2] requires both the backbone module and the attractor module to be pretrained from scratch
- Training becomes challenging under limited computational budgets, making it necessary to leverage pretrained model weights

In this presentation, we propose a parameter-efficient attractor tuning method that uses a pretrained SLM as a backbone
- Freezes the backbone parameters, adds small Low-Rank Adaptation(LoRA)[3] weights, and attaches an attractor module
- Only the LoRA weights and the attractor module are trained to improve the reasoning performance of the pretrained SLM

# Related Works
## Attractor(2)
Attractor Models consist of two modules: the backbone module and attractor module
- The backbone module is typically a larger Transformer network, which generate initial output embedding
- The attractor module is typically a smaller Transformer network, which refines the initial output until convergence

## Attractor module(2)
- Starting from the backbone proposal $\tilde{y}_0$, it repeatedly refines the output embedding
  - $ \tilde{y}_{t+1} = T_{\theta_a}(\tilde{y}_t,\tilde{y}_0), where \tilde{y}_0 = T_{\theta_b}(x) $
  - Persistently inject the initial guess at every refinement step

- Rather than rolling out recurrent steps to reach a fixed point, attractor solve for the convergence
  - $\mathcal{A}_{\theta_a}(\tilde{y}^{*};\tilde{y}_0) := T_{\theta_a}(\tilde{y}^{*},\tilde{y}_0) - \tilde{y}^{*} = 0\Rightarrow \tilde{y}^{*} = \mathrm{RootFind}(\mathcal{A}_{\theta_a}(\cdot;\tilde{y}_0); y_0)$

- The **RootFind** algorithm uses Anderson acceleration, which combines a small window of past iterates and residuals to reach the fixed point faster than plain recursion
  - $y_{t+1} = \sum \alpha_i y_i$

- The solver exits when
  - $\|\mathcal{A}_{\theta_a}(\tilde{y}_t;\tilde{y}_0)\|_2 / \|\tilde{y}_t\|_2 < \epsilon$
  - or after max steps

# Methodology
## Appending attractor module to the end of the pretrained model
Appending 1-2 Transformer layers as attractor module to the end of the pretrained model
- We remove the pretrained LM head(classifier) because attractor module uses unembedding(𝐸^𝑇) for token classification
- Argument of the transformer layers in attractor module are same as pretrained model
- We use rolling out recurrent steps to reach a fixed point instead of RootFind algorithm

Attaching LoRA adapter to each layer of the pretrained model

The LoRA adapter and Attractor module are trained, while the pretrained layers are frozen.

[![attractor-tuning-architecture.png](https://i.postimg.cc/s2yGLWfR/attractor-tuning-architecture.png)](https://postimg.cc/BjYvLX1h)

(Figure 1. Architecture of parameter-efficient attractor tuning)

## Token Superposition Continued Pretraining
During the first 20% of the optimizer steps, we use token-superposition[4] training
- Raw token sequences are grouped into non-overlapping bags of six tokens
- During token superposition training, The embeddings in each bag are averaged, The model is trained to - predict the next token bag using uniform loss weighting across bag position

Enables efficient pretraining because model can learn multiple token with one back propagation

# Experiments
## Training
We use the nanochat[5] training recipe, training on FineWeb-Edu[6].
We train LoRA and Attractor module while freezes the SmolLM2-135M[7] backbone

(Table 1. Training details)

| Parameter | Value |
|---|---:|
| Backbone model | SmolLM2-135M |
| LoRA rank | 16 |
| LoRA alpha | 32 |
| Attractor layers | 2 |
| Max_iter | 8 |
| Min_iter | 2 |
| Tolerance | 1e-3 |
| LoRA learning rate | 2e-4 |
| Attractor learning rate | 8e-4 |
| Batch size | 262144 |
| Step | 1000 |
| Token superposition bag size | 6 |
| Token superposition ratio | 0.2 |
| Total Training tokens | 0.52B |

## Evaluation metrics
PerPLexity(PPL)
- Measures how well a language model predicts a sequence of text, and it reflects the model’s uncertainty when predicting next token(Lower is better)

CORE
- Measures in-context task accuracy under the nanochat base evaluation process(higher is better)

**For Attractor tuned model, We evaluate the model without attractor module for same inference time**

## Some Tasks of CORE evaluation

(Table 2. Some Tasks of CORE evaluation)

| Tasks | Explanation |
|---|---|
| ARC Challenge | A difficult subset of the AI2 **Reasoning Challenge**, consisting of grade-school science multiple-choice questions that often require multi-step reasoning and external knowledge. |
| Lambda-OpenAI | A benchmark dataset used to evaluate language models on **natural-language understanding and reasoning**, typically through prompt-based question answering. |
| BigBench Dyck Language | A BIG-bench task that tests whether models can recognize and generate correctly balanced sequences of brackets, **measuring hierarchical and formal-language reasoning**. |
| AGI Eval LSAT AR | The Analytical Reasoning portion of AGIEval based on LSAT-style logic games, requiring constraint satisfaction, **deduction, and multi-step logical reasoning**. |
| CoQA | A conversational question-answering dataset in which models answer a sequence of **context-dependent questions** about a passage while maintaining dialogue context. |
| BigBench Operators | A BIG-bench **reasoning task** that evaluates whether models can infer and correctly apply unfamiliar or artificially defined operators according to given rules. |
| OpenBookQA | A multiple-choice science QA dataset designed to **test reasoning with a small “open book”** of elementary scientific facts combined with broader commonsense knowledge. |
| HellaSwag Zero-shot | A multiple-choice benchmark that **evaluates commonsense reasoning** by asking models to choose the most plausible continuation of a given context without task-specific examples. |

# Results
## Loss by Training step
[![Loss-by-training-steps.png](https://i.postimg.cc/L8qMMQX8/Loss-by-training-steps.png)](https://postimg.cc/dhKf2Brb)

## Main Results
LoRA-Attractor tuned model achieve higher accuracy than baseline or LoRA-only tuned model on some tasks of CORE evaluation.
- However, due to catastrophic forgetting during LLM fine-tuning, the LoRA-Attractor-tuned model shows lower accuracy on commonsense knowledge tasks. As a result, its overall CORE score is 0.2134, compared with 0.2213 for the LoRA-only model.

| Model | Test ppl | Arc_challenge | Lambda_openai | Bigbench Dyck language | Agi Eval Lsat ar | coqa | Bigbench operator | Openbook qa | Hellaswag zeroshot |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| baseline | 14.4298 | 0.3106 | 0.4357 | 0.165 | 0.2826 | 0.1979 | 0.2333 | 0.356 | 0.4312 |
| LoRA-Only-tuned | **13.47 (-6.62%)** | 0.3166 (+1.92%) | 0.4298 (-1.34%) | 0.157 (-4.85%) | 0.2870 (+1.54%) | **0.2454 (+23.99%)** | 0.2333 (+0.00%) | 0.362 (+1.69%) | **0.4304 (-0.18%)** |
| LoRA-Attractor-tuned-(RootFind) | 14.8912 (+3.2%) | 0.3174 (+2.20%) | 0.4815 (+10.51%) | 0.177 (+7.27%) | 0.3043 (+7.69%) | 0.2429 (+22.72%) | 0.2286 (-2.04%) | 0.362 (+1.69%) | 0.4214 (-2.26%) |
| **LoRA-Attractor-tuned(ours)** | 14.8483 (+2.9%) | **0.3208 (+3.30%)** | **0.4817 (+10.56%)** | **0.177 (+7.27%)** | **0.3087 (+9.23%)** | 0.2434 (+22.97%) | **0.2381 (+2.34%)** | **0.364 (+2.25%)** | 0.4233 (-1.82%) |

## equilibrium internalization
[![internalization.png](https://i.postimg.cc/SN2ZSB6J/internalization.png)](https://postimg.cc/ZCmPjDDS)

(Figure 2. MSE and cosine similarity of initial output and attractor output)

# Conclusion
We propose Parameter-Efficient Attractor Tuning to improve the reasoning ability of pretrained SLMs.
- Appending 1-2 Transformer layers as attractor module to the end of the pretrained model
- We only train the LoRA adapter and Attractor module while the pretrained weights are frozen

With training on only 0.5B tokens, the model's reasoning ability improves for some tasks.

However, due to limited computational resources, we trained on only 0.5B tokens and evaluated models with only 135M parameters.
- Further experiments are needed to verify whether the proposed method remains effective with more training tokens and larger models.

# References
[1] Prairie, H., Novack, Z., Berg-Kirkpatrick, T., & Fu, D.Y. (2026). Parcae: Scaling Laws For Stable Looped Language Models. ArXiv, abs/2604.12946.

[2] Fein-Ashley, J., & Rashidinejad, P. (2026). Solve the Loop: Attractor Models for Language and Reasoning. ArXiv, abs/2605.12466.

[3] Hu, J.E., Shen, Y., Wallis, P., Allen-Zhu, Z., Li, Y., Wang, S., & Chen, W. (2021). LoRA: Low-Rank Adaptation of Large Language Models. ArXiv, abs/2106.09685.

[4] Peng, B., Gigant, T., & Quesnelle, J. (2026). Efficient Pre-Training with Token Superposition. ArXiv, abs/2605.06546.

[5] Karpathy, A. (2025). nanochat: The best ChatGPT that $100 can buy [Computer software]. GitHub. https://github.com/karpathy/nanochat

[6] Penedo, G., Kydlícek, H., Allal, L.B., Lozhkov, A., Mitchell, M., Raffel, C., Werra, L.V., & Wolf, T. (2024). The FineWeb Datasets: Decanting the Web for the Finest Text Data at Scale. ArXiv, abs/2406.17557.

[7] Allal, L.B., Lozhkov, A., Bakouch, E., Bl'azquez, G.M., Penedo, G., Tunstall, L., Marafioti, A., Kydl'ivcek, H., Lajar'in, A.P., Srivastav, V., Lochner, J., Fahlgren, C., Nguyen, X.S., Fourrier, C., Burtenshaw, B., Larcher, H., Zhao, H., Zakka, C., Morlon, M., Raffel, C., Werra, L.V., & Wolf, T. (2025). SmolLM2: When Smol Goes Big - Data-Centric Training of a Small Language Model. ArXiv, abs/2502.02737.

