---
layout  : wiki
title   : Self Attention
summary : 
date    : 2026-01-22 14:56:26 +0900
updated : 2026-01-22 17:34:56 +0900
tag     : 
resource: FF/667E6983EE4427B2095D84BEFB87C1
toc     : true
public  : true
parent  : [[/review/2017_Attention_is_all_you_need]]
latex   : true
---
* TOC
{:toc}

# Self Attention

Self-attention은 

* 하나의 sequence 안에서 각 token이 
* 같은 sequence에 속한 모든 token들을 참조하여 
* 자신의 representation(표현, dense vector)을 계산하는 메커니즘이다.

각 token은 `query`, `key`, `value`로 변환(linear transform)되며,  
* token 간의 관련성은 scaled dot-product 방식으로 
* $q \cdot k / \sqrt{d_k}$를 계산한 뒤 softmax로 정규화.

이 과정을 통해 sequence 내에서 중요한 token에는 더 큰 가중치가 부여되고, context(문맥)을 반영한 token representation이 생성

$$\mathrm{Attention}(Q, K, V)
= \mathrm{softmax}\!\left(\frac{QK^{\top}}{\sqrt{d_k}}\right)V$$

- $Q \in \mathbb{R}^{T_q \times d_k}$ 는 쿼리 벡터들을 행 단위로 쌓아 구성한 행렬이며, $T_q$는 쿼리의 개수, 즉 쿼리 시퀀스의 길이를 의미한다.
- $K \in \mathbb{R}^{T_k \times d_k}$ 는 키 벡터들을 행 단위로 쌓아 구성한 행렬이며, $T_k$는 각 쿼리가 참조할 수 있는 위치의 개수, 즉 참조 대상이 되는 시퀀스의 길이를 의미한다.
- $V \in \mathbb{R}^{T_k \times d_v}$ 는 값 벡터들을 행 단위로 쌓아 구성한 행렬이며, 키와 동일하게 $T_k$개의 원소를 가진다.
- $d_k$는 쿼리와 키 벡터의 차원, $d_v$는 값 벡터의 차원을 의미한다.

실제로, 각 token은 적절한 key, query, value가 되도록 trainable parameter인 $W^Q, W^k, W^V$를 통한 linear transform (로 다른 역할의 표현 공간으로 projection)을 함.
입력이 $X$ 일때 $Q,K,V$는 다음과 같음:

- $Q = W^Q X$
- $K = W^K X$
- $V = W^V X$

입력 $X \in \mathbb{R}^{T \times d_{\text{model}}} $는  
* $ W^Q, W^K, W^V $를 통해 각각  
* $Q \in \mathbb{R}^{T \times d_k}$ ("이 토큰이 무엇을 찾는가"를 표현하는 공간),  
* $K \in \mathbb{R}^{T \times d_k}$ ("이 토큰이 어떤 기준으로 참조되는가"를 표현하는 공간,  
* $V \in \mathbb{R}^{T \times d_v}$ ("실제로 전달할 정보 내용")로 사상(projection)되며,  
* 이 분리를 통해 Transformer는 토큰 간 관계를 학습 가능한 방식으로 모델링.
* 이는 주로 multi-head attention에서 사용됨:
	* $d_q = d_k = d_k = d_model/h$ 가 성립.
	* $d_q$ 는 $d_k$ 와 inner product를 하므로 거의 같은게 관례임. 
	* $h$ : head의 수.


또한 $\mathrm{softmax}(\cdot)$ 함수는 각 쿼리에 대해 키 차원 방향으로 적용되어, 어텐션 가중치의 합이 1이 되도록 정규화한다.

<img src="/resource/FF/667E6983EE4427B2095D84BEFB87C1/43855de8-ab28-450a-8e6a-e6034d8dbc85.png" sytle="display:block; margin:0 auto; max-width:50%" />

[ref: Self-Attention. Ramendra Kumar](https://medium.com/@ramendrakumar/self-attention-d8196b9e9143)

다음 그림이 Transformer에서의 Self Attention을 구하기 위한 scaled dot product의 computational graph임.

<img src="/resource/FF/667E6983EE4427B2095D84BEFB87C1/1b273a75-f1da-446b-ac04-68fc5a42b9ee.png" stye="display:block; margin:0 auto; max-width:50%;" />

[original: Attention is All You Need, 2017](https://arxiv.org/pdf/1706.03762.pdf)

## 예제


"Time flies like an arrow" 라는 sentense의 self attention 을 구하여 시각화 한 예임.

<img src="/resource/FF/667E6983EE4427B2095D84BEFB87C1/16993377-7f09-41a9-8523-bc03465b1157.png" style="display:block; margin:0 auto; max-width:50%;" />

* flies 가 파리들, 날다 의 뜻을 가질 수 있는데
* 위의 경우 날다 이므로 "arrow"와 매우 큰 관계를 가짐을 확인할 수 있음.

"Fruit flies likes a banana" 의 sentense의 예는 다음과 같음:

<img src="/resource/FF/667E6983EE4427B2095D84BEFB87C1/23aebdaf-3da7-4681-8c2b-637812be769b.png" style="display:block; max-width:50%; margin:0 auto;" />

## code

```python
    def scaled_dot_product_attention(
        self, 
        Q: torch.Tensor, 
        K: torch.Tensor, 
        V: torch.Tensor,
        mask: Optional[torch.Tensor] = None
    ) -> torch.Tensor:
        """
        스케일드 닷-프로덕트 어텐션 계산
        
        Attention(Q, K, V) = softmax(QK^T / sqrt(d_k))V
        
        Args:
            Q: Query (batch_size, num_heads, seq_len, d_k)
            K: Key (batch_size, num_heads, seq_len, d_k)
            V: Value (batch_size, num_heads, seq_len, d_k)
            mask: 마스크 (batch_size, 1, seq_len, seq_len) or (batch_size, 1, 1, seq_len)
        Returns:
            어텐션 출력 (batch_size, num_heads, seq_len, d_k)
        """
        # QK^T / sqrt(d_k)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # 마스크 적용 (필요한 경우)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Softmax 적용
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        # Value와 곱하기
        output = torch.matmul(attention_weights, V)
        
        return output
```	

## 참고

[Natural Language Processing with Transformers, O'Reilly](https://colab.research.google.com/github/nlp-with-transformers/notebooks/blob/main/03_transformer-anatomy.ipynb#scrollTo=N1CswhxaU1ko)
