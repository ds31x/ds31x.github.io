---
layout  : wiki
title   : HF-Processor
summary : 전처리를 수행하는 Processor를 설명. Tokenizer와 ImageProcessor 도 같이 다룸.
date    : 2026-02-12 13:59:26 +0900
updated : 2026-02-19 12:48:11 +0900
tag     : hf
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
# %%
from transformers import AutoTokenizer

# 1. 모델에 맞는 토크나이저 로드
# AutoTokenizer는 모델 이름만 주어지면 적절한 토크나이저 클래스를 자동으로 선택함.
# (예: bert-base-uncased의 경우 BertTokenizer를 로드)
# 로드된 데이터(tokenizer.json, vocab.txt 등)는 ~/.cache/huggingface에 캐싱됨.
# use_fast=True: Rust로 구현된 Fast Tokenizer를 명시적으로 사용하도록 설정
tok = AutoTokenizer.from_pretrained("bert-base-uncased", use_fast=True)

# Fast Tokenizer 사용 여부 확인
print(f"Using Fast Tokenizer: {tok.is_fast}")

# %%
# 2. 텍스트를 입력 데이터(Batch)로 변환
# - padding: 배치 내의 문장 길이를 가장 긴 문장에 맞춤
# - truncation: 모델의 최대 입력 길이를 초과하는 경우 자름
# - return_tensors: "pt"(PyTorch), "np"(NumPy), "tf"(TensorFlow) 중 선택
batch = tok(
    ["hello world", "this is a test"],
    padding=True,
    truncation=True,
    return_tensors="pt",
)

# batch는 dict 형태이며 보통 다음 키를 포함함:
# - input_ids: 토큰의 정수 인덱스
# - attention_mask: 실제 토큰(1)과 패딩 토큰(0)을 구분
# - token_type_ids: (BERT 등에서) 문장 구분용 (A 문장인지 B 문장인지)
print(batch.keys())
print("Input IDs shape:", batch["input_ids"].shape)

# %%
# 3. 디코딩 (Token ID -> String)
# [tok.decode vs tok.batch_decode]
# - tok.decode(): 단일 시퀀스(1D 리스트/텐서)를 문자열 하나로 변환.
#   예: tok.decode(batch["input_ids"][0])
# - tok.batch_decode(): 시퀀스 목록(2D 리스트/텐서)을 문자열 리스트로 변환.
#   배치 단위 데이터를 처리할 때 적합함.

# skip_special_tokens=True 옵션을 사용하면 [CLS], [SEP], [PAD] 등을 제거하고 원문만 확인 가능
decoded = tok.batch_decode(batch["input_ids"], skip_special_tokens=False)
print("Decoded strings (with special tokens):", decoded)

decoded_clean = tok.batch_decode(batch["input_ids"], skip_special_tokens=True)
print("Decoded strings (clean):", decoded_clean)
```

* `padding=True`는 배치 텐서화를 위해 사실상 필수 (batch내에 가장 긴 샘플에 맞추어짐.)
* `truncation=True`는 최대 길이 초과 시 안전장치 (model의 최대 context length 제한을 넘지 않도록)

출력은 다음과 같음:
```
KeysView(
{'input_ids': tensor([
				[ 101, 7592, 2088,  102,    0,    0],
				[ 101, 2023, 2003, 1037, 3231,  102]
			]),
 'token_type_ids': tensor([
				[0, 0, 0, 0, 0, 0],
				[0, 0, 0, 0, 0, 0]
			]),
 'attention_mask': tensor([
				[1, 1, 1, 1, 0, 0],
				[1, 1, 1, 1, 1, 1]
			])
})
Input IDs shape: torch.Size([2, 6])
```

Tokenizer는 
* 텍스트를 토큰화하고 (token sequence)
* 각 token을 vocabulary에 정의된 정수 ID로 바꾸고, (token ID sequence)
* 이를 tensor 형태로 반환하는 (tensor)
* "모델 입력 생성기" 임

### 1.2 저장/로드: tokenizer 아티팩트

Tokenizer를 저장하면 출력 디렉토리에 tokenizer를 다시 복원하는 데 필요한 파일들이 생성됨.

일반적으로 다음과 같은 파일들이 포함될 수 있음:

* `tokenizer_config.json`, 
* `special_tokens_map.json`,
* `tokenizer.json`
* vocabulary 관련 파일
	* `vocab.txt`
 	* `vocab.json`
	* `merges.txt`
	* SentencePiece `.model` (vocabulary와 segmentation model 정보를 담은 파일 확장자)

다만 실제 생성 파일은 tokenizer 종류에 따라 다름.  

* Transformers 5.x에서는 fast tokenizer와 `tokenizers` backend의 비중이 커짐.
* 이와 관련된 `tokenizer.json`이 핵심 serialization 파일로 사용되는 경우가 많아짐.
* 그러나 모든 tokenizer가 항상 같은 파일 구성을 가지는 것은 아니므로, 특정 파일의 생성 여부를 일반화하기 어려움.
	* 예를 들어 BERT 계열은 `vocab.txt`가 중요하며,
	* BPE 계열은 `vocab.json`, `merges.txt`가 쓰이는 경우가 많으며,
	* fast tokenizer에서는 `tokenizer.json`이 핵심 serialization 파일로 사용되는 경우가 많음.

> Transformers 5.x에서는 tokenizer 구현이 tokenizers 기반의 fast tokenizer 쪽으로 정리되면서,
> `tokenizer.json`이 tokenizer의 핵심 직렬화(serialization) 파일로 더 중요해짐.
> `tokenizer.json` 파일은 vocabulary, normalizer, pre-tokenizer, post-processor, decoder 등의 정보를 함께 담을 수 있음.

```python
save_dir = "./tmp_tok"

# tokenizer의 설정, special token 정보, vocabulary 관련 파일 등을 저장함.
tok.save_pretrained(save_dir)

# 저장된 디렉토리에서 tokenizer를 다시 로드함.
tok2 = AutoTokenizer.from_pretrained(save_dir)

print(f"{type(tok2)      = }")
print(f"{tok2.is_fast    = }")
print(f"{tok2.vocab_size = }")
```
결과는 다음과 같음:

```Text
type(tok2)      = <class 'transformers.models.bert.tokenization_bert.BertTokenizer'>
tok2.is_fast    = True
tok2.vocab_size = 30522
```

Fast Tokenzier의 경우 `./tmp_tok` 디렉토리에 다음의 2개의 파일이 생성됨:

```Text
❯ tree tmp_tok
tmp_tok
├── tokenizer_config.json
└── tokenizer.json
```

### 1.3 주의사항

특수 토큰(Special Token) 추가 후에는 `save_pretrained()`로 결과를 고정해두는 습관이 필요함.

* Special Token 추가는 단순 문자열 추가 가 아니라
* token string과 token ID의 mapping, special token map, added vocabulary를 바꾸는 작업임.
* 때문에 저장하지 않으면 나중에 tokenizer를 다시 로드했을 때
* 추가한 token이 사라지거나 ID mapping이 달라져, 학습 시점과 추론 시점의 입력 표현이 달라질 수 있음.
* 새 token을 추가한 뒤 model의 embedding matrix 의 row의 크기를 거기에 맞추어줘야 함: `model.resize_token_embeddings(len(tok))`

> 길이가 다른 여러 문장을 하나의 batch tensor로 만들려면 padding이 필요함.  
> `padding=True` 또는 `padding="longest"`를 사용하면  
> batch 안에서 가장 긴 sequence 길이에 맞추어 padding이 적용됨.
> 
> 단, `DataCollatorWithPadding`을 사용하는 경우에는  
>
> * tokenizer 호출 시점이 아니라
> * collator 단계에서 동적 padding(dynamic padding)을 수행할 수도 있음.

Special Token 을 추가하고 그 token을 model 입력에 사용할 예정이라면 다음과 같은 처리를 수행:

```python
# %%
from transformers import AutoTokenizer, AutoModelForCausalLM

base_model = "bert-base-uncased"

tok = AutoTokenizer.from_pretrained(base_model)
model = AutoModelForCausalLM.from_pretrained(base_model)

# 1. tokenizer에 새 special token 추가
num_added = tok.add_special_tokens({
    "additional_special_tokens": ["<image>", "<bbox>", "<ocr>"]
})

# 2. 새 token이 실제로 추가된 경우에만 embedding matrix 크기 조정
if num_added > 0:
    print(f"Before resizing: {model.get_input_embeddings().weight.shape[0]}")
    model.resize_token_embeddings(len(tok))
    print(f"After resizing: {model.get_input_embeddings().weight.shape[0]}")

# 3. 학습 전 초기 상태 저장
save_dir = "./model_with_added_tokens_init"
tok.save_pretrained(save_dir)
model.save_pretrained(save_dir)
```

이후 다음과 같이 저장된 디렉토리에서 다시 시작하면 됨:

```Python
tok = AutoTokenizer.from_pretrained("./model_with_added_tokens_init")
model = AutoModelForCausalLM.from_pretrained("./model_with_added_tokens_init")

print(f"Reloaded model embedding size: {model.get_input_embeddings().weight.shape[0]}")
print(f"Tokenizer vocab size: {len(tok)}")
```

---

## 2. 이미지(Image) - ImageProcessor 사용법

### 2.1 예제: AutoImageProcessor로 pixel_values 만들기

```python
# %%
import requests
from transformers import AutoImageProcessor
from PIL import Image

# 1. 모델에 맞는 이미지 프로세서 로드
# AutoImageProcessor는 모델 이름에 따라 적절한 전처리 설정(Resize, Normalize 등)을 자동으로 로드함.
proc = AutoImageProcessor.from_pretrained("google/vit-base-patch16-224")
print(f"Image Processor type: {type(proc)}")
print(f"{proc.is_fast = }")  # 이미지 프로세서에는 일반적으로 is_fast 속성이 없음

# 샘플 이미지 로드 (URL에서 스트리밍으로 읽어와 RGB로 변환)
img_url = "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/cats.png"
img = Image.open(requests.get(img_url, stream=True).raw).convert("RGB")

# img = Image.open("cat.jpg").convert("RGB")
print(f"Original image size: {img.size}")  # (width, height) 형태로 출력됨

# %%
# 2. 이미지를 모델 입력 데이터(Batch)로 변환
# - images: 이미지 객체 리스트 전달
# - return_tensors: "pt"(PyTorch) 텐서 형식으로 반환
batch = proc(
    images=[img], 
    return_tensors="pt",
    # return_tensors="np",
    )
# dict_keys(['pixel_values'])  (모델에 따라 추가 key가 있을 수 있음)

for k in batch.keys():
    print(f"{k}: {batch[k].shape}")
# pixel_values: (1, 3, 224, 224) 같은 shape의

print(f"{type(batch["pixel_values"]) = }")
print(batch["pixel_values"].shape)
# (1, 3, 224, 224) 같은 shape의 tensor객체임

```

ImageProcessor는 
* resize/normalize 등 이미지 변환을 수행하고 
* vision 모델 입력 텐서를 생성.

### 2.2 저장/로드: preprocessor_config.json

ImageProcessor는 
* `save_pretrained()` 시 보통 `preprocessor_config.json`을 생성
* `from_pretrained()`로 동일 설정을 복원

```python
# %%
save_dir = "./tmp_imgproc"
proc.save_pretrained(save_dir)

proc2 = AutoImageProcessor.from_pretrained(save_dir)
print(type(proc2))
```

### 2.3 실전 팁: numpy로 변환, use_fast, resize/normalize

* `datasets.map()`과 결합할 때는 `pixel_values`를 **numpy로 변환** 하여 
* 저장 안정성을 확보하는 방식이 많이 사용함.
* 이는 일부 환경에서 torch 텐서 저장 을 사용하는 경우보다 보다 높은 호환성을 보이기 때문임.
* 일부 모델은 `use_fast=True` 옵션으로 보다 빠른 ImageProcessor 를 지원


참고: [[/hf_dataset_dict/dd_map]]

---

## 3. Multi-modal - Processor 사용법 (CLIP 등)

Tokenizer 와 ImageProcessor 가 둘 다 필요한 경우에 사용되는 **보다 추상화된 클래스**.

CLIP (Contrastive Language-Image Pretraining) 등과 같은 multi-modal model 에서 주로 사용됨.

### 3.1 AutoProcessor의 목적

`AutoProcessor`는 
* Multi-modal 입력(예: 텍스트+이미지)을 
* 한 번에 처리하기 위해 존재. 
* 내부적으로 Tokenizer와 ImageProcessor 로 구성.
* 한 번 호출로 모델 입력 딕셔너리를 구성

### 3.2 CLIP처럼 “통합 Processor”가 있는 경우

CLIP은 "텍스트 인코더" + "이미지 인코더" 를 함께 쓰는 대표적 Multi-modal 모델:

* Contrastive: 서로 다른 쌍을 구분하도록 학습하는 대조 학습 방식
* Language–Image: 텍스트와 이미지를 함께 다루는 멀티모달 구조
* Pretraining: 대규모 데이터로 사전학습을 수행한 모델임.

```python
# %%
from transformers import AutoProcessor
from PIL import Image
import requests

# 샘플 이미지 로드 (URL에서 스트리밍으로 읽어와 RGB로 변환)
img_url = "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/cats.png"
img = Image.open(requests.get(img_url, stream=True).raw).convert("RGB")
print(f"Original image size: {img.size}")  # (width, height) 형태로 출력됨
img.save("cat.jpg")
# img = Image.open("cat.jpg").convert("RGB")

# %%
# 샘플 이미지 로드 (URL에서 스트리밍으로 읽어와 RGB로 변환)
url = "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/model_doc/dog-sam.png"
img = Image.open(requests.get(img_url, stream=True).raw).convert("RGB")
print(f"Original image size: {img.size}")  # (width, height) 형태로 출력됨
img.save("dog.jpg")
# img = Image.open("dog.jpg").convert("RGB")

# %%
processor = AutoProcessor.from_pretrained("openai/clip-vit-base-patch32")

out = processor(
    text=["a photo of a cat", "a photo of a dog"],
    images=[Image.open("cat.jpg").convert("RGB"), Image.open("dog.jpg").convert("RGB")],
    return_tensors="pt",
    padding=True,
)

for k in out.keys():
    print(f"{k}: {type(out[k]) = }{out[k].shape = }")
# input_ids, attention_mask, pixel_values ... 같은 형태

# %%
# 3. 프로세서 저장 및 로드
# processor는 tokenizer와 image_processor의 기능을 모두 포함함.
# 저장 시 설정 파일들과 각 구성 요소(tokenizer, image_processor 등)의 정보가 함께 저장됨.
save_dir = "./tmp_proc"
processor.save_pretrained(save_dir)

# 저장된 디렉토리에서 프로세서를 다시 로드함.
processor2 = AutoProcessor.from_pretrained(save_dir)

print(f"{type(processor2) = }")

```

결과는 다음과 같음:
```text
pixel_values: type(out[k]) = <class 'torch.Tensor'>out[k].shape = torch.Size([2, 3, 224, 224])
input_ids: type(out[k]) = <class 'torch.Tensor'>out[k].shape = torch.Size([2, 7])
attention_mask: type(out[k]) = <class 'torch.Tensor'>out[k].shape = torch.Size([2, 7])
```

**주의사항**

* Processor가 제공되는 경우, 이를 사용하면 간단히 처리가능함.
* 하지만 모델에서 어떤 Tokenizer와 ImageProcessor를 요구하는지 등을 파악해보기 위해 Processor의 I/O 및 설정(`.json`)을 살펴보는 것이 좋음.
* 해당 모델의 커스터마이징을 위해서도 내부 구성을 파악하고 있는 것이 좋음.
* 만약, 특정한 조합이 요구되면서 Auto로 Processor가 제공되지 않는 경우라면, 이를 조합하여 만들어야 함.

다음의 확인이 필요함:

* `processor_config.json` 을 살펴볼 것.
* 흔히 `processor.tokenizer` 및 `processor.image_processor` 등의 속성으로 실제 전처리 컴포넌트가 지원됨.

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

| Auto 클래스           | 1차 기준 파일                                                            | 실제 사용되는 Key             |
| :----------------: | :-----------------------------------------------------------------: | :-----------------------------: |
| `AutoConfig`         | `config.json`                                                         | `model_type` / `auto_map`   |
| `AutoModel`          | `config.json`                                                         | `model_type` / `auto_map`   |
| `AutoTokenizer`      | `config.json`<br/>(+ `tokenizer_config.json` 보조)                            | `model_type` / `auto_map` |
| `AutoImageProcessor` | `preprocessor_config.json`<br/>(없으면 `config.json` fallback)                 | `image_processor_type` / `auto_map` |
| `AutoProcessor`      | `processor_config.json`<br/>(없으면 `preprocessor_config.json` 또는 `config.json`) | `processor_class` / `auto_map` |

* `AutoImageProcessor`는 `config.json`을 fallback으로 사용할 수 있음
* `AutoTokenizer`는 `tokenizer_config.json`도 보조로 참고함.
* `AutoProcessor`는 `processor_config.json`이 없으면 다른 config를 참고함

> `AutoProcessor`는  
> tokenizer, image processor 등을 직접 대체하는 단일 전처리 객체라기보다,
> 이들을 하나로 묶어 제공하는 wrapper 성격의 객체임.
> 예를 들어 CLIP의 경우
> processor_config.json에는 processor_class: "CLIPProcessor"와 image_processor 설정만 저장되어 있음.  
>
> `processor_config.json` 안에 tokenizer 설정이 없더라도 문제가 되진 않음.  
> `AutoProcessor`는  
> 먼저 `processor_config.json`의 `processor_class` 값을 통해 `CLIPProcessor`를 선택하고,
> 이후 선택된 `CLIPProcessor.from_pretrained()` 내부에서
> tokenizer와 image processor를 각각 필요한 설정 파일로부터 로드하는 구조이기 때문임.  
>
> 따라서 CLIP에서 tokenizer 관련 설정은 `processor_config.json`이 아니라
> tokenizer_config.json과 tokenizer.json을 통해 로드됨.
> 반면 image processor 관련 설정은 현재 `processor_config.json` 내부의 image_processor 항목에 포함됨.
>
> 1. `AutoProcessor.from_pretrained(path)`를 호출함.
> 2. `AutoProcessor`가 `processor_config.json`을 읽음.
> 3. `processor_config.json`에서 `processor_class = "CLIPProcessor"` 를 확인함.
> 4. `AutoProcessor`가 실제 processor class로 `CLIPProcessor`를 선택함.
> 5. `AutoProcessor`가 `CLIPProcessor.from_pretrained(path)`를 호출함.
> 6. `CLIPProcessor.from_pretrained()` 내부에서 필요한 하위 구성요소를 로드함.
>    - image processor: `processor_config.json` 내부의 `image_processor` 설정을 바탕으로 `CLIPImageProcessor`를 구성함.
>    - tokenizer: 같은 directory의 `tokenizer_config.json`과 `tokenizer.json`을 바탕으로 `CLIPTokenizer` 또는 `CLIPTokenizerFast`를 구성함.
> 7. 최종적으로 다음 형태의 객체가 반환됨
>    `CLIPProcessor(image_processor=CLIPImageProcessor(...),tokenizer=CLIPTokenizerFast(...) 또는 CLIPTokenizer(...))`


### 4.3 `AutoModel`의 class 선택 기준과 입력 구조

`AutoModel` 또는 `AutoModelForXxx`는 

* `config.json`을 읽고,
* 해당 config에 맞는 실제 model class를 선택하여 로드함.

이때 표준 모델과 custom model은 class를 선택하는 방식이 조금 차이가 있음

HF Transformers에서 기본적으로 제공하는 **표준 모델** 들은 

* `config.json`의 `model_type` 값을 기준으로
* 내부 mapping을 통해 class를 고름.

예를 들어 BERT의 경우 `config.json`에 `"model_type": "bert"`가 저장되어 있고, 이를 통해 다음 class들이 선택됨.

* `AutoConfig` → `BertConfig`
* `AutoModel` → `BertModel`
* `AutoTokenizer` → `BertTokenizer` 또는 `BertTokenizerFast`
* `AutoModelForMaskedLM` → `BertForMaskedLM`

> 즉 `AutoModel` 계열은
> class 이름을 직접 지정하지 않아도,
> `config.json`의 metadata를 바탕으로 적절한 class를 선택함.

반면 Transformers에 기본 등록되어 있지 않은  
custom model, custom tokenizer, custom config를 사용하는 경우에는  
`config.json`의 `auto_map` 필드를 참고함.  

예를 들어 custom model의 `config.json`에는 다음과 같은 정보가 들어가야 함.

```json
{
  "auto_map": {
    "AutoConfig": "configuration_xxx.MyConfig",
    "AutoModel": "modeling_xxx.MyModel",
    "AutoTokenizer": "tokenization_xxx.MyTokenizer"
  }
}
```

* `AutoModel`, `AutoTokenizer`, `AutoConfig`는
* Transformers 내부 mapping만으로 해당 class를 찾을 수 없기 때문에,
* `auto_map`에 기록된 경로를 참고하여 필요한 class를 import함.

또한 외부 repository의 Python code를 실행해야 하므로 일반적으로 `trust_remote_code=True` 옵션이 필요함.

```python
from transformers import AutoModel

model = AutoModel.from_pretrained(
    "repo_id_or_local_path",
    trust_remote_code=True,
)
```

다만 `AutoModel`이 model class를 자동으로 골라준다고 해서 입력 형식까지 임의로 맞춰 주는 것은 아님.  
실제로 model이 어떤 입력 key를 받는지는 선택된 model class의 `forward(...)` 메서드가 결정함.

예를 들어 text model은 보통 다음과 같은 입력 key를 기대함.

```python
input_ids
attention_mask
token_type_ids
```

image model은 보통 다음과 같은 입력 key를 기대함.

```python
pixel_values
```

multi-modal model은 text와 image 입력을 함께 받을 수 있음.

```python
input_ids
attention_mask
pixel_values
```

따라서 다시 한번 강조하지만,  
model만 단독으로 로드하기보다, 해당 model에 맞는 전처리 객체를 같이 로드하는 패턴이 일반적임.

* text model → `AutoTokenizer`
* image model → `AutoImageProcessor`
* multi-modal model → `AutoProcessor`

이 전처리 객체들은 입력 text나 image를 model이 기대하는 tensor와 key 구조로 변환해 줌.  

* 따라서 사용자가 직접 `input_ids`, `attention_mask`, `pixel_values`를 하나씩 구성하기보다,  
* 해당 model과 함께 저장된 tokenizer, image processor, processor를 사용하는 것이 안전함.

일반적인 사용 흐름은 다음과 같음.

```python
from transformers import AutoModel, AutoProcessor

model = AutoModel.from_pretrained("repo_id_or_local_path")
processor = AutoProcessor.from_pretrained("repo_id_or_local_path")

inputs = processor(
    text="a photo of a dog",
    images=image,
    return_tensors="pt",
)

outputs = model(**inputs)
```

* 여기서 `processor(...)`가 반환하는 dictionary의 key가 곧 model의 `forward(...)`에 전달되는 입력 key가 됨.
* 즉 `AutoProcessor`는 model과 직접 연결된 학습 파라미터를 가지는 것은 아니지만, **model이 기대하는 입력 형식을 맞춰 주는 중요한 역할을 함.**

### 전처리와 AutoClass 및 AutoModel 에서 반드시 기억할 내용

다음을 반드시 기억할것

* 표준 model은 `model_type` 값을 기준으로 Transformers 내부 mapping에서 class가 선택됨.
* custom model은 `auto_map` 값을 기준으로 custom class가 import됨.
* 실제 입력 key는 선택된 model class의 `forward(...)` 메서드가 결정함.
* 입력 tensor와 key 구조는 해당 model에 맞는 `AutoTokenizer`, `AutoImageProcessor`, `AutoProcessor`를 사용하여 맞추는 것이 일반적임.

---

## 5. `ImageProcessingMixin`으로 Custom ImageProcessor 만들기

### 5.1 언제 Custom ImageProcessor가 필요한가

`transformers`에는 `CLIPImageProcessor`,
`ViTImageProcessor` 등
다양한 Built-in ImageProcessor가
이미 제공되고 있음.
이들만으로 충분하지 않은 경우,
즉 **기존 processor가 지원하지 않는
전처리가 필요한 경우**에
Custom ImageProcessor를 구현함.

구체적으로 다음과 같은 경우가 해당됨:

* **비표준 입력 형식**:
  6-channel image, 의료영상 전용
  windowing, modality-specific
  preprocessing 등
  기존 processor의 파이프라인으로
  처리할 수 없는 입력을 다뤄야 하는 경우.
* **고유한 전처리 파이프라인**:
  모델 구조에 맞는 특수한
  normalization, cropping, tiling 등
  기존 processor에 없는
  전처리 로직이 필요한 경우.
* **새로운 모델 아키텍처 구현**:
  처음부터 모델을 설계하면서,
  대응하는 기존 processor가
  존재하지 않는 경우.

Custom ImageProcessor를 구현하면
다음과 같은 이점을 자동으로 얻게 됨:

* **전처리 재현성**:
  전처리 로직과 설정값이
  하나의 클래스에 캡슐화되므로,
  학습과 추론에서
  동일한 입력 변환을 보장할 수 있음.
* **모델과 함께 저장/배포**:
  `save_pretrained()`를 통해
  전처리 설정이
  `preprocessor_config.json`에 저장되고,
  `AutoImageProcessor.from_pretrained()`로
  다시 로드할 수 있음.

즉 Custom ImageProcessor는
단순히 resize나 normalization을
수행하기 위한 객체가 아니라,
**기존 processor로 커버되지 않는
전처리 로직**을
HuggingFace 생태계의
저장/로드/Auto 클래스 체계 안에서
관리하기 위한 구성요소임.

### 5.2 Custom ImageProcessor 최소 구현 골격

Custom `ImageProcessor`를 만들기 위해 알아야 할 핵심은 딱 두 가지임:

1. **`ImageProcessingMixin`** 을 상속하면, `save_pretrained()` / `from_pretrained()` 등의 **저장-로드** 기능을 그대로 사용할 수 있음.
2. 때문에 실질적인 **전처리 로직** 을 `__call__` 메서드에서만 구현하면 됨.

#### 직렬화(Serialization)가 자동으로 되려면?

`ImageProcessingMixin`은

* `save_pretrained()` 호출 시 내부적으로
* `to_dict()`를 사용하여
* `preprocessor_config.json`을 생성함.

이 기본 `to_dict()`는 **`self.__dict__`** 기반으로 동작하므로, `__init__`에서 `self.xxx = value` 형태로 속성을 저장하기만 하면 **`to_dict()`를 오버라이드할 필요가 없음.**

단, 저장되는 속성의 타입이 다음과 같이 **JSON 직렬화가 가능한 단순 타입**이어야 함:

* `int`, `float`, `bool`, `str`
* `list`, `dict` (내부 원소도 단순 타입이어야 함)

이처럼 `__init__`에서 `self.xxx = value`로 **인스턴스의 루트 레벨에 직접 저장**하는 속성들을 **flat attributes** 라고 부름.  

**flat attributes**의 경우, 

* 별도의 중첩 객체(nested object)를 거치지 않고
* 바로 `self.__dict__`에 들어가기 때문에,
* 기본 `to_dict()`가 그대로 직렬화 가능함.

> **주의:**
> `self.cfg = SomeConfig(...)` 처럼
> 별도 객체에 속성을 묶어두면,
> 기본 `to_dict()`가 그 내부까지
> 풀어주지 못하므로
> 직접 `to_dict()`를 오버라이드해야 함.  
> 따라서 **flat attributes 방식**
> (= `self.xxx`로 직접 저장)이
> 가장 간단하고 권장되는 방법임.

#### 최소 구현 예제

아래 예제는
* **"Resize → Normalize → CHW Tensor"** 를 수행하여
* `pixel_values`를 반환하는 Custom `ImageProcessor`임.

```python
# image_processing_simple_vision.py

from __future__ import annotations

from typing import Any

import numpy as np
import torch
from PIL import Image

from transformers.image_processing_base import (
    ImageProcessingMixin,
)
from transformers.utils.generic import TensorType


class SimpleVisionImageProcessor(
    ImageProcessingMixin,
):
    """
    Custom ImageProcessor
    (ImageProcessingMixin 기반)
    - images → pixel_values (B, C, H, W)
    - save_pretrained() 호출 시
      preprocessor_config.json 자동 생성
    """

    # HuggingFace 관례:
    # 모델에 전달할 입력 key 이름을 지정
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
        # ── 반드시 super().__init__() 호출 ──
        # ImageProcessingMixin 내부 초기화 +
        # kwargs 처리를 위해 필수임.
        super().__init__(**kwargs)

        # ── flat attributes로 저장 ──
        # self.xxx 형태 + JSON 직렬화 가능 타입
        #   부모의 기본 to_dict()가
        #   알아서 직렬화해줌
        self.size = int(size)
        self.do_resize = bool(do_resize)
        self.do_normalize = bool(do_normalize)
        self.image_mean = (
            image_mean
            if image_mean is not None
            else [0.485, 0.456, 0.406]
        )
        self.image_std = (
            image_std
            if image_std is not None
            else [0.229, 0.224, 0.225]
        )

    # ────────────────────────────
    # 내부 헬퍼 메서드
    # ────────────────────────────

    def _ensure_pil(
        self,
        img: Image.Image | np.ndarray,
    ) -> Image.Image:
        """입력을 PIL Image로 변환."""
        if isinstance(img, Image.Image):
            return img
        if isinstance(img, np.ndarray):
            arr = img
            if arr.ndim == 2:  # grayscale→3ch
                arr = np.stack(
                    [arr, arr, arr], axis=-1,
                )
            if arr.dtype != np.uint8:
                arr = (
                    np.clip(arr, 0, 255)
                    .astype(np.uint8)
                )
            return Image.fromarray(arr)
        raise TypeError(
            f"Unsupported image type:"
            f" {type(img)}"
        )

    def _resize(
        self, img: Image.Image,
    ) -> Image.Image:
        """self.size x self.size 로 리사이즈."""
        if not self.do_resize:
            return img
        return img.resize(
            (self.size, self.size),
            resample=Image.BILINEAR,
        )

    def _to_chw_float01(
        self, img: Image.Image,
    ) -> np.ndarray:
        """PIL Image → (C,H,W) float32 [0,1]."""
        x = (  # HWC, [0,1]
            np.asarray(img, dtype=np.float32)
            / 255.0
        )
        return np.transpose(x, (2, 0, 1))  # CHW

    def _normalize(
        self, x: np.ndarray,
    ) -> np.ndarray:
        """ImageNet mean/std 기반 정규화."""
        if not self.do_normalize:
            return x
        mean = np.asarray(
            self.image_mean, dtype=np.float32,
        )[:, None, None]
        std = np.asarray(
            self.image_std, dtype=np.float32,
        )[:, None, None]
        return (x - mean) / std

    # ────────────────────────────
    # __call__: 실제 전처리 진입점
    # ────────────────────────────

    def __call__(
        self,
        images: (
            Image.Image
            | np.ndarray
            | list[Image.Image | np.ndarray]
        ),
        return_tensors: (
            str | TensorType | None
        ) = "pt",
        **kwargs: Any,
    ) -> dict[str, torch.Tensor | np.ndarray]:
        """
        Parameters
        ----------
        images :
            단일 이미지 또는 이미지 리스트
        return_tensors :
            "pt" (PyTorch) 또는 "np" (NumPy)

        Returns
        -------
        dict with "pixel_values":
            shape (B, C, H, W)
        """
        if not isinstance(images, list):
            images = [images]

        batch = []
        for im in images:
            im = (
                self._ensure_pil(im)
                .convert("RGB")
            )
            im = self._resize(im)
            x = self._to_chw_float01(im)
            x = self._normalize(x)
            batch.append(x)

        pixel_values = (
            np.stack(batch, axis=0)
            .astype(np.float32)
        )

        if return_tensors in (
            "pt", TensorType.PYTORCH,
        ):
            pixel_values = (
                torch.from_numpy(pixel_values)
            )
        elif (
            return_tensors
            in ("np", TensorType.NUMPY)
            or return_tensors is None
        ):
            pass
        else:
            raise ValueError(
                "Unsupported return_tensors:"
                f" {return_tensors}"
            )

        return {"pixel_values": pixel_values}


# ── AutoImageProcessor 등록 ──
# 이 한 줄을 추가하면,
# config.json의 auto_map에
# "AutoImageProcessor" 매핑이 자동 포함됨.
SimpleVisionImageProcessor \
    .register_for_auto_class(
        "AutoImageProcessor",
    )
```

#### 핵심 정리

| 항목 | 설명 |
|---|---|
| 상속 대상 | `ImageProcessingMixin` |
| `super().__init__` | 필수. 내부 초기화용 |
| 속성 저장 방식 | `self.xxx = value` |
| `to_dict()` 오버라이드 | 불필요 (flat attrs) |
| 전처리 로직 위치 | `__call__` 메서드 |
| `model_input_names` | `["pixel_values"]` |
| Auto 등록 | `register_for_auto_class()` |


#### 저장 및 Auto 클래스를 통한 로드

위 코드를 `image_processing_simple_vision.py`로 저장한 뒤,  
다음과 같이 `.save_pretrained()`를 수행하면  
지정 디렉토리에 `preprocessor_config.json`이 생성됨:

```python
from image_processing_simple_vision import (
    SimpleVisionImageProcessor,
)

# 1. 인스턴스 생성
proc = SimpleVisionImageProcessor(size=224)

# 2. save_pretrained()가
#    preprocessor_config.json을 자동 생성
proc.save_pretrained(
    "./simple_vision_proc",
)
```

이때 `.save_pretrained()`가 수행하는 작업은 다음 두 가지임:

1. **`preprocessor_config.json` 생성**:
	* `to_dict()` 결과가 JSON으로 저장됨.
	* 앞서 `.register_for_auto_class()`를 호출해두었기 때문에,
    * `auto_map` 항목이 함께 기록됨.
2. **Python 소스 파일 복사**:
	* 해당 processor 클래스가 정의된 `.py` 파일이
	* 지정 디렉토리에 같이 복사됨.

저장된 `preprocessor_config.json`의 내용은 다음과 같음:

```json
{
  "auto_map": {
    "AutoImageProcessor":
      "image_processing_simple_vision"
      ".SimpleVisionImageProcessor"
  },
  "do_normalize": true,
  "do_resize": true,
  "image_mean": [0.485, 0.456, 0.406],
  "image_std": [0.229, 0.224, 0.225],
  "image_processor_type":
    "SimpleVisionImageProcessor",
  "size": 224
}
```

`auto_map`에 모듈 경로와 클래스명이 기록되어 있으므로,  
`AutoImageProcessor`가 이 정보를 읽어 해당 `.py` 파일을 import할 수 있게 됨.

이후 저장된 디렉토리를 지정하여
* `AutoImageProcessor.from_pretrained()`로
* 로드할 수 있음:

```python
from transformers import AutoImageProcessor

p = AutoImageProcessor.from_pretrained(
    "./simple_vision_proc",
    trust_remote_code=True,
)
print(type(p), p.size)
# <class '...SimpleVisionImageProcessor'> 224
```

> **참고: `trust_remote_code=True`가 필요한 이유**
> 
> `auto_map`이 가리키는 `.py` 파일은
> 
> * `transformers` 라이브러리에 내장된 코드가 아니라,
> * 사용자가 작성한 커스텀 코드임.
> 
> 따라서 보안상의 이유로 이 옵션을 명시적으로 켜줘야 해당 코드의 import가 허용됨.

### 5.3 저장/로드 실습: `preprocessor_config.json` round-trip

5.2에서 구현한
SimpleVisionImageProcessor를 실제로
저장하고 다시 로드하여,
전처리 결과가 동일한지 확인하는
round-trip 실습임.

#### 1단계: 저장하기

```python
from image_processing_simple_vision import (
    SimpleVisionImageProcessor,
)
from PIL import Image

# 인스턴스 생성
p = SimpleVisionImageProcessor(
    size=224,
    do_resize=True,
    do_normalize=True,
)

# 전처리 수행
img = Image.open("cat.jpg").convert("RGB")
out = p(img, return_tensors="pt")
print(out["pixel_values"].shape)
# torch.Size([1, 3, 224, 224])

# 저장
save_dir = "./tmp_custom_imgproc"
p.save_pretrained(save_dir)
```
`save_pretrained()` 수행 후 `save_dir` 내부에는 다음 파일들이 생성됨:

* `preprocessor_config.json`: `to_dict()` 결과가 JSON으로 저장됨. `auto_map` 항목이 포함되어 있음.
* `image_processing_simple_vision.py`: `register_for_auto_class()`를 호출해두었기 때문에, processor 클래스가 정의된 `.py` 파일이 자동으로 복사됨.

#### 2단계: 로드 및 검증

로드 방법은 두 가지가 있음:

**(A) 클래스 직접 지정 로드**

```python
p2 = SimpleVisionImageProcessor \
    .from_pretrained(save_dir)
```
* 이미 해당 클래스를 import한 상태이므로,
* `trust_remote_code` 없이 `from_pretrained()`만으로 로드 가능함.

**(B) AutoImageProcessor 로드**

```python
from transformers import AutoImageProcessor

p2 = AutoImageProcessor.from_pretrained(
    save_dir,
    trust_remote_code=True,
)
```
* `auto_map` 정보를 통해 클래스명과 모듈 경로를 자동으로 찾아 import함.
* 해당 `.py` 파일이 `save_dir`에 함께 저장되어 있으므로, 별도로 모듈을 import하지 않아도 됨.
* 단, 사용자 코드를 동적으로 `import`하는 것이므로 `trust_remote_code=True`가 필수임.

**3단계: round-trip 검증**

어떤 방식으로 로드했든,
동일 이미지에 대해 전처리 결과가
완전히 일치하는지 확인:

```python
out2 = p2(img, return_tensors="pt")

diff = (
    out["pixel_values"]
    - out2["pixel_values"]
).abs().max().item()

print(diff)  # 0.0
```

* `diff`가 `0.0`이면 저장/로드 round-trip이 정상적으로 동작한 것임.

> `register_for_auto_class()`를 호출한 상태에서 `save_pretrained()`를 수행하면, `auto_map` + `.py` 파일이 함께 저장됨.
>  따라서 (A) 직접 클래스 지정, (B) AutoImageProcessor 두 가지 방식 모두로 로드가 가능함.
> 만약 `register_for_auto_class()`를 호출하지 않았다면, `auto_map` 필드와 `.py` 파일 복사가 이루어지지 않으므로,
> (A) 방식만 사용 가능하고 해당 모듈이 Python의 module search path에 있어야 함.

### 5.4 Custom `AutoImageProcessor`를 AutoClass로 로드되게 만드는 방법

Transformers가 기본 제공하는 standard image processor의 경우, 

* 저장된 `preprocessor_config.json`의
* `image_processor_type` 등의 metadata를 기반으로
* `AutoImageProcessor`가 내부 mapping에서 적절한 class를 선택하여 로드함.

> 반면,
> **custom image processor** 를 `AutoImageProcessor.from_pretrained(...)`으로 로드되게 하려면,
> AutoClass 연동 정보를 별도로 설정해줘야 함.

이를 위한 Custom class의 AutoClass 연동 방식은 크게 두 가지로 구분됨:

* **`AutoImageProcessor.register(...)`**
    * 현재 Python session의 AutoClass mapping에 custom class를 직접 등록하는 방식임.    
    * 이미 해당 custom class를 `import`할 수 있는 local code 환경에서 사용되는 방식임.
* **`register_for_auto_class("AutoImageProcessor")`**
    * 저장 및 배포 시, `config`(정확히는 `preprocessor_config.json`)에 `auto_map` entry를 남기기 위한 방식임.
    * `save_pretrained()` 또는 `push_to_hub()`와 함께 사용되며, custom class가 정의된 Python file도 설정 json파일들과 함께 저장됨.    
    * 이 저장된 코드를 로드하여 사용해야하므로, 다시 로드할 때는 보통 `trust_remote_code=True` 옵션을 사용해야 함.

정리하면, 
* `register(...)`는 현재 실행 환경 내에서의 runtime 등록이고,
* `register_for_auto_class(...)`는 저장/배포 후 다른 환경에서 AutoClass를 통해 로드하기 위한 등록임.

배포 가능한 custom image processor 저장 directory에는 보통 다음 항목들이 함께 포함되어야 함.

* `preprocessor_config.json` : image processor 설정 및 `auto_map` 정보를 포함하는 JSON file
* custom image processor class가 정의된 Python file (e.g., `image_processing_custom.py`)

로드는 다음처럼 수행함:

```python
from transformers import AutoImageProcessor

image_processor = AutoImageProcessor.from_pretrained(
    "repo_id_or_local_path",
    trust_remote_code=True,
)
```

결론적으로, 
* standard class는 `image_processor_type` 등의 metadata와 Transformers 내부 mapping으로 자동 선택되고,
* custom class는 `register_for_auto_class(...)`를 통해 저장된 `auto_map` 정보에 의해 로드되는 구조임.

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
ip.save_pretrained(save_dir)  # preprocessor_config.json 생성

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
# input_ids, attention_mask, pixel_values ... (모델/프로세서에 따라 변형) 
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


