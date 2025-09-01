---
title: "K-Wav2vec 2.0: Automatic Speech Recognition based on Joint Decoding of Graphemes and Syllables"
last_modified_at: 2025-08-01T17:53:13+09:00
categories:
    - paper-review
tags:
    - paper-review

toc: true
toc_label: "My Table of Contents"
author_profile: true
header:
  teaser: "/assets/images/kwav2vec/thumbnail.PNG"
---
This article is review and derivation of [Kim et al., K-Wav2vec 2.0: Automatic Speech Recognition based on Joint Decoding of Graphemes and Syllables](https://arxiv.org/abs/2110.05172)

# Introduction
In recent years, self-supervised methodology has shown success in various fields, including natural language processing, image recognition and auto speech recognition

The Wav2vec 2.0 model is an end-to-end framework of self-supervised learning of Automatic Speech Recognition(ASR), which is an effective pre-training method to learn speech representations
- When followed by fine-tuning with small amounts of labeled data, the Wav2vec2.0 model has shown remarkable performance in English ASR tasks
- It is still an open question whether this method can be effective with other language

In the Korean writing system, letters are written in syllabic blocks, where one sound is made at once
- This unique writing system allows us to build a Korean ASR model that is based on either graphemes(자소) or syllable(음절) blocks
- Graphemes: ㅇㅏㄴㄴㅕㅇㅎㅏㅅㅔㅇㅛ
- Syllable: 안녕하세요
- Most Korean ASR model were developed with syllables, and syllable-based models outperform grapheme-based models
- Grapheme require more combinations to predict(11172)
- However, syllable-based models also have data sparseness problem for infrequently used syllables and the Out-Of-Vocabulary(OOV) problem when the training data is insufficient
    - OOV problem arises when syllables that do not appear in the training set appear in the test set

![fig1](/assets/images/kwav2vec/fig1.PNG)

(Figure 1. Figure 1. Grapheme(자소) and syllables(음절) in korean)

To overcome these problem, using Multi-Task Learning(MTL) approach
- Sharing some representations from different tasks
- Chen et al., 2021
    - By learning the shared representations between high-level and low-level modeling units, multi-task models alleviate data sparseness issues and achieve better performance
- Ueno et al., 2018
    - Recovering methods to substitute OOV words produced for high-level decoding with segments generated in low-level outputs

- Propose a multi-task hierarchical fine-tuning architecture of Wav2Vec2.0 to reflect the unique relationship that exists in Korean writing between syllables and graphemes
    - In the inference step, we used a joint decoding strategy that considers high-level and low-level units to find the best sequence from a set of candidates instead of using additional language models
    - Additionally, we also experimented with the cross-lingual transfer of the English Wav2Vec2.0 pretrained model to Korean ASR tasks to overcome limited dataset

# Background
The Wave2Vec 2.0 consists of three networks: a feature encoder, a contextual transformer, and a quantization module
- The feature encoder, which is composed of a multi-layer Convolutional Neural Network(CNN), encodes the raw audio 𝑋 and outputs the latent speech representation 𝑍
- The contextual transformer, which is a stack of transformer encoders, learns the context representation 𝐶 by taking latent speech representation as input

Pre-training and fine-tuning of Wav2Vec2.0
- A certain portion of the latent representation 𝑍 are randomly masked
- Solving a contrastive task, distinguishing the true quantized vector
    - Discrete latent vectors are randomly sampled from the other time steps

$$ L_m = -\log{\frac{\exp(\frac{sim(c_t,q_{true})}{k})}{\sum_{q~Q_t} \exp (\frac{k}{sim(c_t,q)})}},$$
where
$$sim=\frac{a^Tb}{||a||||b||} $$

- In fine-tuning, randomly initialized linear layer is added
    - Takes the contextualized representation and generate the words

![fig2](/assets/images/kwav2vec/fig2.png)

(Figure 2. Wave2Vec2.0 architecture)

# Method
Multi-task hierarchical architecture
- In the grapheme encoder, a linear layer and softmax is adopted to project the encoded features into a grapheme vocabulary $g\in G = \left\{ ㄱ,ㄴ,ㄷ,...,ㅏ,ㅑ,...\right\}, p(g_f|c_f)$

- In the syllable encoder, a stack of transformer encoder is adopted for converting the encoded features to a sequence of hidden vector ℎ to capture the relationship between low-level and high-level information


![fig3](/assets/images/kwav2vec/fig3.png)

(Figure 3. Overview of proposed ASR framework)

We used the Connectionist Temporal Classification(CTC) loss when training the model
- The CTC is an approach for sequence labeling, wherein the lengths of the label sequence and the output frames are different

The objective function of the proposed architecture is the weighted sum of grapheme and syllable CTC losses to learn the two tasks simultaneously
- $L_{MTL} = \lambda \log p_{syll}^{ctc}(Y|X) + (1-\lambda) \log p_{graph}^{ctc}(Y|X)$

To leverage the ASR performance with limited data, we explore cross-lingual transfer by further English pre-training model with a Korean dataset
- We use pre-trained Wave2Vec2.0 with English dataset(960h)
    - Additionally, we further pre-train this model on the Korean dataset(965h)

## Joint Decoder
The join decoder combines low-level and high-level beam search results to find the best sequence within a limited amount of time
- We used the CTC beam search decoder, which decodes iteratively to find candidates over time-steps of CTC output and scores them with given probability of each time-step
- The objective of the joint decoder is to find the most probable sequence 𝑌 ̂ among the candidates
    - $\hat Y = argmax$




