---
layout  : wiki
title   : HF Start
summary : 
date    : 2026-01-22 23:12:26 +0900
updated : 2026-01-23 00:10:36 +0900
tag     : hf
resource: D8/AA2903BCE24DD3B682DC77B0D5EC30
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# Hugging Face 시작하기

hugging face 를 시작하는 방법으로 `hf auth login` 과 `hf auth logout`을 살펴본다.

Hugging Face Hub에 접근하기 위해 개인 Access Token을 로컬에 저장하여 이후 명령과 라이브러리에서 authentication을 자동으로 사용할 수 있음.
이는 비공개 모델과 데이터셋 접근, 모델 업로드·업데이트, Spaces 배포, CLI·라이브러리에서의 인증 자동 사용가능해짐.

> 공개모델만 사용하는 경우엔, 굳이 필요하지않긴 함.

## huggingface.co 상단 메뉴 설명:

주로 Hub들이 놓임: 모델·데이터·앱을 저장하고 공유하는 서비스

1. Models
	* Models Hub
	* 사전학습 및 파인튜닝된 모델 저장소
	* 모델 체크포인트 + 설정 + 모델 카드 제공
	* PyTorch, TF, JAX 지원.
2. Datasets
	* Datasets Hub
	* 표준 데이터셋 저장소
	* 버전관리, split, streaming 지원.
	* `datasets` 라이브러리와 동일한 인터페이스 기준
3. Spaces
	* Spaces
	* 모델 데모 및 앱 배포 공간
	* Gradio, Streamlit 기반 웹 앱
	* interface + UI 통합 배포.
4. Docs
	* Hub 아님
	* 라이브러리 문서 및 사용 가이드 모음

## 주요 Library

모델 구현, 전처리, 학습, 추론을 담당하는 라이브러리

1. Transformers (Model Layer)
	* Transformer 모델이 중심이나,
	* CNN, RNN, Hybrid 모델들도 포함되어 있음.
	* Trainder도 포함됨.
2. Processor (Preprocessing Layer)
	* Tokenizer : text → ids
	* ImageProcessor : image → tensor 
	* FeatureExtractor : audio → feature 
3. Accelerate (Training/Execution Infra layer)
	* 학습 및 실행 환경 추상화
	* device, distributed, mixed precision 처리
	* single GPU / multi GPU / TPU / multi-node 대응
	* Transformers의 Trainer 및 사용자 정의 학습 루프에서 공통 사용


## WSL 에서 로그인. 

```
conda create -n hf_env python
conda activate hf_env
conda install pip
pip install transformers datasets huggingface_hub torch
```

`transformers` 는 라이브러리로 백앤드가 필요하므로 다음으로 확인:

```
python - << 'EOF'
import transformers
print("torch:", transformers.utils.is_torch_available())
print("tf   :", transformers.utils.is_tf_available())
print("flax :", transformers.utils.is_flax_available())
EOF
```

자기 고유 장비라면 `hf auth login` 을 통해 로그인하고 다음으로 실행을 확인.
```
from transformers import pipeline
clf = pipeline("sentiment-analysis")
print(clf("Hugging Face works well on WSL.")
```

공용장비라면, 잠시 사용할 AccessToken을 생성하고 다음으로 처리하는 것을 권장

`python` interactive shell을 실행하고 다음과 같이 실행:

```
>>> import os
>>> os.environ["HF_TOKEN"] = "hf_xxxxxxxxxxxxxxxxx"
>>> from transformers import pipeline
>>> clf = pipeline("sentiment-analysis")
>>> print(clf("Hugging Face works well on WSL.")
>>> exit()
```

* `export HF_TOKEN=hfxxx` 등을 사용시 history에 남을 수 있음.
* 때문에 일시적으로 사용하는 Access Token 만 사용할 것 (공용PC에서 작업 종료후 해당 Toekn 제거)
* 다중 session 등의 문제가 있으므로 위와 같이 Python session에서만 사용하고 종료하는 형태로 이용할 것.

사실 공용장비라도 `hf auth login`을 해도 되긴 함(끝나고 다음의 처리를 제대로 해준다면...)

```
hf auth logout || true
rm -rf ~/.huggingface
rm -rf ~/.cache/huggingface*
conda deactivate
```

이후 다음으로 로그아웃 여부 확인할 것:
```
$ hf auth whoami
Not logged in
```

## Google Colab 에서 로그인.

```
!hf auth login
```
* 아니면 ssh로 처리해도 됨: `$ hf auth login`
* 이후 Access Token을 입력.
* git credential로 지정할지 물어보는데 `git config --global credential.helper cache` 등을 사용할 거라면 `y`
* 위 명령어 전에 다음의 명령어 실행 권장: `!git config --global credential.helper "cache --timeout=10800"`

이후 `/root/.cache/huggingface/token` 에 Access Token이 저장됨.

백앤드 확인:

```
import transformers
print("torch:", transformers.utils.is_torch_available())
print("tf   :", transformers.utils.is_tf_available())
print("flax :", transformers.utils.is_flax_available())
```

다음의 코드쉘을 작성하여 실행 (`hf_xxxx`인 Access Token은 공유되선 안됨.)

```
import os
os.environ["HF_TOKEN"] = "hf_xxxxxxxxxxxxxxxxx"

from transformers import pipeline
clf = pipeline("sentiment-analysis")
print(clf("Hugging Face works well on Colab."))

del os.environ["HF_TOKEN"]
```

## 참고

일반적으로 다음의 경고가 발생가능함:

* `No model was supplied` : pipeline에 `model`을 지정 안한 경우로 자동 선택될 때 발생.
* revision 미지정 : 재현성 문제 경고
* `HF_TOKEN` 없음 : 공개 모델이라도 인증 권장 안내
* `torchao` + `triton` 없음 : CPU Colab에서는 정상, 제거 불필요

다음과 같이 하면 warning이 최소화됨

```
clf = pipeline(
    task="sentiment-analysis",
    model="distilbert/distilbert-base-uncased-finetuned-sst-2-english",
    revision="714eb0f",        # 재현성 극대화.
)
```

