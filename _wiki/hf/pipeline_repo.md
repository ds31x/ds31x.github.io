---
layout  : wiki
title   : HF-Pipeline Upload
summary : 
date    : 2026-02-02 16:47:28 +0900
updated : 2026-02-19 12:18:26 +0900
tag     : hf pipeline repo
resource: 57/48E6A5A80146F48B0FDD78ADA3766B
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# Pipeline 객체 HF Repo.로 업로드.

Pipeline은 `task`, `config.json`, `processor`, `model` 의 조합으로 구성된다.

HF Repo.에 업로드되는 것은 

* pipeline 객체 자체가 아니라,
* model을 중심으로 한 config 및 processor를 포함한 artifacts의 집합이며,
* 이는 **Model Repo. 형태로 저장** 된다.

> Repo. = Repository (저장소)

주의할 점은,

* Pipeline의 실제 구성 방식은 `config.json`과 같은 설정 파일에 정의되어 있고,
* `AutoConfig`, `AutoProcessor`, `AutoModel` 계열의 **AutoClass** 들이 이를 로딩하여
* 최종적으로 pipeline 객체를 조립한다.

> 이 문서에서는
> pipeline 객체를 HF Repo.에 업로드 하는 것을 다룸.

다음은 이 문서의 과정으로 만들어놓은 HF Model Repo.임:

* [dsaint31/tmp-pl-image-classification](https://huggingface.co/dsaint31/tmp-pl-image-classification)

## HF Repository 에 업로드하기

Pipeline을 업로드하는 건 사실 Model Repo. (모델 대상)에 올리는 것임.
  
> 저장소에 올려진 pipeline의 모델은
> pipeline으로 로딩시,  
> task, config.json, processor, model class 를 확인하여 재구성되는 것임.

### Pipeline 객체 를 직접 업로드

Pipeline 객체 `img_clf`가 있다면 다음의 코드로 업로드.

```python
img_clf.push_to_hub("dsaint31/tmp-pl-image-classification")
```
* repo가 없으면 **자동 생성**
* 있으면 overwrite
* model + config + processor + pipeline metadata 까지 같이 업로드

---

### 참고: 

참고로 다음의 에러 발생시:
```
Writing model shards: 100%|██████████████████████████████████████████████████████████████████████████| 1/1 [00:00<00:00,  3.96it/s]
Traceback (most recent call last):
  File "<python-input-2>", line 1, in <module>
    img_clf.push_to_hub("dsaint31/tmp-pl-image-classification1")
    ~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/dsaint31/miniconda3/envs/hf_env/lib/python3.14/site-packages/transformers/utils/hub.py", line 776, in push_to_hub
    self.save_pretrained(tmp_dir, max_shard_size=max_shard_size)
    ~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/dsaint31/miniconda3/envs/hf_env/lib/python3.14/site-packages/transformers/pipelines/base.py", line 995, in save_pretrained
    if self.modelcard is not None:
       ^^^^^^^^^^^^^^
AttributeError: 'ImageClassificationPipeline' object has no attribute 'modelcard'
```
다음의 코드를 통해 비어있는 `modelcard`  attribute 처리(추가하고 값은 `None`으로 설정)를 해주고 나서 `push_to_hub`를 호출

```python
setattr(img_clf, "modelcard", None)
img_clf.push_to_hub("dsaint31/tmp-pl-image-classification")
```

---

### 로컬에 먼저 저장후 이를 저장소에 업로드

로컬에 먼저 저장 후 push (선택) 하는 방법도 가능함.

```python
img_clf.save_pretrained("./tmp-pl-image-classification")
```

다음의 내용으로 구성됨:

```text
tmp-pl-image-classification/
├── config.json
├── model.safetensors (또는 pytorch_model.bin)
├── preprocessor_config.json
└── README.md (없으면 나중에 생성 가능)
```

이후 업로드:

```
huggingface-cli upload ./tmp-pl-image-classification dsaint31/tmp-pl-image-classification
```


## HF Repository 기본 구성.

업로드된 Model Repo. 에서의 구성을 보면 다음과 같음.

```
repo/
 ├─ config.json              : 모델의 정의
 ├─ model.safetensors        : 가중치
 ├─ tokenizer.json           : 입력 전처리 정의 (text)
 ├─ preprocessor_config.json : 입력 전처리 정의 (image)
 └─ README.md                : 모델 카드
```

> 가장 기본적인 형태임.  
> 모델에 따라 추가적인 파일들이 있을 수 있음.

이들 각각을 살펴보자.

## config.json

Hugging Face(HF)에서  
**repository의 config.json** 은 다음을 위한 핵심 meta-data임.

이 파일의 meta-data를 통해

* **modle 의 구조적 정의** 와 
* 재현 가능한 로딩이 보장됨.

HF는 
* 기본 모델 각각에 Config 클래스 (`BertConfig`, `ViTConfig` 등) 를 제공하며,
* 이들은 대응되는 `config.json` 으로 serialization됨.

> Custom Model의 경우엔 `PretrainedConifg` 클래스를 상속받아서 대응하는 Config 클래스를 만들 수 있음:  
> 자세한 건 [[/hf/hf_config]] 를 참고.

다음이 `config.json`의 핵심요소임:

### Architecture Blueprint

`config.json`은 model의 architecture (or computational graph) 를 결정하는 모든 hyper-parameters 를 정의.

**example:**

* Transformer depth: `num_hidden_layers`
* embedding dimension: `hidden_size`
* attention 구조: `num_attention_heads`
* ViT의 경우 입력 분해 방식: `image_size`, `patch_size`

> blueprint는 원래  설계도를 햇빛으로 복사하던 옛 인쇄 방식에서 나온 용어임.
> 옛날엔 설계도가 파란색 바탕의 도면이었음.
> 이후 의미가 확장되어, 건축·공학의 설계도뿐 아니라
> 어떤 시스템이나 계획의 전체 구조를 미리 정리한 기본 설계안이라는 뜻으로도 널리 사용되고 있음.

### AutoClass 또는 Pipeline 로딩의 기준점

HF의 자동 로딩은 전적으로 `config.json`에 의존함:

AutoClass 중 모델을 로딩하는 예:

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("repo_id")
```

1. `config.json` 다운로드
2. `model_type` 확인 (예: `"vit"`)
3. `architectures` 또는 `AutoModel` 매핑으로 정확한 모델 클래스 결정
4. config로 모델 `skeleton` 생성
5. weight 파일 로드

> config 없이는 AutoModel 은 동작이 불가능 함.

이는, AutoClass 들을 사용하는 Pipeline도 마찬가지임.

```python
from transformers import pipeline

img_clf = pipeline(
    task="image-classification",
    model="dsaint31/tmp-pl-image-classification"
)
```

* `pipeline(task="image-classification", model=repo_id)`를 호출하면,
  먼저 image-classification task에 대응하는 pipeline 클래스가 선택됨
* 이어서 repository의 설정 파일과 auto mapping 정보를 바탕으로, 해당 task에 맞는 image classification model 클래스가 자동으로 결정
	* `AutoConfig.from_pretrained(repo_id)`를 통해 `config.json`을 먼저 로드
	* 해당 파일의 `model_type` 및 `architectures` 등을 바탕으로
	* 해당 task에 맞는 모델 클래스(예: `AutoModelForImageClassification` 계열)를 선택함
* 선택된 클래스에 대해 `from_pretrained(repo_id, config=config)`를 호출됨.
	* 모델 skeleton(구조)가 `config` 를 기반으로 생성되고
	* repo. 에 저장된 가중치(`model.safetensors`/`pytorch_model.bin`)가 로드
* task가 요구하는 전처리기도 같이 로드함
	* 텍스트: `tokenizer_config.json`, `tokenizer.json` 등
	* 비전: `preprocessor_config.json` (또는 `image processor` 관련 파일) 등
* image-classification은 vision task이므로
	* tokenizer 대신 image processor가 자동으로 로드되며,
	* 이때 `preprocessor_config.json` 등 전처리 설정 파일이 활용됨

사용자는 
* `AutoModelForImageClassification`, `AutoImageProcessor`를 직접 명시하지 않아도,
* repository 안의 설정과 저장 파일 구성을 바탕으로
* image-classification용 추론 파이프라인을 바로 사용할 수 있음

물론 다음과 같이 명시적으로 model과 processor를 따로 불러들여서 pipeline을 구성할 수도 있음:

```python
from transformers import pipeline, AutoModelForImageClassification, AutoImageProcessor

model = AutoModelForImageClassification.from_pretrained(
    "dsaint31/tmp-pl-image-classification"
)

processor = AutoImageProcessor.from_pretrained(
    "dsaint31/tmp-pl-image-classification"
)

img_clf = pipeline(
    task="image-classification",
    model=model,
    image_processor=processor
)
```

### model 구조 명세서.

Hugging Face repository의 `config.json`은

* 모델의 구조적 명세(architecture specification) 를 가짐. 
* 학습된 파라미터(weight)를 분리하기 위한 파일.

**구조적 명세** 란?

* 모델 코드 내부에 구현된 연산을 **어떤 형태로 인스턴스화할지를 결정** 하는 설정
* layer 수, hidden size, attention head 구성, 입력 처리 방식 등의 하이퍼파라미터를 포함


`config.json`은 

* 모델 코드를 변경 및 대체 하지 않으면서,
* 이미 라이브러리에 존재하는 모델 코드에 
* 구조 정보를 주입하여 모델의 형태를 결정할 수 있음.

단, 가중치 파일(parameters의 값들) 의 경우, 

* ***특정 구조 명세를 전제로 저장된 파라미터 상태*** 이므로,
* `config.json`이 정의한 구조와 일치할 때만 정상적으로 로딩된다.

### 학습 및 추론 관련 hyper-parameter 정의

`config.json`은 모델의 구조적 정의와 함께, 

* 학습(training) 및 추론(inference) 단계에서 사용되는 
* 주요 hyper-parameter를 포함함

이들 hyper-parameter는 

* 실행 환경이나 학습 전략과는 무관하게, 
* 모델 자체의 동작 방식을 결정함.

**examples**:

* `hidden_dropout_prob`, `attention_probs_dropout_prob`
	* 훈련 모드에서만 적용되는 dropout 확률을 정의함
	* 추론 시에는 비활성화되지만, 설정 자체는 모델 정의의 일부로 유지됨
* `hidden_act`, `pooler_act`
	* 학습과 추론 전반에 걸쳐 사용되는 activation 함수를 지정함
* `layer_norm_eps`, `initializer_range`
	* 수치 안정성, 초기화 특성 등 학습 거동에 영향을 주는 hyper-parameter를 규정함

### 프레임워크 독립적 모델 정의

`config.json` 은 다음의 framework들에서 모두 사용가능함:

* PyTorch
* TensorFlow
* Flax

HF는 동일 config로 다음의 framework용 AutoClass에서 사용됨:

* `AutoModel`
* `TFAutoModel`
* `FlaxAutoModel`


### Example

ViT config.json 항목 설명 표


| Key                            | 의미                                                                 |
| ------------------------------ | ------------------------------------------------------------------ |
| `architectures`                | 이 설정으로 생성되는 모델 클래스 이름. 보통 `["ViTForImageClassification"]` 등으로 명시됨  |
| `attention_probs_dropout_prob` | Self-Attention에서 **attention weight**에 적용되는 dropout 비율             |
| `dtype`                        | 모델 파라미터의 기본 데이터 타입 (`float32`, `float16` 등)                        |
| `encoder_stride`               | Feature map을 만들 때 encoder 출력의 stride 값. 주로 downstream task용        |
| `hidden_act`                   | Transformer MLP 블록에서 사용하는 activation 함수 (`gelu`, `relu`, `silu` 등) |
| `hidden_dropout_prob`          | Transformer block 내부에서 hidden state에 적용되는 dropout 비율               |
| `hidden_size`                  | Transformer의 **embedding dimension**. 각 token vector의 차원           |
| `id2label`                     | 분류 문제에서 **class id → label name** 매핑 딕셔너리                          |
| `image_size`                   | 입력 이미지의 한 변 크기 (정사각형 가정, 예: 224)                                   |
| `initializer_range`            | 가중치 초기화 시 사용하는 정규분포의 표준편차                                          |
| `intermediate_size`            | MLP block의 hidden dimension. 보통 `hidden_size * 4`                  |
| `label2id`                     | 분류 문제에서 **label name → class id** 매핑 딕셔너리                          |
| `layer_norm_eps`               | LayerNorm에서 수치 안정성을 위한 epsilon 값                                   |
| `model_type`                   | 모델 계열 식별자. ViT의 경우 `"vit"`                                         |
| `num_attention_heads`          | Multi-Head Self-Attention의 head 개수                                 |
| `num_channels`                 | 입력 이미지의 채널 수 (RGB = 3)                                             |
| `num_hidden_layers`            | Transformer encoder block의 개수 (depth)                              |
| `patch_size`                   | 이미지를 나누는 patch의 한 변 크기                                             |
| `pooler_act`                   | CLS token pooling 후 적용되는 activation 함수                             |
| `pooler_output_size`           | Pooler 출력 차원. 보통 `hidden_size`와 동일                                 |
| `qkv_bias`                     | Q, K, V projection에 bias를 사용할지 여부                                  |
| `transformers_version`         | 이 config를 생성한 `transformers` 라이브러리 버전                              |

## model.safetensors

* 학습되거나 fine-tuning된 **모델 파라미터(weight)** 를 저장한 파일
* 특정 `config.json`에 정의된 모델 구조를 전제로 한 tensor 값들의 집합
* 과거엔 `.bin` 확장자의 파일로 저장 : `safetensors` 가 보다 안전하고 빠른 로딩을 위해 사용되는 권장 포맷
* 해당하는 `config.json`이 없으면 정상적으로 로딩될 수 없음

## **`tokenizer.json`**

* 텍스트 입력을 모델이 처리 가능한 token ID로 변환하기 위한 **토크나이저 정의**
* vocab, tokenization 규칙, special token 정보 등을 포함
* 텍스트 기반 pipeline에서 입력 전처리를 재현하기 위해 필수적인 파일
* `AutoTokenizer` 또는 `pipeline()` 로딩 시 자동으로 참조됨

## **`preprocessor_config.json`**

* 이미지, 오디오 등 **비텍스트 입력에 대한 전처리 규칙**을 정의한 설정 파일
* 예: 이미지 resize 크기, normalization 방식, 채널 순서 등
* Vision pipeline에서는 `AutoImageProcessor` 또는 `ImageProcessor`가 이 파일을 참조함
* 입력 데이터가 학습 시와 동일한 방식으로 변환되도록 보장함

## **`README.md`**

* Hugging Face Hub에서 표시되는 **모델 카드(model card)**
* 모델의 목적, 학습 데이터, 사용 방법, 라이선스, 제한 사항 등을 문서화
* 사용자가 모델을 이해하고 재현 가능하게 사용하는 데 필요한 설명을 제공
* Hub 공개 및 공유를 위한 사실상 필수 문서





