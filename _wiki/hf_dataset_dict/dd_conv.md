---
layout  : wiki
title   : From Torch Dataset To HF Dataset
summary : 
date    : 2026-01-26 19:35:50 +0900
updated : 2026-01-26 19:55:07 +0900
tag     : 
resource: EF/4EF331C2884EFF9FD6FEECF18F48CB
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}


## PyTorch Dataset에서 HF Dataset으로 전환

### 학습 목표

* 기존 `torch.utils.data.Dataset`을 Hugging Face(HF) 학습 파이프라인에 연결하는 방법 이해
* 데이터 규모에 따라 적절한 변환 전략을 선택하는 기준 이해


### 학습 내용

* `torch.utils.data.Dataset`의 구조적 특성 이해
* HF `Dataset`의 column 기반 데이터 모델 이해
* 소규모 실습 경로와 대용량 실무 경로의 구분 필요성 이해


### 실습 핵심

* (소규모) PyTorch Dataset → `list[dict]` → `Dataset.from_list`
* (대용량) Generator 기반 변환 또는 파일 기반 변환
* `DatasetDict` 구성

itermediate datatype 으로 `list`를 사용하면 간단하지만, RAM에 모든 데이터를 적재하는 부담이 있음.


## 1 torch.utils.data.Dataset의 구조와 한계

`torch.utils.data.Dataset`은 다음과 같은 특성을 가짐.

* index 기반 접근 구조 (`__len__`, `__getitem__`)
* 샘플 단위 반환 구조
* 컬럼 개념이 명시적으로 존재하지 않음

전형적인 PyTorch Dataset 구조는 다음과 같음.

```python
from torch.utils.data import Dataset

class MyTorchDataset(Dataset):
    def __init__(self, texts, labels):
        self.texts = texts
        self.labels = labels

    def __len__(self):
        return len(self.texts)

    def __getitem__(self, idx):
        return {
            "text": self.texts[idx],
            "label": self.labels[idx],
        }
```

* `torch.utils.data.Dataset`의 경우, PyTorch `DataLoader`에는 적합
* HF 생태계에서는 직접적인 호환성 부족 문제가 발생함.


## 2 HF Dataset의 column 기반 구조

HF `Dataset`은 다음과 같은 구조적 특징을 가짐.

* **column 중심** 데이터 표현
* 동일 길이 컬럼 배열 구조
* Arrow 기반 **캐시 및 메모리 관리**
* `map`, `filter` 중심 전처리 파이프라인 지원 (개발자 편의 기능 풍부)

개념적 구조는 다음과 같음.

```text
Dataset
 ├─ text  : [...]
 └─ label : [...]
```


## 3 변환 전략 선택의 기준

PyTorch Dataset을 HF Dataset으로 전환할 때의 핵심 판단 기준은 **데이터 규모**임.

| 데이터 규모            | 권장 변환 전략              |
| :---: | --- |
| 소규모(실습, 수만 샘플 이하) | `list[dict]` 기반 변환    |
| 대용량(수십만~수백만 샘플)   | **Generator** 또는 **파일 기반** 변환 |

* [Generator 참고자료](https://dsaint31.tistory.com/501#Generator-1-4)

## 4 소규모 실습 경로: `list[dict]` 기반 변환

### 목적

* 구조 이해
* 빠른 실습
* 코드 단순성 확보

### 변환 과정

```python
def torch_dataset_to_list(torch_dataset):
    records = []
    for i in range(len(torch_dataset)):
        records.append(torch_dataset[i])
    return records
```

```python
records = torch_dataset_to_list(torch_ds)
```

위에서 만든 `list`객체를 바탕으로 HF Dataset 객체 생성.


### HF Dataset 생성

```python
from datasets import Dataset

hf_dataset = Dataset.from_list(records)
```

> 이 방식은 **전체 데이터를 메모리에 적재**하므로 대용량에는 부적합함.


## 5 대용량 실무 경로 (1): Generator 기반 변환

### 목적

* 메모리 사용 최소화
* PyTorch Dataset 구조 유지

### 변환 방식

```python
from datasets import Dataset

def gen_from_torch(torch_dataset):
    for i in range(len(torch_dataset)):
        yield torch_dataset[i]

hf_dataset = Dataset.from_generator(
    gen_from_torch,
    gen_kwargs={"torch_dataset": torch_ds},
)
```

### 특징

* 메모리 사용량 안정적임
* 변환 속도는 앞서의 `list` 방식보다 느림.
* 이후 `map`, `filter` 사용 가능함


## 6 대용량 실무 경로 2: 파일 기반(JSONL) 변환

### 목적

* 재현성 확보
* 반복 실험에 유리한 구조 확보
* 장기 저장 및 공유 가능성 확보

JSONL (JSON Lines) 형식을 많이 이용함.

* 배열 기반의 일반 JSON과 달리, 한 줄(line)이 한 JSON 객체인 형태.
* 한 줄씩 처리하는 스트리밍 처리가 가능함.

### JSONL 기반 예시

```python
import json

with open("train.jsonl", "w", encoding="utf-8") as f:
    for i in range(len(torch_ds)):
        f.write(json.dumps(torch_ds[i], ensure_ascii=False) + "\n")
```

```python
from datasets import load_dataset

dd = load_dataset(
    "json",
    data_files={"train": "train.jsonl"},
)
```

### 특징

* 디스크 기반 처리로 메모리 안정성 확보
* 대규모 데이터셋에 가장 안전한 경로
* Appach의 Arrow Table을 사용하는 HF의 Dataset에 적합.


## 7 DatasetDict 구성

### PyTorch Dataset 단계에서 분리하는 경우

```python
dd = DatasetDict({
    "train": Dataset.from_generator(gen_from_torch, gen_kwargs={"torch_dataset": train_ds}),
    "validation": Dataset.from_generator(gen_from_torch, gen_kwargs={"torch_dataset": val_ds}),
})
```

이미 나누어져 있는 경우에 유리함.


### HF Dataset 이후 분리하는 경우

```python
dd = hf_dataset.train_test_split(
    test_size=0.2,
    seed=42,
)
```

## 8 Summary

* PyTorch Dataset은 index 기반 구조임
* HF Dataset은 column 기반 구조임
* `list[dict]` 변환은 소규모 실습용 경로임
* 대용량에서는 Generator 또는 파일 기반 변환이 필수임
* 기존 PyTorch Dataset 구현을 유지한 채 HF Trainer로 확장 가능함




