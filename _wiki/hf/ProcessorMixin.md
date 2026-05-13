---
layout  : wiki
title   : ProcessorMixin 요약 및 예제
summary : 
date    : 2026-05-13 21:13:56 +0900
updated : 2026-05-13 22:07:21 +0900
tag     : 
resource: A4/DBF56E35CD4B3C8B41EBF424FAC397
toc     : true
public  : true
parent  : [[/hf/ProcessorMixin]]
latex   : false
---
* TOC
{:toc}

# ProcessorMixin

> `ProcessorMixin` 은  
> 
> * Hugging Face에서 custom processor를 만들 때 사용하는 mixin class임.
> * MultiModal Model 에서 tokenizer와 imager processor를 같이 묶을 때 사용됨.

주된 역할은 다음과 같음:

* processor가 `save_pretrained()`와 `from_pretrained()` 기반의 저장·로드 방식을 따르도록 해주는 것임.
* processor ***내부의 설정값을 config 파일로 저장하고, 다시 로드*** 할 수 있게 해줌.
* `AutoProcessor`와 연결하면 `AutoProcessor.from_pretrained()`로 custom processor를 복원할 수 있음.

즉, 

* raw input을 model input으로 바꾸는 로직은 사용자가 구현하고, 
* 저장·로드와 AutoClass 연동은 `ProcessorMixin`이 담당하는 구조임.

## 주의 사항:

* `__init__(self,...)` 구현
	* `ProcessorMixin` 기반 class에서는 저장되어야 하는 **설정값** 을 
	* JSON 직렬화 가능한 self attribute로 유지하고, 
	* `from_pretrained()`로 복원될 수 있도록 
	* 해당 값을 `__init__()`에서도 받을 수 있게 두는 것이 좋음.
	* 저장된 설정으로 같은 processor 객체를 다시 구성할 수 있어야 함.
* `__call__(self,)` 구현
	* raw input을 model이 받을 수 있는 입력 `dict`로 변환.
	* 필요한 전처리 수행이 여기서 이루어짐.
	* 반환 `dict`의 key는 model의 `forward()` parameter 이름과 일치해야 함.
	* 예: `'input_ids'`, '`attention_mask'`, `'pixel_values'`, `'input_features'`, `'labels'`
* 필요한 경우 상태 계산 메서드 구현
	* 학습 데이터나 기준 데이터로부터 전처리 상태를 계산해야 한다면 `fit()` 같은 메서드를 추가.
	* 계산된 상태값 등은 반드시 JSON 직렬화 가능한 attribute로 저장.
	* 예: 평균, 표준편차, label mapping, vocabulary 정보 등
* `model_input_names` 클래스 속성 지정
	* processor가 생성하는 주요 model input 이름을 지정.
	* model의 `forward()` 입력 이름과 일치시킴. 
* `register_for_auto_class()` 호출
	* `AutoProcessor`로 로드하려면 class 정의 파일 하단에서 호출.
	* `import` 시점에 등록되도록 둠.

## Simple Example:

다음은 Regression Model을 위한 Standardization을 수행하는 전처리기임.

내부에서 `scikit-learn` 을 사용하여, 일종의 Wrapper로 간단히 작성함.

* csv 파일을 읽어들이고,
* feature column만 선택하여
* PyTorch 의 `Tensor` 객체를 넘김.
* `ProcessorMixin`의 기본 직렬화를 그대로 사용하려면 `self.xxx` (instance attribute) 에는 ***JSON 직렬화 가능한 flat attribute만 사용** 해야함.
* scikit-learn의 `StandardScaler` 객체 자체를 `self.scaler`로 저장해선 안됨.
* `StandardScaler`의 fitted state인 `mean_`, `scale_`, `var_`만 `list` 형태로 저장함.


```python

# src/custom_mlp_hf/processing_mlp.py

from __future__ import annotations

import numpy as np
import pandas as pd
import torch
from sklearn.preprocessing import StandardScaler
from transformers import ProcessorMixin


class MLPRegressionProcessor(ProcessorMixin):
    """
    CSV tabular regression용 Custom Processor.
    """
    
    # ProcessorMixin 를 주로 사용하는
    # 기본 composite processor (tokenizer, image_processor 등을 묶음)에서 
    # 사용하는 class attributes 임.
    # 이 예제는 tokenizer/image_processor를 묶는 processor가 아니라,
    # tabular CSV -> tensor 변환 processor이기 때문에 
    # 사용하지 않음.
    attributes          = []
    optional_attributes = []

    # 모델의 forward()가 받을 입력 key 이름.
    # modeling_mlp.py의 
    # forward(input_features=...)와 이름을 맞추기 위한 값.
    model_input_names = ["input_features"]

    def __init__(
        self,
        feature_columns,
        label_column="y",
        scaler_mean=None,
        scaler_scale=None,
        scaler_var=None,
        n_features_in=None,
        **kwargs,
    ):
        """
        ProcessorMixin 기반 class에서는 
	저장되어야 하는 값을
        __init__ parameter 로 입력받고, 
	같은 이름의 self attribute로 저장하는 것이 일반적임.
	
	__init__ 의 parameter 인 경우,
	from_pretrained() 에서 저장된 값을 복원할 수 있음.

        Parameters
        ----------
        feature_columns:
            입력 feature로 사용할 CSV column 이름 목록.
            예: ["x1", "x2", "x3"]

        label_column:
            regression target으로 사용할 CSV column 이름.
            예: "y"

        scaler_mean:
            StandardScaler.fit() 이후의 mean_ 값.
            JSON 직렬화를 위해 list로 저장함.

        scaler_scale:
            StandardScaler.fit() 이후의 scale_ 값.
            JSON 직렬화를 위해 list로 저장함.

        scaler_var:
            StandardScaler.fit() 이후의 var_ 값.
            JSON 직렬화를 위해 list로 저장함.

        n_features_in:
            StandardScaler.fit() 이후의 n_features_in_ 값.
            int로 저장함.

        kwargs:
            ProcessorMixin 내부에서 사용하는 
	    추가 keyword argument를 받기 위한 자리.
        """

        # ProcessorMixin 내부 초기화를 위해 호출함.
        # save_pretrained(), from_pretrained(), 
	# register_for_auto_class() 흐름과 연결됨.
        super().__init__(**kwargs)

        # CSV에서 어떤 column을 입력 feature로 볼지 저장.
        # list[str] 형태이므로 JSON 직렬화 가능함.
        self.feature_columns = feature_columns

        # CSV에서 어떤 column을 label로 볼지 저장.
        # str 형태이므로 JSON 직렬화 가능함.
        self.label_column = label_column

        # 아래 값들은 StandardScaler의 fitted state임.
        # StandardScaler 객체 자체는 
	# JSON 직렬화 대상이 아니므로 저장하지 않음.
        # 대신 fitted state만 list/int로 저장함.
        self.scaler_mean   = scaler_mean
        self.scaler_scale  = scaler_scale
        self.scaler_var    = scaler_var
        self.n_features_in = n_features_in

    def fit(self, csv_path):
        """
        train CSV를 기준으로 StandardScaler의 fit을 수행.

        주의:
        - 이 메서드는 StandardScaler 객체 자체를 저장하지 않고,
        - fit 결과로 생긴 mean_, scale_, var_만 self attribute에 저장함.
        """

        # CSV를 DataFrame으로 읽음.
        df = pd.read_csv(csv_path)

        # 지정된 feature column만 선택함.
        # to_numpy()를 사용하면 StandardScaler가 feature name에 묶이지 않음.
        # column 이름은 processor가 관리하고,
        # scaler는 순수 numeric array에만 적용되게 함.
        x = df[self.feature_columns].to_numpy(dtype=np.float32)

        # scikit-learn의 StandardScaler를 사용함.
        # standardization 을 직접 구현하지 않음.
        scaler = StandardScaler()
        scaler.fit(x)

        # ProcessorMixin의 기본 저장 로직이 처리할 수 있도록
        # JSON 직렬화 가능한 list/int로 변환하여 저장함.
        self.scaler_mean   = scaler.mean_.tolist()
        self.scaler_scale  = scaler.scale_.tolist()
        self.scaler_var    = scaler.var_.tolist()
        self.n_features_in = int(scaler.n_features_in_)

        return self

    def _build_scaler(self):
        """
        저장된 fitted state로 StandardScaler 객체를 복원하는 헬퍼 메서드.

        이 helper는 저장 대상이 아님.
        self.__dict__에 들어가는 값이 아니므로 
	processor config에 저장되지 않음.

        왜 필요한가?
        - __call__()에서 StandardScaler.transform()을 쓰기 위함.
        - standardization은 직접 구현하지 않고 scikit-learn에 맡기기 위함.
        - 동시에 self.scaler 같은 비직렬화 객체를 processor attribute로 두지 않기 위함.
        """

        # fit()이 호출되지 않은 상태에서 
	# __call__()을 하면 전처리에 필요한 통계량이 없음.
        if self.scaler_mean is None or self.scaler_scale is None:
            raise ValueError(
                "Processor is not fitted. "
                "Call processor.fit(train_csv_path) before using it."
            )

        scaler = StandardScaler()

        # StandardScaler가 transform()을 수행할 수 있도록
        # fit 이후 생성되는 주요 attribute를 복원함.
        scaler.mean_          = np.asarray(self.scaler_mean, dtype=np.float64)
        scaler.scale_         = np.asarray(self.scaler_scale, dtype=np.float64)
        scaler.var_           = np.asarray(self.scaler_var, dtype=np.float64)
        scaler.n_features_in_ = int(self.n_features_in)

        return scaler

    def __call__(self, csv_path, return_labels=True):
        """
        CSV 파일을 model input dict로 변환함.

        반환 dict의 key는 
	model.forward()의 parameter 이름과 맞아야 함.

        Returns
        -------
        batch:
            {
                "input_features": torch.FloatTensor,
                "labels"        : torch.FloatTensor  # return_labels=True인 경우
            }
        """

        # CSV를 읽음.
        df = pd.read_csv(csv_path)

        # feature column만 선택하고 numeric array로 변환함.
        x = df[self.feature_columns].to_numpy(dtype=np.float32)

        # fit()에서 저장해둔 scaler state로 StandardScaler를 복원함.
        scaler = self._build_scaler()

        # scikit-learn의 transform()을 사용하여 standardization 수행.
        x = scaler.transform(x)

        # 모델 입력 dict 생성.
        # MLPForRegression.forward(input_features=...)와 key 이름이 일치해야 함.
        batch = {
            "input_features": torch.tensor(
                x,
                dtype=torch.float32,
            ),
        }

        # 학습/평가에서는 labels가 필요함.
        # 추론만 할 때는 return_labels=False로 호출 가능함.
        if return_labels:
            y = df[[self.label_column]].to_numpy(dtype=np.float32)

            batch["labels"] = torch.tensor(
                y,
                dtype=torch.float32,
            )

        return batch


# 이 파일이 import되는 시점에 
# AutoProcessor 저장/배포용 metadata를 등록하는 코드.
#
# 이 코드를 통해 processor.save_pretrained(save_dir)를 수행할 때
# processor 설정 파일에 auto_map 정보가 들어가고,
# custom processor가 정의된 이 .py 파일도 저장 directory로 복사될 수 있게 됨.
#
# 이후 로드 시에 trust_remote_code=True를 사용할 수 있음:
# AutoProcessor.from_pretrained(save_dir, trust_remote_code=True)
MLPRegressionProcessor.register_for_auto_class("AutoProcessor")
```
