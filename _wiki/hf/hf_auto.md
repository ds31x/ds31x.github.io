---
layout  : wiki
title   : HF-Auto클래스
summary : 
date    : 2026-02-19 12:55:48 +0900
updated : 2026-02-19 13:24:17 +0900
tag     : hf
resource: 0C/82FF95B3F245478E080F5DCDCB30BA
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# Hugging Face `Auto` 클래스 튜토리얼

Hugging Face Transformers의 **Auto 클래스**는

* [[/hf/hf_model]]{Model}/[[/hf/hf_processor]]{토크나이저/프로세서}를 
* **직접 클래스 이름을 명시하지 않고** 로드할 수 있도록 
* 설계된 Dynamic Factory Class 임.

핵심 개념은 다음과 같음:

> 저장된 `config.json` 또는 관련 JSON 메타데이터를 읽고
> 그에 맞는 실제 Python 클래스를 자동 선택하여 인스턴스를 생성.

Auto 클래스들을 통해

* 다양한 모델들의 실제 관련 클래스명을 모르더라도
* 특정 모델을 식별하는데 사용되는 일종의 식별자인  `model_type`의 문자열만 알면
* 간편하고 일관된 API를 통해 해당 모델 및 관련 클래스들을 사용할 수 있음.
* 이는 다양한 모델을 일관된 API로 사용하게 함으로써 
  특정 task에 대해 여러 모델을 비교하거나, 추론 및 미세조정 등의 작업을 
  매우 쉽게 수행하게 해 줌.
* [[/hf/pipeline]]도 이 Auto 클래스에 의존하고 있음.  

---

# 1. 왜 Auto 클래스가 필요한가

일반적인 방식:

```python
from transformers import BertModel, BertTokenizer

model = BertModel.from_pretrained("bert-base-uncased")
tokenizer = BertTokenizer.from_pretrained("bert-base-uncased")
```

문제점:

* 모델 타입(`BertModel`, `BertTokenizer`)을 **사전에 알고 있어야 함**
* generic pipeline 의 작성이 쉽지 않음
	* Hub 상의 수천 개 모델을 일일이 매핑할 수 없음

Auto 방식:

```python
from transformers import AutoModel, AutoTokenizer

model = AutoModel.from_pretrained("bert-base-uncased")
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
```

이 경우:

* 내부에서 인자로 넘겨진 문자열에 해당하는 `config.json`을 읽고
* `model_type`을 확인한 뒤
* 적절한 클래스를 자동 선택

---

# 2. Auto 클래스의 종류

| Auto 클래스                             | 목적                  |
| ------------------------------------ | ------------------- |
| `AutoConfig`                         | 모델 구조 메타데이터 로드      |
| `AutoModel`                          | base model 로드       |
| `AutoModelForSequenceClassification` | task-specific 모델 로드 |
| `AutoTokenizer`                      | 텍스트 토크나이저           |
| `AutoImageProcessor`                 | 이미지 전처리기            |
| `AutoProcessor`                      | 멀티모달 통합 processor   |

---

# 3. 내부 동작 원리

## 3.1 AutoConfig

```python
from transformers import AutoConfig

config = AutoConfig.from_pretrained("bert-base-uncased")
print(config.model_type)
```

`config.json` 내부:

```json
{
  "model_type": "bert",
  ...
}
```

* `model_type="bert"` 이 핵심 field.
* 이를 보고
* **내부 매핑 테이블** (`transformers`의 기본 모델인 경우이고, custom model이리면 `auto_map`필드를 또 읽어야 함)
* 에서 `BertConfig` 선택

---

## 3.2 AutoModel

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("bert-base-uncased")
```

동작 순서:

1. `AutoConfig` 로 config 로드
2. config.model_type 확인
3. 내부 매핑에서 `BertModel` 선택
4. 해당 클래스의 `from_pretrained()` 호출

즉:

```
repo → config.json → model_type → class 선택 → weight 로드
```

---

# 4. AutoModel vs Task-specific AutoModel

## 4.1 Base 모델

```python
AutoModel
```

* backbone만 로드

## 4.2 Task-specific

```python
AutoModelForSequenceClassification
AutoModelForImageClassification
AutoModelForCausalLM
AutoModelForTokenClassification
```

* backbone 과 head가 같이 로드됨.
* task에 적절한 head가 존재.

예:

```python
model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-uncased",
    num_labels=3
)
```

이 경우 내부적으로:

```
BertForSequenceClassification
```

이 선택됨.

---

# 5. AutoTokenizer

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
```

`config.json` 또는 tokenizer config를 읽고 다음이 결정됨:

* fast tokenizer 사용 여부
* vocab 경로
* tokenizer class

---

# 6. AutoImageProcessor

이미지 모델의 전처리과정에서 사용.

```python
from transformers import AutoImageProcessor

processor = AutoImageProcessor.from_pretrained("google/vit-base-patch16-224")
```

`preprocessor_config.json` 내부 예:

```json
{
  "image_processor_type": "ViTImageProcessor"
}
```

이 값을 보고 실제 클래스 선택.

---

# 7. AutoProcessor (멀티모달)

예: CLIP (Contrastive Language-Image Pretraining) 모델.

```python
from transformers import AutoProcessor

processor = AutoProcessor.from_pretrained("openai/clip-vit-base-patch32")
```

이 경우:

* tokenizer + image processor를 통합한 processor

`processor_config.json`의 `processor_class` 필드 기준으로 선택.

---

# 8. Custom 모델에서 Auto 사용하기

기본 내장 모델이 아닌 custom 모델의 경우 두 가지 방법이 있음.

## 방법 1 - model_type 기반 등록

```python
AutoConfig.register("my_model", MyConfig)
AutoModel.register(MyConfig, MyModel)
```

* Python의 현재 세션에서 Auto클래스가 해당 클래스들을 찾아낼 수 있음.

이 경우:

* `config.json`에 `"model_type": "my_model"` 필요

---

## 방법 2 - auto_map 기반 (권장)

`config.json`에 `auto_map` 필드를 이용하는 방법:

```json
{
  "model_type": "my_model",
  "auto_map": {
    "AutoConfig": "configuration_my.MyConfig",
    "AutoModel": "modeling_my.MyModel"
  }
}
```

* 해당 custom model 및 config 클래스가 정의된 모듈 python 코드파일의 끝에
* `.register_for_auto_class("AutoConfig")`
* `.register_for_auto_class("AutoModel")`
* 를 호출하는 코드를 추가하면
* 위와 같이 config.json에 `auto_map` 이 추가됨.

주의할 점은 반드시 custom config클래스의 등록도 잊으면 안된다는 점임:
```python
# configuration_my.py
from transformers import PretrainedConfig

class MyConfig(PretrainedConfig):
    model_type = "my_model"

MyConfig.register_for_auto_class("AutoConfig")
```
모델에선 `config_class` 속성으로 해당 custom config 클래스를 지정.
```python
# modeling_my.py
from transformers import PreTrainedModel

class MyModel(PreTrainedModel):
    config_class = MyConfig

MyModel.register_for_auto_class("AutoModel")
```

이같은 처리를 해주면 custom model 클래스의 객체에서 `.save_pretrained()`를 호출하면
```python
model.save_pretrained("my_repo")
```
* `config.json` 에 `auto_map` 필드가 생성되며
* 저장하는 디렉토리에 관련 모듈의 소스코드 파일도 저장됨.

이 경우 `config.json`엔 다음의 `auto_map`필드가 생성됨:
```json
"auto_map": {
  "AutoConfig": "configuration_my.MyConfig",
  "AutoModel": "modeling_my.MyModel"
}
```

그리고 로드 시:

```python
AutoModel.from_pretrained("repo", trust_remote_code=True)
```

---

# 9. AutoBackbone

```python
from transformers import AutoBackbone

backbone = AutoBackbone.from_pretrained("microsoft/resnet-50")
```

* CNN
* ViT
* Swin
* ConvNeXt

등 feature extractor 역할만 수행하는 모델을 통합적으로 로드.

(`transformers` 의 4.47.x 이후 정상적으로 사용가능)

---

# 10. 동작 요약!

```
from_pretrained()
      ↓
config.json 로드
      ↓
model_type / auto_map 확인
      ↓
적절한 클래스 선택
      ↓
해당 클래스의 from_pretrained 호출
```

---

# 11. Example-ViT

```python
from transformers import (
    AutoConfig,
    AutoModelForImageClassification,
    AutoImageProcessor
)

repo = "google/vit-base-patch16-224"

config = AutoConfig.from_pretrained(repo)
processor = AutoImageProcessor.from_pretrained(repo)
model = AutoModelForImageClassification.from_pretrained(repo)

print(type(model))
print(type(processor))
```

---

# 12. 핵심 개념 요약

Auto 시스템은

* JSON 기반 메타데이터 중심
* model_type 매핑
* auto_map 확장
* trust_remote_code 옵션
* task-specific model 자동 선택

을 통해

**Hub 중심의 플러그인 아키텍처**를 구현한 시스템이다.


 
