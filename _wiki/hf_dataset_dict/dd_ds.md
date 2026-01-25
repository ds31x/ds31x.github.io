---
layout  : wiki
title   : load_dataset
summary : 
date    : 2026-01-25 14:39:24 +0900
updated : 2026-01-25 16:26:46 +0900
tag     : 
resource: F9/EC2A8F313A4C07ADBE2F55F3AB31C8
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}


# `load_dataset`으로 Dataset과 DatasetDict 익히기

`DatasetDict`는 split(train, validation, test 등)을 key로 갖는 딕셔너리 구조

* 각 value는 `Dataset` 객체임.
* HF 에서의 표준 컨테이너 역할임.

`load_dataset` 은 가장 많이 사용되는 Hugging Face의 `DatasetDict` 및 `Dataset`을 얻는 기본방식임.


## 1. 학습 목표:

* Hugging Face Dataset(`Dataset`, 데이터셋)과 DatasetDict(`DatasetDict`, 데이터셋딕트)의 구조 이해
* `load_dataset`의 동작 방식 이해
* 학습(Training)을 위한 데이터 입력 단위 인식
* Trainer(트레이너)가 요구하는 데이터 형태 사전 인식

## 2. 실습 환경 준비 단계

### 2.1 필수 라이브러리 확인 단계

```bash
python -V
python -c "import datasets, transformers; print(datasets.__version__, transformers.__version__)"
```

미설치 상태일 경우 다음의 패키지를 설치:

```bash
pip install -U datasets transformers
```

## 3. 공개 데이터셋으로 DatasetDict 구조 확인

### 3.1 `load_dataset` 호출

```python
from datasets import load_dataset

dd = load_dataset("imdb")
print(type(dd))
print(dd)
```

* 반환 타입이 `DatasetDict`
* `train`, `test` split 자동 포함


### 3.2 DatasetDict의 split 접근 방식 확인

```python
print(dd.keys())
print(type(dd["train"]))
print(dd["train"])
```

* `DatasetDict`는 `dict` 인터페이스 제공
* 각 value는 `Dataset` 객체

### 3.3 샘플 접근 방식 확인

```python
sample = dd["train"][0]
print(type(sample))
print(sample)
```

* `Dataset`의 한 행은 `dict` 형태
* column 기반 접근 방식


## 4. Dataset의 column과 features 구조 이해

### 4.1 컬럼 이름 확인 단계

```python
dd["train"].column_names
```

### 4.2 features(피처) 확인 단계

```python
dd["train"].features
```

* `features`는 Dataset의 스키마(schema, 스키마)
* `Value`, `ClassLabel` 등의 타입 정의 포함

## 5. split을 직접 생성해보는 실습

### 5.1 단일 `Dataset`에서 split 생성

```python
train_only = load_dataset("imdb", split="train")
dd2 = train_only.train_test_split(test_size=0.2, seed=42)

print(dd2)
print(dd2.keys())
```

* `train_test_split`의 결과는 `DatasetDict` 객체.
* `train`, `test` 자동 생성

### 5.2 validation split 명시적 생성

```python
from datasets import DatasetDict

dd3 = DatasetDict({
    "train": dd2["train"],
    "validation": dd2["test"],
})

print(dd3)
print(dd3.keys())
```

## 6. 로컬 텍스트 파일을 Dataset으로 로딩

### 6.1 로컬 텍스트 파일 생성

```bash
mkdir data_unit1
echo "This movie was great." > data_unit1/train.txt
echo "This movie was terrible." >> data_unit1/train.txt
```


### 6.2 `load_dataset("text")` 사용 실습

```python
ds_text = load_dataset(
    "text",
    data_files={"train": "data_unit1/train.txt"}
)

print(ds_text)
print(ds_text["train"][0])
```


* text 파일 한 줄이 하나의 샘플
* 기본 컬럼 이름은 `"text"`


## 7. CSV 기반 Dataset 생성 실습

### 7.1 CSV 파일 생성

```bash
cat << EOF > data_unit1/train.csv
text,label
I love this movie,1
I hate this movie,0
EOF
```


### 7.2 CSV 로딩 실습

좀 더 자세한 건 [[/hf_dataset_dict/dd_csv]]{DatasetDict와 CSV} 문서를 참고할 것:

```python
ds_csv = load_dataset(
    "csv",
    data_files={"train": "data_unit1/train.csv"}
)

print(ds_csv)
print(ds_csv["train"][0])
print(ds_csv["train"].features)
```

* CSV 컬럼명이 `Dataset` 컬럼명으로 사용됨
* label은 기본적으로 int64 타입


## 8. DatasetDict 저장 및 재로딩 실습

### 8.1 디스크 저장

```python
ds_csv.save_to_disk("saved_unit1_dataset")
```

* Apach Arrow 포맷 기반 저장
* `Dataset` 객체도 저장 가능함.

일반적인 데이터 구조가 다음과 같음:

```text
my_dataset/
 ├─ dataset_info.json
 ├─ state.json
 ├─ train/
 │   ├─ data-00000-of-00001.arrow
 │   └─ indices.arrow
 └─ validation/
     ├─ data-00000-of-00001.arrow
     └─ indices.arrow
```


### 8.2 디스크에서 재로딩

```python
from datasets import load_from_disk

ds_loaded = load_from_disk("saved_unit1_dataset")
print(ds_loaded)
```

* 전처리 결과까지 함께 보존 가능


## 9. Trainer 입력 구조 사전 확인

### 9.1 Trainer에 Dataset 연결 구조 확인

```python
from transformers import (
	AutoModelForSequenceClassification, 
	Trainer, 
	TrainingArguments
	)

model = AutoModelForSequenceClassification.from_pretrained(
    "distilbert/distilbert-base-uncased",
    num_labels=2
)

args = TrainingArguments(
    output_dir="./out_unit1",
    per_device_train_batch_size=8,
    per_device_eval_batch_size=8,
    report_to="none",
)

trainer = Trainer(
    model=model,
    args=args,
    train_dataset=dd3["train"],
    eval_dataset=dd3["validation"],
)

trainer.train_dataset[0]
```

* Trainer는 Dataset 객체를 그대로 받음
* 현재는 전처리 미적용 상태: 학습 불가
* 좀더 자세한 전처리는 "[[/hf_dataset_dict/dd_map]]{map을 활용한 전처리와 학습데이터 처리} 문서" 참고.

