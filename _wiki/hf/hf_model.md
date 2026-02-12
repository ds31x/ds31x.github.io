---
layout  : wiki
title   : HF-Model
summary : 
date    : 2026-02-11 23:54:23 +0900
updated : 2026-02-12 09:45:52 +0900
tag     : 
resource: 96/D94CEC777847F2A858477156E4D7BD
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# PreTrainedModel 튜토리얼 

Text / Image 의 경우를 나누어서 간단히 소개


# 0. `PreTrainedModel` 이란?

`PreTrainedModel`은 **Hugging Face Transformers에서 “모델 본체”의 공통 기능을 제공하는 기반 클래스**임.

> 여기서 “모델 본체”란,  
> 곧, **신경망 module(=layers) + 가중치 로딩/세이브 규약 + HF Hub 호환 인터페이스**를 의미함.

`PretrainedConfig`가 “구조 정의(설계도)”라면,  
`PreTrainedModel`은 

* 그 설계도(`config`)를 받아서 
* **실제 레이어들을 만들고** (`__init__`)
* 연결하고 (`forward`), 
* 가중치를 붙여서 동작 가능한 모델로 만드는 구현체**임.

모델을 완전히 복원하려면 보통 아래 2가지를 함께 사용함.

* 구조 정의: `config.json` (=`PretrainedConfig`가 직렬화된 결과)
* 가중치: `model.safetensors` (또는 shard들)

그리고 이 둘(`config.json`, `model.safetensors`)을 함께 취급해서

* `save_pretrained()`로 저장하고
* `from_pretrained()`로 로드하는

기능을 제공하는 쪽이 `PreTrainedModel`임.


## Config / Model / 입력 객체의 역할 분리

여기서 매우 중요한 구조적 분리가 존재함.

* **Config (`PretrainedConfig`)**: 모델 구조를 재현하기 위한 메타데이터(하이퍼파라미터)
* **Model (`PreTrainedModel`)**: 레이어 구현, forward, 저장/로드 규약(가중치 포함)
* **입력 전처리 객체**:
  * Text: `Tokenizer`
  * Image: `ImageProcessor`
  * Multimodal: `Processor`

즉, 다음을 기억할 것:

* `config`는 **모델이 어떻게 생겼는가**
* `processor/tokenizer`는 **입력이 어떻게 들어와야 하는가**
* `model`은 **실제로 계산을 수행하는 본체**

이 역할 분리를 정확히 지키는 것이 HF 생태계에서 재현성과 HF Hub 호환성의 핵심임.


## 0.1 `PreTrainedModel`이 실제로 제공하는 기능

`PreTrainedModel`은 단순한 `nn.Module`이 아니라,  
다음의 기능들을 제공하는 고수준 추상화 모델임.

* **Hub 로부터 복원 contract**
* **Config와 결합 구조**
* Auto클래스와 연동   

주요 기능은 다음과 같음:

### 1) Config와의 결합

다음과 같이 `PreTrainedModel`의 자식 클래스는 `config_class`를 가짐:

```python
class MyModel(PreTrainedModel):
    config_class = MyConfig
```

* 이 모델이 어떤 `PretrainedConfig`를 받는지 명시
* `AutoModel` 에서 dispatch의 기준이 됨
* Hub의 `auto_map` 메타데이터와 연결됨
* `from_pretrained()`가 config 기반으로 올바른 모델 클래스를 선택할 수 있게 함

### 2) `from_pretrained()`의 내부 복원 절차

보통은 PreTrainedModel 이 제공하는 메서드를 그냥 사용하면 되지만,  
특별한 기능 등이 필요하다면 overridding 가능함.

```python
model = MyModel.from_pretrained("repo_id")
```

동작 단계:

1. `config.json` 로드
2. 해당하는 `Config` 객체 생성
3. 모델 인스턴스 생성 (`__init__(config)`)
4. 가중치 파일(`model.safetensors` 또는 shard) 로드
5. `state_dict` 매핑 및 로딩
6. missing / unexpected key 검사
7. dtype / device 설정
8. `tie_weights()` 및 내부 초기화 정리
9. 기본적으로 `eval()` 모드 설정

즉, `from_pretrained()`는 단순한 `load_state_dict` 이 아니라  
**Hub 의 contract 전체를 복원하는 고수준의 복합 복원 메서드** 에 해당함.

### 3) state_dict 로딩 전략

HF는 일반 PyTorch의 경우와 달리 다음을 처리함:

* missing keys 처리
* unexpected keys 처리
* sharded weight 로딩
* safetensors 지원
* `low_cpu_mem_usage=True` 옵션 통해 메모리 최적화 로딩.
* `device_map="auto"` 기반 분산 로딩
* `dtype` 자동 캐스팅.

이는 일반 PyTorch 모델의 `load_state_dict` 와 구조적으로 다른 중요한 차이점임.

### 4) `post_init()`의 역할

Custom 모델 작성 시 생성자(`__init__`)에서 다음의 hook 호출이 필요.:

```python
self.post_init()
```

이 호출을 통해 수행되는 내용은 다음과 같음:

* weight initialization 수행: `self.init_weights()` 실행
* `self.tie_weights()` 실행
* gradient checkpointing 관련 초기화
* 내부 hook 정리

이를 호출하지 않으면 HF 기대 동작과 어긋날 수 있음.

### 5) 저장 규약

```python
model.save_pretrained("./out")
```

결과는 다음과 같음:
```text
out/
 ├── config.json
 ├── model.safetensors
 └── (필요 시 shard 파일들)
```

Config와 Weight는 항상 분리 저장됨.


## 0.2 `PretrainedConfig`와의 구조적 관계

구조-구현 분리 관점에서:

```
PretrainedConfig  :  구조 메타데이터
PreTrainedModel   :  실제 레이어 구현
```

* Config: 
	* 레이어 수, 
	* hidden size, 
	* dropout, 
	* label mapping,
	* model_type 
	* 등의 구조 및 라벨 관련 **선언적 정의** 담당.
* Model: 
	* `nn.Linear`, `nn.LayerNorm`, backbone 연결, 
	* `forward` 등 
	* 실제 **실행을 구현** 하는 객체.

즉,

> Config는 “설계도”,
> Model은 “설계도를 구현한 실행 객체”.


## 0.3 `nn.Module`과의 차이

| 항목                   | nn.Module | PreTrainedModel     |
| :---:                  | :---:     | ------------------- |
| config 결합            | 없음      | 강제 결합           |
| Hub 저장               | 수동 구현 | `save_pretrained()` |
| Hub 복원               | 수동 구현 | `from_pretrained()` |
| AutoModel 연동         | 불가      | 가능                |
| sharded loading        | 없음      | 지원                |
| dtype/device 자동 처리 | 제한적    | 지원                |

따라서,

* HF 생태계에서 모델을 배포하거나 재현 가능하게 만들려면 
* `PreTrainedModel` 상속이 필수적임.


## 0.4 Hub 복원 전체 흐름

```
repo_id
   ↓
config.json
   ↓
Config 객체 생성
   ↓
AutoModel dispatch (또는 명시적 클래스)
   ↓
PreTrainedModel __init__
   ↓
model.safetensors 로딩
   ↓
완전 복원된 모델
```

Processor는 별도의 경로로 복원되며, 모델 내부에 포함되지 않음.


---

# 1. 텍스트(Text)용 PreTrainedModel

텍스트 모델은 보통

* 입력: `input_ids`, `attention_mask` (필요 시 `token_type_ids`)
* 출력: `logits` 또는 `BaseModelOutput` 계열

형태를 가짐.


## 1.1 텍스트에서 Tokenizer의 역할

Tokenizer는 텍스트에서의 전처리 객체임.

이미지에서의 `ImageProcessor` 와 동일한 개념적 역할을 수행함.

다음의 기능을 제공:

* 문자열을 subword token 단위로 분해
* 각 token을 vocabulary index로 매핑
* special token 들 추가 삽입
* padding 수행
* attention mask 생성
* segment id 생성(필요 시)

예:

```python
tokens = tokenizer("Hello world", return_tensors="pt")
```

* 이때 반환되는 것은 이미 모델이 처리 가능한 텐서 구조임.
* Model은 문자열을 직접 처리하지 않음.

즉,

* Tokenizer는 **입력 규격 정의 및 변환 수행 객체** 이고
* Model은 "계산 객체"임.


## 1.2 최소 예제: 커스텀 텍스트 분류 모델


```python
import torch
import torch.nn as nn
from transformers import PreTrainedModel, AutoModel
from transformers.modeling_outputs import SequenceClassifierOutput

class MyTextForSequenceClassification(PreTrainedModel):
    config_class = MyTextConfig

    def __init__(self, config: MyTextConfig):
        super().__init__(config)

        self.backbone = AutoModel.from_pretrained(config.backbone_name_or_path)
        hidden_size = self.backbone.config.hidden_size

        self.classifier = nn.Linear(hidden_size, config.num_labels)

        self.post_init()

    def forward(self, input_ids=None, attention_mask=None, labels=None, **kwargs):
        out = self.backbone(input_ids=input_ids, attention_mask=attention_mask, **kwargs)
        pooled = out.last_hidden_state[:, 0]
        logits = self.classifier(pooled)

        loss = None
        if labels is not None:
            loss = nn.CrossEntropyLoss()(logits, labels)

        return SequenceClassifierOutput(loss=loss, logits=logits)
```

여기서 중요한 점은 다음과 같음: 

* `config_class = MyTextConfig`로 "이 모델은 이 Config를 받는다"를 명시
* `super().__init__(config)` 호출로 HF 내부 규약 초기화
* `post_init()` 호출로 HF가 기대하는 초기화 루틴 정리

Tokenizer는 모델 내부에 포함되지 않음.  
전처리는 항상 외부에서 수행됨.

## 1.3 텍스트 모델에서 ModelOutput 사용 이유

위 예제에서:

```Python
return SequenceClassifierOutput(loss=loss, logits=logits)
```

을 사용한 이유는:

HF Trainer가 다음 키를 기대하기 때문임:

* `loss`
* `logits`
* `hidden_states`
* `attentions`

ModelOutput은 다음의 특징을 가짐:

* `dict`처럼 접근 가능
* `attribute` 접근 가능
* `tuple` 처럼 언패킹 가능

> 즉,  
> ModelOutput은  
> Trainer / pipeline / AutoModel 계층과의 인터페이스를 맞추기 위한 구조임.

---

# 2. 이미지(Image)용 PreTrainedModel

이미지 모델은 보통

* 입력: `pixel_values` (B,C,H,W)
* 출력: `logits` 또는 `ImageClassifierOutput`

형태를 가짐.


## 2.1 ImageProcessor의 역할

ImageProcessor는 이미지에서의 전처리 객체임.

수행하는 일:

* resize
* crop
* normalize
* tensor 변환
* 채널 순서 정리
* batch dimension 처리

예:

```python
inputs = processor(image, return_tensors="pt")
```

출력:

* `pixel_values`

이미지의 경우도 Model은 이미지 원본을 직접 처리하지 않음.


## 2.2 최소 예제: 커스텀 이미지 분류 모델


```Python
from transformers.modeling_outputs import ImageClassifierOutput

class MyImageForImageClassification(PreTrainedModel):
    config_class = MyImageConfig

    def __init__(self, config: MyImageConfig):
        super().__init__(config)

        self.backbone = AutoModel.from_pretrained(config.backbone_name_or_path)

        hidden_size = getattr(self.backbone.config, "hidden_size", None)
        if hidden_size is None:
            raise ValueError("backbone.config.hidden_size 를 찾을 수 없습니다.")

        self.classifier = nn.Linear(hidden_size, config.num_labels)
        self.post_init()

    def forward(self, pixel_values=None, labels=None, **kwargs):
        out = self.backbone(pixel_values=pixel_values, **kwargs)

        pooled = out.last_hidden_state[:, 0]
        logits = self.classifier(pooled)

        loss = None
        if labels is not None:
            loss = nn.CrossEntropyLoss()(logits, labels)

        return ImageClassifierOutput(loss=loss, logits=logits)
```


## 2.3 이미지 모델에서 Backbone 설계 시 고려사항


이미지 모델의 경우 backbone 종류가 다양함:

* ViT (Transformer 기반)
* CNN (ResNet, DenseNet 등)
* Hybrid

따라서 다음의 방식이 항상 동작한다고 기대하기 어려움:

```
hidden_size = getattr(self.backbone.config, "hidden_size", None)
```

CNN의 경우:

* feature map flatten 필요
* global pooling 필요
* config에 hidden_size가 없을 수 있음

즉, Image 모델 wrapper는 backbone 출력 규격을 정확히 이해해야 함.

이는 Text 모델보다 구현 난이도가 높음.

---

# 3. Tokenizer와 ImageProcessor를 Processor로 추상화할 수 있는가?

## 3.1 개념적 구조

**공통 구조:**

```
Raw Input → Processor → Tensor → Model
```

**Text:**

```
문자열 → Tokenizer → input_ids → Model
```

**Image:**

```
이미지 → ImageProcessor → pixel_values → Model
```

* Tokenzier와 ImageProcessor 둘은 모두 "전처리 객체"라는 동일한 개념적 역할을 수행함.
* 이는 Processor 라는 클래스로 추상화됨.


## 3.2 실제 구현 클래스들

구현 계층에서는 차이점이 존재함:

| 항목          | Tokenizer                 | ImageProcessor         |
| :---:         | :---:                     | :---:                  |
| 기반 클래스   | `PreTrainedTokenizerBase` | `ImageProcessingMixin` |
| 내부 알고리즘 | BPE/WordPiece             | Resize/Normalize       |
| 출력 키       | `input_ids`               | `pixel_values`         |

따라서:

* 추상 개념으로는 Processor로 묶을 수 있음
* 구현 레벨에서는 완전히 다른 클래스 사용. 

## 3.3 Processor와 Model은 절대 결합하면 안 되는 이유

중요 설계 원칙:

* Processor는 입력 규격을 정의
* Model은 계산을 정의

Model 내부에 Tokenizer나 ImageProcessor를 넣으면:

* 저장 시 Hub 구조가 깨짐
* 재현성 저하
* AutoProcessor와 충돌

따라서 HF 구조는 의도적으로 분리되어 있음.

---

# 4. Processor for Multimodal Models

대표 예:

* CLIP (Contrastive Language-Image Pretraining)
* `"openai/clip-vit-base-patch32"`

CLIP은 다음으로 구성됨:

* Text encoder
* Vision encoder
* 통합 Processor


## 4.1 CLIPProcessor 내부 구조

```python
class CLIPProcessor(ProcessorMixin):
    def __init__(self, tokenizer, image_processor):
        self.tokenizer = tokenizer
        self.image_processor = image_processor
```

즉,

> Processor는 실제 전처리를 구현한다기보다  
> 여러 전처리 객체를 보유하는 wrapper임.


## 4.2 Hub 저장 구조

```
repo/
 ├── config.json
 ├── model.safetensors
 ├── tokenizer.json
 ├── tokenizer_config.json
 ├── special_tokens_map.json
 ├── preprocessor_config.json
 ├── processor_config.json
```

processor_config.json은 다음의 정보를 가짐:

* 어떤 tokenizer
* 어떤 image_processor
* 어떤 auto_map


## 4.3 AutoProcessor 동작

```python
from transformers import AutoProcessor
processor = AutoProcessor.from_pretrained("openai/clip-vit-base-patch32")
```

동작 순서:

1. processor_config.json 존재 여부 확인
2. 존재 시 Processor 클래스 로드
3. 내부에서 tokenizer + image_processor 각각 로드

---

# 5. AutoModel과의 관계

`AutoModel`은 모델 클래스 선택을 담당하는 factory 계층임.

```python
from transformers import AutoModel
model = AutoModel.from_pretrained("bert-base-uncased")
```

동작은 다음과 같은 순서로 진행됨:

1. config.json 로드
2. `model_type` 확인
3. 대응되는 모델 클래스 선택
4. 해당 클래스의 `from_pretrained` 호출

Custom 모델도 다음과 같이 `AutoModel` 에 등록 가능함:

```python
MyModel.register_for_auto_class("AutoModel")
```

> AutoModel은 모델을 구현하지 않음.  
> Config 기반으로 적절한 PreTrainedModel 클래스를 선택하는 dispatcher임.

---

# 6. 실제 Hub 로부터 복원 동작 방식

다음의 순서를 따름:

```text
repo_id
   ↓
config.json
   ↓
Config 객체 생성
   ↓
AutoModel dispatch
   ↓
PreTrainedModel __init__
   ↓
model.safetensors 로딩
   ↓
완전 복원된 모델
```
* Processor는 별도로 복원됨.

# 요약

## 단일 모달(Text)

```
Raw Text
   ↓
Tokenizer
   ↓
input_ids
   ↓
PreTrainedModel
   ↓
logits
```

## 단일 모달(Image)

```
Raw Image
   ↓
ImageProcessor
   ↓
pixel_values
   ↓
PreTrainedModel
   ↓
logits
```

## 멀티모달(CLIP)

```
Raw(Text + Image)
       ↓
   Processor
    /      \
Tokenizer   ImageProcessor
       ↓
   PreTrainedModel
```

## 기억하기

1. PreTrainedModel은 Hub 규약을 구현한 `nn.Module` 확장 클래스
2. Config는 구조 정의
3. Model은 실행 구현
4. `from_pretrained`는 단순 weight load가 아님
5. AutoModel은 config 기반 dispatcher
6. Processor는 모델 외부 계층에 해당.
 
