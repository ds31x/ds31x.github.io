---
layout  : wiki
title   : HF - Dataset for Text
summary : 
date    : 2026-01-25 14:18:45 +0900
updated : 2026-01-25 16:39:51 +0900
tag     : 
resource: BC/167B46711A42F3B3DED8C36A5FBDB3
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}

# Text 기반 Dataset 만들기 - HuggingFace 

## 학습 목표 정리

이 문서는

* **텍스트 분류(Text Classification, 텍스트 분류)** 학습을 위한
* 가장 기본적인 **데이터 준비 단계**에 대해 다룸.

딥러닝 모델은 CSV나 텍스트 파일을 그대로 이해하지 못함.  

* 반드시 **정해진 구조의 데이터셋(Dataset)** 형태로 변환  
* Hugging Face의 **Trainer(트레이너)** 는 
* 일반적으로 **DatasetDict(데이터셋딕트)** 형태로 입력받음.

## 2-1. 실습용 CSV 데이터 준비

### 2-1-1. CSV를 사용하는 이유

CSV(Comma-Separated Values, 쉼표 구분 값)는 다음 장점을 가짐.

- 사람이 직접 읽고 수정하기 쉬움
- 엑셀, 구글 시트 등과 호환 가능
- Hugging Face에서 기본적으로 지원하는 포맷

따라서 학습 데이터의 **출발점으로 가장 적합한 형식**임.

### 2-1-2. 데이터 특성 설명

본 예제 텍스트는 다음 특성을 가짐.

- 문장 구조는 한국어
- 핵심 기술 용어는 영어
- 논문 리뷰, 연구 코멘트, 기술 보고서와 유사한 실제 데이터 형태


> 현대의 [[/nlp/nlp/tokenization]]{토크나이저(Tokenizer)}는  
> 이러한 혼합 언어 입력 (다국어처리)을 문제없이 처리하도록 설계됨.


### 2-1-3. train.csv 예시

```text
text,label
이 논문은 transformer architecture의 scalability를 잘 설명하고 있다,1
모델의 convergence 속도가 느리고 training instability 문제가 보인다,0
실험 결과에서 overfitting 현상이 명확하게 관찰된다,0
proposed method는 baseline 대비 높은 accuracy를 달성했다,1
loss function 설계가 충분히 justified되지 않았다,0
data preprocessing 단계가 systematic하게 잘 구성되어 있다,1
```

* text 컬럼: 모델이 입력으로 읽는 문장
* label 컬럼: 모델이 맞혀야 할 정답

### 2-1-4. validation.csv 예시

```text
text,label
evaluation metric 선택이 타당하지 않아 보인다,0
attention mechanism의 효과가 실험으로 잘 검증되었다,1
```
* validation 데이터는 학습에는 사용되지 않으며,
* 학습 중 모델 성능을 점검하기 위한 검증용 데이터임.

## 2-2. CSV에서 DatasetDict 로딩

```python
from datasets import load_dataset

dd = load_dataset(
    "csv",
    data_files={
        "train": "train.csv",
        "validation": "validation.csv"
    }
)
```

`load_dataset` 함수는 다음 역할을 수행함.

* CSV 파일을 읽음
* 표(table) 구조로 변환함
* `train` / `validation` 분할을 자동 관리함

> 이 시점부터 데이터는 Hugging Face Dataset 형식으로 관리됨.

## 2-3. 샘플 및 컬럼 구조 점검

### 2-3-1. 단일 샘플 확인

```python
print(dd["train"][0])
```

* 이 출력은 모델이 학습 과정에서 실제로 보게 될 입력 데이터의 원형임.

### 2-3-2. 컬럼 목록 확인

```python
print(dd["train"].column_names)
```
* 컬럼(column)은 데이터의 종류를 의미함.
	* `"text"`: 입력 데이터
	* `"label"`: 정답 데이터

### 2-3-3. feature 구조 확인

```python
print(dd["train"].features)
```

* features는 각 컬럼의 자료형과 의미를 정의하는 데이터 설계도에 해당함.
* 이 시점에서는 label이 단순한 숫자로만 인식됨.

## 2-4. 영어 혼재 텍스트에 대한 학습 관점 설명

실제로는 토크나이저 단계에서

* 모든 텍스트가 숫자 토큰(token ID) 으로 변환됨.

아래는 **2.4절 전체를 통합하여 다시 작성한 최종본**입니다.
불필요한 설명 확장 없이, **DatasetDict의 `filter`와 `map` 사용법 + 다국어 tokenizer 적용**만 반영한 형태입니다.
그대로 교체하셔도 됩니다.

---

## 2-4. DatasetDict 객체의 filter와 map 사용법

`DatasetDict(데이터셋딕트)`는 `"train"`, `"validation"` 등의 split을 키로 가지며, 각 값이 `Dataset(데이터셋)` 객체인 구조임.
CSV 기반 데이터셋을 학습에 사용하기 위해서는 이 구조 위에서 `filter`와 `map` 연산을 통해 전처리를 수행하는 것이 기본 흐름임.

---

### 2-4-1. filter의 역할과 사용법

`filter`는 **조건을 만족하지 않는 샘플을 제거하는 연산**임.

데이터 정제 단계에서 다음과 같은 목적으로 사용됨.

* 빈 문장 제거 목적
* 너무 짧은 문장 제거 목적
* 특정 라벨만 선택 목적

아래는 `"text"`가 비어 있지 않고, 최소 길이를 만족하는 샘플만 유지하는 예시임.

```python
def keep_valid_example(ex):
    text = ex["text"]
    return (text is not None) and (len(text.strip()) >= 10)

dd_filtered = dd.filter(keep_valid_example)
```

특정 라벨만 남기는 경우의 예시는 다음과 같음.

```python
def keep_positive(ex):
    return ex["label"] == 1

dd_filtered = dd.filter(keep_positive)
```


### 2-4-2. map의 역할과 사용법

* `map`은 각 샘플을 변환하여 **새로운 컬럼을 생성하거나 기존 컬럼을 가공하는 연산**임.
* 텍스트 분류 학습에서는 주로 **토크나이징(tokenization)** 단계에서 사용됨.

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained(
    "xlm-roberta-base"
)
```

각 샘플의 `"text"`를 입력으로 받아 **Token ID** 와 **Attention Mask** 를 생성함.

```python
def tokenize_example(ex):
    return tokenizer(
        ex["text"],
        truncation=True,
        max_length=128,
        padding="max_length",
    )

dd_tok = dd_filtered.map(tokenize_example)
```

이 연산 이후 각 샘플은 다음 컬럼을 포함함.

* `text`
* `label`
* `input_ids`
* `attention_mask`

### 2-4-3. batched map 사용법

대량 데이터 처리 시에는 `batched=True` 옵션을 사용하는 것이 일반적임.
* `batched=True`시 배치로 묶어서 처리하여 함수 호출의 횟수가 줄어듬.
* 이는 대량 데이터 처리시 성능 향상으로 이어짐.
* 이 방식은 전처리 속도와 메모리 사용 측면에서 유리함.

> `num_proc=3` 과 같이 멀티프로세스로 동작시킬 경우, 더 빠른 처리가 가능.

```python
def tokenize_batch(batch):
    return tokenizer(
        batch["text"],
        truncation=True,
        max_length=128,
        padding="max_length",
    )

dd_tok = dd_filtered.map(
    tokenize_batch,
    batched=True,
    batch_size=1000,
)
```
* 참고로, `tokenizer`에서 `padding=False`, `truncation=False`가 기본임.

### 2-4-4. remove_columns 사용법

학습 단계에서 `"text"` 원문 컬럼을 사용하지 않는 경우, 
`remove_columns` 메서들로 특정 컬럼들을 제거 할 수 있음.
이는 데이터 크기를 줄여줌.

```python
dd_tok = dd_tok.remove_columns(["text"])
```

또는 `map` 단계에서 동시에 제거할 수도 있음.

```python
dd_tok = dd_filtered.map(
    tokenize_batch,
    batched=True,
    remove_columns=["text"],
)
```




### 2-4-5. 예제

CSV 기반 `DatasetDict` 객체의 전처리의 전형적인 예는 다음과 같음:

```python
dd_filtered = dd.filter(keep_valid_example)
dd_tok = dd_filtered.map(
    tokenize_batch,
    batched=True,
    remove_columns=["text"],
)
```
* 이후 결과 객체는 `Trainer` 입력으로 바로 사용 가능한 `DatasetDict` 형태를 유지함.













## 2-5. label을 ClassLabel로 명시

* 현재 label은 단순한 숫자이므로 의미가 불분명함.
* `ClassLabel`을 사용하면 숫자와 의미 있는 이름을 공식적으로 연결.

### 2-5-1. 라벨 의미 정의

```text
0 → negative (부정적 평가)
1 → positive (긍정적 평가)
```

### 2-5-2. ClassLabel 적용

```python
from datasets import ClassLabel

label_feature = ClassLabel(names=["negative", "positive"])
dd2 = dd.cast_column("label", label_feature)
```
* `cast_column`은 데이터셋에게
* "이 컬럼은 이런 의미를 가진다" 고 선언하는 과정임.

### 2-5-3. 변환 결과 확인

```python
print(dd2["train"].features)
```
* label 값은 여전히 숫자처럼 보이지만,
* 내부적으로는 숫자와 라벨 이름의 대응 관계가 저장됨.

## 2-6. 라벨 분포 점검

```python
from collections import Counter
print(Counter(dd2["train"]["label"]))
```

* 이 단계는 클래스 불균형(class imbalance)을 확인하기 위한 절차임.
* 불균형이 심하면 모델 성능에 직접적인 영향을 미침.

## 2-7. 학습용 DatasetDict 점검 체크리스트

다음을 통해, Trainer 가 사용할 수 있는 DatasetDict 객체인지를 최소 조건을 검증:

```
assert "train" in dd2
assert "validation" in dd2
assert "text" in dd2["train"].column_names
assert "label" in dd2["train"].column_names
```

