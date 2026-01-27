---
layout  : category
title   : HF - DatasetDict 와 Dataset
summary : 
date    : 2026-01-25 15:07:53 +0900
updated : 2026-01-27 10:46:08 +0900
tag     : dataset hf
resource: 24/020FA19AC243A8BE8ECF070EDFD860
toc     : true
public  : true
parent  : [[/index]]
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

기본:

* [[/hf_dataset_dict/dd_ds]]{load_dataset으로 Dataset과 DatasetDict 익히기}
* [[/hf_dataset_dict/dd_csv]]{Text 기반 Dataset 생성}
* [[/hf_dataset_dict/dd_map]]{map을 이용한 전처리와 학습 데이터 고정}
* [[/hf_dataset_dict/dd_image]]{Image 기반 Dataset 생성}
* [[/hf_dataset_dict/dd_api]]

참고:

* [[/hf_dataset_dict/dd_conv]]
* [[/hf_dataset_dict/dd_map_transform_collate]]

응용:

* [[/hf_dataset_dict/dd_multimodal]]
* [[/hf_dataset_dict/dd_hf_hub]]


---

---


## 작성 예정:

### 1. Detection / Segmentation 학습 Dataset 구조 이해

#### 학습 목표

* 비정형 어노테이션 학습 데이터 구조 이해

#### 학습 내용

* detection의 objects 구조 이해
* segmentation의 mask 구조 이해
* 가변 길이 어노테이션 문제 인식

#### 핵심 개념

* collate_fn의 필수성
* Trainer 기본 data_collator의 한계 인식

#### 도달 목표

* "왜 detection/segmentation은 Dataset 설계가 중요한지" 이해

---

### 2. DatasetBuilder로 재현 가능한 학습 데이터 정의

#### 학습 목표

* 데이터셋을 코드로 정의하는 개념 이해

#### 학습 내용

* DatasetBuilder의 역할 이해
* Features(Features, 피처) 정의 개념 이해
* _generate_examples의 의미 이해

#### 실습 핵심

* 간단한 GeneratorBasedBuilder 구조 파악

#### 도달 목표

* "데이터셋도 하나의 소프트웨어 아티팩트"라는 인식 형성

---

---

## Related Documents
