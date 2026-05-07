---
layout  : wiki
title   : Tokenizer 
summary : 
date    : 2026-01-20 14:09:40 +0900
updated : 2026-02-19 08:47:42 +0900
tag     : token BPE WordPiece UnigramLM 
resource: 6E/305305E7CE447E926358B6D98F19C9
toc     : true
public  : true
parent  : [[/nlp]]
latex   : true
---
* TOC
{:toc}

# Tokenization

## Intro.

Tokenization은 자연어 처리(NLP)에서 텍스트를 모델이 처리 가능한 토큰(token)의 시퀀스(sequence)로 변환하는 과정.

tokenization을 단순한 전처리 단계로 오인되기도하지만, 실제로는 

* 이후 기계학습 모델의 비용 구조,
* 연산 복잡도, 학습 난이도,
* 표현력 전반에 매우 큰 영향을 주는 매우 중요한 단계임.

일반적인 token과 tokenizer에 대한 간단한 내용은 다음의 url을 참고:  

* [Token and Tokenizer](https://ds31x.tistory.com/140)

### Q1: 오늘날 Tokenization의 기본 단위는 왜 character나 word가 아닌 subword 단위를 사용할까?

* Character tokenization은 sequence length 증가로 연산 비용(= $O(L^2)$ )이 폭증하고 단어 경계나 형태소 등 언어 구조가 완전히 소실되어 이후 layer 등의 학습 부담이 큼.
* Word tokenization은 data sparsity와 OOV (Out Of Vocabulary) 문제가 심각함.
* 적절한 vocabulary 크기(30K-50K)로 의미 단위를 보존하면서 OOV를 구조적으로 해결하는 subword가 최적의 균형점임.

> **데이터 희소성(Data Sparsity)** 이란,
> 모델이 다루는 특징 공간(feature space)의 크기에 비해 실제 학습 데이터에서 관측된 사례(instance)가 지극히 적어,
> 대부분의 특징 조합(feature combination)이 0이거나 통계적으로 신뢰할 수 없을 만큼 낮은 빈도를 가지는 현상을 가리킴.

### Q2: Vocabulary 크기는 왜 중요할까?

* Vocabulary 크기(cardinality of vacabulary) $\|V\|$ 는 embedding matrix의 크기( $\|V\| \times d$ )에 직접적으로 영향을 줌.
* 이 크기는 결국 전체 모델의 파라미터 수와 메모리 사용량에 큰 영향을 미침. 
* vocabulary 크기는 tokenization 방식(character/word/subword)의 선택, OOV 처리 능력, 그리고 sequence length 간의 trade-off를 반영하여 결정.

### Q3: Contextual embedding은 tokenization과 어떤 관계일까?

* 현대의 contextual embedding 모델(BERT, GPT 등)은 사실상 모두 subword tokenization을 전제로 설계됨.
* 최소한의 의미 단위를 가진 subword embedding을 초기 입력으로 사용함. 
* 이 subword embedding이 Transformer의 self-attention layer를 거치면서 주변 context 정보를 통합하여 최종적으로 context에 따라 다른 embedding으로 변형됨.

## 1. Tokenization과 Embedding의 구조적 관계

### 1.1 전체 처리 흐름

일반적인 NLP 모델의 처리 흐름은 다음과 같음:

```text
text (문자열)
 ↓
tokenization (텍스트 분할)
 ↓
token IDs (정수 시퀀스)
 ↓
embedding lookup (벡터 변환)
 ↓
model processing (Transformer/RNN/CNN)
 ↓
output (예측 결과)
```

### 1.2 역할의 명확한 구분

**Tokenization**

* 텍스트를 이산적인 토큰 ID 시퀀스로 변환
* 모델이 인식하는 최소 단위를 정의
* 예: `"tokenization" → [5432, 1234, 8901]`

**Embedding**

* 토큰 ID를 연속 벡터 공간의 벡터(이를 dense vector라고 부름)로 매핑
* 예: `5432 → [0.23, -0.45, 0.67, ...]`

> Embedding 에 대한 보다 자세한 내용은 다음을 참고할 것:
>
> * [Embedding이란?](https://dsaint31.tistory.com/821)
> * [Embedding의 수학적 정의 및 Embedding Layer](https://dsaint31.tistory.com/820)

## 2. Vocabulary와 Embedding Matrix의 정의

### 2.1 Vocabulary (어휘집)

**정의:**

* 특정 tokenizer가 사용하는 토큰들의 집합
* Tokenizer 학습 단계(training)에서 corpus로부터 구축됨
* 각 토큰은 고유한 정수 ID를 가짐

**참고사항:**

1. Tokenizer Training (한 번):
   `Corpus → 알고리즘 (BPE/WordPiece 등) → Vocabulary 생성`
2. Tokenization (매번):
   `Text → 이미 정의된 Vocabulary 사용 → Token IDs`


**Vocabulary Size(크기):**

* $\|V\|$ 로 실제 사용가능한 전체 Token의 개수임.
* BERT-base의 vocabulary 크기는 약 30,000개임.

### 2.2 Embedding Matrix


**정의:**

* Vocabulary에 포함된 각 토큰을 벡터로 변환하기 위한 matrix

형태:

$$
\textbf{e} \in \mathbb{R}^{|V| \times d}
$$

* $\textbf{e}$ : embedding 
* $\|V\|$ : vacabulary size
* $d$ : dimension of embedding (예: BERT는 768)

**Embedding lookup 연산:**

* Token ID를 index로 하여
* Embedding matrix의 해당 row를 선택하는 연산
* 수학적으로는 one-hot encoding과 행렬 곱셈이지만
* 실제 구현에서는 indexing으로 효율적으로 처리

다시 한번 애기하지만,  

* Tokenizer 학습 단계에서 vocabulary가 정의됨.
* 이 vocabulary가 embedding matrix의 크기를 결정.

> tokenization 방식의 선택이 단순히 텍스트 처리 방식뿐만 아니라 
> 모델의 메모리 사용량과 파라미터 수에도 직접적인 영향을 미침.

## 3. Static Embedding과 Contextual Embedding

### 3.1 Static Embedding 

2013년부터 2010년대에 많이 사용된 방식.

**개념:**

* 각 token이 context와 무관하게 하나의 **고정된 벡터(=static embedding)**를 가짐
* Token 단위 = Vocabulary 단위 = 최소 의미 단위

**대표 모델:**

* Word2Vec (Mikolov et al., 2013)
* GloVe (Pennington et al., 2014)

**구조:**

```
"bank" → [0.23, -0.45, 0.67, ...] (항상 같은 벡터)
```

**문제점:**

```
"I went to the bank to deposit money."  (은행)
"I sat by the river bank."              (강둑)
```

두 문장에서 "bank"는 다른 의미지만, static embedding에서는 같은 벡터를 가짐.

### 3.2 Contextual Embedding (2018–)

**개념:**

* `Token ID → Embedding lookup` 까지는 static embedding과 동일
* 이후 Transformer 등의 모델 내부 층을 거치며 context에 따라 embedding이 변형

**대표 모델:**

* ELMo (2018)
* BERT (2018)
* GPT 계열 (2018–)

**동작흐름:**

```
text
 → subword tokenization
 → subword IDs
 → embedding lookup (static, 초기 embedding)
 → Transformer layers (self-attention으로 context 반영)
 → contextualized embeddings (최종 출력)
```

**주의:**

"contextual embedding"이라는 이름 때문에
embedding lookup 단계 자체가 context를 반영한다고 오해해선 안됨.

실제 동작:
1. Embedding lookup은 여전히 static (각 token ID → 고정 벡터)
2. Context 정보는 Transformer의 self-attention에서 형성
3. 여러 layer를 거치면서 주변 토큰 정보를 통합
4. 최종 출력 embedding이 contextual embedding

### 3.3 Contextual Embedding과 Subword Tokenization의 관계

**중요한 사실:**

> 현대의 contextual embedding 모델은  
> 사실상 모두 subword tokenization을 전제로 설계되어 있음.

Contextual embedding이 아무리 강력해도:

* 입력 token이 의미 없는 단위라면 학습이 어려움.
* Character tokenization: 구조가 완전히 사라져서 학습 부담이 커짐. ↑
* Word tokenization: OOV 문제와 vocabulary 폭발.

이를 해결하기 위해 subword 의 등장:

```
Subword tokenization
  → 최소한의 의미 단위 제공
  → Contextual embedding이 효과적으로 동작
```

## 4. Tokenization 단위에 따른 분류

Tokenization은 기본 단위에 따라 다음과 같이 구분됩니다:

* Character tokenization: 문자 단위
* Word tokenization: 단어 단위
* Subword tokenization: 단어의 일부 단위

| 단위 | Vocab 크기 | Seq Length | 구조 보존 | OOV | 비고 |
| :---: | :---: | :---: | :---: | :---: | :---: |
| Character |~100 |매우 긴 길이| 없음 | 없음 | 학습 부담 매우 큼| 
| Word      | ~1M | 짧음 | 완벽 | 심각 | Vocab 폭발 |
| Subword | ~30-50K | 적절 | 부분 | 없음 |최적 균형 |

## 5. Character Tokenization

### 5.1 개념

**정의:**

* 토큰 단위: 문자(character)

**예제:**

```
"tokenization"
→ ['t', 'o', 'k', 'e', 'n', 'i', 'z', 'a', 't', 'i', 'o', 'n']
```

### 5.2 Vocabulary와 Embedding Matrix 관점

**장점처럼 보이는 것:**

* Vocabulary 크기가 매우 작음 (수십~수백 개)
* Embedding matrix 크기 $\|V\| \times d$가 작음

**예제:**

```
영어 알파벳: 26개 (소문자)
+ 대문자: 26개
+ 숫자: 10개
+ 특수문자: ~30개
= 약 100개 정도
```

따라서,

```
Embedding matrix = 100 × 768 ≈ 77K parameters
```

이는 BERT의 전체 110M parameters에 비하면 매우 작은 값임.

### 5.3 전체 모델 비용 구조에서의 문제

**오해:**

* "Embedding matrix가 작으니까 character tokenization이 메모리 효율적이다"

**실제:**

* Transformer 모델에서 메모리 크기를 결정하는 요소는 embedding matrix가 아님.


**비교:**

* Self-Attentino 연산
	* 복잡도: $O(L^2 \times d)$
	* $L$ : sequence length
	* $d$ : hidden dimension

* Intermediate hidden states
	* 각 layer 마다  $L \times d$크기의 tensor 저장.
* KV cache (Key Value cache)
	* 각 layer 마다 $2 \times L \times d$ 크기.

요구되는 메모리가 sequence length $L$에 큰 영향을 받음.

### 5.4 구조 소실 문제

Character tokenization의 더 근본적인 문제는 text의 구조 정보가 소실된다는 점임.

**예제:**
```
"unbelievable"
→ ['u', 'n', 'b', 'e', 'l', 'i', 'e', 'v', 'a', 'b', 'l', 'e']
```

이 경우, 다음의 정보가 tokenization을 수행하고 나선 모두 소실됨:
* `un-`: 부정 접두사 (negation prefix)
* `believe`: 핵심의미 (core semantic unit)
* `-able`: 기능성 접미사 (ability suffix)
* 단어 경계 (word boundary)

이후 character token을 입력받는 모델은

* 문자들의 sequence로부터
* 단어 경계,
* 형태소 구조
* 의미 단위 등을
* internal representation에에 담아내기 위해 더 많은 학습이 요구됨.

#### 중요

> Character tokenization은
> Token 단계에서 제공할 수 있었던 구조 정보를
> 전부 모델 학습 부담으로 전가시킴.

이는 다음을 의미:

* 학습 난이도 증가
* 데이터 요구량 증가
* 더 깊은 모델 구조 필요
* 학습 시간 증가

### 5.5 Character Embedding의 의미적 한계

**문제:**

* 문자 하나의 embedding은 의미적으로 거의 해석 불가능
* 의미는 여러 문자, 여러 layer, 여러 step을 거쳐서야 형성

**Static vs Contextual embedding:**

* Static embedding: Character 단위에서는 의미 표현 사실상 불가능
* Contextual embedding: 이론적으로 가능하지만 의미있는 representation을 얻기까지 거쳐야 하는 연산단계(=computational path)가 너무 길어서 비효율적임.

**결론:**

오늘날 Character tokenization은 특수 영역에서만 사용:

OCR (노이즈가 많은 텍스트)
Biological sequence (DNA, protein)
Extremely noisy text

## 6. Word Tokenization

### 6.1 개념

**정의:**

토큰 단위: 단어(word)

예제:
```
"Tokenization is important"
→ ['Tokenization', 'is', 'important']
```

### 6.2 장점

다음의 명확한 장점을 가짐:

* 언어 구조와 의미 단위가 명확히 보존됨
* 사람이 이해하기 쉬움
* 언어학적으로 자연스러움

### 6.3 단점

* Data Sparsity
* OOV
* 매우 큰 Embedding matrix
* 희귀 단어 처리 어려움.


#### 1. Data Sparsity (데이터 희소성)

실제 언어는 단어가 무한정 생성될 수 있음:

```
walk, walks, walked, walking, walker, ...
compute, computes, computed, computing, computer, computation,
```

**매우 많은 수의 영어 단어 수:**

* Oxford English Dictionary: ~170,000 단어
* 실제 사용 (변형형 포함): 수십만~수백만

이는 각 단어의 학습 샘플이 쉽게 부족해진다는 뜻임.

Corpus는 고정된 상태에서, Vocabulary가 커질수록 각 단어가 corpus에 등장하는 횟수가 감소함:

```
# 예시: 1억 단어 corpus
corpus_size = 100,000,000

# Vocabulary = 10,000일 때
평균 등장 횟수 = 100,000,000 / 10,000 = 10,000번
# 이는 충분한 학습 샘플

# Vocabulary = 1,000,000일 때  
평균 등장 횟수 = 100,000,000 / 1,000,000 = 100번
# 이는 학습 샘플 부족을 보임.
```

**Zipf's Law로 인한 문제 악화:**

단어 빈도는 Zipf's law를 따라 극단적으로 불균형합니다:

* 상위 1% 단어: 전체 corpus의 ~60% 차지
* 하위 50% 단어: 전체 corpus의 ~5% 차지

결과적으로:

```
Vocabulary = 1,000,000일 때
- 상위 10,000 단어: 평균 6,000번 등장 (학습 가능)
- 하위 500,000 단어: 평균 10번 이하 (학습 불가능)
```

이는 Embedding matrix의 크기 증가로 이어짐:

```
Embedding matrix = |V| × d
                 = 1,000,000 × 768
                 = 768M parameters (embedding만)
```

* BERT-base 전체가 110M parameters인 것과 비교할 경우,
* 위의 경우, embedding만으로 7배 더 큰 모델이 됨.

#### 2. OOV (Out-of-Vocabulary) 문제

Training에서 보지 못한 단어는 처리 불가:

```
Training: ["cat", "dog", "bird"]
Test: "I saw a giraffe"
→ "giraffe"를 어떻게 처리?
```

일반적인 해결책:

* `<UNK>` 토큰 사용
* 하지만 의미 정보가 완전히 손실됨

### 6.4 Static Embedding 시대의 한계

Word tokenization은 static embedding 시대의 핵심 한계 요소.

Word2Vec, GloVe 등은:

* Word 단위 필수
* OOV 문제 해결 불가
* Vocabulary 크기 제한 필수

이 한계가 subword tokenization 개발의 동기가 됨.

## 7. Subword Tokenization: 설계적 타협

### 7.1 핵심 아이디어

Subword tokenization은 character와 word 사이의 최적의 균형점을 찾음.

**목표:**

* Vocabulary 크기 통제 (보통 30K-50K)
* OOV 문제 구조적 완화
* Embedding에 의미를 실을 수 있는 최소 단위
* Sequence length의 과도한 증가 방지
* 텍스트 구조의 부분적 보존

### 7.2 동작 원리

**기본 개념:**

```
"unbelievable"
→ Word: ['unbelievable']  (OOV 위험)
→ Subword: ['un', '##believ', '##able']  (의미 단위 보존)
→ Character: ['u','n','b','e'...]  (구조 소실)
```

**핵심:**

* 자주 등장하는 단어 → 하나의 토큰
* 희귀 단어 → 더 작은 단위로 분해
* 모든 단어를 기본 단위(character 또는 byte)로 분해 가능 → OOV 제거

### 7.3 Subword Tokenization의 장점

1. OOV 문제 해결
	* 모든 단어를 기본 단위(character/byte)로 분해 가능
	* 이는 이론적으로 OOV 제거를 의미.
2. Vocabulary 크기 제어
	* Character: ~100개 (너무 작음)
	* Word: ~1,000,000개 (너무 큼)
	* Subword: ~30,000-50,000개 (적절)
3. 의미 단위 부분 보존
	* `"unhappiness"` → `['un', '##happiness']`
	* 앞서 예에서 보이듯이 부정 접두사와 핵심 의미 분리가 가능함.
4. Sequence length 적절
	* Character 대비: 5-10배 짧음
	* Word 대비: 1.5-2배 길지만 OOV 없음

## 8. 주요 Subword Tokenization 알고리즘

### 8.1 BPE (Byte Pair Encoding, 1994/2016)

Byte Pair Encoding (BPE)는 빈도 기반의 데이터 압축 알고리즘으로 개발되었으나, 
오늘날에는 Tokenization에서도 유용하게 사용됨. 

**역사:**

* 1994: 데이터 압축 알고리즘으로 개발
* 2016: Sennrich et al.이 NLP에 적용

**동작 원리:**

* 초기 vocabulary = 모든 character
* 가장 자주 등장하는 문자 쌍(pair) 찾기
	* 빈도가 높은 쌍부터 병합	
* 해당 쌍을 하나의 새 토큰으로 병합
* 원하는 vocabulary 크기에 도달할 때까지 반복

### 8.2 WordPiece (2016)

BPE와 유사한 방식이나 빈도(frequency)가 아닌 Language model likelihood 를 기준으로 병합하는 방식으로 Google BERT에서 사용됨.

* Wu et al. (2016)
* Google에서 개발
* BERT, DistilBERT 등에서 사용

**BPE와의 차이:**

* BPE: 빈도 기반 병합
* WordPiece: Language model likelihood 증가 기준

**병합 기준:**

가장 높은 score를 갖는 symbol pair를 병합함.

$$\text{score}(a,b) = \frac{\text{freq}(a,b)}{\text{freq}(a) \times \text{freq}(b)}$$

- 분자 $\text{freq}(a,b)$는 두 symbol이 함께 등장한 빈도를 의미함.
- 분모 $\text{freq}(a) \times \text{freq}(b)$는 각 symbol이 개별적으로 얼마나 자주 등장하는지를 보정함.
- 따라서 단순히 자주 등장하는 pair가 아니라, **각 symbol의 개별 빈도에 비해 함께 등장하는 경향이 강한 pair**가 우선적으로 병합됨.

WordPiece는 단순 빈도만 보지 않고, 두 symbol의 개별 빈도를 분모로 보정하여 “개별적으로 흔한 symbol끼리 우연히 자주 붙은 경우”보다 “서로 결합성이 강한 pair”를 우선 병합함.

**특징:**

* 단어 내부 토큰을 `##`으로 표시
* 예: `["un", "##believ", "##able"]`


### 8.3 Unigram LM (2018)

Unigram LM(Unigram Language Model)은 

* 가능한 여러 subword 분해 후보에 대한 확률을 계산하고,
* 전체 확률이 가장 높은 조합을 선택하는 확률 기반 subword tokenization 방식임.*
* T5, ALBERT 등에서 사용

> **n-gram:** 연속된 n개의 token을 하나의 묶음으로 취급하는 일종의 단위임.
>
> **Unigram:** n-gram에서 n=1인 경우. 즉, 개별 token 하나를 가리킴.
>
> **Unigram LM:**
> * unigram(개별 token)만을 단위로 사용하는 언어 모델.
> * 개별 token의 확률만 다루므로, 결과적으로 각 token이 독립이라는 가정이 됨

* Kudo (2018)
* SentencePiece의 기본 알고리즘

**BPE와의 근본적 차이:**

* BPE: 병합(merge) 방식 - 작은 단위에서 시작하여 확장
    * Bottom-Up 방식 
* Unigram LM: 가지치기(pruning) 방식 - 큰 단위에서 시작하여 축소
    * Top-Down 방식 

**동작 원리:**

* corpus에서 Suffix Array를 이용하여 자주 반복 등장하는 substring을 대량 추출하고,
* 여기에 모든 개별 문자를 합쳐 목표 vocabulary 크기의 2~10배에 해당하는 과잉 후보 집합(seed vocabulary)을 구축하여 시작
* EM 알고리즘으로 각 subword의 확률 $p(x_i)$를 추정
	* 모든 분할 방식을 확률 비중에 따라 고려하여 기대 빈도를 계산하고,
    * 이를 정규화하여 확률을 갱신하는 과정을 수렴할 때까지 반복
* corpus likelihood 기여도가 낮은 subword를 제거(**pruning**)
* 목표 vocabulary 크기에 도달할 때까지 반복

참고로 Unigram LM 에서의  초기 vocabulary는 보통

* 문자(character)
* 자주 등장하는 문자 n-gram

Unigram LM의 핵심은 

* "처음부터 확정된 vocabulary"가 아니라 과잉 생성된 candidate set에서 불필요한 token을 제거해가는 방식이라는 점임.
* BPE/WordPiece와 달리 확률 분포를 가지므로, 학습 시 다양한 분할을 샘플링하여 regularization 효과를 얻을 수 있다는 점(subword regularization)도 차별점임.

> 최신 LLM에서는 Byte-level BPE(BBPE)가 단순성, 속도, 안정성 면에서 주류로 사용되고 있음.
> Unigram LM의 고유한 장점은 확률 분포 기반이므로
> 학습 시 다양한 분할을 샘플링할 수 있다는 점(subword regularization)이며,
> 이것이 BPE의 결정론적 분할과의 근본적 차이임
>
> SentencePiece는 Unigram LM과 BPE를 모두 지원하는 라이브러리이며,
> * T5/XLNet 등은 SentencePiece + Unigram을,
> * Gemma/Gemini 등은 SentencePiece + BBPE를 사용함.
>
> 공백 없는 언어(한국어, 일본어, 중국어 등)의 처리는
> SentencePiece 라이브러리 자체의 특성(raw text 직접 처리)이며,
> Unigram LM 알고리즘 고유의 장점은 아님 (SentencePiece + BBPE도 동일하게 처리 가능).

### 8.4 SentencePiece (2018)

**참고:**

> SentencePiece는 tokenization 단일 알고리즘이 아니라 프레임워크.
> **최대 장점은 pretokenizer 없이 raw text에서 시작할 수 있다는 특징** 임.

* Kudo & Richardson (2018)
* Google에서 개발

**핵심 특징:**

* **언어 독립적: 공백을 포함한 raw text에서 직접 학습 (SentencePeice의 특징)**
* Pre-tokenizer 불필요: 언어별 규칙 없이 동작
* 알고리즘 선택 가능: BPE 또는 Unigram LM 지원

**Tokenization 분류상 위치:**

- Tokenization 단위: Subword
- 알고리즘: BPE (BBPE도 사용가능) 또는 Unigram LM
- 도구/프레임워크: SentencePiece

> SKTBrain의 KoBERT가 Unigram LM기반의 SentencePiece를 사용함.  
> MarianMT도 주로 SentencePiece를 사용.

### 8.5 BBPE (Byte-level BPE, 2019)

* Radford et al. (2019)
* GPT-2에서 도입
* GPT-3, GPT-4 등 모든 GPT 계열에서 사용

**핵심 아이디어:**

* 기본 단위를 character가 아닌 byte로 설정
* UTF-8 인코딩의 256개 byte가 base vocabulary

**장점:**

* 사실상 OOV 제거: 모든 Unicode 문자 표현 가능
* 다국어 처리 강화: 한글, 중국어, 이모지 등 동일하게 처리
* 특수문자 처리: 수학 기호, 특수 문자 등 자연스럽게 처리

## 9. 비교

BPE, WordPiece, Unigram LM, Byte-level BPE는 tokenization algorithm임.

이와 달리, SentencePiece는 whitespace pre-tokenization 없이 raw text를 직접 처리할 수 있는 tokenizer toolkit/framework임.

Tokenization algorithm 비교는 다음과 같음.

| 알고리즘 | 기본 단위 | 방식 | 결정성 | 주요 사용 모델 |
| :---: | :---: | :---: | :---: | :---: |
| BPE | Character | 빈도 기반 병합 | 결정적 | GPT 초기 계열, 일부 MT |
| WordPiece | Character | likelihood 기반 병합 | 결정적 | BERT, DistilBERT, mBERT |
| Unigram LM | Character / Subword candidate | 확률 기반 pruning | 일반적으로 결정적. <br/> sampling 사용 시 확률적 | T5, ALBERT, XLNet |
| Byte-level BPE | Byte | 빈도 기반 병합 | 결정적 | GPT-2, RoBERTa, <br/> GPT-3 계열 |

## 10. References

* BPE: Sennrich et al. (2016). "Neural Machine Translation of Rare Words with Subword Units"
* WordPiece: Wu et al. (2016). "Google's Neural Machine Translation System"
* Unigram LM: Kudo (2018). "Subword Regularization: Improving Neural Network Translation Models with Multiple Subword Candidates"
* SentencePiece: Kudo & Richardson (2018). "SentencePiece: A simple and language independent approach"
* GPT-2 (BBPE): Radford et al. (2019). "Language Models are Unsupervised Multitask Learners"
* HuggingFace Tokenizers: [https://huggingface.co/docs/tokenizers](https://huggingface.co/docs/tokenizers)
* SentencePiece: [https://github.com/google/sentencepiece](https://github.com/google/sentencepiece)
