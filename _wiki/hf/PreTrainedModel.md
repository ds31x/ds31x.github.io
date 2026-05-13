---
layout  : wiki
title   : Custom Model 요약 및 예제
summary : 
date    : 2026-05-13 22:15:41 +0900
updated : 2026-05-13 22:19:30 +0900
tag     : 
resource: A9/74E043124641DC9C9240D3E6926666
toc     : true
public  : true
parent  : [[/hf/]]
latex   : false
---
* TOC
{:toc}

# Custom Model 요약 및 예제.

## 요약

* `PreTrainedModel` 상속
	* HF의 `save_pretrained()` / `from_pretrained()` 기반 model 저장·로드 사용 목적
* `config_class` 지정
	* model이 사용할 config class를 지정
	* 예: `config_class = MLPRegressionConfig`
* `__init__(config)` 구현
	* 반드시 `config`를 인자로 받음
	* 반드시 `super().__init__(config)` 호출
	* `config` 값으로 PyTorch layer 구성
	* layer 정의 후 `self.post_init()` 호출
* `forward()` 구현
	* processor가 만든 입력 key와 같은 이름의 parameter를 받음
	* 필요하면 `labels`를 받아 loss 계산
	* 가능하면 `ModelOutput` 형태로 반환
* AutoClass 등록
	* class 정의 파일 하단에서 호출
	* 예: `MLPForRegression.register_for_auto_class("AutoModel")`
* 저장 대상 구분
	* model 구조 정보: `config.json`
	* model weight: `model.safetensors`
	* 전처리 정보: processor 설정 파일

## Example

```python
# src/custom_mlp_hf/modeling_mlp.py

from __future__ import annotations

from dataclasses import dataclass
from typing import Optional

import torch
import torch.nn as nn
from transformers import PreTrainedModel
from transformers.modeling_outputs import ModelOutput

from .configuration_mlp import MLPRegressionConfig


@dataclass
class MLPRegressionOutput(ModelOutput):
    """
    MLP regression model의 출력 형식.

    Hugging Face 모델의 forward()는 보통 dict-like output을 반환함.
    ModelOutput을 상속하면 다음 두 방식 모두 사용 가능함.

    예:
        outputs.loss
        outputs.logits

    또는:
        outputs["loss"]
        outputs["logits"]

    이 예제에서는 regression이지만 최종 예측값 이름을 logits로 둠.
    HF 모델들에서 최종 layer의 raw output을 logits라는 이름으로 자주 사용하기 때문.
    """

    # labels가 주어진 경우 계산되는 loss.
    # regression 문제이므로 MSELoss를 사용함.
    loss: Optional[torch.Tensor] = None

    # 모델의 예측값.
    # shape: (batch_size, output_dim)
    logits: Optional[torch.Tensor] = None


class MLPForRegression(PreTrainedModel):
    """
    2개의 hidden layer를 가지는 MLP regression model.

    이 class의 목적:
    - 일반 PyTorch nn.Module을 Hugging Face PreTrainedModel 방식으로 감쌈.
    - config.json의 설정값으로 model architecture를 구성함.
    - save_pretrained() / from_pretrained() 기반 저장·로드를 사용함.
    - AutoModel.from_pretrained(..., trust_remote_code=True) 흐름을 지원함.

    모델 구조:
        input_features
        -> Linear(input_dim, hidden_dim1)
        -> ReLU
        -> Dropout
        -> Linear(hidden_dim1, hidden_dim2)
        -> ReLU
        -> Dropout
        -> Linear(hidden_dim2, output_dim)

    주의:
    - 이 파일은 model architecture와 forward 계산만 담당함.
    - CSV 로드, scaling, tensor 변환 같은 전처리는 processor가 담당함.
    """

    # 이 model이 어떤 config class와 연결되는지 지정함.
    # AutoModel.register() 또는 auto_map 로딩 시 이 관계가 중요함.
    #
    # Hugging Face 문서에서도 custom PreTrainedModel을 AutoClass에 연결하려면
    # model의 config_class가 등록에 사용되는 config class와 일치해야 한다고 설명함.
    config_class = MLPRegressionConfig

    # 모델의 주 입력 이름.
    # Processor의 model_input_names 및 forward() parameter 이름과 맞춤.
    main_input_name = "input_features"

    def __init__(self, config: MLPRegressionConfig):
        """
        Parameters
        ----------
        config:
            MLPRegressionConfig 객체.
            config.json에서 복원된 model architecture 설정을 담고 있음.

        중요:
            PreTrainedModel은 config를 기반으로 초기화됨.
            따라서 반드시 super().__init__(config)를 먼저 호출해야 함.
        """

        # PreTrainedModel 초기화.
        # self.config 저장, HF 공통 저장/로드 기능 연결 등을 수행함.
        super().__init__(config)

        # config에 저장된 architecture 설정값으로 PyTorch layer를 구성함.
        self.net = nn.Sequential(
            nn.Linear(config.input_dim, config.hidden_dim1),
            nn.ReLU(),
            nn.Dropout(config.dropout),
            nn.Linear(config.hidden_dim1, config.hidden_dim2),
            nn.ReLU(),
            nn.Dropout(config.dropout),
            nn.Linear(config.hidden_dim2, config.output_dim),
        )

        # regression 문제이므로 MSE loss 사용.
        self.loss_fn = nn.MSELoss()

        # PreTrainedModel의 weight initialization hook.
        # 모든 layer를 만든 뒤 호출하는 것이 일반적임.
        #
        # 이 메서드는 내부적으로 init_weights() 흐름을 수행함.
        # custom model도 HF의 초기화 흐름에 들어가게 하기 위해 호출함.
        self.post_init()

    def forward(
        self,
        input_features: torch.Tensor,
        labels: Optional[torch.Tensor] = None,
    ) -> MLPRegressionOutput:
        """
        forward computation.

        Parameters
        ----------
        input_features:
            processor가 생성한 model input tensor.
            shape: (batch_size, input_dim)

        labels:
            regression target tensor.
            shape: (batch_size, output_dim)
            labels가 주어지면 loss를 계산함.
            labels가 None이면 prediction만 반환함.

        Returns
        -------
        MLPRegressionOutput:
            loss:
                labels가 주어진 경우 MSE loss.
            logits:
                regression prediction.
        """

        # MLP forward.
        logits = self.net(input_features)

        # labels가 주어진 경우에만 loss 계산.
        # 추론 시에는 labels=None으로 호출 가능함.
        loss = None
        if labels is not None:
            loss = self.loss_fn(logits, labels)

        # HF 스타일 output 반환.
        return MLPRegressionOutput(
            loss=loss,
            logits=logits,
        )


# 이 파일이 import되는 시점에 AutoModel 저장/배포용 metadata를 등록함.
#
# 이 줄이 있어야 model.save_pretrained(save_dir)를 수행할 때
# config.json의 auto_map에 AutoModel 항목이 기록되고,
# custom model이 정의된 이 .py 파일도 저장 directory로 복사될 수 있음.
#
# 이후 로드 시에는 다음 흐름을 사용할 수 있음:
# AutoModel.from_pretrained(save_dir, trust_remote_code=True)
MLPForRegression.register_for_auto_class("AutoModel")

```

