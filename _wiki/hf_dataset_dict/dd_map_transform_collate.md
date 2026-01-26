---
layout  : wiki
title   : map, set_transform, collate_fn
summary : 
date    : 2026-01-26 20:02:22 +0900
updated : 2026-01-26 20:19:52 +0900
tag     : 
resource: 85/ADA5140E654A1CB3298C725CB3EEAE
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}


# map vs set_transform vs collate_fn (Training 단계)

* 학습 파이프라인에서 Preprocessing 위치를 선택하는 기준 확립
* HF Trainer에서 `data_collator`가 필요한 구조적 이유 이해


## 학습 내용

* `map`: 사전 전처리 및 캐싱 중심 전처리 방식
* `set_transform`: epoch별 동적 전처리 중심 방식
* `collate_fn`: 배치 단위 처리 중심 방식


## 결론부터?

* 고정 전처리 용: `map`
* Random Data Augmentation : `set_transform`
* 가변 길이 / 멀티모달 → `collate_fn`

## 1 학습 파이프라인에서 전처리 위치 문제

HF 학습 파이프라인은 개념적으로 다음 단계를 가짐.

```text
Dataset
 → (Preprocessing)
 → DataLoader
 → Model
```

이때 "Preprocessing"는 **하나의 위치가 아니라 여러 계층에 분산**될 수 있음.

HF에서는 preprocessing 을 다음 세 위치 중 하나에 둘 수 있음.

1. Dataset 단계 (`map`)
2. Dataset 접근 단계 (`set_transform`)
3. 배치 구성 단계 (`collate_fn`)

각 위치는 **의미와 책임이 명확히 다름**.



## 2 map: 사전 전처리 및 캐싱 중심

* `map`은 **Dataset 전체에 대해 한 번 수행되는 전처리 연산**임.
* 전처리 결과는 **Arrow 테이블로 캐싱**됨.

참고: [[/hf_dataset_dict/dd_map]]{HF의 Dataset, DatasetDict에서의 map 에 대한 자료}

```python
dataset = dataset.map(tokenize_fn, batched=True)
```

### 특성

* 전처리 결과가 고정됨
* epoch가 바뀌어도 결과 동일함
* 캐시 재사용 가능함
* 대용량 데이터에서 효율적임


### 적합한 작업

* tokenizer 적용
* label 변환
* 고정 정규화
* 고정 feature 생성


### 단점

* epoch마다 다른 결과 생성 불가
* Random Data Augmentation 에 사용하기엔 부적합


## 3 set_transform: epoch별 동적 전처리

* `set_transform`은 **Dataset에서 샘플을 꺼낼 때마다 호출되는 함수**를 지정함.
* Arrow 테이블은 변경되지 않음.

참고: 특히, image에서 [torchvision.transforms](https://ds31x.tistory.com/367) 와 같이 사용되기 쉬움.

```python
def transform(example):
    example["text"] = example["text"].upper()
    return example

dataset = dataset.with_transform(transform)
```

### 특성

* epoch마다 다른 결과 가능함
* 랜덤성 포함 가능함
* 캐시 미적용됨
* `__getitem__` 단계에서 수행됨

### 적합한 작업

* Random Data Augmentation 을 구현하는데 최적.
* stochastic masking
* epoch-dependent 변환

### 단점

* CPU 부하 증가 가능성
* 대용량 데이터에서 속도 저하 가능성
* batch 단위 정보 접근 불가 (하나의 sample에 적용됨)


## 4 collate_fn: Batch 단위 처리

* `collate_fn`은 **여러 샘플을 하나의 Batch 로 묶는 함수**임.
* PyTorch `DataLoader` 단계에서 호출됨.
* HF Trainer에서는 이를 `data_collator`라는 이름으로 사용함.

참고: [PyTorch의 DataLoader에서의 `collate_fn` 에 대한 참고 자료](https://ds31x.tistory.com/234#2-1.-dataset%EC%9C%BC%EB%A1%9C%EB%B6%80%ED%84%B0-dataloader-%EB%A7%8C%EB%93%A4%EA%B8%B0-)

```python
def collate_fn(batch):
    texts = [ex["text"] for ex in batch]
    labels = [ex["label"] for ex in batch]
    return tokenizer(
        texts,
        padding=True,
        truncation=True,
        return_tensors="pt",
    ) | {"labels": labels}
```

### 특성

* batch 전체를 동시에 접근 가능함
* 가변 길이 처리 가능함
* 멀티모달 데이터 통합 가능함
* GPU 친화적 입력 구성 가능함



### 적합한 작업

* dynamic padding
* image + text 결합
* audio + text 결합
* batch-level augmentation

### 단점

* Dataset 캐시 불가
* collate_fn 복잡도 증가 가능성



## 5 세 방식의 역할 비교

| 항목    | map    | set_transform | collate_fn |
| --- | ---  | --- | --- |
| 실행 시점 | 사전 전처리 | 샘플 접근 시       | 배치 구성 시    |
| 캐시 사용 | 가능     | 불가            | 불가         |
| 랜덤성   | 부적합    | 적합            | 적합         |
| 배치 정보 | 접근 불가  | 접근 불가         | 접근 가능      |
| 가변 길이 | 제한적    | 제한적           | 적합         |
| 멀티모달  | 제한적    | 제한적           | 적합         |


## 6 Trainer에서 `data_collator`가 필요한 이유

HF Trainer는 다음 전제를 가짐.

* Dataset은 **column 단위로 저장됨**
* 각 샘플은 **독립적으로 관리됨**
* 배치 구성은 Trainer 외부에서 결정됨

따라서 Trainer는 다음 문제를 해결해야 함.

* 서로 길이가 다른 입력 정렬
* batch 단위 padding
* modality별 tensor 결합

이 책임을 수행하는 구성 요소가 **data_collator**임.


## 7 실무적 전처리 분업 원칙

학습 파이프라인에서의 권장 책임 분리는 다음과 같음.

* 고정적이며 반복 가능한 전처리 : `map`
* sample 인스턴스에 접근할 때마다 달라야 하는 전처리 : `set_transform`
* 배치 단위로만 가능한 처리 : `collate_fn`


## 8 핵심 정리

* `map`은 데이터셋 자체를 바꾸는 전처리임
* `set_transform`은 데이터를 꺼내는 방식을 바꾸는 전처리임
* `collate_fn`은 배치를 만드는 규칙임
* Trainer의 존재 이유는 **배치 구성 책임 분리**에 있음


 
