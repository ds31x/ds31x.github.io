---
layout  : wiki
title   : HF - Processor (전처리)
summary : 
date    : 2026-02-12 13:59:26 +0900
updated : 2026-02-19 11:27:56 +0900
tag     : 
resource: 87/038A66A07D49FF9052E19D71612C98
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# Processor

입력 전처리(preprocessing) 객체를 다음으로 나누어 정리.

* **텍스트([[/nlp/tokenization]]{tokenization})**, 
* **이미지(Image preprocessing)**, 
* **Multi-modal 통합(Processor)** 
 
추가적으로 `save_pretrained()` 과  `from_pretrained()` 을 이용한 실습을 추가함.


## 0. 입력 전처리 객체들

모델을 "완전하게 재사용"하려면 보통 다음이 한 set로 있어야함.

* **Model**: 가중치(Model) + 구조(Config)
* **Preprocessor**: 입력을 모델이 입력으로 기대하는 **텐서 형식** 으로 변환

여기서 전처리 객체가 바로 `Tokenizer`, `ImageProcessor`, `Processor` 임.

### 0.1 Tokenizer (텍스트)

* 텍스트를 **토큰 ID(input_ids)**로 바꾸고
* **attention_mask**, (필요시) **token_type_ids** 등을 만들어서
* 텍스트 모델(BERT류, LLM류)이 기대하는 입력 딕셔너리 객체를 생성.

### 0.2 ImageProcessor (이미지)

* 이미지 로딩(옵션), resize/center-crop/normalize 같은 변환을 수행.
* 최종적으로 **pixel_values**(보통 `(B, C, H, W)`) 텐서를 생성. 

### 0.3 Processor (Multi-modal 통합)

* Multi-modal 모델은 보통 “텍스트 전처리”와 “이미지 전처리”가 동시에 필요.
* 그래서 `ProcessorMixin` 기반의 **단일 객체**가 내부적으로 다음을 가짐:
  * Tokenizer
  * ImageProcessor (또는 feature extractor)
* 이를 통해 한 번 호출로 `input_ids` + `pixel_values` 등을 동시에 만들어 모델에 전달

### 0.4 “Processor로 추상화 가능한가?”에 대한 명확한 기준

결론부터 말하면:

* **가능**: 
	* 하나의 모델이 여러 modality 입력을 동시에 받는 경우
	* 예: CLIP(텍스트+이미지), PaliGemma(텍스트+이미지) 등
* **불필요** 한 경우: 
	* "텍스트만" 또는 "이미지만" 받는 순수 단일모달 모델
	* 이 경우 `AutoTokenizer` 또는 `AutoImageProcessor`만으로 충분

Processor는 "새로운 기능"이라기보다,  
**(Tokenizer, ImageProcessor)를 한 객체로 묶어 호출/저장/로드를 일관되게 하는 래퍼** 임.

---

## 1. 텍스트(Text) - Tokenizer사용법

### 1.1 예제: AutoTokenizer

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("bert-base-uncased")
# tokenizer.json, tokenizer_config.json, vocab.txt 등이
# ~/.cache/huggingface 밑에 서브디렉토리에 캐싱됨.

batch = tok(
    ["hello world", "this is a test"],
    padding=True,         # 배치 내 길이 맞춤
    truncation=True,      # 너무 길면 자름
    return_tensors="pt",  # "pt" | "np" | "tf"
)

print(batch.keys())
# dict_keys(['input_ids', 'token_type_ids', 'attention_mask'])  (모델에 따라 다름)
print(batch["input_ids"].shape)
```

Tokenizer는 
* 텍스트를 토큰화하고 
* 토큰의 열을 숫자 텐서로 바꾸는 
* "모델 입력 생성기" 임

### 1.2 저장/로드: tokenizer 아티팩트

Tokenizer를 저장하면 보통 출력 디렉토리에 

* `tokenizer_config.json`, 
* `special_tokens_map.json`, 
* **vocab** 파일 등이 생성 
	* Transformer 5.x에선 `tokenizer.json` 이 vocabrary 정보를 같이 가지는 형태를 취함 
	* `vocab.txt` 는 `.save_pretrained()`로 생성되지 않음 (transformer 5.x기준)
* 이는 모델/토크나이저 종류에 따라 구성은 다름.

```python
save_dir = "./tmp_tok"
tok.save_pretrained(save_dir)

tok2 = AutoTokenizer.from_pretrained(save_dir)
print(type(tok2), tok2.vocab_size)
```

### 1.3 주의사항

* `padding=True`는 배치 텐서화를 위해 사실상 필수 (batch내에 가장 긴 샘플에 맞추어짐.)
* `truncation=True`는 최대 길이 초과 시 안전장치
* 특수 토큰 추가 후에는 `save_pretrained()`로 결과를 고정해두는 습관이 필요함

---

## 2. 이미지(Image) - ImageProcessor 사용법

### 2.1 예제: AutoImageProcessor로 pixel_values 만들기

```python
from transformers import AutoImageProcessor
from PIL import Image

proc = AutoImageProcessor.from_pretrained("google/vit-base-patch16-224")

img_url = "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/cats.png"
img = Image.open(requests.get(img_url, stream=True).raw).convert("RGB")
# img = Image.open("cat.jpg").convert("RGB")

batch = proc(images=[img], return_tensors="pt")
print(batch.keys())
# dict_keys(['pixel_values'])  (모델에 따라 추가 key가 있을 수 있음)
print(batch["pixel_values"].shape)
# (1, 3, 224, 224) 같은 형태
```

ImageProcessor는 
* resize/normalize 등 이미지 변환을 수행하고 
* vision 모델 입력 텐서를 생성.

### 2.2 저장/로드: preprocessor_config.json

ImageProcessor는 
* `save_pretrained()` 시 보통 `preprocessor_config.json`을 생성
* `from_pretrained()`로 동일 설정을 복원

```python
save_dir = "./tmp_imgproc"
proc.save_pretrained(save_dir)

proc2 = AutoImageProcessor.from_pretrained(save_dir)
print(type(proc2))
```

### 2.3 실전 팁: numpy로 변환, use_fast, resize/normalize

* `datasets.map()`과 결합할 때는 `pixel_values`를 **numpy로 변환** 하여 
* 저장 안정성을 확보하는 방식이 많이 사용함.
* 이는 일부 환경에서 torch 텐서 저장 을 사용하는 경우보다 보다 높은 호환성을 보이기 때문임.
* 일부 모델은 `use_fast=True` 옵션으로 빠른 이미지 프로세서를 지원


참고: [[/hf_dataset_dict/dd_map]]

---

## 3. Multi-modal - Processor 사용법 (CLIP 등)

Tokenizer 와 ImageProcessor 가 둘 다 필요한 경우 사용되는 보다 추상화된 클래스.

CLIP (Contrastive Language-Image Pretraining) 등의 multi-modal model 에서 주로 사용됨.

### 3.1 AutoProcessor의 목적

`AutoProcessor`는 
* Multi-modal 입력(예: 텍스트+이미지)을 
* 한 번에 처리하기 위해 존재. 
* 내부적으로 Tokenizer와 ImageProcessor 로 구성.
* 한 번 호출로 모델 입력 딕셔너리를 구성

### 3.2 CLIP처럼 “통합 Processor”가 있는 경우

CLIP은 텍스트 인코더 + 이미지 인코더를 함께 쓰는 대표적 Multi-modal 모델:

* Contrastive: 서로 다른 쌍을 구분하도록 학습하는 대조 학습 방식
* Language–Image: 텍스트와 이미지를 함께 다루는 멀티모달 구조
* Pretraining: 대규모 데이터로 사전학습을 수행하는 단계

```python
from transformers import AutoProcessor

processor = AutoProcessor.from_pretrained("openai/clip-vit-base-patch32")

out = processor(
    text=["a photo of a cat", "a photo of a dog"],
    images=[Image.open("cat.jpg").convert("RGB"), Image.open("dog.jpg").convert("RGB")],
    return_tensors="pt",
    padding=True,
)

print(out.keys())
# input_ids, attention_mask, pixel_values ... 같은 형태
```

Processor가 제공되는 경우, 이를 사용하면 간단히 처리가능하지만 모델에서 어떤 Tokenizer와 ImageProcessor를 요구하는지 등을 파악하는 것이 좋음. 특정한 조합이 요구되면서 Auto로 Processor가 제공되지 않는 경우엔 이를 조합하여 만들어야하며, 커스터마이징을 위해서도 내부 구성을 파악하고 있는 것이 좋음.

* `processor_config.json` 을 살펴볼 것.
* 흔히 `processor.tokenizer` 및 `processor.image_processor` 등의 속성으로 지원됨.

> Processor는 내부적으로 Tokenizer + ImageProcessor를 감싸는 래퍼(wrapper) 객체

---

## 4. AutoClass 및 AutoModel과의 관계

### 4.1 입력 전처리 객체(Processor)와 모델(Model)은 pair임.

* 모델이 기대하는 입력 키는 모델 아키텍처에 따라 다름.
  * 텍스트: `input_ids`, `attention_mask`, ...
  * 비전: `pixel_values`
  * Multi-modal: 둘 다 필요
* 따라서 "모델 - 전처리기"는 사실상 pair(짝)으로 움직여야 안전.
* Config가 "구조", processor가 "입력 규격"을 결정: Config는 모델 내부에 attribute로 존재하는 경우가 일반적.

### 4.2 AutoTokenizer / AutoImageProcessor / AutoProcessor는 무엇을 기준으로 클래스를 고르나

* Auto 계열은 저장된 repository/directory 의 메타데이터 JSON 파일을 읽고, 
* 그에 맞는 **실제 클래스** 를 동적으로 선택해 로드.
* Processor 등록/연동도 AutoClass 체계 안에서 이뤄짐.

주로 메타데이터가 저장된 JSON 파일의 특정 필드에 의존:

| Auto 클래스           | 1차 기준 파일                                                            | 실제 판단 키                         |
| :----------------: | :-----------------------------------------------------------------: | :-----------------------------: |
| AutoConfig         | config.json                                                         | model_type / auto_map           |
| AutoModel          | config.json                                                         | model_type / auto_map           |
| AutoTokenizer      | config.json (+ tokenizer_config.json 보조)                            | model_type / auto_map           |
| AutoImageProcessor | preprocessor_config.json (없으면 config.json fallback)                 | image_processor_type / auto_map |
| AutoProcessor      | processor_config.json (없으면 preprocessor_config.json 또는 config.json) | processor_class / auto_map      |

* AutoImageProcessor는 config.json을 fallback으로 사용할 수 있음
* AutoProcessor는 processor_config.json이 없으면 다른 config를 참고함
* AutoTokenizer는 tokenizer_config.json도 보조로 참고함.

주의할 점은 다음과 같음:
* HF의 transformers에서 기본적으로 제공하는 표준 모델들은 `model_type`의 값에 따라 내장된 매핑으로 클래스를 고름.
* Custom Class 들의 경우, `auto_map` 필드를 참고함.

`BERT` 의 경우 `model_type`가 `bert`로 되어 있으며, 이를 통해 다음이 결정됨:

* AutoConfig → BertConfig
* AutoModel → BertModel
* AutoTokenizer → BertTokenizer / BertTokenizerFast
* AutoModelForMaskedLM → BertForMaskedLM

Custom model 의 경우 `trsut_remote_code=True` 옵션을 사용하며, `config.json`에 다음의 정보가 있음:

```json
{
  "auto_map": {
    "AutoConfig": "configuration_xxx.MyConfig",
    "AutoModel": "modeling_xxx.MyModel",
    "AutoTokenizer": "tokenization_xxx.MyTokenizer"
  }
}
```
* 해당 model은 `auto_map` 필드를 참고하여 관련된 클래스를 import 하게 됨.

### 4.3 AutoModel은 어떤 입력을 기대하나

* `AutoModel`(또는 `AutoModelForXxx`)은 config에 맞는 모델 클래스를 로드 
* `forward(...)`에서 특정 입력 key(키)를 기대
* 실무적으로는 
	* AutoModel을 로드한 뒤, 
	* 해당 모델에 맞는 AutoTokenizer/AutoImageProcessor/AutoProcessor를 같이 로드 하는 패턴이 일반적.

---


## 5. `ImageProcessingMixin`으로 Custom ImageProcessor 만들기

### 5.1 언제 Custom ImageProcessor가 필요한가

다음 중 하나라도 해당되면 Custom ImageProcessor 가  필요.

* **모델 입력 규격이 표준 전처리와 다름**: 예) 6채널, 특수 normalize, 의료영상 전용 windowing 등
* 학습/추론 재현성을 위해 “전처리 설정”을 모델과 함께 배포해야 함
	* ImageProcessor는 `save_pretrained()` 시 
	* 설정을 `preprocessor_config.json`로 저장
* Hub 배포 시 `AutoImageProcessor.from_pretrained()`로 자동 로드 기능 추가.


### 5.2 Custom ImageProcessor 최소 구현 골격

핵심 포인트:

* **`ImageProcessingMixin`**이 `save_pretrained()` / `from_pretrained()` 등 “저장-로드” 루틴을 제공
* 이를 상속하여 해당 기능을 그대로 사용하면 됨.
* 실질적인 전처리 로직은 `__call__`(또는 `preprocess`)에서 구현.
* 주의할 점은 `to_dict()`로 직렬화 가능한 속성만 남겨야 `preprocessor_config.json`이 안정적으로 생성된다는 점임.

아래 예제는 “Resize + Normalize + CHW 텐서”를 만들어 `pixel_values`를 반환.

```python
# custom_image_processor.py
from __future__ import annotations

from dataclasses import dataclass, asdict
from typing import Any, Dict, List, Optional, Union

import numpy as np
import torch
from PIL import Image

from transformers.image_processing_base import ImageProcessingMixin
from transformers.utils.generic import TensorType

@dataclass
class SimpleVisionProcessorConfig:
    size: int = 224
    do_resize: bool = True
    do_normalize: bool = True
    image_mean: List[float] = None
    image_std: List[float] = None

    def __post_init__(self):
        if self.image_mean is None:
            self.image_mean = [0.485, 0.456, 0.406]
        if self.image_std is None:
            self.image_std = [0.229, 0.224, 0.225]


class SimpleVisionImageProcessor(ImageProcessingMixin):
    """
    Custom ImageProcessor (ImageProcessingMixin 기반)
    - images -> pixel_values (B, C, H, W)
    - save_pretrained() => preprocessor_config.json 생성
    """

    model_input_names = ["pixel_values"]  # 관례적으로 지정

    def __init__(self, **kwargs):
        # kwargs는 JSON 직렬화 대상이 되므로 단순 타입 위주로 유지
        self.cfg = SimpleVisionProcessorConfig(**kwargs)

    # ---------
    # 직렬화
    # ---------
    def to_dict(self) -> Dict[str, Any]:
        # ImageProcessingMixin이 save_pretrained()할 때 to_dict()를 사용합니다. :contentReference[oaicite:5]{index=5}
        d = asdict(self.cfg)
        d["processor_class"] = self.__class__.__name__  # 관례적 메타
        return d

    @classmethod
    def from_dict(cls, data: Dict[str, Any], **kwargs) -> "SimpleVisionImageProcessor":
        # from_pretrained() 경로에서 내부적으로 사용될 수 있어 제공해두면 안전합니다.
        data = dict(data)
        data.pop("processor_class", None)
        data.update(kwargs)
        return cls(**data)

    # ---------
    # 전처리
    # ---------
    def _ensure_pil(self, img: Union[Image.Image, np.ndarray]) -> Image.Image:
        if isinstance(img, Image.Image):
            return img
        if isinstance(img, np.ndarray):
            if img.ndim == 2:
                # grayscale -> RGB (단순 예시)
                img = np.stack([img, img, img], axis=-1)
            return Image.fromarray(img.astype(np.uint8))
        raise TypeError(f"Unsupported image type: {type(img)}")

    def _resize(self, img: Image.Image) -> Image.Image:
        if not self.cfg.do_resize:
            return img
        return img.resize((self.cfg.size, self.cfg.size), resample=Image.BILINEAR)

    def _to_chw_float(self, img: Image.Image) -> np.ndarray:
        x = np.array(img).astype(np.float32) / 255.0  # HWC, 0..1
        x = np.transpose(x, (2, 0, 1))                # CHW
        return x

    def _normalize(self, x: np.ndarray) -> np.ndarray:
        if not self.cfg.do_normalize:
            return x
        mean = np.array(self.cfg.image_mean, dtype=np.float32)[:, None, None]
        std = np.array(self.cfg.image_std, dtype=np.float32)[:, None, None]
        return (x - mean) / std

    def __call__(
        self,
        images: Union[Image.Image, np.ndarray, List[Union[Image.Image, np.ndarray]]],
        return_tensors: Optional[Union[str, TensorType]] = "pt",
        **kwargs,
    ) -> Dict[str, Any]:
        if not isinstance(images, list):
            images = [images]

        batch = []
        for im in images:
            im = self._ensure_pil(im).convert("RGB")
            im = self._resize(im)
            x = self._to_chw_float(im)
            x = self._normalize(x)
            batch.append(x)

        pixel_values = np.stack(batch, axis=0)  # (B,C,H,W)

        if return_tensors in ("pt", TensorType.PYTORCH):
            pixel_values = torch.from_numpy(pixel_values)
        elif return_tensors in ("np", TensorType.NUMPY):
            pass
        else:
            raise ValueError(f"return_tensors not supported: {return_tensors}")

        return {"pixel_values": pixel_values}
```

* `preprocessor_config.json`에는 위 `to_dict()` 결과가 저장


위에서 serialization의 `to_dict`와 `from_dict`를 오버라이드 했지만,
* `__init__`에서 `self.xxx = ...` 형태로 값을 직접 저장하는 방식을 사용하고
* 해당 properties 가 JSON 직렬화가 가능한 경우(`int`, `float`, `bool`, `str`, `list`, `dict`) 에는
* 굳이 `to_dict`를 오버라이드 안해도, 부모 클래스의 기본 `to_dict()`가 `self.__dict__`기반으로 직렬화를 수행함.

앞서의 예에선 root property에 직렬화할 속성들이 놓이는 대신에,  
하나의 root property `cfg` 가 존재하고
해당 `cfg` 안에 실제 필요한 속성들이 놓이는 구조였기 때문에  
명시적으로 `to_dict` 를 구현해 주는 것이 좋으나, 다음과 같이 구현하면 굳이 필요하지 않음.

```python
# image_processing_simple_vision.py

from __future__ import annotations

from typing import Any

import numpy as np
import torch
from PIL import Image

from transformers.image_processing_base import ImageProcessingMixin
from transformers.utils.generic import TensorType


class SimpleVisionImageProcessor(ImageProcessingMixin):
    model_input_names = ["pixel_values"]

    def __init__(
        self,
        size: int = 224,
        do_resize: bool = True,
        do_normalize: bool = True,
        image_mean: list[float] | None = None,
        image_std: list[float] | None = None,
        **kwargs: Any,
    ):
        super().__init__(**kwargs)

        # flat attributes only -> 기본 to_dict/save_pretrained로 충분
        self.size = int(size)
        self.do_resize = bool(do_resize)
        self.do_normalize = bool(do_normalize)
        self.image_mean = image_mean if image_mean is not None else [0.485, 0.456, 0.406]
        self.image_std = image_std if image_std is not None else [0.229, 0.224, 0.225]

    def _ensure_pil(self, img: Image.Image | np.ndarray) -> Image.Image:
        if isinstance(img, Image.Image):
            return img
        if isinstance(img, np.ndarray):
            arr = img
            if arr.ndim == 2:
                arr = np.stack([arr, arr, arr], axis=-1)
            if arr.dtype != np.uint8:
                arr = np.clip(arr, 0, 255).astype(np.uint8)
            return Image.fromarray(arr)
        raise TypeError(f"Unsupported image type: {type(img)}")

    def _resize(self, img: Image.Image) -> Image.Image:
        if not self.do_resize:
            return img
        return img.resize((self.size, self.size), resample=Image.BILINEAR)

    def _to_chw_float01(self, img: Image.Image) -> np.ndarray:
        x = np.asarray(img, dtype=np.float32) / 255.0  # HWC
        return np.transpose(x, (2, 0, 1))              # CHW

    def _normalize(self, x: np.ndarray) -> np.ndarray:
        if not self.do_normalize:
            return x
        mean = np.asarray(self.image_mean, dtype=np.float32)[:, None, None]
        std = np.asarray(self.image_std, dtype=np.float32)[:, None, None]
        return (x - mean) / std

    def __call__(
        self,
        images: Image.Image | np.ndarray | list[Image.Image | np.ndarray],
        return_tensors: str | TensorType | None = "pt",
        **kwargs: Any,
    ) -> dict[str, torch.Tensor | np.ndarray]:
        if not isinstance(images, list):
            images = [images]

        batch = []
        for im in images:
            im = self._ensure_pil(im).convert("RGB")
            im = self._resize(im)
            x = self._to_chw_float01(im)
            x = self._normalize(x)
            batch.append(x)

        pixel_values = np.stack(batch, axis=0).astype(np.float32)

        if return_tensors in ("pt", TensorType.PYTORCH):
            pixel_values = torch.from_numpy(pixel_values)
        elif return_tensors in ("np", TensorType.NUMPY) or return_tensors is None:
            pass
        else:
            raise ValueError(f"Unsupported return_tensors: {return_tensors}")

        return {"pixel_values": pixel_values}


# AutoImageProcessor 매핑 정보(auto_map)를 저장되도록 등록
SimpleVisionImageProcessor.register_for_auto_class("AutoImageProcessor")
```

위의 코드를 다음과 같이 저장하면 이후 Auto 클래스를 통해 로드가 가능해짐: 


아래와 같이 `.save_pretrained()` 수행시 지정된 디렉토리에 `preprocessor_config.json`이 저장됨.

```python
from image_processing_simple_vision import SimpleVisionImageProcessor


# 1) 인스턴스 생성
proc = SimpleVisionImageProcessor(size=224)

# 2) save_pretrained()가 preprocessor_config.json을 자동 생성
proc.save_pretrained("./simple_vision_proc")
```
* `auto_map` 항목이 저장 json파일에 생김
* 앞서의 예제와 달리, `.register_for_auto_class()`를 통해 auto 클래스가 찾을 수 있게 됨.
* `.save_pretrained()`의 호출시 관련 processor 에 해당하는 python파일도 해당 디렉토리에 같이 놓임.
* 이를 통해 `trust_remote_code=True` 등을 사용할 수 있고, import정보를 config를 통해 얻은 후 실제 모듈을 import가능해짐.

이후 저장된 `preprocessor_config.json`  이 있는 디렉토리를 지정하여 로드 가능:

```python
from transformers import AutoImageProcessor

p = AutoImageProcessor.from_pretrained(
    "./simple_vision_proc",
    trust_remote_code=True,
)

print(type(p), p.size)
```


### 5.3 저장/로드 실습: `preprocessor_config.json` round-trip

```python
from custom_image_processor import SimpleVisionImageProcessor
from PIL import Image

p = SimpleVisionImageProcessor(size=224, do_resize=True, do_normalize=True)

img = Image.open("cat.jpg").convert("RGB")
out = p(img, return_tensors="pt")
print(out["pixel_values"].shape)  # (1, 3, 224, 224)

save_dir = "./tmp_custom_imgproc"
p.save_pretrained(save_dir)  # preprocessor_config.json 생성 :contentReference[oaicite:7]{index=7}

p2 = SimpleVisionImageProcessor.from_pretrained(save_dir)
out2 = p2(img, return_tensors="pt")
print((out["pixel_values"] - out2["pixel_values"]).abs().max().item())
```

* `SimpleVisionImageProcessor`는 `custom_image_processor.py` 모듈이 module search path에 있는 경우만 로딩이 가능.
* auto_map 필드를 만들어 주지 않았기 때문임.


### 5.4 (선택) AutoImageProcessor로 로드되게 "배포"하는 관점

AutoClass는 

* 저장된 메타데이터 를 보고 
* 적절한 클래스를 고름.

커스텀 클래스까지 자동으로 로드하려면 보통 다음 중 하나를 이용함:

* (권장) 커스텀 클래스에 대해 AutoClass 연동 정보를 저장(예: `register_for_auto_class` / `auto_map` 계열)
* 또는 패키지 형태로 배포하여 import 가능하게 만든 뒤 AutoClass에 등록
* 이 경우 지정한 디렉토리에 JSON 파일 외에도 관련 모듈 python 파일도 저장됨.

AutoClass 확장 개념 자체는 다음등과 동일한 방식을 채택하고 있음: 

* `AutoConfig.register` 
* `AutoModel.register`

---

## 6. 실습 - Tokenizer / ImageProcessor / Processor


### 6.1 실습 A - Tokenizer (텍스트)

#### A-1. 로드

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("bert-base-uncased")
```

#### A-2. 호출(인코딩)

```python
batch = tok(
    ["hello world", "this is a test"],
    padding=True,
    truncation=True,
    return_tensors="pt",
)
print(batch.keys())
print(batch["input_ids"].shape)
```

#### A-3. 저장/복원

```python
save_dir = "./tmp_tok"
tok.save_pretrained(save_dir)

tok2 = AutoTokenizer.from_pretrained(save_dir)
batch2 = tok2(["hello world"], return_tensors="pt")
print(batch2["input_ids"].shape)
```

### 6.2 실습 B - ImageProcessor (비전)

#### B-1. 로드

```python
from transformers import AutoImageProcessor
from PIL import Image

ip = AutoImageProcessor.from_pretrained("google/vit-base-patch16-224")
img = Image.open("cat.jpg").convert("RGB")
```

#### B-2. 호출

```python
x = ip(images=[img], return_tensors="pt")
print(x.keys())  # 보통 pixel_values
print(x["pixel_values"].shape)
```

#### B-3. 저장/복원

```python
save_dir = "./tmp_imgproc"
ip.save_pretrained(save_dir)  # preprocessor_config.json 생성 :contentReference[oaicite:11]{index=11}

ip2 = AutoImageProcessor.from_pretrained(save_dir)
x2 = ip2(images=[img], return_tensors="pt")
print(x2["pixel_values"].shape)
```

ImageProcessor는 모델별 전처리 설정을 `preprocessor_config.json`에 저장


### 6.3 실습 C - Processor 

Multi-modal: CLIP 을 사용한 예제임.

Processor는 Tokenizer + ImageProcessor를 묶어서 "한 번 호출"로 Multi-modal 입력을 생성.

#### C-1. 로드

```python
from transformers import AutoProcessor
from PIL import Image

proc = AutoProcessor.from_pretrained("openai/clip-vit-base-patch32")
img = Image.open("cat.jpg").convert("RGB")
```

#### C-2. 호출(텍스트+이미지 동시)

```python
out = proc(
    text=["a photo of a cat", "a photo of a dog"],
    images=[img, img],
    padding=True,
    return_tensors="pt",
)
print(out.keys())
# input_ids, attention_mask, pixel_values ... (모델/프로세서에 따라 변형) :contentReference[oaicite:14]{index=14}
```

#### C-3. 저장/복원

```python
save_dir = "./tmp_clip_proc"
proc.save_pretrained(save_dir)

proc2 = AutoProcessor.from_pretrained(save_dir)
out2 = proc2(text=["a photo of a cat"], images=[img], return_tensors="pt", padding=True)
print(out2.keys())
```

---

## 7. (권장) 실습 검증 체크리스트

각 실습을 "성공"으로 보기 위한 최소 조건:

* Tokenizer: `input_ids`가 생성되고, 저장/복원 후 동일하게 동작
* ImageProcessor: `pixel_values`가 생성되고, 저장/복원 후 동일하게 동작
* Processor: 텍스트+이미지를 동시에 넣었을 때, 키들이 함께 생성되고 저장/복원 후 동일하게 동작
* Custom ImageProcessor: `preprocessor_config.json`이 생성되고, `from_pretrained()`로 복원 가능


---

## 같이 보면 좋은 자료

* [Tokenizer](https://huggingface.co/docs/transformers/en/main_classes/tokenizer)
* [Convert saved pretrained tokenizers from transformers to](https://github.com/huggingface/tokenizers/issues/230)
* [Image processor-Transformers](https://huggingface.co/docs/transformers/main/ko/image_processors) 
* [Image Processors](https://huggingface.co/docs/transformers/en/main_classes/image_processor)
* [Image processors](https://huggingface.co/docs/transformers/en/image_processors) 
* [transformers/src/transformers/image_processing_base.py](https://github.com/huggingface/transformers/blob/main/src/transformers/image_processing_base.py)
* [Processor](https://huggingface.co/docs/transformers/v4.25.1/main_classes/processors)
* [Processor](https://huggingface.co/docs/transformers/ko/main_classes/processors)
* [Processor](https://huggingface.co/docs/transformers/en/processors)
* [Auto Classes](https://huggingface.co/docs/transformers/en/model_doc/auto) 
* [CLIP](https://huggingface.co/docs/transformers/en/model_doc/clip)


