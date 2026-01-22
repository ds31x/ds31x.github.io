---
layout  : wiki
title   : Transformer
summary : Attention is all you need
date    : 2026-01-22 14:05:18 +0900
updated : 2026-01-22 16:38:10 +0900
tag     : 
resource: 08/A1F11336AE4B348E5C35356B0F88BA
toc     : true
public  : true
parent  : [[/review]]
latex   : false
---
* TOC
{:toc}

# Attentino is all you need

기존의 RNN의 구조에서 사용되던 Attetion 을 아예 중심으로 하는 새로운 구조를 제안.

Sequence data를 다루는데 있어서 가장 강력하게 사용되던 RNN 구조를 버리고, 
철저하게 Attention을 통해 Sequence data를 다루는 model임.

[scaled dot-product](https://dsaint31.me/mkdocs_site/ML/ch16_RNN/RNN_attention_score/#attention-scores) 기반의 attention을 활용하여, Multi-Head Attention (=[[./sefl_attention]]) 와 Masked Multi-Head Attention(=self attention 과 cross attetion) 과 Point-wise FeedForward, LayerNormalization 등을 통해 Encoder-Decoder 구조의 모델을 구성함.
