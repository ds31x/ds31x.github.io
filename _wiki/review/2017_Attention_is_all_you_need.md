---
layout  : wiki
title   : Transformer
summary : Attention is all you need
date    : 2026-01-22 14:05:18 +0900
updated : 2026-04-18 23:04:37 +0900
tag     : transformer attention multi-head-attention
resource: 08/A1F11336AE4B348E5C35356B0F88BA
toc     : true
public  : true
parent  : [[/review]]
latex   : false
---
* TOC
{:toc}

# Attention Is All You Need

2017년 Google의 8명의 저자가  
기존의 RNN의 구조에서 사용되던 attetion 을  
아예 attention 중심으로 하는 새로운 구조를 제안한 논문.

* [Attention is all you need. Ahish Vaswani et al.](https://arxiv.org/abs/1706.03762)

Sequence data를 다루는데 있어서 가장 강력하게 사용되던 RNN 구조를 버리고, 
철저하게 Attention을 통해 Sequence data를 다루는 model임.

[scaled dot-product](https://dsaint31.me/mkdocs_site/ML/ch16_RNN/RNN_attention_score/#attention-scores) 기반의 attention을 활용하여, 

* Multi-Head Attention (=[[/review/self_attention]]) 와
* Masked Multi-Head Attention(self attention) 과 Multi-Head Cross Attetion 과
* Position-wise Feed Forward Network,
* [[/review/LayerNormalization]] 등을 통해
* **[Encoder-Decoder](https://dsaint31.me/mkdocs_site/ML/ch16_RNN/RNN_topologies/#many-to-many-encoder-decoder)** 구조의 모델을 구성함.
