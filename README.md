# ESS Battery Cycle Life Prediction - EDA
본 프로젝트는 MIT-Stanford Battery Dataset의 초기 100 cycle 기반 파생 피처로 배터리의 총 수명(cycle_life)을 예측하는 모델을 구축합니다.

## 프로젝트 개요
- 데이터셋 : MIT-Stanford Battery Dataset (Severson et al., Nature Energy 2019)
- 학습 데이터 : Batch 1 (2017-05-12)
- 평가 데이터 : Batch 2 (2018-02-20)
- 태스크 : Regression (Cycle Life 예측)

---
## 파일 구조
```bash
├── notebooks/
│   ├── eda_v1/
│   │   └── day1_eda.ipynb
│   ├── eda_v2/
│   │   ├── batch1.ipynb
│   │   ├── batch2.ipynb
│   │   └── batch3.ipynb
│   └── modeling/
│       └── day2_eda.ipynb
│       └── day2_modeling_v1.ipynb
│       └── day2_modeling_v2.ipynb
├── requirements.txt
└── README.md
```

## 환경 설정
```bash
git clone https://github.com/dksrkn/ess-battery-project.git

cd ess-battery-project

python3 -m venv .venv

source .venv/bin/activate

pip install -r requirements.txt
```

## 주요 EDA 결과

### 1. Cycle Life 분포

* 배터리 수명은 셀마다 편차 존재
* Batch 간 분포 차이 존재
* 일부 극단값(outlier) 존재

**Insight**
* 단순 평균 기반 모델링 어려움
* robust한 feature 필요

---

### 2. 충전 조건(C-rate)과 수명 관계

* C-rate 증가 → cycle life 감소 경향
* Batch1에서는 강한 음의 상관관계
* Batch2에서는 관계 약화

**Insight**
* 충전 속도는 중요한 feature
* 하지만 batch마다 영향 다름

---

### 3. QD(용량) 감소 패턴

* 일반적으로 cycle 증가 → QD 감소
* 일부 셀에서 **비정상적인 증가(spike)** 발생

**문제**
* 센서 노이즈 또는 측정 오류
* ΔQ feature 계산 시 왜곡 발생

**처리 방향**
* smoothing / monotonic 보정 필요

---

### 4. 열화 곡선 분석

* 초기에는 완만한 감소
* 이후 특정 시점에서 급격한 감소 (knee point)

**Insight**

* 초기 slope가 전체 수명과 밀접한 관계
* `slope_early` 중요 feature

---

### 5. ΔQ(V) 분석

ΔQ(V) = Q(100) - Q(10)

* 초기 열화 패턴을 전압 기준으로 표현
* 다양한 통계 feature 생성

#### 주요 feature

* `delta_log_var`: 열화 변동성 (논문 핵심)
* `delta_mean`: 평균 열화량
* `delta_std`: 열화 불균일성

#### 문제 발견

* `delta_max`, `delta_area` 등은 batch 간 값 차이 매우 큼
* 전압 정렬 차이로 인해 feature 불안정

**Insight**

* ΔQ feature는 강력하지만 민감함
* 일부 feature 제거 필요

---

### 6. Feature 분포 비교 (Batch1 vs Batch2)

| Feature       | 특징        |
| ------------- | --------- |
| delta_max     | 최대 40배 차이 |
| QD_cv         | 약 4배 차이   |
| delta_log_var | 상대적으로 안정  |
| effective_C   | 비교적 안정    |

**Insight**

* batch 간 distribution shift 존재
* 일반화 성능 저하 원인

---

### 7. 다중공선성 분석

* ΔQ 계열 feature 간 강한 상관관계 존재
* VIF 매우 높음 (inf 수준)

**문제**

* 모델 계수 불안정
* 일반화 성능 저하

**해결**

* delta feature 축소 (대표 feature만 사용)

---

## 주요 문제점 정리

1. **Batch 간 분포 차이 (distribution shift)**
2. **QD 이상치 (비정상 증가)**
3. **ΔQ feature의 불안정성**
4. **다중공선성 (특히 delta 계열)**

---

## Feature Engineering 방향

EDA 결과를 기반으로 다음 전략을 적용:

### 파생 변수 생성

```text
- QD_cv
- effective_C
- temp_ir_interaction
- delta_log_var 
```

---

## 핵심 인사이트

* 초기 열화 패턴(ΔQ(V), QD 변화, slope 등)이 수명과 밀접한 관계가 있음을 확인함
* ΔQ(V)는 강력하지만 매우 민감한 feature
* 모델보다 feature 안정성이 더 중요
* batch 간 차이를 고려한 feature 설계 필요

-----

## 모델링 결과
### 사용 feature : delta_log_var, QD_cv, effective_C, temp_ir_interaction

1. ElasticNet + Log
   
| 구분                       | MAPE (%) | 비고                  |
| ------------------------ | -------- | ------------------- |
| Train (Batch 1 CV)       | 9.46     |                     |
| Valid (Batch 1 Hold-out) | 8.45     |                     |
| Test (Batch 2)           | 30.46    |                     |
| Gap (Train-Valid)        | -1.01    | 과적합 없음              |
| Gap (Valid-Test)         | +22.01   | (+) : 배치간 일반화 저하 의심 |
| Gap (Target-Test)        | +21.36   | Target : 원논문 9.1%   |

2. Log Regression + Ridge

| 구분                       | MAPE (%) | 비고                  |
| ------------------------ | -------- | ------------------- |
| Train (Batch 1 CV)       | 9.42     |                     |
| Valid (Batch 1 Hold-out) | 7.61     |                     |
| Test (Batch 2)           | 35.77    |                     |
| Gap (Train-Valid)        | -1.80    | 과적합 없음              |
| Gap (Valid-Test)         | +28.15   | (+) : 배치간 일반화 저하 의심 |
| Gap (Target-Test)        | +26.67   | Target : 원논문 9.1%   |

-----
### 사용 feature : delta_log_var, delta_max, QD_cv, effective_C, temp_ir_interaction

3. LightGBM
   
| 구분                       | MAPE (%) | 비고                  |
| ------------------------ | -------- | ------------------- |
| Train (Batch 1 CV)       | 13.60    |                     |
| Valid (Batch 1 Hold-out) | 8.94     |                     |
| Test (Batch 2)           | 27.65    |                     |
| Gap (Train-Valid)        | -4.66    | 과적합 없음              |
| Gap (Valid-Test)         | +18.71   | (+) : 배치간 일반화 저하 의심 |
| Gap (Target-Test)        | +18.55   | Target : 원논문 9.1%   |

## 결과 해석
1. 성능 해석
   - Train과 Valid 성능 차이가 크지 않아 과적합은 관찰되지 않음
   - Valid 대비 Test 성능이 감소되어 Batch 간 데이터 분포 차이로 인한 일반화 성능 저하 확인
   - 논문 수준의 성능에는 도달하지 못함  
2. 도메인 관점 해석
   - 초기 100 cycle 데이터 만으로 Validation 성능이 양호한 점을 통해 초기 열화 패턴이 수명과 유의미한 관계를 가짐을 확인
   - 배치 간 일반화에는 한계 존재
3. 한계점
   - ΔQ 기반 feature의 민감도 및 불안정성
   - Batch 2,3에서 QD 값이 비정상적으로 증가하는 이상치로 인해 오차 증가
   - 다중공선성으로 변수 선택의 제한
   - 제한된 데이터 수로 인해 일반화 성능 저하
4. lightGBM
   - LightGBM은 비선형 패턴 반영 가능
   - 이상치 및 노이즈에 강건
   - 작은 데이터에서도 안정적 성능

## 팀 구성
- 김한규 : EDA, 피처 엔지니어링, 모델 개발, 성능 평가
- 박지원 : EDA, 피처 엔지니어링, 모델 개발, 성능 평가
- 배수정 : EDA, 피처 엔지니어링, 모델 개발, 성능 평가
- 안가은 : EDA, 피처 엔지니어링, 모델 개발, 성능 평가
