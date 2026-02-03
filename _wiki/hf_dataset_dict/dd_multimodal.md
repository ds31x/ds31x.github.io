---
layout  : wiki
title   : Image와 Text를 동시 사용하는 Multi-modal Dataset 
summary : 
date    : 2026-01-27 08:36:23 +0900
updated : 2026-01-27 09:27:28 +0900
tag     : collate collate_fn collator processor hf
resource: 59/6104258BE041D4B4D893AA91370365
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}


# Image + Text 멀티모달 학습 Dataset 구성

## 학습 목표

* Image + Text 멀티모달 학습에서의 **표준 Dataset 열(column) 구성 이해**
* Trainer가 기대하는 **멀티모달 입력 구조** 이해
* `collate_fn` / `collator` / `map(batched)` 방식의 차이와 역할 이해
* **실제 Hugging Face Hub 모델로 학습까지 수행**


## 1. 멀티모달 학습에서 가장 중요한 관점

학습(특히 multi-modal)에서 `Dataset` 객체에 model이 요구하는 column 이 반드시 있어야 하는 건 아님.

멀티모달 학습에서는 반드시 다음 세 단계를 **분리해서** 처리할 것:

1. **원본 데이터 보관 단계 (`Dataset`)**
2. **모델 입력 변환 단계 (`collate_fn` / `collator` / `map`)**
3. **학습 실행 단계 (`Trainer` + `Model`)**

각 단계에서 처리를 정확히 이해할 필요가 있음

## 2. 표준 멀티모달 Dataset 컬럼 설계

### 2.1 원칙: modality(양식)별 컬럼 분리

멀티모달 Dataset의 원본 컬럼은 반드시 다음처럼 단순해야 함.

| column| 의미                   |
| ----- | -------------------- |
| image | 이미지(`PIL.Image` 또는 경로) |
| text  | 텍스트(`str`)                |
| label | 정답은 흔히 정수형(`int`)     |

최소 형태는 다음과 같음.

```python
{"image": PIL.Image, "text": "some description", "label": 0}
```
### 2.2 Dataset 에서 명심할 점

중요한 점은 다음임.

* `input_ids`, `pixel_values` 같은 **모델이 직접 요구하는 입력 column 은 Dataset에 없어도 상관 없음**
* 사실 Dataset은 “원본 저장소” 역할만 수행해도 충분함.

## 3. 더미 멀티모달 Dataset 생성

```python
from datasets import Dataset
from PIL import Image
import numpy as np

def make_random_image(seed, size=(224, 224)):
    rng = np.random.default_rng(seed)
    arr = (rng.random((size[1], size[0], 3)) * 255).astype(np.uint8)
    return Image.fromarray(arr)

examples = [
    {"image": make_random_image(0), "text": "red 느낌의 이미지", "label": 1},
    {"image": make_random_image(1), "text": "green 느낌의 이미지", "label": 0},
    {"image": make_random_image(2), "text": "blue 느낌의 이미지", "label": 1},
]

ds = Dataset.from_list(examples)
```

* 위의 Dataset에는 **모델이 요구하는 입력 column이 없음**.
* 그럼에도 이후 학습은 문제없이 가능함: `collate_fn`, `collator` 등이 이를 생성.

Dataset 에는 모델이 요구하는 column을 추가하는 전처리가 이루어지기 때문에  
반드시 직접 모델의 입력이 될 column을 가지지 않아도 됨.

이후, `collator` 혹은 `collate_fn` 에 의해 이들이 추가됨.


## 4. Trainer가 기대하는 멀티모달 입력 구조

간단한 예로,  

* 멀티모달 분류 모델의 경우, 
* Trainer에서 실제로 받는 입력은 다음과 같음.

```python
{
  "input_ids": (B, T),
  "attention_mask": (B, T),
  "pixel_values": (B, C, H, W),
  "labels": (B,)
}
```

여기서 핵심은 다음 두 가지임.

* `T`는 텍스트 길이 (token의 수)에 따라 가변적임 : **배치 단위 padding 필요**
* 이 변환은 Dataset이 아니라 **배치 단계**에서 수행되어야 함

이 배치 단위 처리를 수행하는 것이 바로 **`collate_fn` / `collator`**임.

## 5. 배치 전처리 방식 1: collate_fn 함수형 방식

### 5.1 map보다 선호되는 collate_fn 함수형 방식.

`collate_fn` 이 (`map`을 이용한 처리보다) 많이 사용되는 이유는 다음과 같음:

* 텍스트 길이가 sample(~setence)마다 다른데, `collate_fn` 은 **배치 단위로 동적 패딩(dynamic padding)**을 적용하기 쉬운 구조를 가짐.
* 이미지 변환(리사이즈/정규화)도 배치에서 묶어서 일관되게 처리 가능함: `collate_fn`은 batch 에 직접 접근 가능!
* dataset 자체에는 원본만 유지하므로 저장 공간이 절약되는 장점이 있음: trade-off 로 학습 단계에서 요구되는 처리량이 늘어난다.

전처리가 `collate_fn`에서 이루어지는데, 이는 일반적으로는 `AutoProcessor`를 사용하는 방식이 HF에선 가장 편리함.

여기선 `AutoProcessor` 과 같은 Processor를 통한 전처리는 다음과 같이 간단히만 설명하고 `collate_fn` 등의 이해에만 집중한다.

아래의 예제에선 다음과 같이 전처리를 분리해서 처리한다고 생각하자.

* 텍스트 전처리: `AutoTokenizer` 사용이 일반적임.
* 이미지 전처리: `AutoImageProcessor`(또는 계열 클래스) 사용이 일반적임.

### 5.2 예제

Bootstrapping Language-Image Pre-traing (BLIP) 계열은  

* `processor(images=..., text=...)` 와 같이
* 이미지와 텍스트를 한 번에 처리하는 통합 호출이 가장 단순하며
* 일반적으로 권장됨 (BLIP을 사용하는 예제에서 보일 예정임)

통합이라도 많은 경우, **"tokenizer와 image processor를 같이 사용하는 패턴"** 임:

```python
# 7-2. BLIP AutoProcessor에서 tokenizer / image_processor 분리 사용
import torch
from transformers import AutoProcessor

mm_ckpt = "Salesforce/blip-itm-base-coco"
processor = AutoProcessor.from_pretrained(mm_ckpt)

# BLIP에서는 아래 두 줄이 정답
tokenizer       = processor.tokenizer
image_processor = processor.image_processor

def multimodal_collate_fn(batch):
    # batch: List[dict] where dict has {"image", "text", "label"}

    texts  = [x["text"]  for x in batch]
    images = [x["image"] for x in batch]
    labels = [x["label"] for x in batch]

    # 1) 텍스트: batch 단위 padding
    text_inputs = tokenizer(
        texts,
        padding=True,
        truncation=True,
        return_tensors="pt",
    )
    # text_input 는 다음임.
    # BatchEncoding({
    #  'input_ids': tensor(...),
    #  'attention_mask': tensor(...),
    # })


    # 2) 이미지: resize / normalize
    image_inputs = image_processor(
        images=images,
        return_tensors="pt",
    )

    # 3) labels
    labels = torch.tensor(labels, dtype=torch.long)

    # 4) Trainer 입력 형태로 병합
    batch_out                 = dict(text_inputs) # BatchEncoding객체를 dict객체로 변환이 호환성 높음.
    batch_out["pixel_values"] = image_inputs["pixel_values"]
    batch_out["labels"]       = labels

    return batch_out

# 동작 확인
b = multimodal_collate_fn([ds[0], ds[1]])
for k, v in b.items():
    print(k, tuple(v.shape))
```

`collate_fn`이 다음을 반환하는지 확인할 것:

* `input_ids`, `attention_mask`의 shape가 `(B, T)`인지 확인할 것.
* `pixel_values`의 shape가 `(B, 3, H, W)` 형태인지 확인할 것.
	* 여기서 `H, W`는 processor의 리사이즈 설정에 의해 결정되는 값임.
	* 보통 `H=W=224` 형태가 자주 사용되나, 코드에서는 `(B, 3, H, W)`로 확인하는 것이 원칙임.
* `labels`의 shape가 `(B,)`인지 확인할 것.


## 6. 배치 전처리 방식 1: collator(DataCollator) 방식

`collator`는 다음을 책임짐.

* Dataset의 원본 샘플 목록(`List[dict]`)을 입력으로 받음: batch를 입력으로 받음.
* text : tokenizer 적용
* image : image processor 적용
* label : labels 텐서 생성
* Trainer가 바로 전달할 수 있는 `dict` 반환

즉,

> collator는 Dataset과 Model 사이의 "변환기" 역할임.

### collate_fn과 collator의 관계

* Trainer 관점에서 `data_collator`는 “배치 구성기”라는 의미로, 내부적으로는 호출 가능한(callable) 객체를 받는 구조임.
* `collate_fn`은 함수형 구현이고, `collator`는 클래스로 구현한 callable 객체라는 차이만 있는 경우가 많음.
* `collator` 클래스는 **설정값을 생성자에 담아두기 쉬워 재사용성이 좋아지는 장점** 이 있음.

즉, 아래 두 방식은 Trainer에 들어가는 역할이 동일해질 수 있음.

* `multimodal_collate_fn(batch) -> dict` 형태의 함수형 구현임.
* `MultimodalDataCollator(...).__call__(batch) -> dict` 형태의 클래스형 구현

### 예제

예제코드는 다음과 같음:

```python
# 7-2a. DataCollator(클래스)로 배치 전처리 구현
from dataclasses import dataclass
from typing import Any, Dict, List
import torch
from transformers import PreTrainedTokenizerBase

@dataclass
class MultimodalDataCollator:
    """
    image + text + label 배치를 모델 입력 배치로 변환하는 collator 클래스임.

    입력 배치 각 원소 예시:
      {"image": PIL.Image, "text": str, "label": int}

    출력 배치 키(Trainer-friendly):
      input_ids: (B, T)
      attention_mask: (B, T)
      pixel_values: (B, C, H, W)
      labels: (B,)
    """
    tokenizer: PreTrainedTokenizerBase
    image_processor: Any
    padding: bool = True
    truncation: bool = True

    def __call__(self, batch: List[Dict[str, Any]]) -> Dict[str, torch.Tensor]:
        texts = [x["text"] for x in batch]
        images = [x["image"] for x in batch]
        labels = [x["label"] for x in batch]

        text_inputs = self.tokenizer(
            texts,
            padding=self.padding,
            truncation=self.truncation,
            return_tensors="pt",
        )
        image_inputs = self.image_processor(
            images=images,
            return_tensors="pt",
        )

        batch_out = dict(text_inputs)
        batch_out["pixel_values"] = image_inputs["pixel_values"]
        batch_out["labels"] = torch.tensor(labels, dtype=torch.long)
        return batch_out

# collator 동작 확인
collator = MultimodalDataCollator(tokenizer=tokenizer, image_processor=image_processor)
b2 = collator([ds[0], ds[1]])
for k, v in b2.items():
    print(k, tuple(v.shape))
```

Trainer 입장에서는 `data_collator` 에 `collate_fn` 또는 `collator` 의 객체를 할당하면 됨.

해당 객체는 

* callable 객체로
* batch를 인자로 받아서
* 모델이 원하는 tensor 로 구성된 batch (`dict`형)를 넘겨주기만 하면 됨.

## 7. Dataset 자체를 바꾸기: map 방식

### 7.1 map(batched=True)의 의미

* dataset을 **미리 순회** 하면서 "새 컬럼"을 만들어 저장하는 방식임.
* `batched=True`를 사용할 경우, 배치 단위로 처리하므로 tokenizer의 padding 등을 사용할 수 있음: 단 지정하는 함수가 인자를 batch로 받을 수 있어야 함.
* 단, padding을 미리 고정해버리면(예: `max_length=128`) 데이터 크기가 늘어날 수 있음.

### 7.2 예제

```python
# 7-3. map(batched=True)로 전처리 컬럼을 미리 생성
def preprocess_batch(batch):
    # batch: dict with keys "image", "text", "label" and values are lists
    text_inputs = tokenizer(
        batch["text"],
        padding="max_length",    # 미리 고정 길이로 저장하는 예시
        truncation=True,
        max_length=64,
    )
    image_inputs = image_processor(images=batch["image"])

    out = {}
    out["input_ids"] = text_inputs["input_ids"]
    out["attention_mask"] = text_inputs["attention_mask"]
    out["pixel_values"] = image_inputs["pixel_values"]
    out["labels"] = batch["label"]
    return out

ds2 = ds.map(preprocess_batch, batched=True, remove_columns=ds.column_names)

print(ds2)
print(ds2[0].keys())
```

## 8. 실제 모델을 이용한 실습: `Salesforce/blip-itm-base-coco`

**`Salesforce/blip-itm-base-coco`**는 다음 목적의 모델임.

* task: **Image-Text Matching (ITM)**
* 문제 정의:
  * 주어진 image와 text가 **서로 의미적으로 맞는 쌍인지 여부를 분류**
* 출력:
  * `label = 1` : image와 text가 잘 매칭됨
  * `label = 0` : 매칭되지 않음

즉, 이 모델은 다음 질문에 답하는 모델임.

> "이 이미지와 이 문장은 같은 내용을 말하고 있는가?"

이 구조는 **멀티모달 분류 학습의 가장 전형적인 형태** 중 하나임.

이 모델의 `forward()`에서는 다음의 키들을 가진 `dict`객체를 요구함.

```text
input_ids
attention_mask
pixel_values
labels
```

---

## 9. BLIP 모델용 Processor와 Collator

### 9.1 Processor

BLIP는 image + text 전처리를 통합한 `Processor`를 제공함.

```python
from transformers import AutoProcessor

processor = AutoProcessor.from_pretrained(
    "Salesforce/blip-itm-base-coco"
)
```
* 내부적으로 `tokenizer`(text) 와 `image processor`(image)를 포함함.

---

### 9.2 BLIP용 collator 구현

```python
from dataclasses import dataclass
import torch

@dataclass
class BLIPDataCollator:
    processor: any

    def __call__(self, batch: list[dict[str, any]]) -> dict[str, torch.Tensor]:
        texts  = [x["text"]  for x in batch]
        images = [x["image"] for x in batch]
        labels = [x["label"] for x in batch]

        # 앞서의 예제와 달리 한번에 Processor객체로 처리!
        inputs = self.processor(
            images  = images,
            text    = texts,
            padding = True,
            return_tensors="pt",
        )
        inputs["labels"] = torch.tensor(labels, dtype=torch.long)
        return inputs
```

* 많은 경우, 모델에 맞는 `processor` 만을 교체하여 범용적 사용이 가능함.


이 `collator`의 출력은 다음과 키를 가진 `dict 객체임:

```text
input_ids
attention_mask
pixel_values
labels
```

* BLIP 모델의 `forward()` 시그니처와 정확히 일치시킴.

## 10. 실제 학습 실행 (Trainer)

Trainer를 이용하는 HF 에서 일반적인 방식은 다음과 같음:

```python
from transformers import AutoModelForImageTextRetrieval
from transformers import Trainer, TrainingArguments

model = AutoModelForImageTextRetrieval.from_pretrained(
    "Salesforce/blip-itm-base-coco"
)

training_args = TrainingArguments(
    output_dir="./blip_out",
    per_device_train_batch_size=2,
    num_train_epochs=1,
    logging_steps=1,
    remove_unused_columns=False,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=ds,
    data_collator=BLIPDataCollator(processor),
)

trainer.train()
```

다시 강조하지만,  
이 Trainer가 사용하는 모델이 기대하는 입력은 다음과 같음

```text
{
  "input_ids": (B, T),
  "attention_mask": (B, T),
  "pixel_values": (B, C, H, W),
  "labels": (B,)
}
```
* Trainer 가 기대하는(모델의 `forward()` 가 기대하는) 
* 키와 shape의 tensor를 넘겨줘야 함.

### 10.1 중요 포인트

멀티모달의 경우  
TrainArguments 에서 다음의 옵션을 반드시 `False`로 해야함.

```python
remove_unused_columns=False
```

이 option이 없으면 대부분의 멀티모달 학습에서 문제 발생.

이유는 다음과 같음.

* Dataset에는 `image`, `text`, `label`만 있음
* 실제 모델 입력은 `collator` 객체에서 생성됨
* Trainer가 Dataset 컬럼을 보고 "불필요하다"고 잘못 판단하면
  `collator`로 전달되기 전에 데이터가 제거됨


## 11. collator vs map(batched=True) 요약 비교

| 방식           | 특징                          |
| ------------ | --------------------------- |
| collator     | 동적 padding, 메모리 효율적, 실전 최우선 |
| map(batched) | 전처리 결과 고정, 재현성 높음, 저장 공간 증가 |

멀티모달 + Trainer 조합에서는 **collator 방식이 기본 선택**임.

다음의 선택 기준을 참고:

* `collate_fn`이 유리한 경우
	* 텍스트 길이가 다양해서 동적 패딩이 효율적인 경우임.
	* 데이터 저장 공간을 아끼고 싶은 경우임.
	* 이미지 augmentation을 epoch마다 바꾸고 싶은 경우임(예: random crop).
* `collator(DataCollator)`가 유리한 경우
	* collate_fn과 같은 동작을 하되, 설정값을 객체에 보관하여 재사용성을 높이고 싶은 경우임.
	* 실험별로 collator 인스턴스를 바꿔가며 관리하고 싶은 경우임.
	* 코드 구조상 "전처리 로직을 하나의 컴포넌트로 캡슐화"하고 싶은 경우임.
* `map(batched)`가 유리한 경우
	* 전처리 비용이 크고, 매 epoch 반복하고 싶지 않은 경우임.
	* 전처리 결과를 고정해 재현성을 강하게 확보하고 싶은 경우임.
	* 학습 전처리 파이프라인을 dataset에 "고정" 해두고 싶은 경우임.

## Summary

* Dataset은 `image`, `text`, `label`을 원본 형태로 보관함.
* Trainer가 batch를 요청함.
* collator가 batch를 받아 다음의 키의 column을 생성:
	* `text` => `input_ids`, `attention_mask`
	* `image` => `pixel_values`
	* `label` => `labels`
* Trainer는 `collator` 결과만 모델에 전달함.
* 모델은 멀티모달 입력을 받아 loss를 계산함.

## 응용

* text encoder + vision encoder를 직접 결합하는 커스텀 멀티모달 모델
* AutoProcessor 하나로 모든 전처리가 통합되는 구조의 내부 동작 분석


