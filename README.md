# ESS 배터리 수명 예측 
목적 작성 


## 프로젝트 개요
- 데이터셋 : MIT-Stanford Battery Dataset (Severson et al., Nature Energy 2019)
- 학습 데이터 : Batch 1 (2017-05-12)
- 평가 데이터 : Batch 2 (2018-02-20)
- 태스크 : Regression (Cycle Life 예측) / Classification (장단수명 분류)  ← 택1 


## 파일 구조 (sample) 
```
├── data/
│   └── README.md          
├── notebooks/
│   ├── 01_EDA.ipynb
│   ├── 02_feature_engineering.ipynb
│   └── 03_modeling.ipynb
├── src/
│   ├── preprocess.py
│   ├── features.py
│   └── train.py
├── results/
│   └── model_performance.csv
├── requirements.txt
└── README.md
```


## 환경 설정 (sample) 
```bash
git clone https://github.com/팀명/ess-battery-project
cd ess-battery-project
pip install -r requirements.txt
```


## EDA

- Cycle Life 분포
  - 분포 형태 및 장단수명 비율 요약 : 전체 셀(Batch 1~3)의 수명은 150~2,300사이클로 넓은 범위에 걸쳐 오른쪽으로 꼬리가 긴(right-skewed) 분포를 보임. 장수명(>1,000사이클) 셀은 전체의 약 20%, 단수명(<500사이클) 셀은 약 40%를 차지하며 단수명 비중이 높음
  - 핵심 발견 : 배터리 수명 분포는 정규분포가 아닌 비대칭 분포이므로, 회귀 모델링 시 로그 변환(log transform)을 고려해야 하며, 단순 평균보다 중앙값이 대표값으로 더 적합함

- 열화 곡선 분석
  - 장수명 vs 단수명 셀의 열화 속도 차이 : 단수명 셀은 초기 50~100사이클 이내부터 방전 용량(Qd)이 빠르게 감소하는 반면, 장수명 셀은 상대적으로 완만한 선형적 열화를 보임
  - Knee point 존재 여부 및 발생 시점 : 단수명 셀에서 Knee point(급격한 열화 가속 시점)가 뚜렷하게 관찰되며, 대체로 SOH 90% 근방에서 발생. 장수명 셀은 Knee point가 늦게 나타나거나 관측 범위 내에서 나타나지 않음
  - 핵심 발견 : 초기 100사이클의 열화 기울기만으로도 장단수명 셀을 구분할 수 있는 신호가 존재하며, 이는 조기 수명 예측의 핵심 근거가 됨

- ΔQ(V) 곡선 분석
  - Cycle 100 - Cycle 10 차이 곡선 형태 : ΔQ(V) = Q(V, cycle 100) - Q(V, cycle 10)으로 계산하며, 전압 구간 2.0~3.6V에 걸쳐 음의 방향으로 면적이 형성됨. 단수명 셀일수록 곡선의 최솟값이 더 크게 음의 방향으로 이동하고 분산이 큰 형태를 보임
  - 장단수명 셀 간 ΔQ 형태 비교 : 장수명 셀은 ΔQ(V) 곡선의 분산(variance)이 작고 최솟값이 0에 가까운 반면, 단수명 셀은 분산과 최솟값의 절댓값이 모두 크게 나타남
  - 핵심 발견 : ΔQ(V) 곡선의 분산(var), 최솟값(min), 평균(mean)이 cycle_life와 강한 상관관계를 가지며, 원논문의 핵심 피처로 활용됨. 특히 분산(variance of ΔQ)은 단독으로도 수명과 높은 음의 상관관계(-0.87 수준)를 보임

- 충전 속도(C-rate)와 수명의 관계
  - 충전 프로토콜별 평균 수명 비교 결과 : 고속 충전(4C 이상) 프로토콜을 사용한 셀은 저속 충전 셀 대비 평균 수명이 짧은 경향이 있으나, 동일 C-rate 내에서도 수명 편차가 크게 나타남
  - 핵심 발견 : C-rate 단독으로는 수명을 결정짓는 지배적 요인이 아니며, 초기 사이클의 ΔQ(V) 패턴이 C-rate 효과를 내포하는 더 강력한 피처임. 따라서 charging_policy는 보조 피처로 활용하고 ΔQ 기반 피처를 우선시하는 전략이 적합함

- 상관관계 분석 (초기 사이클 피처 vs Cycle Life)
  - 초기 사이클의 QD(방전 용량), IR(내부 저항), Tavg(평균 온도) 등 Summary 피처와 cycle_life 간 상관계수를 확인한 결과, var(ΔQ) > min(ΔQ) > QD_mean 순으로 높은 상관관계를 보임
  - 핵심 발견 : var(ΔQ)와 min(ΔQ)는 서로 높은 다중공선성(multicollinearity)을 가지므로, 두 피처를 동시에 투입할 경우 VIF 확인 또는 PCA 등의 차원 축소 고려가 필요함

## Modeling 

### 피처 엔지니어링 전략
EDA 결과를 바탕으로 선택한 피처와 그 근거를 기술


### 모델 선택 및 근거
- 후보 모델 : 
- 최종 모델 :
- 선택 이유 :


## 성능 결과
Format에 맞춰 작성


## 오류 분석
- 모델이 가장 크게 틀린 셀의 공통점
- 원인 가설 및 개선 방향


## ESS 도메인 해석
분석 결과를 실제 ESS 운영 관점에서 해석

- 이 모델을 실제 BESS에 적용한다면 어떤 의사결정에 활용 가능한가?
- 어떤 한계가 있으며, 실 배포를 위해 추가로 필요한 것은 무엇인가?


## 참고문헌
- Severson et al. (2019). Data-driven prediction of battery cycle life before capacity degradation. *Nature Energy*, 4, 383–391.


## 팀 구성
- 김한규 : EDA, 피처 엔지니어링, 모델 개발, 성능 평가
- 박지원 : EDA, 피처 엔지니어링, 모델 개발, 성능 평가
- 배수정 : EDA, 피처 엔지니어링, 모델 개발, 성능 평가
- 안가은 : EDA, 피처 엔지니어링, 모델 개발, 성능 평가