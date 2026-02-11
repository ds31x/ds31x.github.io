---
layout  : wiki
title   : Pipeline 만들기 - image classifier
summary : 
date    : 2026-02-11 22:07:47 +0900
updated : 2026-02-11 22:46:08 +0900
tag     : hf classification image
resource: DE/F203CAB610435B899DB44433747561
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# image-classification 테스크용 model과 processor

다음이 가능한 `custom_model` 과 `custom_processor` 의 조건을 살펴본다.

```
pipeline(
    task="image-classification",
    model=custom_model,
    image_processor=custom_processor
)
```

## Custom model의 조건.

우선, 모델이 다음 중 하나를 만족해야함:

* `AutoModelForImageClassification` 로 로드 가능
* 또는 `PreTrainedModel` 상속 + `logits` 반환

즉, forward가 다음과 같은 contract를 만족해야 함:

```Python
def forward(self, pixel_values=None, labels=None, **kwargs):
    ...
    return ImageClassifierOutput(
        loss=loss,
        logits=logits
    )
```
* `ImageClassifierOutput`이 권장됨.
* `dict` 객체를 반환해도 되긴 함: `return {"logits": logits}`

`ImageClassifierOutput`은 다음의 구조를 가짐:

```Python
ImageClassifierOutput(
    loss: Optional[Tensor] = None,
    logits: Tensor = None,
    hidden_states: Optional[Tuple] = None,
    attentions: Optional[Tuple] = None,
)
```

> `pixel_values` 를 통해 `(B,C,H,W)`형태의 tensor로 입력을 받아야 함.


모델의 `.config`에 다음의 속성이 필요함:

```Python
config.id2label
config.label2id
config.num_labels
```

## Custom processor의 조건

`processor` 다음과 같은 호출이 가능한 callable 객체여야함:

```Python
processor(images, return_tensors="pt")
```

위와 같은 호출의 return value는 다음의 형태의 `dict`객체여야 함:

```Python
{"pixel_values": tensor}
```
* `pixel_values` 키가 반드시 있어야 함.
* `model(**{"pixel_values": ...})` 의 형태로 모델의 `.forward` signature가 되어있기로 약속됨.

## Custom pipeline 만들기

위의 조건을 만족하는 `model`, `processor` 가 있다면 다음으로 만들 수 있음:

```Python
from transformers import pipeline

pipe = pipeline(
    task="image-classification",
    model=model,
    image_processor=processor,
    device=0
)

out = pipe(image)
```

## Example

```Python
from PIL import Image
import requests

img_url = "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/cats.png"
image = Image.open(requests.get(img_url, stream=True).raw)


from transformers import pipeline
from transformers import AutoModelForImageClassification, AutoImageProcessor

model = AutoModelForImageClassification.from_pretrained(
    "google/vit-base-patch16-224",
    trust_remote_code=True,
)

processor = processor = AutoImageProcessor.from_pretrained(
    "google/vit-base-patch16-224",
    trust_remote_code=True,
    use_fast=False,
)

pipe = pipeline(
    task="image-classification",
    model=model,
    image_processor=processor,
    device=0
)

out = pipe(image)
```

