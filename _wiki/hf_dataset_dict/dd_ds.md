---
layout  : wiki
title   : HF - load_dataset
summary : 
date    : 2026-01-25 14:39:24 +0900
updated : 2026-01-25 16:39:59 +0900
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
* `train`, `test` split 로 나누어지며, 각 split은 `Dataset`객체임

출력
```text
type(dd) = <class 'datasets.dataset_dict.DatasetDict'>
DatasetDict({
    train: Dataset({
        features: ['text', 'label'],
        num_rows: 25000
    })
    test: Dataset({
        features: ['text', 'label'],
        num_rows: 25000
    })
    unsupervised: Dataset({
        features: ['text', 'label'],
        num_rows: 50000
    })
})
```


### 3.2 DatasetDict의 split 접근 방식 확인

```python
print(dd.keys())
print(type(dd["train"]))
print(dd["train"])
```

* `DatasetDict`는 `dict` 인터페이스 제공
* 각 value는 `Dataset` 객체로 split에 해당함

주로 split은 train dataset, test dataset, validation dataset 등에 해당함.

출력
```text
dict_keys(['train', 'test', 'unsupervised'])
<class 'datasets.arrow_dataset.Dataset'>
Dataset({
    features: ['text', 'label'],
    num_rows: 25000
})
```

### 3.3 샘플 접근 방식 확인

```python
sample = dd["train"][0]
print(type(sample))
print(sample)
```

* `Dataset`의 한 행은 `dict` 형태: `'text'`와 `'label`'을 키로 가지고 있음.
* 이들을 `Dataset`의 column이라고 부름 (4.1절 참고)

출력
```text
<class 'dict'>
{'text': 'I rented I AM CURIOUS-YELLOW from my video store because of all the controversy that surrounded it when it was first released in 1967. I also heard that at first it was seized by U.S. customs if it ever tried to enter this country, therefore being a fan of films considered "controversial" I really had to see this for myself.<br /><br />The plot is centered around a young Swedish drama student named Lena who wants to learn everything she can about life. In particular she wants to focus her attentions to making some sort of documentary on what the average Swede thought about certain political issues such as the Vietnam War and race issues in the United States. In between asking politicians and ordinary denizens of Stockholm about their opinions on politics, she has sex with her drama teacher, classmates, and married men.<br /><br />What kills me about I AM CURIOUS-YELLOW is that 40 years ago, this was considered pornographic. Really, the sex and nudity scenes are few and far between, even then it\'s not shot like some cheaply made porno. While my countrymen mind find it shocking, in reality sex and nudity are a major staple in Swedish cinema. Even Ingmar Bergman, arguably their answer to good old boy John Ford, had sex scenes in his films.<br /><br />I do commend the filmmakers for the fact that any sex shown in the film is shown for artistic purposes rather than just to shock people and make money to be shown in pornographic theaters in America. I AM CURIOUS-YELLOW is a good film for anyone wanting to study the meat and potatoes (no pun intended) of Swedish cinema. But really, this film doesn\'t have much of a plot.', 'label': 0}
```


## 4. Dataset의 column과 features (meta-data)

### 4.1 컬럼 이름 확인 단계

```python
dd["train"].column_names
```

* 현재 `Dataset` 객체의 column들의 이름을 가지는 list 객체 반환.

출력
```text
['text', 'label']
```

### 4.2 features(피처) 확인 단계

```python
dd["train"].features
```

* `features`는 해당 Dataset 객체의 스키마(schema) 정보를 나타내는 `Features` 객체임.
* 각 컬럼(column)의 이름과 자료형(type), 그리고 `Value`, `ClassLabel`, `Sequence` 같은 feature type 정의를 포함함.
* 즉, 이 데이터셋의 각 필드가 어떤 구조와 타입으로 저장되는지를 설명하는 메타정보(meta-information)임.
* 여기서 확인하는 대상은 `dd["train"]`이라는 개별 `Dataset`의 스키마임.

주의할 점은, `DatasetDict` 전체의 메타데이터를 직접 반환하는 것은 아님.

* `DatasetDict`는 여러 split을 묶는 컨테이너이고,
* 실제 `features`는 보통 각 split인 `dd["train"]`, `dd["test"]` 등에 대해 확인함.

출력
```text
{'text': Value('string'), 'label': ClassLabel(names=['neg', 'pos'])}
```

## 5. split을 직접 생성해보는 실습

### 5.1 단일 `Dataset`에서 split 생성

```python
# Hugging Face Datasets 라이브러리를 사용하여
# "imdb" 데이터셋의 train 분할(split)만 로드함.
# 결과는 하나의 Dataset 객체로 반환됨.
train_only = load_dataset("imdb", split="train")

# 로드한 train 데이터셋을 다시 학습용(train)과 평가용(test)으로 분할함.
# test_size=0.2 는 전체 데이터의 20%를 test 쪽으로 분리하겠다는 뜻임.
# seed=42 는 난수 시드를 고정하여, 매번 같은 방식으로 분할되게 하기 위한 설정임.
# 결과는 DatasetDict 형태로 반환되며,
# 일반적으로 "train", "test" 두 개의 키를 가짐.
dd2 = train_only.train_test_split(test_size=0.2, seed=42)

# 분할된 전체 구조를 출력함.
# 각 split 이름과 각 split에 포함된 샘플 수, feature 정보 등을 확인할 수 있음.
print(dd2)

# DatasetDict의 key 목록을 출력함.
# 보통 dict_keys(['train', 'test']) 와 같이 표시됨.
print(dd2.keys())
```

* split을 지정하여 특정 split만 로딩하는 것도 가능함: 이 경우 지정한 split에 해당하는 `Dataset` 객체 반환.
* `Dataset`의 `train_test_split`의 결과는 `DatasetDict` 객체인 점에 유의할 것.
	* `"train"`, `"test"` split을 자동 생성

출력
```text
DatasetDict({
    train: Dataset({
        features: ['text', 'label'],
        num_rows: 20000
    })
    test: Dataset({
        features: ['text', 'label'],
        num_rows: 5000
    })
})
dict_keys(['train', 'test'])
```

### 5.2 validation split 명시적 생성

```python

from datasets import DatasetDict # DatasetDict 클래스를 import함.
# 여러 개의 데이터 분할(split)을
# 하나의 사전(dict)처럼 묶어 관리할 때 사용함.
dd3 = DatasetDict({
    # dd2에서 "train" split을 꺼내어
    # 새 DatasetDict의 "train" split으로 넣음.
    "train": dd2["train"],

    # dd2에서 "test" split을 꺼내어
    # 이름을 "validation"으로 바꾸어 넣음.
    # 즉, test 데이터를 validation 데이터처럼 재구성하는 것임.
    "validation": dd2["test"],
})

# dd3 전체 구조를 출력함.
# 각 split 이름, 샘플 수, feature 정보 등을 확인할 수 있음.
print(dd3)

# dd3에 들어 있는 split 이름들만 출력함.
# 보통 dict_keys(['train', 'validation']) 형태로 나타남.
print(dd3.keys())
```

* `DatasetDict`를 여러 `Dataset` 객체로부터 생성하고 있음.
* evaluation에 사용되는 validation 용 `Dataset` 객체를 지정하고 있음.

출력:
```text
DatasetDict({
    train: Dataset({
        features: ['text', 'label'],
        num_rows: 20000
    })
    validation: Dataset({
        features: ['text', 'label'],
        num_rows: 5000
    })
})
dict_keys(['train', 'validation'])
```

참고로, 다음과 같이 하면 `dd2` 에 test를 validation 으로 변경하게 됨.
```python
dd2["validation"] = dd2.pop("test")
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

* Trainer는 `Dataset` 객체를 그대로 받음
* 현재는 전처리 미적용 상태: 학습 불가
* 좀더 자세한 전처리는 "[[/hf_dataset_dict/dd_map]]{map을 활용한 전처리와 학습데이터 처리} 문서" 참고.

