---
layout  : wiki
title   : HF - DatasetBuilder
summary : 
date    : 2026-01-27 11:10:50 +0900
updated : 2026-01-27 13:52:36 +0900
tag     : 
resource: D4/D567D16B7D42C7AC097511F798F3DE
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}


#  DatasetBuilder를 활용한 재현 가능한 학습 데이터 정의

## 1. DatasetBuilder의 필요성

머신러닝 및 딥러닝 학습에서 dataset(데이터셋)은 
* 단순한 파일 집합이 아니라,
* 학습 과정 전반을 규정하는 핵심 요소이다.

이미지 파일, CSV, JSON 등의 형태로만 존재하는 데이터는 다음과 같은 한계를 가짐:

* 데이터의 column 구조가 명확히 정의되지 않음
* label 의 의미가 코드 외부에 암묵적으로 존재함
* train, validation, test 분할 기준이 명확하지 않음
* 동일한 데이터셋을 다시 불러왔을 때 재현성을 보장하기 어려움

Hugging Face Datasets 라이브러리는 이러한 문제를 해결하기 위해
* **데이터셋을 코드로 정의하는 방식**을 제공하며,
* 그 핵심 구성 요소가 **DatasetBuilder** 임.

## 2. DatasetBuilder의 역할

DatasetBuilder는 데이터셋에 대해 다음 사항을 코드 수준에서 명확히 정의:

1. 데이터셋이 가지는 **column 구조**
2. 각 컬럼의 **자료형 및 의미**
3. train, validation, test와 같은 **데이터 분할 방식**
4. `load_dataset()` 호출 시 생성될 **DatasetDict의 형태**

즉, DatasetBuilder는
* "이 데이터셋이 무엇이며, 어떤 구조로 해석되어야 하는가" 를
* 명시적으로 기술하는 **정의서 역할**을 수행.

> DatasetBuiler 는 Dataset Definition!!

이로 인해 DatasetBuilder가 포함된 데이터셋은
단순한 데이터 파일 묶음이 아니라,
재현 가능한 학습 resource로 인식됨.

## 3. DatasetBuilder와 DatasetDict의 관계

`load_dataset()` 함수가 호출되면 내부적으로 다음 과정이 수행됨:

1. DatasetBuilder가 정의된 스크립트를 탐색
2. DatasetBuilder 실행
3. split 규칙에 따라 각 Dataset 생성
4. 생성된 Dataset들을 하나로 묶어 DatasetDict 반환

DatasetBuilder 를 통해 **DatasetDict가 생성됨**.
* DatasetDict는 **결과물** 임.


## 4. GeneratorBasedBuilder의 개념

DatasetBuilder에는 여러 구현 방식이 존재하지만,
가장 기본적이고 널리 사용되는 형태는 `GeneratorBasedBuilder`이다.

이 방식의 특징은 다음과 같다.

* 데이터를 한 번에 메모리에 적재하지 않음
* 파일을 순차적으로 읽으며 샘플을 생성함
* 이미지, 텍스트, CSV 등 다양한 데이터 형식에 적용 가능함

`GeneratorBasedBuilder`를 사용할 경우,
다음 세 가지 메서드를 반드시 오버라이딩해야 한다.

| 메서드                    | 기능                          |
| ---------------------- | --------------------------- |
| `_info()`              | 데이터셋의 컬럼과 자료형 정의            |
| `_split_generators()`  | train/validation/test 분할 정의 |
| `_generate_examples()` | 실제 데이터 샘플 생성                |


## 5. 실습: DatasetBuilder를 이용한 DatasetDict 생성


### (1) 실습용 디렉토리 구조

다음과 같은 디렉토리 구조를 구성한다.

```text
dataset_builder_practice/
 ├─ my_dataset.py
 └─ data/
     ├─ train/
     │   ├─ cat/
     │   └─ dog/
     ├─ validation/
     │   ├─ cat/
     │   └─ dog/
     └─ test/
         ├─ cat/
         └─ dog/
```

* 각 클래스 디렉토리에는 임의의 이미지 파일을 배치.
* 중요한 것은 **split 디렉토리와 클래스 디렉토리의 계층 구조** 임.

### (2) DatasetBuilder 코드 작성

`my_dataset.py` 파일에 다음 코드를 작성.

```python
from pathlib import Path
import datasets

IMAGE_EXTS = {".png", ".jpg", ".jpeg"}


class MyDataset(datasets.GeneratorBasedBuilder):
    VERSION = datasets.Version("1.0.0")

    def _info(self):
        return datasets.DatasetInfo(
            features=datasets.Features({
                "image": datasets.Image(),
                "label": datasets.ClassLabel(names=["cat", "dog"]),
            })
        )

    def _split_generators(self, dl_manager):
        root = Path(self.config.data_dir)
        data_root = root / "data"

        return [
            datasets.SplitGenerator(
                name=datasets.Split.TRAIN,
                gen_kwargs={"split_root": data_root / "train"},
            ),
            datasets.SplitGenerator(
                name=datasets.Split.VALIDATION,
                gen_kwargs={"split_root": data_root / "validation"},
            ),
            datasets.SplitGenerator(
                name=datasets.Split.TEST,
                gen_kwargs={"split_root": data_root / "test"},
            ),
        ]

    def _generate_examples(self, split_root):
        class_to_id = {"cat": 0, "dog": 1}

        for class_name, label in class_to_id.items():
            class_dir = split_root / class_name
            for img_path in class_dir.iterdir():
                if img_path.suffix.lower() not in IMAGE_EXTS:
                    continue

                key = str(img_path.relative_to(split_root))
                yield key, {
                    "image": str(img_path),
                    "label": label,
                }
```

이 코드에서 
* `_info()`는 데이터셋의 스키마를 정의
* `_split_generators()`는 DatasetDict의 분할(split) 구조를 결정,
* `_generate_examples()`는 실제 파일을 읽어 샘플을 생성.

### (3) DatasetDict 생성 확인

다음 코드를 실행한다.

```python
from datasets import load_dataset

dataset_dict = load_dataset(
    path="dataset_builder_practice",
    name=None,
)

print(dataset_dict)
```

출력 결과는 `train`, `validation`, `test` split을 포함하는 DatasetDict 객체임.


### (4) DatasetDict 내부 구조 확인

```python
print(dataset_dict.keys())
print(dataset_dict["train"][0])
print(dataset_dict["train"].features["label"].names)
```

* DatasetDict의 split 구조가 코드에 의해 결정됨
* label의 의미가 `ClassLabel`로 고정됨
* 데이터셋의 구조와 의미가 코드 수준에서 재현 가능함


## 6. 정리

지금까지 DatasetBuilder를 사용하여
* 학습 데이터셋을 코드로 정의하고,
* 그 결과로 DatasetDict를 생성하는 과정을 확인함.

이를 통해 다음과 같은 인식을 갖는 것이 중요하다.

> 데이터셋은 단순한 파일 집합이 아니라,
> **코드로 정의되는 하나의 소프트웨어 아티팩트**이다.
 
