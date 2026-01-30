---
layout  : wiki
title   : Language Model Taxonomy - Pretraining Paradigms and Encoder-Decoder Architectures
summary : 
date    : 2026-01-30 15:50:18 +0900
updated : 2026-01-30 23:37:46 +0900
tag     : 
resource: 7F/408E6E724C430BA5A84F387A28B40A
toc     : true
public  : true
parent  : [[/index]]
latex   : false
---
* TOC
{:toc}

# Post Transformers

## Autoencoding, Autoregressive, Encoder–Decoder 모델의 역할 구분

Transformer 이후의 언어모델은

* 단순히 "큰 모델"로 발전한 것이 아니라,
* **사전학습 목표(pretraining objective)**와
* **모델 구조(model architecture)**의 조합에 따라
* 서로 다른 계열로 분화되어 왔음.

본 글에서는 언어모델을 다음 두 축으로 동시에 정리함.

1. **사전학습 방식**
   * Autoencoding Language Model(자기부호화 언어모델)
   * Autoregressive Language Model(자기회귀 언어모델)
   * Denoising Language Model(잡음 제거 언어모델)

2. **구조적 형태**
   * Encoder-only
   * Decoder-only
   * Encoder–Decoder

이를 통해 각 모델이 

* **왜 특정 태스크에 강한지**, 그리고 
* **어떤 inductive bias(귀납적 편향)**를 가지는지를 설명함.

---

## 1. 사전학습 패러다임의 기원: ULMFiT (2018)

### ULMFiT (Universal Language Model Fine-tuning, 2018)

ULMFiT는 현대 사전학습 언어모델의 **개념적 출발점**에 해당함.

주요 기여는 다음과 같음.

* 대규모 말뭉치 기반 Language Model 사전학습
* Discriminative fine-tuning(판별적 미세조정): 레이어별 학습률 차등 적용
* Gradual unfreezing(점진적 해동): 하위 레이어부터 순차적 학습
* Slanted triangular learning rates(비대칭 삼각 학습률): 빠른 수렴 전략
* "사전학습 → 미세조정" 패러다임 정립

비록 **3-layer AWD-LSTM 기반**이었으나,
BERT와 GPT 계열 모두에 영향을 준 선행 연구로 평가됨.

특히 **transfer learning(전이학습)**이
NLP에서도 Computer Vision과 같이 효과적임을 실증한 점이 핵심임.

> 참고: [ULMFit : Transfer Learning for NLP](https://dsaint31.tistory.com/956)

---

## 2. Autoencoding Language Model + Encoder-only 구조

### 2.1 개념 정의

**Autoencoding Language Model(자기부호화 언어모델)**은

* 입력의 일부를 손상(corrupt)시키고,
* **양방향 문맥(bidirectional context)**을 활용하여 
* 복원하거나 판별하도록 학습된 모델임.

대표적인 사전학습 목표는 다음과 같음.

* **Masked Language Modeling (`MLM`, 마스크 언어모델링)**:  
  입력 토큰의 15%를 `[MASK]`로 치환한 뒤 원본 복원
* **Next Sentence Prediction (`NSP`, 다음 문장 예측)**:  
  두 문장의 연속성 판별 (`BERT`에서 사용, 이후 `RoBERTa`에서 제거됨)
* **Replaced Token Detection (`RTD`, 대체 토큰 탐지)**:  
  `ELECTRA`에서 사용, generator가 생성한 그럴듯한 토큰을 판별

이 계열은 대부분 **Encoder-only Transformer** 구조를 사용함.


### 2.2 구조적 특징

* **Bidirectional Self-Attention(양방향 자기어텐션)**: 모든 토큰이 서로를 참조 가능
* 입력 전체 동시 접근(parallel processing)
* **Dense representation(밀집 표현)** 생성에 최적화
* Token-level classification(토큰 단위 분류)에 적합
* 출력 생성 능력은 구조적으로 제한적
* `[CLS]` 토큰을 통한 **sequence representation(시퀀스 표현)** 추출


### 2.3 대표 모델과 등장 연도

| 모델 | 연도 | 파라미터 | 비고 |
|------|------|---------|------|
| `BERT-base` | 2018 | 110M | MLM + NSP, 12-layer |
| `BERT-large` | 2018 | 340M | 24-layer |
| `RoBERTa` | 2019 | 125M~355M | NSP 제거, 동적 마스킹 |
| `ALBERT` | 2020 | 12M~235M | Parameter sharing, 경량화 |
| `DeBERTa` | 2020 | 134M~1.5B | Disentangled attention |
| `DeBERTa-v3` | 2021 | 86M~1.5B | ELECTRA-style 개선 |
| `ELECTRA` | 2020 | 14M~335M | 판별형 사전학습, 효율적 |
| `KoBERT` | 2019 | 92M | 한국어 특화, SKT |
| `koELECTRA` | 2020 | 14M~110M | 한국어 ELECTRA, KLUE 상위권 |
| `KcBERT` | 2020 | 110M | 댓글 데이터(Beep/News) |
| `HanBERT` | 2020 | 614M | 형태소 기반, 54만 어휘 |


### 2.4 KLUE 및 code-mixed 환경에서의 위치

**KLUE (Korean Language Understanding Evaluation)** 벤치마크는
다음 8개 태스크로 구성됨.

* Topic Classification(주제 분류, `TC`)
* Semantic Textual Similarity(의미 유사도, `STS`)
* Natural Language Inference(자연어 추론, `NLI`)
* Named Entity Recognition(개체명 인식, `NER`)
* Relation Extraction(관계 추출, `RE`)
* Dependency Parsing(의존 구문 분석, `DP`)
* Machine Reading Comprehension(기계 독해, `MRC`)
* Dialogue State Tracking(대화 상태 추적, `DST`)

`Autoencoding + Encoder-only` 계열은 다음 태스크에서 구조적 강점을 가짐.

* **Named Entity Recognition(개체명 인식, `NER`)**: BIO tagging 등 token-level labeling
* **Token Classification(토큰 분류)**: POS tagging, chunking
* **Sequence Classification(시퀀스 분류)**: sentiment analysis, topic classification
* **Span Extraction(구간 추출)**: `MRC`에서 답변 위치 탐지

특히 **`koELECTRA-base-v3`**는
KLUE-NER에서 **F1 87.92**를 기록하며
**한국어+영어 혼재(`code-mixed`) 문장** 에서도
높은 샘플 효율(sample efficiency)을 보이는 것으로 알려짐.

이는 
* **subword tokenization(서브워드 토큰화)** 과
* **양방향 문맥 활용**의 조합이
* OOV(Out-Of-Vocabulary) 문제와
* entity boundary detection(개체 경계 탐지) 에 유리하기 때문임.

> 참고: [[/nlp/tokenization]]


## 3. Autoregressive Language Model + Decoder-only 구조

### 3.1 개념 정의

**Autoregressive Language Model(자기회귀 언어모델)**은

* 이전 토큰들 $x_1, x_2, \ldots, x_{t-1}$에 조건부로
* **다음 토큰 $x_t$를 예측**하도록 학습된 모델임.

수식으로는 다음과 같이 표현됨.

$$P(x_1, x_2, \ldots, x_T) = \prod_{t=1}^{T} P(x_t \mid x_1, \ldots, x_{t-1})$$

대표적인 사전학습 목표는 **Causal Language Modeling (`CLM`, 인과 언어모델링)**임.

이 계열은 **Decoder-only Transformer** 구조를 사용하며,
**GPT (Generative Pre-trained Transformer)** 시리즈가 대표적임.


### 3.2 구조적 특징

* **Causal (Unidirectional) Self-Attention(인과적 단방향 자기어텐션)**: 미래 토큰 참조 불가
* **Autoregressive generation(자기회귀 생성)**: 한 토큰씩 순차 생성
* 학습과 추론 형식의 일치(training-inference consistency)
* 텍스트 생성에 특화된 구조
* **In-context learning(문맥 내 학습)** 능력 발현 (모델 크기 증가 시)
* **Few-shot/Zero-shot learning(소수샷/제로샷 학습)** 가능


### 3.3 대표 모델과 등장 연도

| 모델 | 연도 | 파라미터 | 비고 |
|------|------|---------|------|
| GPT | 2018 | 117M | 초기 생성형, 12-layer |
| GPT-2 | 2019 | 124M~1.5B | 스케일 확장, zero-shot |
| GPT-3 | 2020 | 125M~175B | 초거대 모델, few-shot |
| GPT-3.5 | 2022 | ~175B | RLHF 적용, ChatGPT |
| GPT-4 | 2023 | 미공개 | 멀티모달, 추론 강화 |
| LLaMA | 2023 | 7B~65B | 공개 계열, efficient training |
| LLaMA-2 | 2023 | 7B~70B | 개선판, 상용 라이선스 |
| LLaMA-3 | 2024 | 8B~70B | 확장된 context, 다국어 |
| Mistral 7B | 2023 | 7B | Grouped Query Attention |
| Mixtral 8x7B | 2024 | 47B | Mixture of Experts |


### 3.4 Summarization 태스크에서의 강점 설명

Summarization(요약)은

* 입력 문서 전체를 이해한 뒤
* **새로운 압축된 문장을 생성**하는 태스크임.

Autoregressive 모델은 다음 이유로 요약에 적합함.

1. **Prefix conditioning**: 입력을 완전한 context로 받은 뒤 생성 시작
2. **Fluent generation(유창한 생성)**: 자연스러운 문장 구성
3. **Abstractive summarization(추상적 요약)**: 원문에 없는 표현 생성 가능
4. **Instruction following(지시 수행)**: "다음 문서를 요약하시오" 형태의 프롬프트 처리

다만 **입력 길이 제한**이 있으므로,
긴 문서의 경우 **chunking(분할)**이나
**hierarchical summarization(계층적 요약)**이 필요함.

**GPT-3.5 이후**에는 **instruction tuning(지시 튜닝)**과
**RLHF (Reinforcement Learning from Human Feedback, 인간 피드백 기반 강화학습)**를 통해
요약 품질이 크게 향상됨.

## 4. Denoising Language Model + Encoder–Decoder 구조

### 4.1 독립적 사전학습 패러다임으로 분류하는 이유

Encoder–Decoder 모델은

* **Denoising Autoencoding(잡음 제거 자기부호화)** 또는
* **Sequence-to-Sequence Pretraining(시퀀스-투-시퀀스 사전학습)**이라는
* **독립적인 사전학습 방식**을 사용함.

이는 단순히 Autoencoding과 Autoregressive를 조합한 것이 아니라,  
고유한 학습 목표를 가짐:

* **입력**: 손상된 시퀀스(corrupted sequence)
* **Encoder**: 양방향으로 손상된 입력 이해
* **Decoder**: 단방향으로 원본 시퀀스 순차 복원
* **핵심**: **cross-attention(교차 어텐션)**을 통해 encoder 표현을 decoder가 참조

수식:
$$\mathcal{L} = -\log P(x \mid \tilde{x}) = -\sum_{t=1}^{T} \log P(x_t \mid \tilde{x}, x_{<t})$$

여기서 $\tilde{x} = C(x)$는 손상 함수(corruption function)를 적용한 입력.

### 4.2 대표적 사전학습 목표

#### T5의 Span Corruption

**`T5` (Text-to-Text Transfer Transformer, 2020)**는
**Span Corruption** 방식을 사용함.

**원리**:
1. 입력의 **연속된 토큰들(span)**을 sentinel 토큰으로 치환
2. 평균 span 길이: 3 토큰, 손상 비율: 15%
3. Decoder는 sentinel 순서대로 원본 span 복원

**예시**:
```
원본: "Thank you for inviting me to your party last week"

Encoder 입력: "Thank you <extra_id_0> me to your party <extra_id_1> week"

Decoder 출력: "<extra_id_0> for inviting <extra_id_1> last <extra_id_2>"
```

**특징**:
- BERT의 MLM보다 적은 토큰으로 효율적 학습
- 모든 태스크를 Text-to-Text 형식으로 통일

#### BART의 Denoising Autoencoding

**`BART` (Bidirectional and Auto-Regressive Transformers, 2019)**는
**다양한 손상 기법**을 조합함.

**5가지 손상 유형**:

| 손상 유형 | 설명 | 예시 |
|----------|------|------|
| **Token Masking** | 무작위 토큰을 `[MASK]`로 치환 | "I `[MASK]` pizza" |
| **Token Deletion** | 무작위 토큰 삭제 | "I pizza" (love 삭제) |
| **Text Infilling** | 연속 토큰을 단일 `[MASK]`로 치환 | "I `[MASK]` pizza" |
| **Sentence Permutation** | 문장 순서 섞기 | [Sent3, Sent1, Sent2] |
| **Document Rotation** | 임의 토큰에서 시작하도록 회전 | "pizza. I love" |

**실제 설정**:

- Text Infilling + Sentence Permutation 주로 사용
- 30% 토큰 손상
- Span 길이는 Poisson 분포($\lambda=3$)

#### 기타 방식

**`PEGASUS` (2020)**:
- **Gap Sentence Generation (GSG)**: 중요한 문장을 제거하고 복원
- 요약 태스크에 직접 최적화

**`MASS` (2019)**:
- 연속 span만 마스킹, 해당 부분만 복원
- 번역 태스크 특화

**`UL2` (2022)**:
- **Mixture-of-Denoisers**: 여러 손상 방식 동시 학습
- R-denoising, X-denoising, S-denoising 혼합

### 4.3 세 패러다임의 본질적 차이

**Autoencoding (`BERT`)**:
```
입력: "I `[MASK]` pizza"
목표: P(love | context) 
방식: 각 `[MASK]` 위치에서 독립적으로 병렬 예측
```

**Autoregressive (`GPT`)**:
```
입력: "I love"
목표: P(pizza | I, love)
방식: 다음 토큰 순차 예측
```

**Denoising Encoder-Decoder (`T5`, `BART`)**:
```
Encoder 입력: "I <X> pizza" (손상됨)
Decoder 목표: P(love | corrupted_input, previous_output)
방식: 손상된 전체를 보고 원본을 순차 복원
```


### 4.4 대표 모델과 등장 연도

| 모델         | 연도 | 파라미터  | 비고                     |
|--------------|------|-----------|--------------------------|
| `BART`       | 2019 | 140M~400M | Denoising, 5가지 노이즈  |
| `BART5`      | 2020 | 60M~11B   | Text-to-Text, C4 corpus  |
| `BARmBART`   | 2020 | 610M      | 다국어, 25개 언어        |
| `BARmT5`     | 2021 | 300M~13B  | 다국어 T5, 101개 언어    |
| `BARFLAN-T5` | 2022 | 80M~11B   | 지시 학습, 1,800+ 태스크 |
| `BARUL2`     | 2022 | 20B       | Mixture-of-Denoisers     |
| `BARPEGASUS` | 2020 | 568M      | Gap Sentence Generation  |


### 4.5 강점을 보이는 태스크

Encoder–Decoder 구조는 다음 태스크에서 강점을 보임.

* **Summarization(요약)**: 긴 입력 이해 + 간결한 출력 생성
* **Translation(번역)**: 소스 언어 인코딩 + 타깃 언어 디코딩
* **Question Answering(질의응답)**: 문서 이해 + 답변 생성
* **Data-to-Text**: 구조화 데이터 → 자연어 변환
* **Instruction Following(지시 수행)**: 명령 이해 + 실행
* **Conditional Generation(조건부 생성)**: 제약 조건 하 생성

특히 **입력과 출력의 길이와 형식이 상이**하거나,
**입력 전체를 동시에 참조**해야 하는 태스크에서
Decoder-only 모델 대비 안정적인 성능을 보임.

**요약 태스크 우위 이유**:

1. **Encoder**: 긴 문서(500+ 토큰)를 양방향으로 완전 이해
2. **Cross-attention**: 출력 생성 시 입력 전체 참조
3. **Variable-length output**: 입력과 다른 길이 자유롭게 생성
4. **Explicit conditioning**: 조건(문서)과 생성(요약)의 명확한 분리

**실증**:

- `PEGASUS`: CNN/DailyMail ROUGE-L **44.17**
- `BART`: XSum ROUGE-L **45.14**
- `T5`: 다양한 요약 벤치마크에서 일관된 상위권


### 4.6 한국어 환경에서의 활용

한국어 Encoder–Decoder 모델은 다음과 같음.

* **`KoBART` (2021)**: SKT, `BART` 기반, 요약·생성 태스크
* **`KE-T5` (2021)**: ETRI, 한국어 `T5`, 150GB 말뭉치
* **`mT5` 다국어 모델**: 한국어 포함, zero-shot 전이 가능

KLUE에서는 **`MRC`(기계 독해)** 태스크가  
Encoder–Decoder 구조에 적합하나,  
실제로는 **Encoder-only + span extraction** 방식이  
더 많이 사용됨.


## 5. 사전학습 방식 + 구조 기준 통합 요약

| 구조 | 사전학습 방식 | Attention 패턴 | 대표 모델 | 강점 태스크 | 약점 |
|------|-------------|--------------|---------|----------|------|
| Encoder-only | **Autoencoding** (MLM, RTD) | Bidirectional | `BERT`, `ELECTRA`, `koELECTRA` | `NER`, 분류, 추출 | 생성 불가 |
| Decoder-only | **Autoregressive** (CLM) | Causal (Unidirectional) | `GPT`, `LLaMA` | 생성, 요약, 대화 | 양방향 이해 제한 |
| Encoder–Decoder | **Denoising** (Span Corruption, Text Infilling) | Bidirectional + Causal + Cross | `T5`, `BART` | 요약, 번역, 조건부 생성 | 계산량 증가 |

### 사전학습 목표의 핵심 차이

| 사전학습 방식 | 입력 상태 | 출력 방식 | 학습 신호 | 최적화 대상 |
|-------------|---------|----------|----------|-----------|
| **Autoencoding** | 손상됨 | Parallel prediction | 각 `[MASK]` 위치 | 양방향 이해 |
| **Autoregressive** | 정상 | Sequential generation | 다음 토큰 | 유창한 생성 |
| **Denoising** | 손상됨 | Sequential generation | 전체 원본 시퀀스 | 이해 + 생성 균형 |


## 6. 모델 선택 가이드라인

### 6.1 태스크별 권장 구조

* **Token Classification (`NER`, POS tagging)**: Encoder-only (`koELECTRA`)
* **Sequence Classification (sentiment, topic)**: Encoder-only (`BERT`)
* **Text Generation (creative writing, dialogue)**: Decoder-only (`GPT`)
* **Summarization (short input)**: Decoder-only (`LLaMA`) 또는 Encoder–Decoder (`T5`)
* **Summarization (long input)**: Encoder–Decoder (`BART`, `PEGASUS`)
* **Translation**: Encoder–Decoder (`mBART`, `mT5`)
* **Question Answering (extractive)**: Encoder-only (`BERT`)
* **Question Answering (generative)**: Encoder–Decoder (`FLAN-T5`) 또는 Decoder-only (`GPT`)

### 6.2 한국어 특화 고려사항

* **형태소 분석 필요 시**: `HanBERT` (형태소 기반 어휘)
* **댓글·비속어 처리**: `KcBERT` (댓글 말뭉치)
* **일반 도메인**: `koELECTRA` (효율성), `RoBERTa-large` (성능)
* **생성 태스크**: `KoGPT-2`, `Polyglot-Ko`
* **요약·번역**: `KoBART`, `KE-T5`
* **KLUE 벤치마크**: `koELECTRA`, `klue/roberta-large`

### 6.3 Code-Mixed 환경 전략

한국어+영어 혼재 문장 처리 시 고려사항.

* **Multilingual tokenizer**: `XLM-R`, `mBERT` 활용
* **Language-specific model + romanization**: 영어 부분을 한국어 토크나이저로 처리
* **Separate encoding**: 언어별 인코딩 후 fusion
* **Pretrained on mixed data**: 혼재 말뭉치로 추가 학습

`koELECTRA 는 **한국어 중심이나 영어 subword도 일부 포함**하여  
code-mixed NER에서 실용적 성능을 보임.


## 7. 최근 동향 (2023~2024)

### 7.1 Decoder-only 모델의 지배

**LLaMA, Mistral, Qwen** 등
Decoder-only 모델이 **instruction tuning** 후
다양한 태스크에서 Encoder-only 모델을 능가함.

이는 다음 요인에 기인함.

* **Scaling law**: 모델 크기 증가 시 범용 성능 향상
* **In-context learning**: Few-shot으로 새 태스크 학습
* **Instruction following**: 자연어 명령 이해

### 7.2 Encoder-Decoder의 효율성 재평가

**T5 논문 실험** (동일 데이터, 동일 계산량):

| 모델 구조 | 사전학습 방식 | GLUE 점수 | 비고 |
|----------|-------------|----------|------|
| Encoder-only | MLM (BERT-style) | 82.3 | 기준선 |
| Decoder-only | CLM (GPT-style) | 79.1 | 생성 태스크 강점 |
| Encoder-Decoder | Span Corruption | **84.7** | 균형적 우수성 |

특정 태스크(요약, 번역)에서는
**Encoder-Decoder가 여전히 효율적**임.

### 7.3 Mixture of Experts (MoE)

**Mixtral 8x7B**는  
8개의 전문가(expert) 중 2개를 동적으로 선택하는  
**sparse model(희소 모델)** 구조로  
계산 효율과 성능을 동시에 달성함.  

### 7.4 Long Context Models

**`LLaMA-3` (8K tokens)**, **`Claude 3` (200K tokens)**,  
**`Gemini 1.5` (1M tokens)** 등  
**context window** 확장이 주요 경쟁 요소가 됨.

긴 문서 요약, 코드 분석 등에서 유리함.

### 7.5 Multimodal Integration

**`GPT-4V`, `Gemini`, `Claude 3`**는
텍스트+이미지 동시 처리가 가능하며,
**Vision Transformer (`ViT`)**를 통합한 구조임.


## 8. 정리

### 8.1 핵심 결론

* 언어모델의 본질적 차이는 **사전학습 목표와 구조의 조합**에 있음
* **`Autoencoding + Encoder-only`** 구조는 이해·판별·추출 태스크에 적합함
* **`Autoregressive + Decoder-only`** 구조는 생성·대화·few-shot 학습에 적합함
* **`Denoising + Encoder–Decoder`** 구조는 손상된 입력을 복원하는 독립적 패러다임으로, 번역·요약·조건부 생성에 강점을 보임
* 한국어 `KLUE-NER` 및 `code-mixed` 환경에서는 **Autoencoding 계열**(`koELECTRA`, `klue/roberta`)이 정석적 선택임
* 최근에는 **Decoder-only + instruction tuning**이 범용 모델로 부상하나, 특정 태스크에서는 여전히 Encoder-only 또는 Encoder-Decoder가 효율적
* 이 모든 흐름의 출발점에는 **`ULMFiT`의 전이학습 패러다임**이 존재함

### 8.2 세 패러다임의 관계

```
ULMFiT (2018)
    ↓
전이학습 패러다임 정립
    ↓
    ├─→ Autoencoding (BERT 2018)
    │   - MLM으로 양방향 이해 학습
    │   - Encoder-only 구조
    │   - 이해·분류 태스크 특화
    │
    ├─→ Autoregressive (GPT 2018)
    │   - CLM으로 다음 토큰 예측
    │   - Decoder-only 구조
    │   - 생성 태스크 특화
    │
    └─→ Denoising (BART 2019, T5 2020)
        - Span Corruption으로 복원 학습
        - Encoder-Decoder 구조
        - 조건부 생성·변환 태스크 특화
```

