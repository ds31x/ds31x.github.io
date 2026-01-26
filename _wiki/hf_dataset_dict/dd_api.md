---
layout  : wiki
title   : Dataset 과 DatasetDict 
summary : 
date    : 2026-01-26 19:20:02 +0900
updated : 2026-01-26 19:23:17 +0900
tag     : 
resource: 7D/FA6C86D3FE49DB9BE8F87E90CBDC6F
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}


# Dataset과 DatasetDict 주요 메서드 정리 및 예제

본 문서는 Hugging Face Datasets 라이브러리의 

* `Dataset`과 
* `DatasetDict` 의
* 객체에서 제공하는 **주요 메서드들의 동작 방식**을 소개하는 문서.



## 0. 예제 데이터 및 기본 객체 생성

모든 예제는 아래와 같은 CSV 구조의 데이터를 기반으로 함.

```python
from datasets import Dataset, DatasetDict

data = {
    "text": [
        "이 논문은 transformer architecture의 scalability를 잘 설명하고 있다",
        "모델의 convergence 속도가 느리고 training instability 문제가 보인다",
        "실험 결과에서 overfitting 현상이 명확하게 관찰된다",
        "proposed method는 baseline 대비 높은 accuracy를 달성했다",
        "loss function 설계가 충분히 justified되지 않았다",
        "data preprocessing 단계가 systematic하게 잘 구성되어 있다",
    ],
    "label": [1, 0, 0, 1, 0, 1],
}

dataset = Dataset.from_dict(data)

dataset_dict = DatasetDict({
    "train": dataset.select([0, 1, 2, 3]),
    "validation": dataset.select([4, 5]),
})
```


## 1. 전처리 및 변환 메서드

### 1.1 map

* 지원: **Dataset + DatasetDict**
* 목적: 샘플 단위 또는 배치 단위 변환 수행 목적

#### 단일 샘플 단위 변환

```python
def add_length(example):
    return {"length": len(example["text"])}

dataset_mapped = dataset.map(add_length)
dataset_mapped[0]
```

#### 배치 단위 변환

```python
def add_length_batch(batch):
    return {"length": [len(t) for t in batch["text"]]}

dataset_mapped = dataset.map(
    add_length_batch,
    batched=True,
    batch_size=2,
)
```

#### DatasetDict 전체에 동일 적용

```python
dataset_dict_mapped = dataset_dict.map(
    add_length_batch,
    batched=True,
)
```



### 1.2 filter

* 지원: **Dataset + DatasetDict**
* 목적: 조건을 만족하는 샘플만 유지 목적

```python
def keep_long(example):
    return len(example["text"].strip()) >= 10

dataset_filtered = dataset.filter(keep_long)
```

```python
dataset_dict_filtered = dataset_dict.filter(keep_long)
```



## 2. 순서 및 분할 관련 메서드

### 2.1 shuffle

* 지원: **Dataset + DatasetDict**
* 목적: 데이터 순서 무작위화 목적

```python
dataset_shuffled = dataset.shuffle(seed=42)
```

```python
dataset_dict_shuffled = dataset_dict.shuffle(seed=42)
```



### 2.2 sort

* 지원: **Dataset + DatasetDict**
* 목적: 특정 컬럼 기준 정렬 목적

```python
dataset_sorted = dataset.sort("label")
```



### 2.3 select

* 지원: **Dataset 전용**
* 목적: 인덱스 기반 부분 선택 목적

```python
dataset_selected = dataset.select([0, 3, 5])
```



### 2.4 shard

* 지원: **Dataset 전용**
* 목적: 데이터 분할 및 부분 처리 목적

```python
dataset_shard0 = dataset.shard(num_shards=2, index=0)
dataset_shard1 = dataset.shard(num_shards=2, index=1)
```



### 2.5 train_test_split

* 지원: **Dataset 전용**
* 반환: `DatasetDict`
* 목적: 학습/검증 데이터 분리 목적

```python
split = dataset.train_test_split(
    test_size=0.3,
    seed=42,
)
```



## 3. 컬럼 조작 메서드

### 3.1 remove_columns

* 지원: **Dataset + DatasetDict**
* 목적: 불필요 컬럼 제거 목적

```python
dataset_no_text = dataset.remove_columns(["text"])
```

```python
dataset_dict_no_text = dataset_dict.remove_columns(["text"])
```



### 3.2 rename_column

* 지원: **Dataset + DatasetDict**
* 목적: 단일 컬럼 이름 변경 목적

```python
dataset_renamed = dataset.rename_column("label", "y")
```



### 3.3 rename_columns

* 지원: **Dataset + DatasetDict**
* 목적: 다수 컬럼 이름 일괄 변경 목적

```python
dataset_renamed = dataset.rename_columns({
    "text": "sentence",
    "label": "y",
})
```



### 3.4 select_columns

* 지원: **Dataset 전용**
* 목적: 일부 컬럼만 선택 목적

```python
dataset_text_only = dataset.select_columns(["text"])
```



### 3.5 cast / cast_column

* 지원: **Dataset 전용**
* 목적: feature(schema) 변경 목적

```python
from datasets import ClassLabel, Features, Value

features = Features({
    "text": Value("string"),
    "label": ClassLabel(names=["neg", "pos"]),
})

dataset_cast = dataset.cast(features)
```



## 4. 포맷 및 프레임워크 연동 메서드

### 4.1 with_format

* 지원: **Dataset + DatasetDict**
* 목적: 반환 포맷 지정 목적

```python
dataset_torch = dataset.with_format(
    "torch",
    columns=["label"],
)
```



### 4.2 set_format / reset_format

* 지원: **Dataset + DatasetDict**
* 목적: 포맷 설정 및 초기화 목적

```python
dataset.set_format("numpy", columns=["label"])
dataset.reset_format()
```



### 4.3 with_transform

* 지원: **Dataset 전용**
* 목적: 조회 시점(on-the-fly) 변환 적용 목적

```python
def to_upper(example):
    example["text"] = example["text"].upper()
    return example

dataset_transformed = dataset.with_transform(to_upper)
```



## 5. 저장 및 배포 메서드

### 5.1 save_to_disk / load_from_disk

* 지원: **Dataset + DatasetDict**
* 목적: 로컬 저장 및 재로딩 목적

```python
from datasets import load_from_disk

dataset.save_to_disk("tmp_dataset")
dataset_loaded = load_from_disk("tmp_dataset")
```



### 5.2 to_csv / to_json / to_parquet

* 지원: **Dataset 전용**
* 목적: 외부 포맷 내보내기 목적

```python
dataset.to_csv("dataset.csv")
dataset.to_json("dataset.json")
dataset.to_parquet("dataset.parquet")
```



### 5.3 push_to_hub

* 지원: **Dataset + DatasetDict**
* 목적: Hugging Face Hub 업로드 목적

```python
# dataset.push_to_hub("username/my-dataset")
# dataset_dict.push_to_hub("username/my-dataset-dict")
```



## 6. DatasetDict 컨테이너 동작

```python
dataset_dict.keys()
dataset_dict["train"][0]

for split, ds in dataset_dict.items():
    print(split, len(ds))
```



## 정리

* `Dataset`은 단일 테이블 기반 데이터 처리 객체임.
* `DatasetDict`는 split 단위 Dataset을 묶는 컨테이너임.
* 일부 메서드는 Dataset 전용이며, DatasetDict는 split 단위 위임 방식으로 동작함.
* `map`과 `filter`는 학습 파이프라인의 핵심 전처리 연산임.



필요하시면 다음 단계로

* **Trainer 입력 관점 정리**
* **collate_fn vs map 책임 분리**
* **대규모 데이터에서 num_proc 설계 가이드**

중 하나를 이어서 독립 문서로 정리해 드릴 수 있습니다.
 
