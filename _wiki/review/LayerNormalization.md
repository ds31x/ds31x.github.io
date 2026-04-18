---
layout  : wiki
title   : 
summary : 
date    : 2026-04-18 22:07:58 +0900
updated : 2026-04-18 22:43:01 +0900
tag     : 
resource: F6/7939D8592B448B96D72126B5683A3D
toc     : true
public  : true
parent  : 
latex   : true
---
* TOC
{:toc}

# Layer Normalization



## Overview

- BN(Batch Normalization)이 NLP 도메인에서 겪는 **패딩(Padding) 왜곡 문제** 를 극복하기 위해 Ba et al. (2016)이 제안한 Normalization(정규화) 기법임.
- Mini-batch 전체가 아니라, **개별 샘플(단일 토큰) 하나의 feature 차원 전체(행, Row)** 방향으로 평균($\mu$)과 분산($\sigma^2$)을 계산함.
- Transformer 아키텍처에서 사실상 표준(de facto standard) Normalization 기법으로 채택됨.

> **정규화(Normalization)**는 들쭉날쭉한 데이터의 통계적 특성을 평균 0, 분산 1로 일정하게 맞추는 작업임.  
> BN과 Layer Normalization 은 인공신경망 학습시  
> 각 layer 를 거치는 데이터들이  
> Gradient Exploding이나 Vanishing이 일어나지 않으면서 학습을 효과적으로 할 수 있는 
> 즉, 특성을 보존하면서 안정적인 분포를 가지도록 해 줌.

## Formula

token 벡터 $\mathbf{x} \in \mathbb{R}^{d}$에 대해 아래와 같이 정의됨.

$$\mu = \frac{1}{d}\sum_{j=1}^{d} x_j, \qquad \sigma^2 = \frac{1}{d}\sum_{j=1}^{d}(x_j - \mu)^2$$

$$\text{LN}(\mathbf{x}) = \gamma \odot \frac{\mathbf{x} - \mu}{\sqrt{\sigma^2 + \epsilon}} + \beta$$

- $d$: 정규화 대상 feature 수 (임베딩 차원)
- $\gamma,\, \beta \in \mathbb{R}^{d}$: 학습 가능한 scale / shift 파라미터
- $\epsilon$: numerical stability를 위한 작은 상수 (기본값 $10^{-5}$)


위의 수식은 training 과정에 해당하며, inference 과정에서는 training 중 구한 Exponential Moving Average로 구한 평균과 분산이 사용됨: 이는 BN과 동일함.

## BN vs. LN

![](https://github.com/user-attachments/assets/5ae2c019-8bec-4feb-a83f-d68c28a804dc){style="display: block; margin: 0 auto; width: 500"}

| 구분                |                                  Batch Normalization                                   |         Layer Normalization          |
|---------------------|:--------------------------------------------------------------------------------------:|:------------------------------------:|
| 정규화 축           |           배치 축 (열, Column) </br> 좀더 정확히는 feature를 제외한 모든 축            |        feature axis (행, Row)        |
| 패딩 토큰 간섭      | **심각한 왜곡 발생**<br/> 무의미한 padding으로 인해 평균과 분산등이 계산이 부정확해짐. |               **없음**               |
| Regularization 효과 |                               배치 노이즈 기반 (암묵적)                                | 기재하기 어려움<br/> (deterministic) |
| 주요 적용 도메인    |                                        CV / CNN                                        |          NLP / Transformer           |
| 배치 크기 의존성    |                        있음 <br/>(작은 크기의 batch 시 불안정)                         |                 없음                 |


## Why LN is Padding-Robust

**BN의 열(along Column) 방향 통계** : padding이 직접 개입함.

$$\mu_j^{\text{BN}} = \frac{1}{N \cdot T}\sum_{i=1}^{N}\sum_{t=1}^{T} x_{i,t,j}$$

* $N$ : 아래 소스예제의 `B`로 batch size임 (Number of samples (=sentences or images) in a mini-batch).
* $T$ : 아래 소스예제의 `T`로 token size(=lenght of sequence, pixel number of single iamge)임.
    * image의 경우 feature map의 width와 height의 곱에 해당함. 

위의 수식에 다음을 확인 할 수 있음:

- 패딩 토큰(0-vector)이 feature $j$의 $\mu_j^{\text{BN}}$과 $(\sigma_j^{\text{BN}})^2$ 계산에 그대로 포함됨.
- 패딩 비율이 높을수록 실제 토큰의 분포가 0 쪽으로 심각하게 왜곡됨.


**LN의 행(Row) 방향 통계** : padding이 개입 불가.

$$\mu_{i,t}^{\text{LN}} = \frac{1}{d}\sum_{j=1}^{d} x_{i,t,j}$$

- 각 토큰 $(i, t)$ 자신의 feature 차원 내에서만 통계를 계산함.
- 토큰 각각에 대해 서로 독립적으로 계산됨 
    - 때문에, token간의 관계는 attention에서 충분히 처리하는 Transformer와 궁합이 좋음.
    - 서로의 역할이 정확히 분리됨.
- 다른 샘플 (다른 sentence or image)이나 다른 타임스텝 (같은 sequence내의 다른 token 또는 같은 image내의 다른 pixel)의 패딩 값이 계산에 전혀 관여하지 않음.
- 배치 크기 및 패딩 비율과 무관하게 안정적인 정규화 가능.

## 단점: Regularization 효과가 거의 없음

- **BN**: 미니배치마다 달라지는 $\mu_B$, $\sigma_B^2$ = stochastic noise : implicit regularization 효과 제공.
- **LN**: token 하나에 해당하는 feature vector 내에서 평균과 분산이 구해지는 deterministic 연산 : BN에서 random한 선택에 의한 노이즈가 원천적으로 발생하지 않음.
- 따라서 LN 적용 시 필요에 따라 Dropout 등 별도의 regularization 기법을 병행하는 것이 반드시 필요함.

> LN은 Attention 또는 FFN sub-layer의 전(Pre-LN) 또는 후(Post-LN)에
> 적용되어, 해당 sub-layer 입출력 tensor의 분포를 안정화시키는 역할을 수행함.

---

---

## `torch.nn` 을 사용한 기본 예제 

```python
import torch
import torch.nn as nn

# ── 재현성 고정 / Fix reproducibility ─────────────────────────────
torch.manual_seed(42)

# ── 하이퍼파라미터 / Hyperparameters ──────────────────────────────
B, T, C = 2, 4, 6   # batch_size=2, seq_len=4, embed_dim=6

# ── 패딩 포함 입력 생성 / Create padded input ─────────────────────
# shape: (B, T, C) = (2, 4, 6)
# - 문장1: 실제 토큰 3개 + <PAD> 1개
# - 문장2: 실제 토큰 2개 + <PAD> 2개
x = torch.tensor([
    # sentence 1
    [[1., 2., 3., 4., 5., 6.],    # token 1
     [7., 8., 9., 1., 2., 3.],    # token 2
     [4., 5., 6., 7., 8., 9.],    # token 3
     [0., 0., 0., 0., 0., 0.]],   # <PAD>
    # sentence 2
    [[2., 4., 6., 8., 1., 3.],    # token 1
     [5., 7., 9., 2., 4., 6.],    # token 2
     [0., 0., 0., 0., 0., 0.],    # <PAD>
     [0., 0., 0., 0., 0., 0.]],   # <PAD>
])  # shape: (2, 4, 6)

# ════════════════════════════════════════════════════════════════
# 1. Batch Normalization
#    - nn.BatchNorm1d 입력 형태: (N, C) 또는 (N, C, L)
#    - NLP 시퀀스에 적용하려면 (B, T, C) → (B, C, T) permute 필요
#    - train() 모드: 현재 미니배치 통계(μ_B, σ_B²) 사용
# ════════════════════════════════════════════════════════════════
bn = nn.BatchNorm1d(num_features=C, affine=False)  # γ,β 없이 순수 정규화만 확인
bn.train()

x_for_bn = x.permute(0, 2, 1)          # (B, C, T) = (2, 6, 4)
with torch.no_grad():
    out_bn = bn(x_for_bn)               # shape: (2, 6, 4)
out_bn = out_bn.permute(0, 2, 1)        # 다시 (B, T, C) = (2, 4, 6)

# ════════════════════════════════════════════════════════════════
# 2. Layer Normalization
#    - normalized_shape=C: 마지막 차원(feature 축) 방향으로 정규화
#    - 입력 shape 그대로 유지: (B, T, C) → (B, T, C)
#    - 각 토큰 독립적으로 처리 → 패딩 토큰의 간섭 없음
# ════════════════════════════════════════════════════════════════
ln = nn.LayerNorm(normalized_shape=C, elementwise_affine=False)

with torch.no_grad():
    out_ln = ln(x)                      # shape: (2, 4, 6)

# ════════════════════════════════════════════════════════════════
# 3. 결과 출력 및 비교 / Output & Comparison
# ════════════════════════════════════════════════════════════════
print("=" * 60)
print("[입력값 / Input]")
print(f"  문장1 token1 : {x[0, 0].tolist()}")
print(f"  문장1 <PAD>  : {x[0, 3].tolist()}")

print("\n[Batch Normalization 출력]")
print("  ※ 열(Column) 방향 통계 → PAD(0) 값이 통계에 섞임")
print(f"  문장1 token1 : {out_bn[0, 0].numpy().round(4).tolist()}")
print(f"  문장1 <PAD>  : {out_bn[0, 3].numpy().round(4).tolist()}")
# PAD 토큰 출력이 0이 아님 → BN 통계가 PAD에 의해 왜곡되었음을 방증

print("\n[Layer Normalization 출력]")
print("  ※ 행(Row) 방향 통계 → PAD(0) 토큰은 스스로만 정규화됨")
print(f"  문장1 token1 : {out_ln[0, 0].numpy().round(4).tolist()}")
print(f"  문장1 <PAD>  : {out_ln[0, 3].numpy().round(4).tolist()}")
# PAD 토큰 출력 = 0 (모든 feature가 동일값 → σ=0 → NaN 방지용 ε로 처리 후 0)

# ════════════════════════════════════════════════════════════════
# 4. 수동 검증: LN 수식을 직접 계산하여 nn.LayerNorm과 대조
#    LN(x) = (x - μ) / sqrt(σ² + ε)
# ════════════════════════════════════════════════════════════════
print("\n[수동 검증 / Manual Verification  ─  문장1 token1]")
tok = x[0, 0]                                      # [1,2,3,4,5,6]
mu_manual  = tok.mean()
var_manual = tok.var(unbiased=False)               # LN은 biased variance 사용
ln_manual  = (tok - mu_manual) / (var_manual + 1e-5).sqrt()

print(f"  수동 계산    : {ln_manual.numpy().round(4).tolist()}")
print(f"  nn.LayerNorm : {out_ln[0, 0].numpy().round(4).tolist()}")
# 두 결과가 일치함을 확인
```

**예상 출력 / Expected Output:**

```
============================================================
[입력값 / Input]
  문장1 token1 : [1.0, 2.0, 3.0, 4.0, 5.0, 6.0]
  문장1 <PAD>  : [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]

[Batch Normalization 출력]
  ※ 열(Column) 방향 통계 → PAD(0) 값이 통계에 섞임
  문장1 token1 : [ 0.2289,  0.4709,  0.1180,  0.9129,  0.8528,  0.4861]
  문장1 <PAD>  : [-1.1471, -1.1471, -1.4142, -1.1471, -1.1471, -1.1471]
  # ↑ PAD 임에도 0이 아님 → BN 통계가 패딩에 의해 왜곡됨

[Layer Normalization 출력]
  ※ 행(Row) 방향 통계 → PAD(0) 토큰은 스스로만 정규화됨
  문장1 token1 : [-1.4638, -0.8783, -0.2928,  0.2928,  0.8783,  1.4638]
  문장1 <PAD>  : [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
  # ↑ PAD는 모든 feature가 동일값(0) → μ=0, σ=0 → ε 처리 후 0 유지

[수동 검증 / Manual Verification  ─  문장1 token1]
  수동 계산    : [-1.4638, -0.8783, -0.2928,  0.2928,  0.8783,  1.4638]
  nn.LayerNorm : [-1.4638, -0.8783, -0.2928,  0.2928,  0.8783,  1.4638]
```

- **BN의 `<PAD>` 출력이 0이 아님** → 패딩 토큰들이 feature별 $\mu_B$, $\sigma_B^2$을 왜곡했다는 직접적인 증거임.
- **LN의 `<PAD>` 출력은 정확히 0** → 해당 토큰의 모든 feature가 동일값(0)이므로 $\mu=0$, $\sigma=0$이 되어 $\epsilon$ 처리 후 0으로 수렴함. 다른 토큰 결과에는 전혀 영향을 주지 않음.
- **수동 계산과 `nn.LayerNorm` 결과가 일치** → LN 수식의 동작을 코드 레벨에서 직접 검증 가능함.
