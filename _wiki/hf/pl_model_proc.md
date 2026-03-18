---
layout  : wiki
title   : Pipeline만들기-image classifier
summary : 
date    : 2026-02-11 22:07:47 +0900
updated : 2026-02-19 12:46:39 +0900
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

`"image-classification"` task의  `pipeline()` 에서  
사용 가능한 `custom_model` 과 `custom_processor` 의 조건을 살펴본다.

```python
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

즉, forward가 다음과 같은 contract(=API)를 만족해야 함:

```python
def forward(self, pixel_values=None, labels=None, **kwargs):
    ...
    return ImageClassifierOutput(
        loss=loss,
        logits=logits
    )
```
* 입력으로 image processor가 만든 `pixel_values`를 받아 `logits`를 반환할 수 있어야 함
* 반환값으 `ImageClassifierOutput`이 권장됨.
* `dict` 객체를 반환해도 되긴 함: `return {"logits": logits}`

`ImageClassifierOutput`은 다음의 구조를 가짐:

```python
ImageClassifierOutput(
    loss: Optional[Tensor] = None,
    logits: Tensor = None,
    hidden_states: Optional[Tuple] = None,
    attentions: Optional[Tuple] = None,
)
```

> `pixel_values` 를 통해 `(B,C,H,W)`형태의 tensor로 입력을 받아야 함.


모델의 `.config`에 다음의 분류 속성이 필요함:

```python
config.id2label
config.label2id
config.num_labels
```

## Custom processor의 조건

> `BaseImageProcessor` 계열과 호환되는 image processor 를 권장.
> 연결될 model이 기대하는 출력을 반환해야함: 

`processor` 다음과 같은 호출이 가능한 callable 객체여야함:

```python
processor(images, return_tensors="pt")
```

위와 같은 호출의 return value는 다음의 형태의 `dict`객체여야 함:

```python
{"pixel_values": tensor}
```
* `pixel_values` 키가 반드시 있어야 함.
* `model(**{"pixel_values": ...})` 의 형태로 모델의 `.forward()`의 signature가 되어있기로 약속됨.

> resize, normalize, tensor 변환 등 수행.

## Custom pipeline 만들기

위의 조건을 만족하는 `model`, `processor` 가 있다면 다음으로 만들 수 있음:

```python
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

다음은 간단하게 pipeline을 만드는 예제임:

1. URL에서 예제 이미지를 가져옴
2. 이미지 분류용 ViT 모델을 로드함
3. 해당 모델에 맞는 이미지 전처리기를 로드함
4. `image-classification` 파이프라인을 구성함
5. 이미지를 넣어 분류 결과를 얻음

```python
# PIL(Image)과 requests를 import함.
# - PIL.Image: 이미지를 열고 다루기 위한 라이브러리
# - requests: URL로부터 파일을 다운로드하기 위한 라이브러리
from PIL import Image
import requests


# 예제 이미지를 내려받을 URL임.
# Hugging Face 문서에서 제공하는 고양이 이미지 예제임.
img_url = "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/cats.png"

# requests.get(..., stream=True).raw 로 URL의 원본 바이너리 스트림을 받아옴.
# 이를 Image.open(...)에 넘겨 PIL Image 객체로 엶.
# 결과적으로 image 변수에는 메모리에 로드된 PIL 이미지가 들어감.
image = Image.open(requests.get(img_url, stream=True).raw)


# Hugging Face의 pipeline과 자동 로딩용 클래스들을 import함.
# - pipeline: 추론 파이프라인을 쉽게 구성하는 고수준 API
# - AutoModelForImageClassification: 이미지 분류용 모델을 자동 선택하여 로드
# - AutoImageProcessor: 해당 모델에 맞는 이미지 전처리기 자동 로드
from transformers import pipeline
from transformers import AutoModelForImageClassification, AutoImageProcessor


# 이미지 분류용 pretrained model을 불러옴.
# "google/vit-base-patch16-224"는 Vision Transformer(ViT) 기반 이미지 분류 모델임.
# trust_remote_code=True:
# - 모델 저장소에 custom code가 있을 경우 이를 신뢰하고 로드하겠다는 뜻임
# - 이 예제 모델에는 꼭 필요하지 않은 경우가 많지만, custom repo에 대비해 넣을 수 있음
model = AutoModelForImageClassification.from_pretrained(
    "google/vit-base-patch16-224",
    trust_remote_code=True,
)

# 위 모델과 짝이 맞는 image processor를 불러옴.
# image processor는 입력 이미지를 모델이 기대하는 형태(pixel_values)로 바꾸는 역할을 함.
# 예:
# - resize
# - rescale
# - normalization
# - tensor 변환
#
# use_fast=False:
# - fast processor 대신 일반 processor를 사용하겠다는 뜻임
# - 모델/환경에 따라 fast 버전이 없거나, 일반 버전을 명시적으로 쓰고 싶을 때 사용 가능함
processor = AutoImageProcessor.from_pretrained(
    "google/vit-base-patch16-224",
    trust_remote_code=True,
    use_fast=False,
)

# image-classification task용 pipeline을 생성함.
# 여기서는 문자열 model id를 넘기지 않고,
# 이미 생성해 둔 model 객체와 image_processor 객체를 직접 넘김.
#
# device=0:
# - GPU 0번 장치를 사용하겠다는 뜻임
# - CUDA가 가능한 환경에서만 정상 동작함
# - CPU에서 실행하려면 보통 device=-1 로 지정함
pipe = pipeline(
    task="image-classification",
    model=model,
    image_processor=processor,
    device=0
)

# PIL 이미지 객체를 pipeline에 넣어 추론 수행
# 내부적으로는 대략 다음 순서로 처리됨.
# 1. processor가 image를 모델 입력 형태로 전처리
# 2. model이 logits 출력
# 3. pipeline이 softmax 등을 적용하여 label과 score로 정리
#
# out은 보통 다음과 같은 dict들의 list 형태임.
# [
#   {"label": "...", "score": ...},
#   {"label": "...", "score": ...},
#   ...
# ]
out = pipe(image)
```

