---
layout  : wiki
title   : DatasetDict 허브 업로드.
summary : 
date    : 2026-01-27 09:38:51 +0900
updated : 2026-01-27 10:31:24 +0900
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

다음 호출이 성공하려면,

```python
dataset_dict = load_dataset("username/mm-dataset-exp")
```

Hub Dataset repo는 다음 조건을 만족해야 함.

1. Dataset의 구조를 해석할 수 있는 **Dataset loading script**가 존재
2. loading script가 train / validation split을 생성
3. 각 split이 `Dataset`으로 구성됨

파일만 올린 repo는 이 조건을 자동으로 만족하지 않음.


## 3. Dataset repo의 최종 파일 구조

DatasetDict 생성을 전제로 한 Dataset repo 구조는 다음과 같음.

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
* 텍스트와 라벨은 흔하게 JSONL 메타데이터로 관리
* loading script가 이들을 **DatasetDict 구조로 묶음**


## 4. Dataset loading script와 DatasetDict 생성 원리

`load_dataset()`은 다음 과정을 수행함.

1. Hub에서 Dataset repo 다운로드
2. repo 루트에서 loading script 탐색
3. loading script 실행
4. loading script가 정의한 규칙에 따라
   * `train` split Dataset 생성
   * `validation` split Dataset 생성
5. 이들을 묶어 **DatasetDict 반환**


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


