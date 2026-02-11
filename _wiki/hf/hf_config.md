---
layout  : wiki
title   : HF-Config
summary : 
date    : 2026-02-11 22:56:59 +0900
updated : 2026-02-11 23:34:17 +0900
tag     : hf config
resource: 31/709D1251394DFBA0DBF7E440BE0A97
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# PretrainedConfig 튜토리얼

Text / Image 의 경우를 나누어서 간단히 소개.

---

## 0. `PretrainedConfig` 란? 

`PretrainedConfig`는 **모델의 구조를 정의하고 재현하기 위한 설정(구성) 클래스**임.
가중치가 아니라, 모델이 어떤 아키텍처와 하이퍼파라미터로 구성되어 있는지를 설명하는 메타데이터를 담당하는 역할임.

모델을 완전히 복원하려면 두 가지 요소가 필요함.

* 가중치 파일 (`model.safetensors`)
* 구조 정의 파일 (`config.json`)

이때 `config.json`의 내용을 정의하는 기반 클래스가 바로 `PretrainedConfig`임.

---

### 일반 모델 Config와의 관계

`BertConfig`, `ViTConfig`와 같은 일반 모델의 설정 클래스는 모두 `PretrainedConfig`를 상속한 구체 구현체임.
사용자가 직접 정의하는 Custom Config 클래스 역시 동일하게 `PretrainedConfig`를 상속하여 작성하는 구조임.

즉, 관계는 다음과 같음.

```
PretrainedConfig
    ├── BertConfig        (텍스트 모델 설정)
    ├── ViTConfig         (이미지 모델 설정)
    └── MyCustomConfig    (사용자 정의 설정)
```

* `PretrainedConfig`는 공통 기능을 제공하는 기반 클래스임.
* `BertConfig`, `ViTConfig`는 특정 모델 구조에 맞게 필드를 확장한 공식 구현체임.
* Custom Config는 사용자가 동일한 방식으로 확장한 사용자 정의 구현체임.
* 세 클래스의 차이는 상속 구조가 아니라 “어떤 구조 정보를 담고 있는가”에 있음.

따라서,

> Custom Config는 일반 모델 Config의 하위 개념이 아니라, 같은 기반 클래스를 공유하는 형제 클래스 관계임.

---

### 주요 기능

* **모델 아키텍처 하이퍼파라미터 저장**
  hidden_size, num_layers, image_size, num_labels 등 모델 구조를 결정하는 구성 정보를 저장하는 역할임.

* **`save_pretrained()` : `config.json` 생성**
  현재 설정을 JSON 파일로 저장하여 구조를 보존하는 기능임.

* **`from_pretrained()` : 동일 구조 재구성**
  저장된 `config.json`을 기반으로 동일한 Config 객체를 복원하는 기능임.

* **라벨 매핑 유지**
  `id2label`, `label2id` 정보를 저장하여 분류 결과 해석이 가능하도록 유지하는 역할임.

---

요약하면,

> `PretrainedConfig`는 모든 모델 설정 클래스의 공통 기반이며,
> 일반 모델 Config와 Custom Config는 이를 상속하여 구조 정보를 정의하는 구현체임.


---

# 1. 텍스트(Text)용 PretrainedConfig

## 1.1 최소 예제: 커스텀 텍스트 분류 Config

```python
from transformers import PretrainedConfig

class MyTextConfig(PretrainedConfig):
    model_type = "my_text"

    def __init__(
        self,
        vocab_size=30522,
        hidden_size=256,
        num_hidden_layers=4,
        num_attention_heads=4,
        intermediate_size=1024,
        num_labels=2,
        id2label=None,
        label2id=None,
        **kwargs
    ):
        super().__init__(**kwargs)

        self.vocab_size = vocab_size
        self.hidden_size = hidden_size
        self.num_hidden_layers = num_hidden_layers
        self.num_attention_heads = num_attention_heads
        self.intermediate_size = intermediate_size

        self.num_labels = num_labels
        self.id2label = id2label or {i: f"LABEL_{i}" for i in range(num_labels)}
        self.label2id = label2id or {v: k for k, v in self.id2label.items()}
```

---

## 1.2 텍스트에서 config 의 주요 요소들

* **vocab_size**
  토크나이저가 사용하는 전체 어휘 집합의 크기이며, 입력 토큰 임베딩 행렬의 첫 번째 차원을 결정하는 요소임.

* **hidden_size**
  모델 내부 표현 벡터의 차원이며, 임베딩·어텐션·FFN 출력의 기본 feature 크기를 규정하는 값임.

* **num_attention_heads**
  Multi-Head Attention에서 병렬로 사용하는 어텐션 헤드의 개수이며, hidden_size가 이 값으로 나누어떨어지도록 설정하는 것이 일반적임.

* **num_hidden_layers**
  Transformer encoder 또는 decoder 블록의 총 개수이며, 모델의 깊이(depth)를 결정하는 하이퍼파라미터임.

* **intermediate_size**
  각 Transformer 블록의 Feed-Forward Network 내부에서 확장되는 차원 크기이며, 일반적으로 hidden_size보다 크게 설정되는 값임.

* **max_position_embeddings**
  모델이 처리할 수 있는 최대 시퀀스 길이이며, positional embedding 테이블의 크기를 정의하는 항목임.

* **type_vocab_size**
  segment embedding의 종류 수이며, 문장 쌍 입력 등에서 구분 정보를 표현하기 위한 차원 설정 값임.

* **pad_token_id**
  패딩 토큰에 해당하는 정수 ID이며, attention mask 및 손실 계산 시 무시할 위치를 식별하는 기준 값임.

* **num_labels**
  분류 문제에서 예측해야 할 클래스의 개수이며, 최종 분류 헤드의 출력 차원을 결정하는 요소임.
---

## 1.3 실제 예제: BERT의 config.json 분석

다음은 `bert-base-uncased`의 대표적인 config 구조임.

```json
{
  "model_type": "bert",
  "hidden_size": 768,
  "num_hidden_layers": 12,
  "num_attention_heads": 12,
  "intermediate_size": 3072,
  "vocab_size": 30522,
  "max_position_embeddings": 512,
  "type_vocab_size": 2,
  "hidden_dropout_prob": 0.1,
  "attention_probs_dropout_prob": 0.1,
  "layer_norm_eps": 1e-12,
  "pad_token_id": 0
}
```

### 항목 해석

| 항목                    | 의미                          |
| :---:                   | -------------------------     |
| model_type              | AutoConfig가 복원 시 참조     |
| hidden_size             | Transformer embedding 차원    |
| num_hidden_layers       | encoder block 수              |
| num_attention_heads     | multi-head attention 수       |
| intermediate_size       | FFN 내부 확장 차원            |
| vocab_size              | tokenizer와 직결              |
| max_position_embeddings | 절대 positional encoding 크기 |
| type_vocab_size         | token_type_ids 차원           |

### 중요한 점

* 이 config만 있으면 BERT 아키텍처를 재구성 가능
* 가중치는 별도로 `pytorch_model.safetensors` (과거 `pytorch_model.bin`) 에 저장
* 구조 정의는 전부 config에 존재

> 과거의 `.bin` 파일은 pickle 기반으로 실행가능 코드를 포함할 수 있는데, 이는 보안에 매우 위험한 요소임.  
> 현재는 `.safetensors` 파일을 사용하며, 순수한 텐서 데이터만 저장됨.
> 추가적으로 zero-copy 로딩 등이 가능해서 로딩 속도가 매우 개선되었음.

---

## 1.3 save/load 실습

```python
cfg = MyTextConfig(num_labels=3)
cfg.save_pretrained("./tmp_cfg")

from transformers import AutoConfig
cfg2 = AutoConfig.from_pretrained("./tmp_cfg", trust_remote_code=True)
```

### 실습 목표

Config가

1. 정상적으로 저장되고
2. 동일 구조로 복원되는지
3. AutoConfig가 올바른 클래스를 선택하는지
   확인하는 과정임.

---

### 1단계: 생성 확인

```python
print(cfg)
print(cfg.num_labels)
```

* `num_labels=3`이 제대로 설정되었는지 확인하는 단계임.
* 내부 속성이 예상대로 구성되었는지 점검하는 과정임.

---

### 2단계: 저장 결과 확인

```python
import json
print(json.load(open("./tmp_cfg/config.json")))
```

확인할 것:

* `config.json` 파일이 생성되었는지
* `"model_type": "my_text"`가 기록되었는지
* `"num_labels": 3`이 반영되었는지

Config가 JSON으로 정상 직렬화되었는지 검증하는 단계임.

---

### 3단계: 복원 확인

```python
print(type(cfg2))
print(cfg2.num_labels)
```

확인할 것:

* 타입이 `MyTextConfig`인지 확인
* `num_labels` 값이 유지되는지 확인

`model_type`을 기반으로 AutoConfig가 올바른 클래스를 선택했는지 점검하는 과정임.

---

### 4단계: round-trip 검증

```python
print(cfg.to_dict() == cfg2.to_dict())
```

* True가 나오면 저장 전후 구조가 동일함을 의미함.
* Config 설계가 안정적임을 확인하는 단계임.

---

### 실습 요약

* Config가 JSON으로 저장 가능함을 검증
* 저장된 정보만으로 동일 구조 복원 가능함을 확인
* AutoConfig가 `model_type`을 올바르게 해석함을 점검
* Hub 업로드 전 기본 안정성 테스트 역할 수행


---

# 2. 이미지(Image)용 PretrainedConfig

## 2.1 최소 예제

```python
class MyImageConfig(PretrainedConfig):
    model_type = "my_image"

    def __init__(
        self,
        backbone_name_or_path="google/vit-base-patch16-224",
        image_size=224,
        num_channels=3,
        num_labels=2,
        id2label=None,
        label2id=None,
        **kwargs
    ):
        super().__init__(**kwargs)

        self.backbone_name_or_path = backbone_name_or_path
        self.image_size = image_size
        self.num_channels = num_channels

        self.num_labels = num_labels
        self.id2label = id2label or {i: f"LABEL_{i}" for i in range(num_labels)}
        self.label2id = {v: k for k, v in self.id2label.items()}
```

---

## 2.2 이미지에서 config의 주요 요소들

* **image_size**
  입력 이미지의 한 변 길이이며, 패치 분할 또는 convolution 연산 전 입력 해상도를 정의하는 기준 값임.

* **patch_size**
  Vision Transformer 계열에서 이미지를 분할하는 패치의 한 변 크기이며, 토큰 개수와 패치 임베딩 차원 계산에 직접 영향을 주는 요소임.

* **num_channels**
  입력 이미지의 채널 수이며, 일반적으로 RGB의 경우 3으로 설정되는 모델 입력 구조 정의 값임.

* **hidden_size**
  **"패치 임베딩"** 또는 backbone 출력 특징 벡터의 차원이며, Transformer 블록 또는 분류 헤드 입력 차원을 규정하는 값임.

* **num_hidden_layers**
  Transformer 블록 또는 backbone 내부 반복 블록의 개수이며, 모델의 깊이(depth)를 결정하는 하이퍼파라미터임.

* **num_attention_heads**
  Vision Transformer에서 사용하는 multi-head attention의 헤드 개수이며, hidden_size와의 정합성이 요구되는 구조적 설정 값임.

* **intermediate_size**
  Transformer 블록의 Feed-Forward Network 내부 확장 차원 크기이며, hidden_size 대비 확장 비율을 결정하는 요소임.

* **hidden_dropout_prob**
  은닉층 출력에 적용되는 dropout 확률이며, 과적합 방지를 위한 정규화 강도를 조절하는 하이퍼파라미터임.

* **attention_probs_dropout_prob**
  attention score에 적용되는 dropout 확률이며, attention 단계에서의 정규화 수준을 정의하는 값임.

* **num_labels**
  이미지 분류 문제에서 예측해야 할 클래스 수이며, 최종 분류 헤드의 출력 차원을 결정하는 핵심 요소임.


---

## 2.3  실제 예제: ViT의 config.json

다음은 `google/vit-base-patch16-224`의 대표적 config 구조임:

```json
{
  "model_type": "vit",
  "hidden_size": 768,
  "num_hidden_layers": 12,
  "num_attention_heads": 12,
  "intermediate_size": 3072,
  "image_size": 224,
  "patch_size": 16,
  "num_channels": 3,
  "hidden_dropout_prob": 0.0,
  "attention_probs_dropout_prob": 0.0,
  "layer_norm_eps": 1e-12,
  "initializer_range": 0.02
}
```

### 항목 해석

| 항목                | 의미                   |
| :---:               | -------------------    |
| image_size          | 입력 이미지 한 변 크기 |
| patch_size          | patch 분할 크기        |
| num_channels        | RGB=3                  |
| hidden_size         | patch embedding 차원   |
| num_hidden_layers   | Transformer block 수   |
| num_attention_heads | attention head 수      |

### 참고: BERT와 ViT 주요의 주요 속성비교

| BERT                    | ViT                     |
| ----------------------- | ----------------------- |
| vocab_size              | image_size / patch_size |
| token embedding         | patch embedding         |
| max_position_embeddings | 2D positional embedding |
| type_vocab_size         | 없음                      |

구조는 둘 다 Transformer 기반이지만
입력 토큰 생성 방식에서 차이가 있음.

## 2.4 

---

# 3. 참고 - AutoClass 에 등록하기


## 3.1 로컬 registry 방식

```python
AutoConfig.register("my_text", MyTextConfig)
```

* 현재 세션에서만 유효한 등록임.
* Hub와 무관함 (Hub에 등록하려면 다음을 참고)

---

## 3.2 Hub 배포용 방식

Custom config의 python 소스 코드 파일 마지막에 다음을 추가:

```python
MyTextConfig.register_for_auto_class("AutoConfig")
```

이렇게 하면 `config.json`에 `_auto_map` 필드가 추가됨.

이는 Hub에서 다음을 통해 config를 로드할 수 있게 해 줌:

```python
AutoConfig.from_pretrained("repo", trust_remote_code=True)
```

이는 이미지의 경우도 동일:

```python
MyImageConfig.register_for_auto_class("AutoConfig")
```

---

# 4. 주의

* `__init__`에서 속성 추가시 반드시 JSON 직렬화 가능한 값만 사용할 것.
* `model_type`은 가급적 변경하지 않는 것이 안전: 처음부터 좋은 이름으로...
* `num_labels` / `id2label` / `label2id` 는 거의 항상 필요함.

보통 `config`는 "구조", `processor`는 "입력 규격" 을 결정.


 
