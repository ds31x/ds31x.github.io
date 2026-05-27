---
layout  : wiki
title   : HF-Trainer
summary : PyTorch 학습 루프를 추상화한 Hugging Face Trainer의 A to Z. TrainingArguments, compute_metrics를 사용한 기본 설정부터, Colab에서 Google Drive를 활용해 학습을 재개하고, AdamW와 스케줄러의 동작 원리를 이해하며 단계별 미세 조정을 수행하는 고급 기법까지 설명함.
date    : 2026-02-21 07:35:12 +0900
updated : 2026-02-21 07:35:12 +0900
tag     : huggingface hf transformers trainer accelerate datasets ml ai finetuning colab
resource: 8A/1DD88F028F4A62B0A65D3F48589E61
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# 1. Trainer란 무엇인가

`transformers.Trainer`는 PyTorch 학습 루프를 표준화한 ***고수준 API*** 이다.

직접 학습 루프를 작성하면 보통 다음 요소들을 포함해야 한다.

* forward
* loss 계산
* optimizer step
* scheduler step
* backward
* evaluation loop
* checkpoint 저장
* best 모델 관리
* logging

Trainer는 위 과정을 하나의 일관된 추상화 인터페이스로 구현하게 해 줌.

핵심 개념은 다음과 같음:

1. `TrainingArguments`로 학습 전략을 정의:
   * 주요 hyper-parameters의 설정이 이루어짐.
2. `Trainer`에 model, datasetdict, metric 을 인자로 넘김.
3. `trainer.train()`을 호출.

Trainer를 사용하는 이유는 위의 단계를 일관된 API로 구현할 수 있다는 것도 있지만, 

* 학습 중 현재 모델 및 optimizer의 상태를 저장하고, 
* 이를 다시 불러와 학습을 재개하는 등의 기능을 매우 쉬운 방식으로 구현할 수 있다.
* 이는 사전학습된 모델을 사용하는 것과 매우 유사하게 구현됨.

즉, 다음의 기능을 Trainer를 사용하면 매우 간단하게 사용할 수 있다:

* Colab에서 끊겨도 학습 이어가기
* Best checkpoint 표시
* 재개된 학습과정을 포함하는 하나의 learning curve 생성.

---

# 2. Trainer 기본 사용법

---

## 2-1. 설치

```bash
pip install transformers datasets evaluate accelerate
```

* `datasets`: 데이터셋 로딩/전처리 라이브러리
* `evaluate`: metric (평가지표) 지원 라이브러리
* `accelerate`: 분산/Multi-GPU 학습 지원 라이브러리 


## 2-2. 데이터 준비

Internet Movie Database (IMDb) 텍스트 분류 예제를 사용.

```python
from datasets import load_dataset

raw_datasets = load_dataset("imdb")
```


## 2-3. Tokenizer 설정

```python
from transformers import AutoTokenizer

checkpoint = "bert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(checkpoint)

def preprocess_function(examples):
    return tokenizer(
        examples["text"],
        truncation = True,
        padding    = "max_length",
        max_length = 512,
        return_tensors = "pt",
    )

tokenized_datasets = raw_datasets.map(
    preprocess_function, 
    batched=True,
)
```

## 2-4. 모델 준비

```python
from transformers import AutoModelForSequenceClassification

model = AutoModelForSequenceClassification.from_pretrained(
    checkpoint,
    num_labels=2,
)
```

## 2-5. TrainingArguments 설정

Trainer의 동작을 결정하는 대부분의 선택은 여기서 이루어짐:

```python
from transformers import TrainingArguments

training_args = TrainingArguments(
    output_dir          = "./results",
    eval_strategy       = "epoch",
    learning_rate       = 2e-5,
    per_device_train_batch_size = 16,
    per_device_eval_batch_size  = 16,
    num_train_epochs = 3,
    weight_decay     = 0.01,
    save_strategy    = "epoch",
    load_best_model_at_end = True
)

from transformers import TrainingArguments

training_args = TrainingArguments(
    output_dir="./results",         # 학습 결과물, checkpoint, log 등이 저장될 directory 지정.
    eval_strategy="epoch",          # evaluation을 언제 수행할지 지정. 여기서는 매 epoch마다 평가 수행.
    learning_rate=2e-5,             # optimizer가 parameter를 업데이트할 때 사용할 learning rate 지정.
    per_device_train_batch_size=16, # 각 device(GPU/CPU)당 training에 사용할 batch size 지정.
    per_device_eval_batch_size=16,  # 각 device(GPU/CPU)당 evaluation에 사용할 batch size 지정.
    num_train_epochs=3,             # 전체 training dataset을 몇 번 반복해서 학습할지 지정.
    weight_decay=0.01,              # overfitting을 줄이기 위한 L2 regularization 계수 지정.
    save_strategy="epoch",          # model checkpoint를 언제 저장할지 지정. 여기서는 매 epoch마다 저장.
    load_best_model_at_end=True,    # 학습 종료 후 evaluation 기준으로 가장 좋은 checkpoint를 model에 load.
)
```

이후 다룰 학습 재개 기능을 제대로 사용하기 위해 중요한 것은 다음과 같음:

* `output_dir` : 학습 checkpoint 저장 위치 
  * colab 등을 사용할 경우, 일시적 사용되는 스토리지가 아닌 본인이 언제나 접근 가능한 스토리지를 사용해야 함.
* `save_strategy`
* `load_best_model_at_end`

> `load_best_model_at_end=True`를 쓰려면 보통 `eval_strategy`와 `save_strategy`가 서로 호환되어야 함.
> 여기서는 둘 다 같은 `"epoch"`를 사용하여 적절히 설정.

## 2-6. metric 정의

Trainer에서 `compute_metrics()`는 딕셔너리(`dict`) 형태로 여러 metric을 동시에 반환할 수 있음.

* `compute_metrics()`는 `dict` 객체를 반환함.
* 적절한 key들로 metrics를 넘겨주면 됨.

```python
import evaluate
import numpy as np

metric_acc = evaluate.load("accuracy")
metric_precision = evaluate.load("precision")
metric_recall = evaluate.load("recall")
metric_f1 = evaluate.load("f1")

def compute_metrics(eval_pred):
    logits, labels = eval_pred
    preds = np.argmax(logits, axis=-1)

    acc = metric_acc.compute(
        predictions=preds, 
        references=labels,
    )
    precision = metric_precision.compute(
        predictions=preds, 
        references=labels, 
        average="binary", # multi-class classification의 경우 macro, weighted, micro 중 하나.
    )
    recall = metric_recall.compute(
        predictions=preds, 
        references=labels, 
        average="binary", # multi-class classification의 경우 macro, weighted, micro 중 하나.
    )
    f1 = metric_f1.compute(
        predictions=preds, 
        references=labels, 
        average="binary",
    )

    return {
        "accuracy": acc["accuracy"],
        "precision": precision["precision"],
        "recall": recall["recall"],
        "f1": f1["f1"],
    }
```

앞서 다룬 `TrainingArguments`에서 

* `metric_for_best_model` 을 `"eval_f1"` 으로 설정할 경우,
* F-1 score로 best checkpoint가 선택됨.
* 지정하지 않으면(`metric_for_best_model=None`)
* 기본적으로 `eval_loss` 기준으로 best checkpoint가 선택됨.

`metric_for_best_model`와 관련된 기본 설정은 다음이라고 보면 됨.

```
metric_for_best_model="eval_loss"
greater_is_better=False
```

> imbalanced class 인 경우엔, macro average가 보다 선호됨.

## 2-7. Trainer 생성

```python
from transformers import Trainer

trainer = Trainer(
    model = model,
    args  = training_args,
    train_dataset = tokenized_datasets["train"],
    eval_dataset  = tokenized_datasets["test"],
    tokenizer = tokenizer, # 구버전 방식: 저장 및 padding 처리에 사용됨.
                           # 최신 버전에서는 processing_class 사용 권장.
    compute_metrics = compute_metrics
)
```

전통적으로는 Trainer에 `tokenize`r를 넘겨서 다음 용도로 사용함:

* `trainer.save_model()` 또는 학습 종료 시 model 저장과 함께 `tokenizer` 저장
* padding이 필요한 batch 생성 시 data collator가 tokenizer의 pad token 정보를 참고
* `model`, `tokenizer`, `config`를 같은 directory에 저장해서 나중에 `from_pretrained()`로 다시 load 가능하게 함

최신 코드 스타일은 다음과 같음:

```
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["test"],
    processing_class=tokenizer,
    compute_metrics=compute_metrics,
)
```

## 2-8. 학습

```python
trainer.train()
```

이 한 줄로:

* 학습
* 평가
* checkpoint 저장
* best 모델 추적

이 전부 자동 처리됨.

## 2-9. 평가 및 예측

```python
trainer.evaluate()
trainer.predict(tokenized_datasets["test"])
```

## 2-10. 모델 저장

```python
trainer.save_model("./final_model")
```

---

# 3. Colab에서 중단된 학습 이어가기

앞서 다룬 기본 사용법만으로도 Trainer의 가치는 충분하지만,  
Colab 등을 사용하는 개인 연구자의 경우 중단된 학습 이어가는 기능이야말로 정말 유용함.


## 3-1. 학습 재개 기능의 중요성.

Colab 런타임은 갑자기 종료될 수 있는 단점을 가짐.

만약 앞서 살펴본 것 처럼 `output_dir`을 `./results`로 한 경우,
Colab 에선  `/content` 아래에 있기 때문에
Colab 런타임이 꺼지면 체크포인트도 같이 사라짐.

따라서 반드시:

* `output_dir`를 Google Drive 밑의 디렉토리로 설정해야 한다.

## 3-2. Drive에 output_dir 설정

```python
from google.colab import drive
drive.mount("/content/drive")

OUT_DIR = "/content/drive/MyDrive/hf_runs/trainer_resume_demo"
```

---

## 3-3. Resume 가능하도록 TrainingArguments 수정

중단 대비를 위해 `step` 기반 저장을 사용.

```python
training_args = TrainingArguments(
    output_dir=OUT_DIR,

    save_strategy="steps",
    save_steps=200,
    save_total_limit=2,

    eval_strategy="steps",
    eval_steps=200,

    logging_strategy="steps",
    logging_steps=50,

    load_best_model_at_end=True,
    metric_for_best_model="eval_f1",
    greater_is_better=True,

    report_to="none",
)
```

## 3-4. 학습 재개 방법

런타임이 꺼진 뒤 다음을 통해 학습을 이어서 재개:

1. Drive mount
2. 데이터/모델/Trainer 재구성
3. 아래 실행

```python
trainer.train(
    resume_from_checkpoint=True,
)
```

Trainer는 `output_dir` 안의 마지막 checkpoint를 자동으로 찾음.

## 3-5. JSONL 기반 Curve Logger Callback

재개된 학습을 이전의 학습 로그와 이어서 learning curve를 만들기 위해선 다음과 같은 callback 이 필요:

```python
import os
import json
from transformers import TrainerCallback

class CurveLoggerCallback(TrainerCallback):

    def __init__(self, out_dir, filename="learning_curve.jsonl", stage="stage1"):
        self.path = os.path.join(out_dir, filename)
        self.stage = stage
        os.makedirs(out_dir, exist_ok=True)

    def _append(self, record):
        with open(self.path, "a", encoding="utf-8") as f:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")

    def _base(self, state):
        return {
            "stage": self.stage,
            "step": int(state.global_step),
            "epoch": float(state.epoch) if state.epoch else None,
        }

    def on_log(self, args, state, control, logs=None, **kwargs):
        if logs:
            rec = self._base(state)
            rec.update(logs)
            self._append(rec)

    def on_evaluate(self, args, state, control, metrics=None, **kwargs):
        if metrics:
            rec = self._base(state)
            rec.update(metrics)
            self._append(rec)
```

Trainer 생성 시 `callbacks` 에 넘겨줌:

```python
curve_logger = CurveLoggerCallback(
    OUT_DIR, 
    stage="stage1",
)

trainer = Trainer(
    ...
    callbacks=[curve_logger]
)
```

## 3-6. 러닝커브 그리기

```python
import pandas as pd
import matplotlib.pyplot as plt
import json, os

def load_curve(out_dir):
    records = []
    with open(os.path.join(out_dir, "learning_curve.jsonl")) as f:
        for line in f:
            records.append(json.loads(line))
    df = pd.DataFrame(records)
    if "step" in df.columns:
        df = df.sort_values("step")
        df = df.groupby("step").last().reset_index()
    return df

df = load_curve(OUT_DIR)

fig, axes = plt.subplots(4, 1, figsize=(10, 12), sharex=True)

if "loss" in df.columns:
    axes[0].plot(df["step"], df["loss"], label="train_loss", color="tab:blue")
    axes[0].set_ylabel("Loss")
    axes[0].legend()
    axes[0].grid(True)

if "eval_accuracy" in df.columns:
    axes[1].plot(df["step"], df["eval_accuracy"], label="eval_accuracy", color="tab:orange")
    axes[1].set_ylabel("Accuracy")
    axes[1].legend()
    axes[1].grid(True)

if "eval_precision" in df.columns:
    axes[2].plot(df["step"], df["eval_precision"], label="eval_precision", color="tab:green")
    axes[2].set_ylabel("Precision")
    axes[2].legend()
    axes[2].grid(True)

if "eval_f1" in df.columns:
    axes[3].plot(df["step"], df["eval_f1"], label="eval_f1", color="tab:red")
    axes[3].set_ylabel("F1")
    axes[3].legend()
    axes[3].grid(True)

axes[3].set_xlabel("Step")
plt.tight_layout()
plt.show()
```

step이 끊기지 않고 증가하면
resume이 제대로 된 것임.

## 3-7. Best Checkpoint 표시

Trainer는 `trainer_state.json`에 best 정보를 남기도록 설계되어 있음.

```python
import re, json, os

with open(os.path.join(OUT_DIR, "trainer_state.json")) as f:
    state = json.load(f)

best_ckpt = state.get("best_model_checkpoint")
best_step = int(re.search(r"checkpoint-(\d+)", best_ckpt).group(1))
```

이를 그래프에 표시:

```python
plt.axvline(best_step, linestyle="--", label="best_checkpoint")
```

# 4. Fine Tuning의 경우

Fine Tuning의 경우 다양한 stage를 거침:

* Stage1 : head 부분만 학습.
* Stage2 : backbone의 downstream layer를 일부 열어서 학습.

이 경우에도 하나의 OUT_DIR을 유지하면,
learning curve를 이어갈 수 있음.


단, Stage1에서 head만 학습하고
Stage2에서 backbone을 열어 학습하는 구조에서
가장 많이 헷갈리는 부분이 **AdamW의 state 동작**임.

> HF의 Trainer의 기본 optimizer는 `AdamW`임.    
> 이에 대해 보다 자세한 내용은 다음을 참고[Adaptive Moment Estimation with Weight decay (AdamW)](https://dsaint31.me/mkdocs_site/ML/ch09/op_adamw/)

여기서 정확히 정리해야 할 것은 세 가지임.

1. freeze는 optimizer에서 파라미터를 제거하는 동작이 아님
2. HF의 Trainer가 기본으로 사용하는 AdamW state는 lazy하게 생성되는 구조임
3. resume은 `optimizer.state`까지 그대로 복원하는 동작임

## 4-1 Stage1 - backbone freeze의 실제 의미

일반적으로 freeze는 다음 방식임.

```python
for p in model.backbone.parameters():
    p.requires_grad = False
```

이 동작의 의미는 다음임.

* autograd가 해당 파라미터의 gradient를 계산하지 않음
* 그러나 `optimizer.param_groups`에는 그대로 존재함
* optimizer에서 자동 제거되지 않음

즉,

> freeze는 "optimizer에서 제거"가 아니라
> "gradient 생성 중단"에 불과한 동작임.

## 4-2 AdamW의 state 생성 방식

* 참고: [Adaptive Moment Estimation with Weight decay (AdamW)](https://dsaint31.me/mkdocs_site/ML/ch09/op_adamw/)

AdamW는 각 파라미터마다 다음 state를 가짐.

* `exp_avg`
* `exp_avg_sq`
* `step`

중요한 점은 다음임.

> AdamW는 gradient가 처음 등장하는 시점에 `state`를 생성함.  
> 이를 lazy initialization 구조라 부름

Stage1에서 backbone이 freeze 상태라면:

* gradient가 발생하지 않음
* 따라서 `exp_avg` / `exp_avg_sq` 생성되지 않음
* `optimizer.state`에는 아직 해당 파라미터 entry가 없음

Stage2에서 unfreeze하면:

* 첫 backward에서 gradient 생성됨
* 그 시점에 AdamW `state`가 새로 초기화됨

즉,

> backbone은 "optimizer에 등록은 되어 있었지만"
> state는 Stage2 첫 step에서 생성되는 구조임.


## 4-3 resume을 사용하는 경우의 실제 의미

`trainer.train(resume_from_checkpoint=True)`는 다음을 복원함.

* 모델 가중치
* `optimizer.state`
* `param_groups`
* scheduler 상태
* `global_step`

Stage1이 끝난 상태에서 resume을 하면:

* head는 이미 `exp_avg` / `exp_avg_sq`가 존재
* backbone은 `state`가 없는 상태 그대로 복원

Stage2에서 unfreeze하면:

* backbone은 새로 `state` 초기화
* head는 누적된 `state` 유지

결과:

* head와 backbone의 업데이트 동역학이 다름
* head는 stage1의 `momentum` 누적 상태
* backbone은 초기 상태

이것이 staged fine-tuning에서 자연스럽게 발생하는 구조임.


## 4-4 Stage2에서 learning rate를 낮췄는데 이상하게 동작하는 이유

resume을 사용하면 scheduler state도 복원됨.

예를 들어 cosine scheduler 사용 시:

* Stage1이 이미 step 5000까지 진행한 상태라면,
* Stage2에서 `learning_rate=5e-6` 와 같이 새로 설정하더라도
* 하지만 scheduler는 "step 5000 위치"에서 `lr` 계산하게 됨.

즉,

> Stage2가 "새 곡선 시작"이 아니라
> 기존 곡선의 중간 지점에서 계속 진행되는 구조임.

쉽게 말하면, 
`learning_rate`만 변경한다고
스케줄이 초기화되는 것이 아님.

## 4-5 Stage2를 어떻게 설계해야 하는가

### Option 1 — (완전) 연속형 Stage (resume 유지)

특징:

* `global_step` 이어짐
* scheduler 진행 이어짐
* learning curve 한 곡선 유지
* Stage2는 **정책** 만 변경

적합한 경우:

* Stage2가 Stage1의 연장선인 경우
* scheduler를 다시 시작할 필요가 없는 경우

### Option 2 — 분리형 Stage (가중치만 로드)

Stage2는 완전히 새로운 optimizer dynamics

* 새 warmup 가능
* 새 scheduler 가능
* 논문용 실험 분리 명확
* 러닝커브는 stitching으로 통합 가능

```python
import os
import json
import re
from transformers import AutoModelForSequenceClassification
from transformers import Trainer

# Stage1
OUT_DIR_STAGE1 = "./runs/stage1"

curve_logger1 = CurveLoggerCallback(
    OUT_DIR_STAGE1,
    stage="stage1"
)

trainer1 = Trainer(
    model=model,
    args=training_args_stage1,
    train_dataset=train_ds,
    eval_dataset=val_ds,
    compute_metrics=compute_metrics,
    callbacks=[curve_logger1],
)

trainer1.train()

# Stage1 종료 시점에 다음이 존재함:
# ./runs/stage1/checkpoint-xxxx/ 존재
# learning_curve.jsonl 존재
# trainer_state.json 존재

# Stage1의 Best Checkpoint 찾기
def get_best_checkpoint(out_dir):
    state_path = os.path.join(out_dir, "trainer_state.json")
    with open(state_path, "r") as f:
        state = json.load(f)
    return state["best_model_checkpoint"]

best_ckpt = get_best_checkpoint(OUT_DIR_STAGE1)
print("Best checkpoint:", best_ckpt)

# 모델 가중치만 로드
model_stage2 = AutoModelForSequenceClassification.from_pretrained(
    best_ckpt
)

# Backbone Unfreeze
# 필요하다면 일부 layer만 열어도 됨.
for p in model_stage2.backbone.parameters():
    p.requires_grad = True


from transformers import TrainingArguments

# Stage2용 TrainingArguments
# 완전히 새 optimizer / scheduler를 생성
OUT_DIR_STAGE2 = "./runs/stage2"
training_args_stage2 = TrainingArguments(
    output_dir=OUT_DIR_STAGE2,
    per_device_train_batch_size=32,
    per_device_eval_batch_size=32,
    num_train_epochs=5,
    learning_rate=5e-6,              # 낮은 lr
    weight_decay=0.01,
    evaluation_strategy="steps",
    save_strategy="steps",
    logging_steps=50,
    load_best_model_at_end=True,
    metric_for_best_model="eval_f1",
    greater_is_better=True,
)

# Curve Logger 추가
curve_logger2 = CurveLoggerCallback(
    OUT_DIR_STAGE2,
    stage="stage2"
)

# Trainer 구성
trainer2 = Trainer(
    model=model_stage2,
    args=training_args_stage2,
    train_dataset=train_ds,
    eval_dataset=val_ds,
    compute_metrics=compute_metrics,
    callbacks=[curve_logger2],
)

trainer2.train()
```

여기서 중요한 점:
* `resume_from_checkpoint` 사용하지 않음
* `global_step = 0` 부터 시작
* optimizer state 새로 생성
* scheduler 새로 생성

이 구조가 완전 분리형 Stage2임.

적합한 경우:

* Stage2에 새 warmup 필요
* 완전히 다른 `lr` schedule 필요
* optimizer 정책이 크게 바뀌는 경우

## 4-6 분리형 Stage에서도 러닝커브를 한 곡선으로 만드는 방법

`global_step`이 0부터 시작하므로
***step stitching*** 필요함.

다음은 이를 수행하는 간단한 코드임:

```python
import pandas as pd
import json

def load_curve(path):
    rows = []
    with open(path) as f:
        for line in f:
            rows.append(json.loads(line))
    df = pd.DataFrame(rows).sort_values("step")
    df = df.groupby("step").last().reset_index()
    return df

df1 = load_curve(f"{OUT_DIR_STAGE1}/learning_curve.jsonl")
df2 = load_curve(f"{OUT_DIR_STAGE2}/learning_curve.jsonl")

offset = df1["step"].max()

df1["step_global"] = df1["step"]
df2["step_global"] = df2["step"] + offset

df_all = pd.concat([df1, df2]).sort_values("step_global")

```

이후 `step_global` 기준 plot 수행하면
두 단계가 하나의 곡선처럼 연결됨.

## 4-7 최종 핵심 정리

* freeze는 optimizer에서 제거가 아님
* AdamW state는 gradient 발생 시 생성됨
* resume은 optimizer.state와 scheduler까지 복원함
* Stage2에서 lr만 바꿔도 scheduler는 재시작되지 않음
* 연속형과 분리형 중 의도에 맞는 설계 선택 필요함

좀 더 고급 테크닉을 사용한다면,

* backbone/head를 서로 다른 `param_group`으로 관리하는 구조를 만들고,
* Stage2에서 `param_group`을 재구성하는 안전한 방법이 있음.
* 동시에, 실제 lr을 매 step 기록해 scheduler 동작을 검증하는 코드도 가능함.
