---
layout  : category
title   : DatasetDict 와 Dataset
summary : 
date    : 2026-01-25 15:07:53 +0900
updated : 2026-01-25 16:25:20 +0900
tag     : 
resource: 24/020FA19AC243A8BE8ECF070EDFD860
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}


# Hugging Face Dataset 학습 튜토리얼


## 학습 목표

* Hugging Face 기반 학습 파이프라인의 전체 구조 이해

## 핵심 개념

* `Dataset`(데이터셋): 학습 데이터의 표준 컨테이너
* `DatasetDict`(데이터셋딕트): `train`/`validation`/`test` 분할 컨테이너
* `Processor`(프로세서): 모델 입력 전처리 표준 인터페이스
* `Trainer`(트레이너): 학습 루프 표준 구현체


## Index

* [[/hf_dataset_dict/dd_ds]]{load_dataset으로 Dataset과 DatasetDict 익히기}
* [[/hf_dataset_dict/dd_csv]]{Text 기반 Dataset 생성}
* [[/hf_dataset_dict/dd_map]]{map을 이용한 전처리와 학습 데이터 고정}


---

## 작업할 내용.

## Unit 4. Image 기반 Dataset을 학습용으로 변환

### 학습 목표

* 이미지 학습을 위한 HF Dataset 구조 이해

### 학습 내용

* image 컬럼과 Image(Image, 이미지 타입) 개념 이해
* imagefolder 방식 이해
* 이미지 경로 기반 Dataset 구성 원리 이해

### 실습 핵심

* imagefolder 또는 CSV + image 경로 Dataset 생성
* cast_column("image", Image()) 적용

### 도달 목표

* “이미지 Dataset도 map과 Trainer에 동일하게 연결 가능함”의 이해

---

## Unit 5. PyTorch Dataset에서 HF Dataset으로 전환

### 학습 목표

* 기존 PyTorch Dataset을 HF 학습 파이프라인에 연결하는 방법 이해

### 학습 내용

* torch.utils.data.Dataset의 한계 이해
* HF Dataset의 column 기반 구조 이해
* 경로 기반 변환 전략의 필요성 이해

### 실습 핵심

* PyTorch Dataset → list[dict] → Dataset.from_list 변환
* DatasetDict 구성

### 도달 목표

* “PyTorch Dataset을 버리지 않고 HF Trainer로 연결 가능함”의 이해

---

## Unit 6. map vs set_transform vs collate_fn (학습 관점 비교)

### 학습 목표

* 학습 파이프라인에서 전처리 위치 선택 기준 확립

### 학습 내용

* map: 사전 전처리 및 캐싱 중심
* set_transform: epoch별 동적 전처리 중심
* collate_fn: 배치 단위 처리 중심

### 학습 정리 규칙

* 고정 전처리 → map
* 랜덤 증강 → set_transform
* 가변 길이/멀티모달 → collate_fn

### 도달 목표

* “왜 Trainer에서 data_collator가 필요한지”의 이해

---

## Unit 7. Image + Text 멀티모달 학습 Dataset 구성

### 학습 목표

* 멀티모달 학습 데이터의 표준 컬럼 설계 이해

### 학습 내용

* modality(modality, 양식)별 컬럼 분리 원칙 이해
* image + text + label 구조 이해
* 배치 전처리 필요성 이해

### 실습 핵심

* 멀티모달 Dataset 생성
* collate_fn 또는 map(batched) 적용

### 도달 목표

* “멀티모달 Trainer 입력 구조”에 대한 명확한 그림 형성

---

## Unit 8. Detection / Segmentation 학습 Dataset 구조 이해

### 학습 목표

* 비정형 어노테이션 학습 데이터 구조 이해

### 학습 내용

* detection의 objects 구조 이해
* segmentation의 mask 구조 이해
* 가변 길이 어노테이션 문제 인식

### 핵심 개념

* collate_fn의 필수성
* Trainer 기본 data_collator의 한계 인식

### 도달 목표

* “왜 detection/segmentation은 Dataset 설계가 중요한지” 이해

---

## Unit 9. DatasetBuilder로 재현 가능한 학습 데이터 정의

### 학습 목표

* 데이터셋을 코드로 정의하는 개념 이해

### 학습 내용

* DatasetBuilder의 역할 이해
* Features(Features, 피처) 정의 개념 이해
* _generate_examples의 의미 이해

### 실습 핵심

* 간단한 GeneratorBasedBuilder 구조 파악

### 도달 목표

* “데이터셋도 하나의 소프트웨어 아티팩트”라는 인식 형성

---

## Unit 10. Hub 업로드와 학습 파이프라인 재사용

### 학습 목표

* 학습 데이터의 공유 및 재사용 흐름 이해

### 학습 내용

* push_to_hub의 의미 이해
* load_dataset으로 재다운로드 흐름 이해
* AutoModel + Trainer와의 연결 이해

### 실습 핵심

* DatasetDict → Hub 업로드
* Hub → load_dataset → Trainer 연결

### 도달 목표

* “학습 파이프라인의 재현성과 배포성” 이해


## Related Documents
