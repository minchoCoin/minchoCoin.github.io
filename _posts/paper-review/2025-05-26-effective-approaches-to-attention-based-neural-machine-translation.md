---
title: "Effective Approaches to Attention-based Neural Machine Translation"
last_modified_at: 2025-05-26T10:53:13+09:00
categories:
    - paper-review
tags:
    - paper-review

toc: true
toc_label: "My Table of Contents"
author_profile: true
header:
  teaser: "/assets/images/attention_nmt/thumbnail.PNG"
---

# paper-review
This article is review of [Luong et al.,  Effective Approaches to Attention-based Neural Machine Translation](https://arxiv.org/abs/1508.04025)

# Introduction
- Neural Machine Translation(NMT) achieved state-of-the-art performances in large-scale translation tasks

![fig1](/assets/images/attention_nmt/fig1.PNG){: .align-center}

(Figure 1. Neural machine translation - a stacking recurrent architecture for translating a source sequence A B C D into a target sequence X Y Z. Here, eos marks the end of a sentence)

- Typical NMT uses sequence-to-sequence model.
- sequence-to-sequence model consists of encoder and decoder
- In figure 1, the encoder and the decoder is 2-layer LSTMs each
    - This means that the first layer receive embedded text and previous hidden state of first layer as input, and the second layer receive output of first layer and previous hidden state of second layer as input
    - The output of decoder is translated result
    - The encoder encoded the original text and final hidden vector is fed into decoder as source sentence representation
    - The decoder predict next word probability using previous hidden state and previous predicted word
        - with eos as input, decoder predict the first word(X)
        - with first word(X) as input, decoder predict the second word(Y)
    - Sequence-to-sequence model is trained with Teacher forcing
        - Teacher forcing is that the predicted output of the decoder cell from the previous is not input to the decoder cell from the current, but the real(target) value from the previous is used as the input value of the decoder cell from the current
- In parallel, the concept of attention has gained popularity recently in training neural networks, allowing models to learn alignments between different modalities, e.g., between speech frames and text in the speech recognition, or between picture and its text description
- In the context of NMT,Bahdanau et al. (2015) has successfully applied such attentional mechanism to jointly translate and align words.
- In this work, we design, with simplicity and effectiveness in mind, two novel type of attention based models: a global approach in which all source words are attended and a local one whereby only a subset of source words are considered at a time

# Neural Machine Translation
A neural machine translation system is a neural network that directly models the conditional probability $ p(y|x) $ of translating a source sentence, $ x_1,..., x_n $ to a target sentence, $ y_1,...,y_m $. The conditional probability is

$$ \log p(y|x)=\sum_{j=1}^{m} \log p(y_j|y_{<j}, \textbf{s})$$

,where $s$ is the source representation of encoder

- In previous works, RNN, LSTM, GRU and CNN was used for encoder, and RNN, LSTM and GRU was used for decoder
- Probability of word $y_i$ is calculated from current hidden state(with weight $W$):

$$ p(y_i|y_{<j},\textbf{s}) = softmax(g(\textbf{h}_j ))$$

- outputs of g function are vocabulary-sized vector.

$\textbf{h}_j $ is the RNN hidden unit, abstractly computed as

$$\textbf{h}_j=f(\textbf{h}_{j-1},\textbf{s}) $$

$f$ computes the current hidden state given the previous hidden state(+ usually with previous output as current input)

In Kalchbrenner and Blunsom, 2013, Sutskever et al., 2014, Cho et al., 2014, Luong et al., 2015, the source representation $\textbf{s}$ is only used once to initialize the decoder hidden state

On the other hand, in  (Bahdanau et al., 2015, Jean et al., 2015), and this work, $\textbf{s}$  implies a set of source hidden states(encoder hidden states) which are referred throughout the entire course of the translation(decoding) process. 

In this paper, the author uses the 2-layer LSTM architecture.

Training objective is

$$ J_t=\sum_{(x,y)\in D} - \log p(y|x) $$

with D being parallel training corpus, $ x $ being input sentence, $ y $ being target sentence. The higher the probability the model gives the correct answer, the smaller the log value

By minimizing the sum of these values in all training data, the performance of the model is improved.

# Attention-based model

Attention models are classified into global and local.

these models differ in how the context vector $c_t$ is derived. context vector $c_t$ captures relevant source-side(encoder-side hidden state) information to help predict the current target word $y_t$.

Attentional hidden state is produced as follows

$$ \tilde{\textbf{h}}_t = \tanh (W_c[\textbf{c}_t;\textbf{h}_t])$$

, with concatenating the context vector and hidden state

The attentional hidden state is then fed through the softmax layer to produce the predictive distribution formulated as

$$ p(y_t|y_{<t},x)=softmax(\textbf{W}_s \tilde{\textbf{h}}_t)$$

## Global Attention
Consider all the hidden states of the encoder when deriving the context vector.

align vector $\textbf{a}_t(s)$ indicates the proportion of attention that the decoder's hidden vector at time step $t$ should pay to the encoder's hidden vector at position $s$ among all encoder hidden vectors



$$ \textbf{a}_t(s) = \frac{\exp (\textbf{score}(\textbf{h}_t, \textbf{h}_s))}{\sum_{s'}\exp (\textbf{score}(\textbf{h}_t, \textbf{h}_{s'}))} $$

![fig2](/assets/images/attention_nmt/fig2.PNG){: .align-center}


score function indicates that how much attention the hidden vector at time $t$ on the decoder should pay to $s$ hidden vector on the encoder

there are 3 different alternatives score function.

first is $dot$ score function

$$ \textbf{score}(\textbf{h}_t, \textbf{h}_s)= \textbf{h}_t^\top \textbf{h}_s$$

second is $general$ score function

$$ \textbf{score}(\textbf{h}_t, \textbf{h}_s)= \textbf{h}_t^\top \textbf{W}_a\textbf{h}_s$$

third is $concat$ score function

$$ \textbf{score}(\textbf{h}_t, \textbf{h}_s)= \textbf{v}_a^\top\tanh(\textbf{W}_a[\textbf{h}_t ;\textbf{h}_s])$$

, where $\textbf{v}_a$ and $\textbf{W}_a$ are learnable parameters

The authors also attempts to use $location-based$ function in which the alignment scores are computed from solely the decoder's hidden state

$$ \textbf{a}_t = softmax(\textbf{W}_a \textbf{h}_t)$$

Given the alignment vector as weights, the context vector is computed as the weighted average over all the source hidden states in each top layer of encoder $E$

$$ \textbf{c}_t=\sum_{i \in E} \textbf{a}_t(i)\textbf{h}_i$$
