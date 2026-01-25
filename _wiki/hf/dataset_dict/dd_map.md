---
layout  : wiki
title   : 
summary : 
date    : 2026-01-25 15:14:44 +0900
updated : 2026-01-25 16:01:04 +0900
tag     : 
resource: D0/8C9B62FC5046DE831A8092239AFAC4
toc     : true
public  : true
parent  : 
latex   : false
---
* TOC
{:toc}



# map을 이용한 전처리와 학습 데이터 고정


## 1 학습 목표

* `map`(매핑)의 역할과 필요성 이해
* preprocessing (전처리)가 학습 파이프라인에서 차지하는 위치 이해
* Trainer (트레이너)가 요구하는 입력 컬럼 구조 이해
* 전처리 결과를 `Dataset`에 고정 반영하는 방법 이해 
* image 데이터에서도 `map`이 동일하게 사용됨을 이해
* `collate_fn` (collate function, 배치 결합 함수)과의 역할 차이 이해

## 2 실습을 위한 Dataset 확보

### 2.1 Text 기반 실습용 Dataset 확보

가장 단순하고 재현 가능한 방법은 Hugging Face Hub의 공개 데이터셋을 사용하는 방식임.

```python
from datasets import load_dataset

dataset = load_dataset("imdb")
```

이 호출의 결과는 다음 구조를 가짐.

```text
DatasetDict({
  train: Dataset(features=['text', 'label'])
  test:  Dataset(features=['text', 'label'])
})
```

Trainer 학습을 고려하여 validation split을 명시적으로 구성함.

```python
dataset = load_dataset(
              "imdb", 
	      split="train",
	  )
dataset = dataset.train_test_split(
              test_size=0.2, 
	      seed=42,
	  )

dataset
```

이제 dataset은 다음 구조를 가짐.

```text
DatasetDict({
  train: Dataset
  test:  Dataset   # validation으로 사용
})
```

### 2.2 Image 기반 실습용 Dataset 확보

image 기반 `map` 예제를 실행하기 위해 `image` 컬럼을 포함한 `Dataset`이 필요함.

실습 재현성을 위해 **최소 더미 이미지 Dataset을 직접 생성**함.

```python
from datasets import Dataset, Image
from PIL import Image as PILImage
import numpy as np
import os

os.makedirs("tmp_images", exist_ok=True)

for i in range(2):
    img = PILImage.fromarray(
        (np.random.rand(224, 224, 3) * 255).astype("uint8")
    )
    img.save(f"tmp_images/{i}.png")

image_dataset = Dataset.from_dict({
    "image": ["tmp_images/0.png", "tmp_images/1.png"],
    "label": [0, 1],
}).cast_column("image", Image())
```

이 image_dataset은 이후 image map 예제에서 그대로 사용가능.

### 2.3 Hugging Face Hub에서 MNIST 로딩

```python
from datasets import load_dataset

mnist = load_dataset("mnist")
```

이 호출의 결과는 다음 구조를 가짐.

```python
DatasetDict({
  train: Dataset(features=['image', 'label'])
  test:  Dataset(features=['image', 'label'])
})
```

* `image`: `Image`(이미지 타입), PIL `Image`로 lazy loading
* `label`: `int`, 0부터 9까지의 숫자 클래스

Validation을 다음과 같이 training data에서 구분할 수 있음:
```python
mnist_train = mnist["train"].train_test_split(
    test_size=0.1,
    seed=42
)

mnist_dataset = {
    "train": mnist_train["train"],
    "validation": mnist_train["test"],
}

# 필요한 경우 다음과 같이
# DatasetDict로 명시적으로 감쌀 수 있음.
from datasets import DatasetDict
mnist_dataset = DatasetDict(mnist_dataset)
```


## 3 Dataset에서 map 의 역할.

Trainer는 모델에 다음과 같은 입력을 전달함.

* text 모델 (Transformer 계열): `input_ids`, `attention_mask`, `labels`
* image 모델: `pixel_values`, `labels`

그러나 원본 Dataset에는 일반적으로 다음 컬럼만 존재함.

* text Dataset: `text`, `label`
* image Dataset: `image`, `label`

즉, **모델이 요구하는 입력 컬럼이 Dataset에 존재하지 않는 상태** 이기 쉬움.

이 불일치를 해결하는 단계가 전처리이며,
HF Dataset에서는 전처리를 **`map`을 통해 `Dataset` 자체에 반영** 가능함.

`map`은

* **전처리를 코드 수준이 아닌 데이터 수준으로 고정하는 메커니즘**임.
* 일괄적으로 적용함.


## 4 Text 전처리에서 map 사용

### 4.1 AutoTokenizer 준비

Raw Text 데이터는 [[/nlp/tokenization]]{Tokenization} 이 필요함.

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained(
    "distilbert/distilbert-base-uncased"
)
```

* Tokenizer는 raw text를 
* `input_ids`와 `attention_mask`로 변환하는 규칙 집합임.

> HF 는 `AutoTokenizer`에서 사전학습된 토큰나이저를 제공함.

### 4.2 단일 샘플 토큰화 확인

```python
tokenizer(
    "This movie was very interesting.",
    truncation=True
)
```
* tokenizer 는 callable 객체.
* 이를 호출하여 반환한 객체가 Trainer가 기대하는 입력 구조를 가짐.
	* `input_ids`와 `attention_mask` 가 추가됨.

> padding을 사용하지 않는 경우  
> `attention_mask`는 모든 값이 1인 상태로 자동 생성


### 4.3 map용 전처리 함수 정의

```python
def preprocess_text(batch):
    return tokenizer(
        batch["text"],
        truncation=True
    )
```
* `batch["text"]`는 문자열 리스트임.

### 4.4 map 적용

앞서 만든 `preprocess_text(batch)`함수를 `.map`으로 적용.

```python
encoded_dataset = dataset.map(
    preprocess_text,
    batched=True
)
```
* map 적용 후 컬럼은 다음과 같음.

```text
["text", "label", "input_ids", "attention_mask"]
```

### 4.5 remove_columns 적용

앞서 tokenization 이 이루어진 이후엔,  
`text` 컬럼은 더 이상 필요하지 않음.

다음의 `remove_columns`를 통해 제거할 column을 제거할 수 있음.

```python
encoded_dataset = dataset.map(
    preprocess_text,
    batched=True,
    remove_columns=["text"]
)
```

위 예제의 결과는 다음과 같음.

```text
["label", "input_ids", "attention_mask"]
```

위의 columns는 일반적으로 Text 데이터를 사용하는 모델 학습을 위한  Trainer에 바로 전달 가능한 상태임.


## 5 Image 전처리에서 map 사용

`map`은 image 데이터에도 동일하게 적용됨.


### 5.1 AutoImageProcessor 준비

`map`의 image 데이터에 적용하기 위해서, 이미지전처리를 `AutoImageProcessor`로 가져옴.

```python
from transformers import AutoImageProcessor

image_processor = AutoImageProcessor.from_pretrained(
    "google/vit-base-patch16-224-in21k"
)
```


### 5.2 image map 전처리 함수 정의

```python
def preprocess_image(batch):
    out = image_processor(
        batch["image"],
        return_tensors="np" # NumPy의 ndarray로 반환.
    )
    batch["pixel_values"] = out["pixel_values"]
    return batch
```

* `return_tensors="np"` 설정은 `Dataset` 저장 안정성을 위한 설정임.



### 5.3 map 적용

```python
encoded_image_dataset = image_dataset.map(
    preprocess_image,
    batched=True,
    remove_columns=["image"]
)
```

결과 컬럼은 다음과 같음.

```text
["pixel_values", "label"]
```

* 이 `Dataset`은 image Trainer 에 입력 가능한 상태임.


## 6 map과 collate_fn의 차이

### 6.1 전처리 시점의 차이

* `map`: `Dataset` 자체에 전처리
* `collate_fn`: `DataLoader` 배치 생성 시점 전처리

### 6.2 결과 저장 여부의 차이

* `map`: 전처리 결과가 Dataset 컬럼으로 저장됨
* `collate_fn`: `Dataset` 객체는 변경되지 않음

### 6.3 사용 목적의 차이

`map`이 적합한 경우는 다음과 같음.

* 토큰화처럼 결정적 전처리
* `resize`, `normalize` 같은 고정 이미지 전처리
* HF Hub 업로드를 위한 전처리 고정

`collate_fn`이 적합한 경우는 다음과 같음.

* 동적 padding
* 랜덤 증강
* detection, segmentation의 가변 어노테이션
* 멀티모달 배치 동기화

### 6.4 collate_fn 이미지 예제

```python
import torch

def collate_images(batch):
    images = [x["image"] for x in batch]
    labels = [x["label"] for x in batch]

    out = image_processor(images, return_tensors="pt")
    out["labels"] = torch.tensor(labels)
    return out
```

이 방식은 매 배치마다 전처리를 수행함.



## 7 map 결과 캐시와 재현성

map 결과는 자동으로 캐시됨.

다음의 경우, 캐시된 데이터를 사용함.

* 동일 입력
* 동일 함수
* 동일 파라미터

디버깅 에서 코드의 변경사항을 제대로 확인하기 위해서 다음 옵션 사용 가능함.

```python
encoded_dataset = dataset.map(
    preprocess_text,
    batched=True,
    load_from_cache_file=False
)
```

 
