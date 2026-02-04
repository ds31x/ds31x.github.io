---
layout  : wiki
title   : DatasetDict 허브 업로드.
summary : 
date    : 2026-01-27 09:38:51 +0900
updated : 2026-01-27 14:06:52 +0900
tag     : 
resource: AE/6280DD224F4CC5AA16BE73567EF658
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}

# DatasetDict의 Hub 업로드와 학습 파이프라인 재사용


## 학습 목표

* Hugging Face Hub에 업로드된 Dataset을 **DatasetDict** 형태로 재사용하는 흐름 이해
* `DatasetDict`를 기준으로 학습 파이프라인을 고정하고 반복 실행하는 방법 이해

> HF Hub에 Dataset을 공개할 때도 반드시 필요함.

## 학습 내용

* Hub Dataset과 `load_dataset`의 관계 이해
* DatasetDict가 Hugging Face 학습 파이프라인의 표준 단위임을 이해
* Dataset loading script의 역할과 규칙 이해
* DatasetDict => collate_fn => Trainer 연결 구조 이해
* Dataset 변경 후 revision / tag로 DatasetDict를 다시 로드하는 방법 이해


## 실습 핵심

* image + text + label Dataset을 **DatasetDict로 로드**
* DatasetDict를 그대로 Trainer에 연결
* DatasetDict revision / tag만 바꿔 재학습


## 도달 목표

* DatasetDict를 기준으로 학습 파이프라인을 재사용할 수 있는 상태 도달


## 0. 문제 설정 (DatasetDict 관점)

이 Unit에서 다루는 문제는 다음과 같음.

* 로컬에 **image + text + label**로 구성된 데이터가 존재함
* 이 데이터를 Hugging Face Hub에 Dataset으로 업로드함
* 이후 다음 코드로 Dataset을 다시 불러오고자 함

```python
from datasets import load_dataset

dataset_dict = load_dataset("username/mm-dataset-exp")
```

다음을 주의할 것:

* Hugging Face에서 "Dataset"의 실제 사용 단위는 **DatasetDict**임
* `load_dataset()`의 반환값은 대부분 `DatasetDict`
* Trainer는 `DatasetDict["train"]`, `DatasetDict["validation"]`을 입력으로 사용함


## 1. DatasetDict란 무엇인가

`DatasetDict`는 다음과 같은 구조를 가지는 컨테이너임.

```text
DatasetDict
 ├─ train        -> Dataset
 ├─ validation   -> Dataset
 └─ test         -> Dataset (선택)
```

각 split(`train`, `validation`)은 개별 `Dataset` 객체이며,
Trainer는 이 split 단위로 학습과 평가를 수행함.

따라서 Hub에 업로드되는 Dataset은
**결국 DatasetDict로 재구성되는 것을 전제로 설계**해야 함.


## 2. DatasetDict로 로드되기 위한 조건

> `load_dataset("username/mm-dataset-exp")` 호출이 성공하려면,  
> Hub Dataset repo는
>   
> **(1) DatasetDict가 Arrow 기반으로 이미 직렬화되어 있거나**,
> **(2) 또는 raw 데이터를 해석하는 Dataset loading script를 제공해야 한다.**


다음 호출이 일반적으로 HF Hub로부터 `DatasetDict` 객체를 local로 가져옴.

```python
dataset_dict = load_dataset("username/mm-dataset-exp")
```

이를 위해선 Hub Dataset repo는 **아래 조건 중 하나를 만족** 해야 함.

### 2-1. Case-1. DatasetDict가 이미 직렬화되어 있는 경우 (권장, 최신 방식)

다음 조건을 만족하면 **Dataset loading script 없이도** 자동으로 `DatasetDict`로 로드됨.

* repo 루트에 `dataset_dict.json` 이 존재
* 각 split 디렉토리(`train/`, `validation/`, `test/` 등)가 존재
* 각 split 디렉토리가 다음을 포함

  * Arrow 데이터 파일 (`data-xxxxx-of-yyyyy.arrow`)
  * `dataset_info.json`
  * `state.json`
* 각 split은 이미 `Dataset`으로 직렬화된 상태

이 경우:

* Hugging Face Datasets는 **loading script를 사용하지 않음**
* Hub는 `dataset_dict.json`을 기준으로 split을 복원
* `load_dataset()` 호출 시 즉시 `DatasetDict` 반환

이 같은 구조는 `DatasetDict` 객체의 

* `save_to_disk()` 를 통해 local에 저장한 결과(directory)를 그대로 업로드 한 경우에 해당.
* `push_to_hub()` 를 통해 직접 업로드도 가능함.

> `save_to_disk()` 결과를 그대로 업로드한 repo는
> **그 자체로 완전한 DatasetDict 정의임**

### 2-2. Case-2 Raw 데이터만 존재하는 경우 (이전 방식, loading script 필요)

다음과 같은 repo 구조의 경우에는 **Dataset loading script가 반드시 필요함**.

* 이미지, json, csv, jsonl 등 raw 파일만 존재
* split 정보가 파일 구조나 메타데이터로 명시되지 않음
* Dataset schema가 정의되지 않음

이 경우 repo는 다음 조건을 만족해야 함.

* Dataset의 구조를 해석할 수 있는 Dataset loading script가 존재
* loading script가 `train / validation / test` split을 생성
* 각 split이 `Dataset` 객체로 구성됨

> 2026년 현재,
> 
> `save_to_disk()` 호출 이후 만들어진 디렉토리를 그대로 올리는 경우엔
> loading script가 필요없음.
> 단, raw파일만 올린 repo의 경우엔 loading script가 필요함.

## 3. Dataset repo의 최종 파일 구조

Hugging Face Dataset repo의 최종 파일 구조는

* **어떤 시점의 datasets 라이브러리 기준을 따르느냐**와
* **DatasetDict를 어디에서 구성하느냐**에 따라 달라짐.

특히 datasets **4.0 이후**를 기준으로는
Dataset loading 단계의 역할과 허용 범위가 과거와 명확히 달라졌음.

이에 따라 Dataset repo 구조는 다음 두 가지 방식으로 구분됨.

### 3-1. DatasetDict를 `save_to_disk()`로 직렬화하여 업로드하는 경우

*(datasets 3.x 후반 ~ 4.x 이후, 최신 권장 방식)*

Dataset을 로컬 환경에서 `DatasetDict` 형태로 완전히 구성한 뒤,
`save_to_disk()` 결과를 그대로 Hub에 업로드하는 방식임.

이 방식은 datasets **3.x 이후 점진적으로 도입**되었고,
datasets **4.0 이후에는 사실상의 표준 방식**으로 사용됨.

이 경우 Dataset repo의 구조는 다음과 같음.

```text
mm-dataset-exp/
 ├─ README.md
 ├─ dataset_dict.json          # DatasetDict split 정의
 ├─ train/
 │   ├─ data-00000-of-00001.arrow
 │   ├─ dataset_info.json
 │   └─ state.json
 ├─ validation/
 │   ├─ data-00000-of-00001.arrow
 │   ├─ dataset_info.json
 │   └─ state.json
 ├─ test/
 │   ├─ data-00000-of-00001.arrow
 │   ├─ dataset_info.json
 │   └─ state.json
 └─ .gitattributes
```

이 구조의 핵심은 다음임.

* 각 split(`train`, `validation`, `test`)은 이미 **Dataset 객체로 직렬화됨**
* split 간의 관계는 `dataset_dict.json`에 명시됨
* 실제 데이터는 Apache Arrow 포맷으로 저장됨
* feature, schema, column 정보는 `dataset_info.json`에 포함됨
* Dataset loading script가 필요 없음
* `load_dataset()` 호출 시 Hub가 이 구조를 그대로 복원하여 `DatasetDict` 반환

즉,

> datasets 4.0 기준에서
> Dataset repo는 **실행 로직이 아닌, 완성된 데이터 구조**를 제공하는 것이 기본 전제임

멀티모달 `Dataset`(image, text, label 등)에서도
이 방식이 동일하게 적용됨.

### 3-2. Raw 데이터 + Dataset loading script를 사용하는 경우

*(datasets 2.x ~ 3.x 초반 기준의 전통적 방식)*

Raw 데이터 파일을 업로드하고,
Dataset loading script를 통해 DatasetDict를 구성하는 방식임.

이 방식은 datasets **2.x 및 3.x 초반까지는 일반적으로 사용**되었음.

이 경우 Dataset repo의 구조는 다음과 같음.

```text
mm-dataset-exp/
 ├─ README.md
 ├─ mm_dataset_exp.py          # Dataset loading script
 ├─ data/
 │   ├─ train.jsonl            # image + text + label
 │   ├─ validation.jsonl
 │   └─ images/
 │       ├─ 0000.png
 │       ├─ 0001.png
 │       └─ 0002.png
 └─ .gitattributes
```

이 구조의 핵심은 다음임.

* 이미지 파일은 **파일 그대로 보관**
* 텍스트와 라벨은 **JSONL** 등의 메타데이터로 관리
* Dataset loading script가 raw 데이터를 파싱하여 DatasetDict를 구성
* split(`train / validation / test`) 정의가 script에 포함됨

다만, datasets **4.0 이후 기준에서는**  
이 방식에서 허용되는 역할이 명확히 제한됨.

* loading script는 **정적 파싱과 split 정의**까지만 담당
* 실행 시점의 동적 전처리, 랜덤 변환, 데이터 생성은 **Dataset loading 단계의 책임에서 제외됨**

이러한 처리들은 현재 기준으로

* `map`,
* `set_transform`,
* collator,
* Trainer 단계로 이동됨.

즉, 이 구조 자체가 “금지”된 것은 아니지만, 

> **datasets 4.0 이후에는
> loading script의 역할이 크게 축소되었다** 는 점을 주의할 것.

### 3-3. 기준 시점에 따른 정리

* datasets **2.x ~ 3.x 초반**

  * Raw 데이터 + loading script 방식이 일반적
  * loading 단계에서 전처리 로직을 포함하는 경우도 많았음
* datasets **3.x 후반 ~ 4.x 이후**

  * DatasetDict 직렬화(`save_to_disk`) 방식이 표준
  * Dataset repo는 정적 데이터 구조 제공에 집중
  * 처리 로직은 학습 파이프라인으로 분리됨


## 4. Dataset loading script와 DatasetDict 생성 원리

**Dataset loading script란** 

* Hugging Face에서 데이터를 읽고 구조화하는 파이썬 스크립트 전체를 가리킴.
* 그 안에는 DatasetBuilder (대부분 GeneratorBasedBuilder) 클래스를 정의하는 코드가 들어 있음.
* 즉, 로딩 스크립트의 실제 핵심은 DatasetBuilder 클래스임.

> A dataset loading script is a Python file that defines a DatasetBuilder class. 
> DatasetBuilder specifies metadata, splits, and how to generate examples.
> 참고: [Writing a dataset loading script](https://huggingface.co/docs/datasets/v1.2.1/add_dataset.html?utm_source=chatgpt.com)

`load_dataset()`은 다음 과정을 수행함:

> `load_dataset()`은 Hub Dataset repo(또는 로컬 경로)를 받아 **Dataset loading script(DatasetBuilder)**를 찾아 실행하고, 그 결과로 DatasetDict를 생성한다.

1. **source** 를 결정: (Hub repo, 로컬 경로, 내장 로더 중 하나 선택)
2. **loading script 탐색** - 적용 규칙은 다음과 같음:
	* Hub repo 루트에 loading script가 있어야 함.
	* 파일명은 repo 이름과 일치해야 함.
		* repo의 `-`는 파일명에서 `_`로 변환됨.
		* 예: `mm-dataset-exp` → `mm_dataset_exp.py`
3. **loading script 실행** 
	* script 내부에 정의된 DatasetBuilder를 로드하고 초기화함.
4. DatasetBuilder 정의에 따라 DatasetDict 생성.
	* `_info()`로 Features(스키마) 확정함.
	* `_split_generators()`로 split(`train`/`validation`/`test`) 정의함.
	* `_generate_examples()`로 각 split의 샘플을 생성.
5. split별 Dataset을 묶어 DatasetDict 반환을 수행함.

## 5. Dataset loading script 파일명 규칙

Dataset loading script는 다음 규칙을 반드시 만족해야 함.

* repo 루트에 위치
* 파일명은 repo 이름과 동일
* `-`는 `_`로 변환

예:

| Hub Dataset repo | loading script      |
| ---------------- | ------------------- |
| `mm-dataset-exp` | `mm_dataset_exp.py` |

이 규칙이 어긋나면 `load_dataset()`은 DatasetDict를 생성할 수 없음.


## 6. Dataset loading script 예시 (DatasetDict 생성)

```python
import json
from pathlib import Path
import datasets

class MmDatasetExp(datasets.GeneratorBasedBuilder):
    VERSION = datasets.Version("1.0.0")

    def _info(self):
        return datasets.DatasetInfo(
            features=datasets.Features({
                "image": datasets.Image(),
                "text": datasets.Value("string"),
                "label": datasets.Value("int64"),
            })
        )

    def _split_generators(self, dl_manager):
        data_dir = Path(self.config.data_dir) / "data"
        return [
            datasets.SplitGenerator(
                name=datasets.Split.TRAIN,
                gen_kwargs={"path": data_dir / "train.jsonl"},
            ),
            datasets.SplitGenerator(
                name=datasets.Split.VALIDATION,
                gen_kwargs={"path": data_dir / "validation.jsonl"},
            ),
        ]

    def _generate_examples(self, path):
        with open(path, encoding="utf-8") as f:
            for idx, line in enumerate(f):
                yield idx, json.loads(line)
```

이 script의 결과는 항상 다음 형태임.

```text
DatasetDict(
  train: Dataset
  validation: Dataset
)
```

## 7. Hub 업로드와 DatasetDict 고정

이 repo를 Hub에 업로드하면,
Hub에는 **DatasetDict를 생성할 수 있는 규칙 전체**가 저장됨.

```bash
git add .
git commit -m "initial image+text dataset"
git push origin main
```

이 commit 하나가 **DatasetDict의 특정 상태**를 의미함.


## 8. DatasetDict 로드 확인

```python
from datasets import load_dataset

dataset_dict = load_dataset("username/mm-dataset-exp")

print(dataset_dict)
print(dataset_dict["train"][0])
```

확인 포인트:

* `DatasetDict` 타입인지 확인
* `train`, `validation` split 존재 여부 확인
* 각 샘플에 `image`, `text`, `label` 존재 여부 확인


## 9. DatasetDict와 Trainer의 연결 구조

Trainer는 DatasetDict를 다음 방식으로 사용함.

```python
trainer = Trainer(
    train_dataset=dataset_dict["train"],
    eval_dataset=dataset_dict["validation"],
)
```

즉, **Trainer는 DatasetDict를 직접 받지 않고**,
DatasetDict의 각 split을 받음.

DatasetDict는 Trainer 연결을 위한 **전제 구조**임.


## 10. DatasetDict => collate_fn => batch tensor

DatasetDict의 각 Dataset은 여전히
`{"image": PIL.Image, "text": str, "label": int}` 형태임.

이를 batch tensor로 바꾸는 역할이 **collate_fn**임.

> 이같은 멀티 모달의 경우,  
> BLIP processor 기반 collate_fn 사용을 예제로 삼아 이해하면 쉬움.


## 11. Dataset 변경 후 DatasetDict 재생성 (revision)

Dataset 파일을 수정하고 Hub에 다시 push하면,

* Hub에는 새로운 commit이 생성됨
* 해당 commit은 **새로운 DatasetDict 상태**를 의미함

```python
dataset_dict = load_dataset(
    "username/mm-dataset-exp",
    revision="91bd210"
)
```

## 12. DatasetDict 상태에 tag 부여

tag는 DatasetDict 상태에 이름을 붙이는 행위임.

```bash
git tag v1.1-cleaned
git push origin v1.1-cleaned
```

```python
dataset_dict = load_dataset(
    "username/mm-dataset-exp",
    revision="v1.1-cleaned"
)
```

## 13. 핵심 정리 (DatasetDict 관점)

* Hub Dataset의 실체는 **DatasetDict**
* `load_dataset()`의 반환값은 DatasetDict
* Trainer는 DatasetDict의 split을 사용
* Dataset 변경 → revision / tag → 새로운 DatasetDict 생성
* DatasetDict만 바꿔 학습 파이프라인 재사용 가능

## 응용

* **DatasetDict + collate_fn + BLIP Trainer를 하나의 `train.py`로 묶는 실습**


