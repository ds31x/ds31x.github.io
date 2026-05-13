---
layout  : wiki
title   : PretrainedConfig 요약 및 예제
summary : 
date    : 2026-05-13 21:54:17 +0900
updated : 2026-05-13 22:14:02 +0900
tag     : 
resource: C6/DA82686F024743A7CC748852112B78
toc     : true
public  : true
parent  : [[/hf]]
latex   : false
---
* TOC
{:toc}

# Custom Config 요약 및 예제.

* `PretrainedConfig` 상속
	* custom config는 `transformers.PretrainedConfig`를 상속하여 구현.
	* `save_pretrained()` / `from_pretrained()` 기반의 config.json 저장·로드 구조를 사용하기 위함.
* `model_type` 지정
	* config class의 class attribute로 지정.
	* `AutoConfig`가 config.json을 읽고 어떤 config class를 사용할지 판단하는 식별자.
	* `model_type = "custom-mlp-regression"`
* `__init__()` 구현
	* model architecture에 필요한 custom field를 parameter로 받음.
	* 예: `input_dim`, `hidden_dim`, `num_labels`, `dropout` 등.
	* 받은 값을 같은 이름의 instance attribute로 저장.
	* `**kwargs` 처리
		* `__init__()`은 반드시 `**kwargs`를 받을 수 있게 구성.
		* HF 공통 config field와 추가 field를 처리하기 위함.
		* 반드시 `super().__init__(**kwargs)`를 호출하여 PretrainedConfig 기본 초기화 수행.
* 저장 대상 구분
	* model 구조에 필요한 값만 config에 저장.
	* 참고로 전처리 정보는 processor 설정 파일(`preprocessor_config.json` 등)에 저장됨.
	* 학습 hyperparameter는 보통 training script나 training args에서 관리.
* AutoClass 등록
	* custom config를 `AutoConfig`로 로드하려면 등록 필요.식
	* 배포 가능한 형태로 저장하려면 class 정의 파일 하단에서 호출.
	* `CustomConfig.register_for_auto_class()`

## Processor 설정 관련

* HF의 processor 계열에서는 model 쪽의 `PretrainedConfig`와 달리, 
* 보통 processor 자체가 자기 설정값을 instance attribute로 가지고 있으면서,
* `save_pretrained()`를 통해 별도의 `preprocessor_config.json` 파일로 저장

`ProcessorMixin`은 processor class에 `save_pretrained()`, `from_pretrained()`, `push_to_hub()` 같은 저장·로드 기능을 제공

* 참고: [[/hf/ProcessorMixin]]


## Example

```python
# src/custom_mlp_hf/configuration_mlp.py

from transformers import PretrainedConfig


class MLPRegressionConfig(PretrainedConfig):
    """
    MLP regression model의 구조 정보를 저장하는 Custom Config.

    이 class의 역할:
    - model architecture에 필요한 설정값을 저장함.
    - save_pretrained() 호출 시 config.json으로 저장됨.
    - from_pretrained() 호출 시 config.json에서 다시 복원됨.
    - AutoConfig / AutoModel이 이 config를 기준으로 custom model class를 찾을 수 있게 함.

    주의:
    - processor의 전처리 정보는 여기에 넣지 않음.
    - 이 config는 model을 어떻게 만들지에 대한 정보만 가짐.
    """

    # AutoConfig가 config.json을 읽을 때 어떤 config class를 사용할지 식별하는 이름.
    # config.json 안에 "model_type": "custom-mlp-regression" 형태로 저장됨.
    #
    # AutoClass를 사용할 계획이면 model_type을 반드시 명확히 지정하는 것이 좋음.
    model_type = "custom-mlp-regression"

    def __init__(
        self,
        input_dim=3,
        hidden_dim1=32,
        hidden_dim2=16,
        output_dim=1,
        dropout=0.0,
        **kwargs,
    ):
        """
        Parameters
        ----------
        input_dim:
            입력 feature의 개수.
            예: x1, x2, x3을 사용하면 input_dim=3.

        hidden_dim1:
            첫 번째 hidden layer의 neuron 수.

        hidden_dim2:
            두 번째 hidden layer의 neuron 수.

        output_dim:
            출력 차원.
            regression에서 scalar y를 예측하면 output_dim=1.

        dropout:
            hidden layer 뒤에 적용할 dropout 비율.

        kwargs:
            PretrainedConfig가 내부적으로 사용하는 추가 설정값을 받기 위한 인자.
            from_pretrained()로 config를 다시 읽을 때,
            config.json 안의 추가 field들이 여기를 통해 들어올 수 있음.

        중요:
            Custom Config는 **kwargs를 받고,
            반드시 super().__init__(**kwargs)로 넘겨야 함.
        """

        # PretrainedConfig의 기본 초기화 수행.
        # name_or_path, transformers_version, torch_dtype 등
        # HF 공통 config field를 처리하기 위해 필요함.
        super().__init__(**kwargs)

        # 아래 값들은 model architecture를 재구성하는 데 필요한 custom field임.
        # save_pretrained() 호출 시 config.json에 저장됨.
        self.input_dim = input_dim
        self.hidden_dim1 = hidden_dim1
        self.hidden_dim2 = hidden_dim2
        self.output_dim = output_dim
        self.dropout = dropout


# 이 파일이 import되는 시점에 AutoConfig 저장/배포용 metadata를 등록함.
#
# 이 줄이 있어야 config.save_pretrained() 또는 model.save_pretrained() 시
# config.json의 auto_map에 AutoConfig 항목이 기록될 수 있음.
#
# 이후 다음과 같은 방식으로 custom config를 자동 로드할 수 있음:
# AutoConfig.from_pretrained(save_dir, trust_remote_code=True)
MLPRegressionConfig.register_for_auto_class()
```
