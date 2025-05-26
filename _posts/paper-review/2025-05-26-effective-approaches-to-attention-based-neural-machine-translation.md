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

![fig1](/assets/images/attention_nmt/fig1.PNG)

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
- A neural machine translation system is a neural network that directly models the conditional probability $p(y|x)$ of translating a source sentence, $x_1,..., x_n$ to a target sentence, $y_1,...,y_m$. The conditional probability is:

$$ \log p(y|x)=\sum_{j=1}^{m} \log p(y_j|y_{<j}, s)$$

,where $s$ is the source representation of encoder

- In previous works, RNN, LSTM, GRU and CNN was used for encoder, and RNN, LSTM and GRU was used for decoder
- Probability of word $y_i$ is calculated from current hidden state(with weight $W$):

$$ p(y_i|y_{<j},s) = softmax(g(h_j))$$