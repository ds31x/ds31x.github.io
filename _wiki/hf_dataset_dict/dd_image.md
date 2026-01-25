---
layout  : wiki
title   : HF - DatasetDict and Image
summary : 
date    : 2026-01-25 17:18:36 +0900
updated : 2026-01-25 20:04:31 +0900
tag     : 
resource: 15/95778599354FDF83056ABE2DD54537
toc     : true
public  : true
parent  : [[/hf_dataset_dict]]
latex   : false
---
* TOC
{:toc}


# Image 를 HF DatasetDict 로 다루기.

* `DatasetDict` 기반 학습 파이프라인 구성 
* `"image"` (이미지 원본 컬럼)와 `"pixel_values"`(모델 입력 텐서 컬럼) 구분
* `map` 기반 고정 전처리 구성 능력 확보
* `set_transform` 기반 random data augmentation(`torchvision` 사용) 이해.
* `collate_fn` 기반 배치 전처리 구성 능력 확보
* Trainer (트레이너)로 end-to-end 학습 실행해 보기

이 문서에서 dataset은 PyTorch의 `Dataset`이 아닌 HF의 `Dataset` 을 지칭함.
* PyTorch의 Dataset and DataLoader 은 다음을 참고: [[PyTorch] Dataset and DataLoader](https://ds31x.tistory.com/234)

## 사전 개념 정리

`"image"`와 `"pixel_values"`의 차이 정리

* `image`: `Dataset` 계층의 원본 입력 표현(`PIL.Image`) 의미
* `pixel_values`: Transformers 모델이 받는 입력 텐서(`torch.Tensor`) 의미
	* `"pixel_values"`는 보통 `AutoImageProcessor` (ImageProcessor)가 생성
	* Trainer는 `image`를 직접 사용하지 않고, `model(**batch)` 형태로 `"pixel_values"` 키를 요구.

## 실습 0. 환경 점검과 공통 준비

### 0-1. 패키지 버전 점검 단계

```python
import datasets
import transformers

print("datasets:", datasets.__version__)
print("transformers:", transformers.__version__)
```

### 0-2. 실습용 더미 이미지 데이터 생성 단계

실제 데이터가 이미 있다면 이 절 생략 가능함.

* 다음의 [`torchvision.datasets.ImageFolder` 예제](https://ds31x.tistory.com/463)의 데이터를 이용해되 됨: [test.zip](https://blog.kakaocdn.net/dna/bS07lS/btsOFOCM3H8/AAAAAAAAAAAAAAAAAAAAAAYJV_D-rBX5EByPczHoNZsi5q_yo_BLp823HqwImUp1/test.zip?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1769871599&allow_ip=&allow_referer=&signature=o6VHhtb1Ez15xnC1ePgx51dHJLw%3D&attach=1&knm=tfile.zip) 

이 절은 `imagefolder`로더를 위한 최소 폴더 구조 생성 절(classifcation에서 많이 사용됨)임.

```python
from pathlib import Path
from PIL import Image
import numpy as np

def make_dummy_image(path: Path, seed: int, size=(256, 256)):
    rng = np.random.default_rng(seed)
    arr = rng.integers(0, 256, size=(size[1], size[0], 3), dtype=np.uint8)
    img = Image.fromarray(arr, mode="RGB")
    path.parent.mkdir(parents=True, exist_ok=True)
    img.save(path)

dataset_root = Path("dataset_root")
for split in ["train", "validation"]:
    for cls in ["class0", "class1"]:
        n = 24 if split == "train" else 8
        for i in range(n):
            make_dummy_image(
                dataset_root / split / cls / f"{i:04d}.png",
                seed=hash((split, cls, i)) % (2**32)
            )

print("dataset_root ready")
```

## 실습 1. imagefolder로 DatasetDict 만들기

### 1-1. DatasetDict 로딩 단계

```python
from datasets import load_dataset

dd_raw = load_dataset("imagefolder", data_dir="dataset_root")
print(dd_raw)
print(dd_raw["train"].features)
```

### 1-2. 샘플 접근과 타입 확인 단계

```python
ex = dd_raw["train"][0]
print(ex.keys())
print(type(ex["image"]))     # PIL.Image.Image 기대
print(ex["label"])           # int 기대
```

### 1-3. ClassLabel(names) 확인 단계

```python
label_feature = dd_raw["train"].features["label"]
print(label_feature)
print("names:", label_feature.names)
```

## 실습 2. Trainer 학습을 위한 최소 구성 이해

### 2-1. 핵심 규칙 확인 단계

Trainer는 모델 입력을 `model(**batch)` 형태로 전달하는 구조임.

Vision 모델은 보통 `pixel_values` 키를 요구하므로  
따라서 학습 `batch`에는 최소한 다음이 필요함.

* `"pixel_values"` 키의 데이터 : `torch.Tensor`
* `"labels" 키의 데이터 : `torch.LongTensor` 또는 label 컬럼에서 유도되는 값

여기서 labels 키 이름은 일반적으로 `labels`가 사용됨.

* `Dataset` 컬럼명은 `label` 인 경우, 
* `collate_fn`에서 `labels`로 만들어 전달하는 구조가 애용됨.


## 실습 3. map 사용 방식 실습 

고정 전처리 에는 `map` 을 사용.
* 데이터셋 사전 변환
* 캐싱 가능
* 대량 처리

이 절에서는 

* image를 `pixel_values`로 미리 변환하여 
* `DatasetDict`를 학습용 텐서로 고정 전처리.

`map`으로 적용되는 callable객체는 인자를 두가지 방식으로 선택가능함:

* `batched=False` 인 경우 : sample을 인자로 받음 (`dict`).
* `batched=True` 인 경우 : batch를 인자로 받음 (`dict` or `list`).
* 기본적으로 `batched=False`임.



### 3-1. AutoImageProcessor 준비 단계

```python
from transformers import AutoImageProcessor

ckpt = "google/vit-base-patch16-224-in21k"
processor = AutoImageProcessor.from_pretrained(ckpt)
print("processor ready")
```

### 3-2. map 전처리 함수 작성 단계

```python
def preprocess(batch):
    out = processor(batch["image"], return_tensors="pt")
    batch["pixel_values"] = out["pixel_values"]
    return batch
```

### 3-3. DatasetDict 전체에 map 적용 단계

`map`은 split별 `Dataset` 객체에 따로 적용하는 것도 가능하나,  
일반적으론 `DatasetDict` 객체에 같이 적용함.

```python
dd_map = dd_raw.map(
    preprocess,
    batched=True,
    batch_size=8,
    remove_columns=["image"],   # 고정 전처리형에서 권장
)
print(dd_map)
print(dd_map["train"].features)
```

### 3-4. Trainer 입력을 위한 torch 포맷 설정 단계

```python
dd_map.set_format(type="torch", columns=["pixel_values", "label"])
item = dd_map["train"][0]

print(item["pixel_values"].shape)
print(item["label"])
```

### 3-5. 주의할 점

* `remove_columns=["image"]` 이후 `set_transform` 을 사용할 경우 `image`를 참조하기 때문에 문제가 발생함.
* `batched=False`로 두고 processor를 호출할 경우 성능 저하가 발생하니 주의할 것.
* `load_from_cache_file=True` 동작이 기본이므로 디버그 단계에서 함수 수정이 반영되지 않을 수 있으니 주의 해야함.

디버깅 시 캐시 무시 옵션 (`load_from_cache_file=False`)을 사용할 것.

```python
dd_map = dd_raw.map(
    preprocess,
    batched=True,
    load_from_cache_file=False,
    remove_columns=["image"],
)
```

## 실습 4. set_transform 사용 방식 실습 

`set_transform`은 **`Dataset`(HF의 `Dataset`)의 `__getitem__`을 살짝 감싸는 것"** 으로 이해할 것.

* `Dataset`의 `__getitem__` 시점 처리
* epoch마다 반복 호출
* `PyTorch`의 `DataLoader`와 결합
* 주로 `torchvision.transforms`와 연동됨: [`torchvision.transforms` 에 대한 참고자료](https://ds31x.tistory.com/461)
	* `torchvision.transforms.v2` 사용 권장.

Random Data Augmentation 에 주로 사용.

* epoch마다 달라지는 랜덤 증강을 image 기준으로 적용하여 `pixel_values`를 on-the-fly 생성 하는 방식.
* 이를 위해서 `"image"` 키의 column을 유지해야 함.

주의할 점은 
* `set_transform`에 넘겨지는 callable 객체는 반드시 하나의 sample로 넘겨짐
* batch 전체가 넘겨지지 않음.

다음은 넘겨지는 하나의 sample `example`의 일반적인 형태: 

```python
example = {
    "image": PIL.Image,
    "label": int
}
```

### 4-1. torchvision transform (v2) 준비 단계

```python
from torchvision.transforms import v2 as T

train_tfms = T.Compose([
    T.RandomResizedCrop(224),
    T.RandomHorizontalFlip(),
])
```

### 4-2. train용 transform 함수 작성 단계

```python
def train_transform(example):
    img = train_tfms(example["image"])
    example["pixel_values"] = processor(img, return_tensors="pt")["pixel_values"][0]
    return example
```

### 4-3. validation용 transform 함수 작성 단계

validation에는 랜덤 증강을 보통 적용하지 않는 것이 권장됨.

```python
def val_transform(example):
    example["pixel_values"] = processor(
    			example["image"], 
			return_tensors="pt")["pixel_values"][0]
    return example
```

### 4-4. split별 set_transform 적용 단계

```python
dd_aug = dd_raw
dd_aug["train"].set_transform(train_transform)
dd_aug["validation"].set_transform(val_transform)

x = dd_aug["train"][0]
y = dd_aug["validation"][0]
print(x["pixel_values"].shape, x["label"])
print(y["pixel_values"].shape, y["label"])
```

### 4-5. 주의 사항

* `"image"` 컬럼이 `map` 등에서 제거된 경우 `set_transform`을 적용시 문제가 발생.
* `train`과 `validation`에 동일한 랜덤 증강을 적용하는 것은 피할 것.
* `transform` 함수에서 batch 입력이 아님을 기억.


## 실습 5. collate_fn 사용 방식 실습 (배치 전처리)

* `Dataset`은 원본 image를 유지하면서  
* batch 구성 시점에 processor를 적용하여  
* `pixel_values`와 `labels`를 생성하는 방식

이 방식은 segmentation, detection으로 확장에 많이 사용됨.

### 5-1. collate_fn 정의 단계

```python
import torch

def collate_fn(batch):
    images = [b["image"] for b in batch]
    labels = torch.tensor([b["label"] for b in batch], dtype=torch.long)

    out = processor(images, return_tensors="pt")
    out["labels"] = labels
    return out
```

### 5-2. DataLoader로 동작 검증 단계

`PyTorch`의 `DataLoader` 객체 와 함께 사용됨.

```python
from torch.utils.data import DataLoader

loader = DataLoader(dd_raw["train"], batch_size=4, shuffle=True, collate_fn=collate_fn)
batch = next(iter(loader))
print(batch["pixel_values"].shape)
print(batch["labels"].shape)
```

## 5-3. 주의 사항

* `collate_fn`에서 `"label"` 키를 `"labels"`로 만들지 않을 경우 HT의 모델 입력 불일치 발생 가능성
	* ViT 등의 경우, `pixel_values`를 입력텐서로 `labels`는 라벨텐서로 기대.
	* `DatasetDict` 객체의 경우 `"label"`을 키로 사용하는 경우가 일반적임.
	* 이는 키의 불일치로 문제가 일어남: 일반적으로 기본 collator 가 이를 처리해주는데 커스텀 `collate_fn`을 쓰면 문제가 일어나기 쉬움.
* 이미지 크기 가변 상태에서 모델이 기대하는 크기 불일치 발생 가능성
* `DataLoader` 단계에서 처리되므로 CPU 에서 연산 병목이 커질 수 있음.


## 실습 6. Trainer로 classification 학습 실행 실습

앞의 방식으로 구성된 DatasetDict를 Trainer에 연결하여 학습이 되는지를 확인하는 실습.

(연습 목적임.)

### 6-1. 모델 준비 단계

```python
from transformers import AutoModelForImageClassification

# clf 의 경우 
ckpt = "microsoft/swin-tiny-patch4-window7-224"
# ckpt = "facebook/convnext-tiny-224"

num_labels = len(dd_raw["train"].features["label"].names)
model = AutoModelForImageClassification.from_pretrained(
    ckpt,
    num_labels=num_labels
)
print("model ready")
```

### 6-2. TrainingArguments 설정 단계

```python
from transformers import TrainingArguments

training_args = TrainingArguments(
    output_dir = "./out_test",
    per_device_train_batch_size = 8,
    per_device_eval_batch_size  = 8,
    num_train_epochs = 1,
    eval_strategy = "epoch",
    save_strategy = "no",
    logging_strategy = "steps",
    logging_steps = 5,
    report_to = "none",
)
```

### 6-3. Trainer 실행 방식 3가지 연결 패턴

#### 패턴 A: map 기반 dd_map 사용 방식

```python
from transformers import Trainer

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=dd_map["train"],
    eval_dataset=dd_map["validation"],
)
trainer.train()
```

#### 패턴 B: set_transform 기반 dd_aug 사용 방식

* `dataset` 항목이 이미 `pixel_values`를 포함하므로 그대로 사용 가능함.
* 다만 Trainer가 `"labels"` 키를 기대하므로, `"label"`을 `"labels"`로 맞추는 `data_collator` 사용이 권장됨.
* 즉, `collate_fn`을 같이 쓰는게 좋음.

```python
from transformers import Trainer

def collate_from_pixel_values(batch):
    import torch
    pixel_values = torch.stack([b["pixel_values"] for b in batch], dim=0)
    labels = torch.tensor([b["label"] for b in batch], dtype=torch.long)
    return {"pixel_values": pixel_values, "labels": labels}

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=dd_aug["train"],
    eval_dataset=dd_aug["validation"],
    data_collator=collate_from_pixel_values,
)
trainer.train()
```

#### 패턴 C: collate_fn 기반 dd_raw 사용 방식

```python
from transformers import Trainer

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=dd_raw["train"],
    eval_dataset=dd_raw["validation"],
    data_collator=collate_fn,
)
trainer.train()
```

## 실습 7. 선택 기준 정리 연습

### 7-1. 선택 기준 표기 연습

* `map`: 전처리 결과 고정, 캐싱 활용, 실험 재현성 이 필요한 경우.
* `set_transform`: data augmentation이 필요한 경우, epoch마다 다른 입력이 필요한 경우.
* `collate_fn`: 배치 시점 처리, 가변 구조 태스크 확장 용도

### 7-2. 자가 점검 질문 3개

* “image 컬럼을 제거해도 되는 상황” 판별 연습
* “validation에 랜덤 증강을 적용하면 안 되는 이유” 설명 연습
* “Trainer가 pixel_values 키를 요구하는 이유” 설명 연습


## 실습 8. Object Detection 테스크 간단 실습 (작성중)

* `DatasetDict` + `collate_fn` 기반 구성
* object detection은

  * image와 bbox의 동시 변환이 핵심
  * bbox는 좌표 체계(format)와 canvas_size 관리가 핵심
  * v2에서는 `tv_tensors.BoundingBoxes`를 사용하면 변환 동기화가 정석

이 예제는 이해를 돕기 위한 간단한 더미 annotation 기반 예제임.

### 8-0. detection 실습용 ckpt 준비 단계

Detection은 분류/segmentation과 processor 규칙이 다르므로 **detection용 ckpt**를 사용해야 함.

```python
from transformers import AutoImageProcessor, AutoModelForObjectDetection

ckpt = "facebook/detr-resnet-50"
processor = AutoImageProcessor.from_pretrained(ckpt)
model = AutoModelForObjectDetection.from_pretrained(ckpt)
```

### 8-1. 더미 bbox annotation 생성 단계

* 각 이미지마다 박스 1개를 랜덤 생성하는 예시
* bbox는 transform 전에 일단 **XYXY**로 만들고, transform 후 processor에 줄 때 **XYWH(COCO)**로 변환하는 구성이 안정적임

```python
import numpy as np
from PIL import Image

def make_dummy_boxes_for_image(img_pil: Image.Image, seed: int):
    rng = np.random.default_rng(seed)
    w, h = img_pil.size

    # 1개 박스 생성: (x1, y1, x2, y2)
    x1 = int(rng.integers(0, max(1, w // 2)))
    y1 = int(rng.integers(0, max(1, h // 2)))
    x2 = int(rng.integers(max(x1 + 1, w // 2), w))
    y2 = int(rng.integers(max(y1 + 1, h // 2), h))

    boxes_xyxy = [[x1, y1, x2, y2]]
    class_ids = [0]  # 단일 클래스 예시 (0)
    return boxes_xyxy, class_ids
```

### 8-2. detection DatasetDict 생성 단계

* `"image"`는 `datasets.Image()`로 캐스팅하여 PIL.Image로 로딩
* `"objects"` 컬럼에 bbox와 category를 저장
* 여기서는 bbox를 **XYXY**로 저장하고, collate_fn에서 `BoundingBoxes`로 감싸 v2 변환 적용

```python
from pathlib import Path
from datasets import Dataset, DatasetDict, Image as HFImage

def list_rows_for_detection(img_dir: Path):
    rows = []
    for ip in sorted(img_dir.glob("**/*.png")):
        rows.append({"image": str(ip)})
    return rows

train_rows = list_rows_for_detection(dataset_root / "train")
val_rows   = list_rows_for_detection(dataset_root / "validation")

train_ds = Dataset.from_list(train_rows).cast_column("image", HFImage())
val_ds   = Dataset.from_list(val_rows).cast_column("image", HFImage())

dd_det = DatasetDict({"train": train_ds, "validation": val_ds})
print(dd_det["train"].features)
```

* 이 단계에서는 일단 image만 구성함
* bbox는 “이미지를 실제로 열어 크기를 알아야” 만들기 쉬우므로, 더미 bbox는 collate_fn에서 생성하는 방식으로 단순화함
* 실제 데이터에서는 bbox를 dataset 컬럼으로 보관하는 것이 정석임


### 8-3. v2 + tvtensor 기반 detection collate_fn 검증 

* v2 변환은 `tv_tensors.Image`와 `tv_tensors.BoundingBoxes`를 함께 전달해야 **동일 랜덤 파라미터**가 공유됨
* bbox는 반드시 `canvas_size=(H, W)`가 필요함
* 변환 후 HF processor에는 **COCO 형식(XYWH)** annotations를 제공하는 방식이 DETR 계열에서 정석적임

```python
import torch
from torch.utils.data import DataLoader
from torchvision.transforms import v2 as T
from torchvision import tv_tensors

# train용 기하 변환 예시 (간단 버전)
train_geo = T.Compose([
    T.Resize((512, 512)),
    T.RandomHorizontalFlip(p=0.5),
])

# tvtensor -> PIL 변환용 (processor에 PIL로 주는 구성이 안전)
to_pil = T.ToPILImage()

def xyxy_to_xywh(box_xyxy):
    x1, y1, x2, y2 = box_xyxy
    return [float(x1), float(y1), float(x2 - x1), float(y2 - y1)]

def collate_det(batch):
    images_pil = []
    annotations = []

    for i, b in enumerate(batch):
        img_pil = b["image"]               # PIL.Image
        boxes_xyxy, class_ids = make_dummy_boxes_for_image(img_pil, seed=hash(("det", i)) % (2**32))

        # tvtensor wrapping
        img_tv = T.ToImage()(img_pil)      # tv_tensors.Image, shape (C,H,W)
        H, W = img_tv.shape[-2], img_tv.shape[-1]

        boxes_tv = tv_tensors.BoundingBoxes(
            torch.tensor(boxes_xyxy, dtype=torch.float32),
            format="XYXY",
            canvas_size=(H, W),
        )

        # v2 transform 동기 적용
        img_tv, boxes_tv = train_geo(img_tv, boxes_tv)

        # processor 입력 준비: PIL 이미지로 변환
        img_pil2 = to_pil(img_tv)
        images_pil.append(img_pil2)

        # processor annotations(COCO) 준비: XYWH + category_id
        ann_list = []
        for box, cid in zip(boxes_tv.tolist(), class_ids):
            x, y, w, h = xyxy_to_xywh(box)
            area = float(max(w, 0.0) * max(h, 0.0))
            ann_list.append({
                "bbox": [x, y, w, h],
                "category_id": int(cid),
                "area": area,
                "iscrowd": 0,
            })

        annotations.append({
            "image_id": i,
            "annotations": ann_list,
        })

    # HF processor가 detection 학습 배치를 생성
    # DETR 계열에서는 processor가 pixel_values와 labels(타깃 dict 리스트)를 만들어 줌
    enc = processor(images=images_pil, annotations=annotations, return_tensors="pt")

    # enc.keys() 예: pixel_values, pixel_mask, labels 등
    return enc

det_loader = DataLoader(dd_det["train"], batch_size=2, shuffle=True, collate_fn=collate_det)
det_batch = next(iter(det_loader))

print(det_batch.keys())
print(det_batch["pixel_values"].shape)
print(type(det_batch["labels"]), len(det_batch["labels"]))
print(det_batch["labels"][0].keys())
```

* `tv_tensors.BoundingBoxes`를 쓰는 이유

  * v2 transform이 bbox를 “의미 있는 객체”로 인식
  * resize/flip 등에서 bbox가 자동으로 올바르게 갱신됨
  * image와 bbox에 동일 랜덤 파라미터가 적용됨

* processor에 annotations를 주는 이유

  * DETR 계열은 `labels`가 단순 정수 텐서가 아니라, bbox/클래스 정보를 포함한 구조체(dict) 리스트 형태임
  * `processor(images, annotations=...)`가 학습 입력(`pixel_values`)과 타깃(`labels`)을 함께 구성하는 것이 정석임


## 실습 9. segmentation 테스크 간단 실습 (작성 중...)

아래는 “HF 모델(Transformers) + DatasetDict + Trainer” 조합에서 **정석적으로** 쓰는 형태에 가깝게, 

* `torchvision.transforms.v2`와 `tvtensor`를 이용해 
* **image와 mask에 동일한 랜덤 기하 변환을 동기 적용**하도록 작성한 예제임. 
* 핵심은 `Mask`를 tvtensor로 만들면 **보간(NEAREST)과 변환 동기화가 자연스럽게 보장** 됨을 확인하는 것.


### 핵심 개념

* v2 변환은 **입력이 `tvtensor`일 때** task-aware 동작이 가장 안정적임
* `tv_tensors.Image`와 `tv_tensors.Mask`를 함께 변환 함수에 넣으면
  **같은 랜덤 파라미터**(crop 위치, flip 여부 등)가 **image와 mask에 동일 적용**됨
* `Mask`는 라벨 맵이므로 변환 시 **최근접 보간(NEAREST)**이 유지되어야 함
  tvtensor를 쓰면 이 규칙이 자연스럽게 지켜지도록 설계됨


### 0) 준비: processor와 (선택) 모델

```python
from transformers import AutoImageProcessor, AutoModelForSemanticSegmentation

ckpt = "nvidia/segformer-b0-finetuned-ade-512-512"  # 예시
processor = AutoImageProcessor.from_pretrained(ckpt)
model = AutoModelForSemanticSegmentation.from_pretrained(ckpt)
```

### 1) v2 transform 정의: train과 validation 분리

* train: resize + random crop + random flip 같은 “기하 증강” 중심 구성 권장
* validation: resize 같은 “결정적(deterministic) 변환”만 적용 권장

```python
from torchvision.transforms import v2 as T

# 입력 크기 정책(예시)
RESIZE_TO = (512, 512)
CROP_TO   = (448, 448)

train_geo = T.Compose([
    T.Resize(RESIZE_TO),          # image/mask 모두 동일 resize
    T.RandomCrop(CROP_TO),        # 동일 crop 파라미터 공유
    T.RandomHorizontalFlip(p=0.5) # 동일 flip 파라미터 공유
])

val_geo = T.Compose([
    T.Resize(RESIZE_TO)
])
```

### 2) tvtensor 변환 + collate_fn (train용 / val용)

* HF Dataset의 `"image"`, `"mask"`는 보통 PIL.Image로 들어옴
* v2 변환을 제대로 쓰려면 `Image`/`Mask`를 tvtensor로 감싸는 편이 정석임
* 변환 후 HF `processor`로 `pixel_values`를 만들고, mask는 `(H, W)`의 `int64` 텐서로 유지함

```python
import numpy as np
import torch
from torchvision import tv_tensors
from torchvision.transforms import v2 as T

to_pil = T.ToPILImage()   # tvtensor(Image) -> PIL 변환용

def _wrap_as_tvtensors(img_pil, mask_pil):
    # Image: v2 ToImage()로 tv_tensors.Image로 변환
    img_tv = T.ToImage()(img_pil)  # tv_tensors.Image

    # Mask: 클래스 id 라벨 맵이므로 정수 텐서로 tv_tensors.Mask로 변환
    mask_np = np.array(mask_pil, dtype=np.int64)  # (H, W)
    mask_tv = tv_tensors.Mask(torch.from_numpy(mask_np))  # tv_tensors.Mask

    return img_tv, mask_tv


def make_collate_seg(geo_transform, processor):
    def collate_seg(batch):
        images = []
        masks  = []

        for b in batch:
            img_pil = b["image"]
            msk_pil = b["mask"]

            img_tv, msk_tv = _wrap_as_tvtensors(img_pil, msk_pil)

            # v2 transform에 (img, mask)를 같이 넣으면 동일 랜덤 파라미터가 공유됨
            img_tv, msk_tv = geo_transform(img_tv, msk_tv)

            # processor는 PIL 입력이 가장 안전하므로 PIL로 변환 후 전달
            img_pil2 = to_pil(img_tv)
            images.append(img_pil2)

            # labels는 (H, W) int64 유지
            masks.append(msk_tv.to(torch.int64))

        out = processor(images, return_tensors="pt")  # pixel_values 생성
        labels = torch.stack(masks, dim=0)            # (B, H, W)

        return {
            "pixel_values": out["pixel_values"],
            "labels": labels
        }

    return collate_seg


collate_seg_train = make_collate_seg(train_geo, processor)
collate_seg_val   = make_collate_seg(val_geo, processor)
```

### 3) DataLoader로 검증

```python
from torch.utils.data import DataLoader

seg_loader = DataLoader(
    dd_seg["train"],
    batch_size=2,
    shuffle=True,
    collate_fn=collate_seg_train
)

seg_batch = next(iter(seg_loader))
print(seg_batch["pixel_values"].shape)  # (B, 3, H, W) - processor 설정에 따름
print(seg_batch["labels"].shape)        # (B, CROP_H, CROP_W) 예: (2, 448, 448)
print(seg_batch["labels"].dtype)        # torch.int64
```

### Trainer에 연결할 때의 정석 포인트

* segmentation은 보통 train/eval에 서로 다른 collate를 쓰고 싶지만, `Trainer`는 `data_collator`를 하나만 받음
* 초보자 연습에서는 다음 중 하나를 선택하는 방식이 안전함

  1. **eval도 train과 동일한 크기 변환을 사용**(증강만 빼고 크기만 동일)
  2. eval은 `trainer.evaluate()` 전에 임시로 collator를 바꾸는 실험(초보자에겐 비권장)

> 연습용으로는  
> "train용 collate를 eval에도 동일하게 사용하되, geo_transform만 deterministic으로 구성"이 가장 단순. 
> 즉 `Random*`를 제외한 `Resize`만 쓰는 `val_geo` 대신, 아예 train에서도 `Resize`만 쓰는 버전을 권장.


### 요약

* v2를 “정석적으로” 쓰려면 tvtensor(`tv_tensors.Image/Mask`) 기반이 가장 안정적임
* `(image, mask)`를 함께 transform에 전달하여 **랜덤 파라미터 동기화** 확보가 핵심임
* mask는 라벨 맵이므로 dtype과 보간 규칙(NEAREST)이 깨지지 않도록 설계하는 것이 핵심임
* HF 모델 입력은 `pixel_values`, 타깃은 `labels` 키로 맞추는 것이 가장 안전한 습관임

---


## 참고: `"image"` 열이 필요한 경우

* 랜덤 증강을 적용할 때
	* `set_transform` 사용 시
	* epoch마다 다른 변형이 필요할 때
* 배치 시점 전처리를 할 때
	* `collate_fn`에서 processor를 적용할 때
	* detection, segmentation, multimodal로 확장할 때
* Segmentation / Detection 태스크
	* image와 mask, box를 같이 변형해야 할 때
	* resize, crop을 동기화해야 할 때
* 다른 모델로 재학습 가능성을 남길 때
	* processor가 바뀔 수 있을 때
	* 해상도나 정규화 규칙이 달라질 때
* 디버깅·시각화가 필요할 때
	* 원본 이미지를 직접 확인해야 할 때

참고로 image 컬럼이 필요 없는 경우

* map으로 pixel_values를 고정 생성하고 나서
* 추가 증강이나 재전처리 계획이 없을 때
* 주로 단순 분류 학습만 할 때

