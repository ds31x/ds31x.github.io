---
layout  : category
title   : HF - DatasetDict 와 Dataset
summary : 
date    : 2026-01-25 15:07:53 +0900
updated : 2026-01-26 20:39:40 +0900
tag     : 
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
