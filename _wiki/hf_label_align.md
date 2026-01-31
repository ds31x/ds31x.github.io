---
layout  : wiki
title   : Lable Alignment in NER
summary : 
date    : 2026-01-30 21:41:20 +0900
updated : 2026-01-30 23:23:20 +0900
tag     : 
resource: EC/7960EA6C084F68813213F398502C89
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# Label Alignment in Named Entity Recognition (NER)

> NER Dataset의 `Dataset`의 4.0부터 이 전 방식이 안 되는 경우 많음.  
> `revision`을 `refs/convert/parquet` 로 바꾸는 게 가능한 데이터셋을 사용할 것.

[관련 gist 의 ipynb](https://gist.github.com/dsaint31x/5ca5062bcf34d65b879d32ae1c5d1a2e)

## 1. Named Entity Recognition (NER)와 BIO Tagging

**Named Entity Recognition (NER, 개체명 인식)**은 

* 입력 문장의 각 token(토크)에 대해 
* entity(개체) 범주를 예측하는 **Token Classification (토큰 분류)** 문제.  

### 1-1. IOB2 Taggking Sceme

NER에서 가장 널리 사용되는 라벨링 체계는 **IOB2 scheme (IOB2 태깅 체계, BIO)** 이다.

- **B-X (Begin)**: 개체 X의 시작 토큰
- **I-X (Inside)**: 개체 X 내부 토큰
- **O (Outside)**: 개체가 아닌 토큰

### 1-2. 주의 사항

대부분의 NER 데이터셋에서 
* BIO 라벨은 **이미 주어진 상태** 임.

## 2. Label Alignment (라벨 정렬)의 필요성

현대의 Transformer 기반 언어 모델은 
* **Subword Tokenization (서브워드 토크나이제이션)**을 사용.  
* 대표적인 예로 
	* **WordPiece**, 
	* **BPE**, 
	* **SentencePiece** 방식이 있다.

이는 NER 데이터셋의 라벨과 다른 기준 좌표계(label coordinate system)를 가질 수 있다는 문제를 일으킴:

- **word-level (단어 단위) 라벨**
- **character-level (문자 단위) 라벨**


Transformer 모델 입력은 항상 **subword-level (서브워드 단위)** `Token Sequence` 임.  

* 라벨 좌표계와 모델 입력 좌표계가 일치하지 않는 문제가 발생할 수 있음,  

> 이를 해결하기 위한  
> 전처리 단계가 바로 **Label Alignment (라벨 정렬)** 이다.

Hugging Face의 (Fast) Tokenizer는 이를 위해 두 가지 핵심 정보를 제공한다.

- `word_ids()` : word-level 라벨 정렬을 위한 메타정보
- `offset_mapping` : character-level 라벨 정렬을 위한 문자 오프셋 정보

> NER 에선 사실상 Fast Tokenizer 를 사용해야 함.
> Tokenization 과정에서 alignment metadata를 함께 생성하기 때문임.


## 3. Test 용 유틸함수

이 문서에서 NER 전처리의 Label Alignment가 잘 되었는지 여부를 다음을 확인한다:

1. **subword 토큰과 label을 인덱스 별로 나란히 출력**
2. **학습에서 제외할 토큰은 `-100` (ignore index)로 표시**

```python
def show_alignment(encoded, tokenizer, id2label, n=80):
    tokens = tokenizer.convert_ids_to_tokens(encoded["input_ids"])
    labels = encoded["labels"]
    m = min(len(tokens), len(labels), n)

    for i in range(m):
        lab = "IGN" if labels[i] == -100 else id2label[labels[i]]
        print(f"{i:3d} {tokens[i]:16s} {lab}")
````


## 4. Case 1: 영어 · word-level BIO → `word_ids()` 정렬

### 4.1 데이터셋: CoNLL-2003

**CoNLL-2003**은 대표적인 **word-level BIO** 데이터셋.

* `tokens[i]` : 단어(word)
* `ner_tags[i]` : 해당 단어의 BIO 라벨

[https://huggingface.co/datasets/tner/conll2003](https://huggingface.co/datasets/tner/conll2003)

`datasets` 4.0.0 부터 parquet 가 없이 script코드만으로 구성된 데이터셋은 다운로드가 안된다.
다음의 코드로 받을 것.
```python
from datasets import load_dataset

ds_en = load_dataset(
    "tner/conll2003",
    revision="refs/convert/parquet" #Parquet으로 변환된 브랜치.
)

print(ds_en)
print(ds_en['train'].features) # ner_tags 가 없다. int인 tasg만 존재.
print(ds_en['train'].column_names)
print(ds_en['train'][0])

example_en = ds_en['train'][0]
```

> 보안의 문제로 인해서임. (쓸데없이 악용하는 인간들 때문에 불편함만 늘어난다.)
> 스크립트 기반 데이터셋은 `refs/convert/parquet` 브랜치로 우회 필요

위의 `tner/conll2003` 의 경우, `ClassLabel` 메타데이터가 깨져서 단순 `int32` 리스트임. ==;;
다음의 코드로 수동 매핑을 정의한다 (Parquet Conversion에서 생각보다 이런 경우 많음).

```python
# tner/conll2003 레이블 정의
label_names = ['O', 'B-ORG', 'B-MISC', 'B-PER', 'I-PER', 'B-LOC', 'I-LOC', 'I-ORG', 'I-MISC']

id2label_en = {i: label for i, label in enumerate(label_names)}
label2id_en = {label: i for i, label in enumerate(label_names)}

print(id2label_en)
```


### 4.2 전처리 (word-level BIO 표준 패턴)

```python
def tokenize_and_align_labels_wordlevel(example, tokenizer):
    tokenized = tokenizer(
        example["tokens"],
        is_split_into_words=True,
        truncation=True
    )

    word_ids = tokenized.word_ids()
    labels = []
    prev_word_id = None

    for word_id in word_ids:
        if word_id is None:
            labels.append(-100)
        elif word_id == prev_word_id:
            labels.append(-100)
        else:
            # labels.append(example['ner_tags'][wid]) # 'conll2003' 사용시
            labels.append(example['tags'][wid]) # 'tner/conll2003' 사용시
        prev_word_id = word_id

    tokenized["labels"] = labels
    return tokenized
```

참고: [https://huggingface.co/learn/llm-course/en/chapter7/2](https://huggingface.co/learn/llm-course/en/chapter7/2)


### 4.3 전처리 결과 확인

다음은 전처리 결과를 보기 위한 코드

```python
tokenizer_en = AutoTokenizer.from_pretrained('bert-base-cased', use_fast=True)
encoded_en = tokenize_and_align_labels_wordlevel(example_en, tokenizer_en)
show_alignment(encoded_en, tokenizer_en, id2label_en)
```

결과는 다음과 같음:
```txt
  0 [CLS]            IGN
  1 EU               B-ORG
  2 rejects          O
  3 German           B-MISC
  4 call             O
  5 to               O
  6 boycott          O
  7 British          B-MISC
  8 la               O
  9 ##mb             IGN
 10 .                O
 11 [SEP]            IGN
```


## 5. Case 2: 영어 · character-level BIO → `offset_mapping` 정렬

### 5.1 데이터셋: EMBO/sd-character-level-ner

이 데이터셋은 영어 문장에 대해 **character-level BIO 라벨**을 제공.

* `text` : 원문 문자열
* `labels` : 각 문자(character)에 대응하는 BIO 라벨

참고: [https://huggingface.co/datasets/EMBO/sd-character-level-ner](https://huggingface.co/datasets/EMBO/sd-character-level-ner)

이 경우 라벨의 기준 좌표계는 문자 인덱스이므로,
`word_ids()` 기반 정렬은 논리적으로 적합하지 않다.

```python
ds_char = load_dataset(
    'EMBO/sd-character-level-ner', 
    revision='refs/convert/parquet',
    )
example_char = ds_char['train'][0]
label_names_char = ds_char['train'].features['labels'].feature.names
id2label_char = {i: s for i, s in enumerate(label_names_char)}
```

* 여기엔 메타 데이터가 남아있어서 수동으로 넣을 이유가 없음.

### 5.2 전처리 (character-level BIO 표준 패턴)

```python
def tokenize_and_align_labels_charlevel(example, tokenizer):
    tokenized = tokenizer(
        example["text"],
        return_offsets_mapping=True,
        truncation=True
    )

    offsets = tokenized["offset_mapping"]
    char_labels = example["labels"]

    labels = []
    for start, end in offsets:
        if start == end:
            labels.append(-100)
        elif start < 0 or start >= len(char_labels):
            labels.append(-100)
        else:
            labels.append(char_labels[start])

    tokenized["labels"] = labels
    return tokenized
```

이 방식은 Fast Tokenizer가 제공하는 문자 오프셋 정보를 가장 직접적으로 활용하는 정렬 방식임.

참고: [https://huggingface.co/docs/transformers/main/en/main_classes/tokenizer](https://huggingface.co/docs/transformers/main/en/main_classes/tokenizer)

### 5.3 전치리 결과 확인

다음은 전처리 결과를 보기 위한 코드

```python
tokenizer_char = AutoTokenizer.from_pretrained('bert-base-cased', use_fast=True)
encoded_char = tokenize_and_align_labels_charlevel(example_char, tokenizer_char)
show_alignment(encoded_char, tokenizer_char, id2label_char)
```
결과는 대략 다음과 같음.

```
  0 [CLS]            IGN
  1 (                O
  2 E                O
  3 )                O
  4 Q                O
  5 ##uant           O
  6 ##ification      O
  7 of               O
  8 the              O
  9 number           B-EXP_ASSAY
 10 of               O
 11 cells            O
 12 without          O
 13 γ                B-GENEPROD
 14 -                I-GENEPROD
 15 Tu               I-GENEPROD
 16 ##bul            I-GENEPROD
 17 ##in             I-GENEPROD
 18 at               O
 19 cent             B-SUBCELLULAR
 20 ##ros            I-SUBCELLULAR
 21 ##ome            I-SUBCELLULAR
 22 ##s              I-SUBCELLULAR
 23 (                O

이하 생략.
```

Entity Type에 대한 참고 표임:

| ENTITY_TYPE | 의미 (영문/한글)                    | 등장 예시       | BIO 패턴 |
| ----------- | -----------------------------   | -----------     | ------   |
| EXP_ASSAY   | Experimental Assay (실험/분석 지표) | number          | B        |
| GENEPROD    | Gene Product (유전자 산물, 단백질)   | γ-Tubulin   | B + I* |
| SUBCELLULAR | Subcellular Location (세포내 위치) | centrosomes | B + I* |


## 6. Case 3: 한글 · character-level BIO → `offset_mapping` 정렬

### 6.1 데이터셋: KLUE NER

> **KLUE (Korean Language Understanding Evaluation) NER**는 한국어 NER 벤치마크로,  
> `tokens` 자체가 **character-level (문자 단위 토큰)** 로 제공됨.

* `tokens[i]` : 문자
* `ner_tags[i]` : 해당 문자의 BIO 라벨

KLUE의 NER 데이테셋에는 공백문자도 token임.

참고: [https://huggingface.co/datasets/klue/klue](https://huggingface.co/datasets/klue/klue)

```python
ds_ko = load_dataset('klue', 'ner')
label_names_ko = ds_ko['train'].features['ner_tags'].feature.names
id2label_ko = {i: s for i, s in enumerate(label_names_ko)}
example_ko = ds_ko['train'][0]
```

### 6.2 전처리 (KLUE character-level BIO 표준 패턴)

```python
def tokenize_and_align_labels_klue_charlevel(example, tokenizer):
    text = "".join(example["tokens"]).replace("\xa0", " ")
    char_labels = example["ner_tags"]

    tokenized = tokenizer(
        text,
        return_offsets_mapping=True,
        truncation=True
    )

    labels = []
    for start, end in tokenized["offset_mapping"]:
        if start == end:
            labels.append(-100)
        elif start < 0 or start >= len(char_labels):
            labels.append(-100)
        else:
            labels.append(char_labels[start])

    tokenized["labels"] = labels
    return tokenized
```

KLUE 논문에서는 NER 태깅이 문자 단위(character-level)로 정의되어 있음을 명시한다.

참고: [https://arxiv.org/abs/2105.09680](https://arxiv.org/abs/2105.09680)


### 6.3 전치리 결과 확인


다음은 전처리 결과를 보기 위한 코드
```python
tokenizer_ko = AutoTokenizer.from_pretrained('klue/bert-base', use_fast=True)
encoded_ko = tokenize_and_align_labels_klue(example_ko, tokenizer_ko)
show_alignment(encoded_ko, tokenizer_ko, id2label_ko)
```

결과는 다음과 같음:

```text
  0 [CLS]            IGN
  1 특히               O
  2 영동고속도로           B-LC
  3 강릉               B-LC
  4 방향               O
  5 문                B-LC
  6 ##막              I-LC
  7 ##휴              I-LC
  8 ##게              I-LC
  9 ##소              I-LC
 10 ##에서             O
 11 만                B-LC
 12 ##종              I-LC
 13 ##분              I-LC
 14 ##기              I-LC
 15 ##점              I-LC
 16 ##까              O
 17 ##지              O
 18 5                B-QT
 19 ##㎞              I-QT
 20 구간               O
 21 ##에              O
 22 ##는              O

이하 생략.
```

## 7. 정리

| 데이터셋                        | BIO 기준 좌표계      | Label Alignment 방식 |
| --------------------------- | --------------- | ------------------ |
| CoNLL-2003                  | word-level      | `word_ids()`       |
| EMBO/sd-character-level-ner | character-level | `offset_mapping`   |
| KLUE NER                    | character-level | `offset_mapping`   |

**Label Alignment의 본질은 BIO 여부가 아니라,
BIO 라벨이 정의된 좌표계임.**

이를 사용한 Tokenizer 의 token들에 맞춰야 함.

---

## References

1. Hugging Face Datasets – CoNLL-2003
   [https://huggingface.co/datasets/conll2003](https://huggingface.co/datasets/conll2003)
2. Hugging Face LLM Course – Token Classification & NER
   [https://huggingface.co/learn/llm-course/en/chapter7/2](https://huggingface.co/learn/llm-course/en/chapter7/2)
3. Hugging Face Transformers Documentation – Fast Tokenizer
   [https://huggingface.co/docs/transformers/main/en/main_classes/tokenizer](https://huggingface.co/docs/transformers/main/en/main_classes/tokenizer)
4. EMBO/sd-character-level-ner Dataset
   [https://huggingface.co/datasets/EMBO/sd-character-level-ner](https://huggingface.co/datasets/EMBO/sd-character-level-ner)
5. KLUE Benchmark Dataset
   [https://huggingface.co/datasets/klue/klue](https://huggingface.co/datasets/klue/klue)
6. Park et al., “KLUE: Korean Language Understanding Evaluation”
   [https://arxiv.org/abs/2105.09680](https://arxiv.org/abs/2105.09680)

