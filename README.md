**이전에 진행된 컴페티션에서 우수했던 방법들을 정리한 것입니다.** 

**제가 속한 팀은 분반 2등을 기록했으며, 시행착오는 생략하고 핵심적인 접근법만 기록했습니다.**

# 🏆온라인 설문조사 응답 예측

## 📌 프로젝트 개요
프로젝트 배경 : **"각 패널에게 어떤 온라인 설문조사를 요청해야 응답할까?"** 

대회 목표 : 패널정보와 설문 데이터를 바탕으로 **응답 여부(`STATUS`)를 예측하는 머신러닝 모델을 개발**

- **평가 척도:** Accuracy (정확도)
- **데이터 구성:**  
  - `train.csv` (813,650개) - 학습 데이터  
  - `test.csv` (541,867개) - 평가 데이터  
  - 총 44개의 컬럼(패널 정보, 설문 정보, 응답 여부 포함)  

---

## 데이터 전처리 & Feature Engineering

### 🛠 1. 데이터 전처리
- **결측치 처리:**  
  - 범주형 변수 → 최빈값(`mode`)으로 대체  
  - 수치형 변수 → 평균값(`mean`)으로 대체
    
- **피처 정리:**  
  - 결측치 비율이 30% 이상인 컬럼 제거  
  - 일부 피처에서 불필요한 공백(`,` 포함)으로 인해 같은 값이 다르게 분류되는 문제 해결  

### 2. Feature Engineering
**모델 성능을 높이기 위해 다양한 피처를 추가**  
직관적으로 타당한 가정을 바탕으로 피처를 생성하고, 추가할 때마다 submission을 제출하여 public 성능을 기준으로 선택.

1️⃣ **응답률 관련 피처**
   - 패널별 응답률 (`RES_RATE_user`)
   - 설문별 응답률 (`RES_RATE_survey`)
     
   ** 패널별 응답률은 개인성향을 반영, 설문별 응답률은 설문자체의 응답경향 
   
   
   → 서로 다른 차원에서 보완적인 정보를 제공하여 모델을 일반화하는데 기여할 수 있음
   - 두 피처를 추가했을 때의 public Score 증가폭이 가장 컸음
   - 모델 학습을 복잡도를 줄이기 때문에 소수점 3자리까지 반올림한 값 사용
   - 반올림 적용후 피처고유값 개수 감소
       - RES_RATE_USER 2441 -> 933
       - RES_RATE_Survey 1038 -> 519
   - 패널 타입별 응답률 (`RES_RATE_type`)


2️⃣ **설문 정보 관련 피처**
   - 설문 제목(`TITLE`)에서 주요 단어(해외, 소비자, 일반인 등) 포함 여부를 피처로 추가  
   - 제목에서 등장하는 단어 빈도를 기반으로 Encoding  

3️⃣ **리워드 & 설문 시간 관련 피처** 
   - `리워드 / 응답 시간` → 리워드가 많을수록 응답 가능성이 높다고 가정
   - `-(설문 시간 + 난이도)` → 시간이 길수록 난이도가 높을수록 응답 가능성이 낮다고 가정  

4️⃣ **패널별 성향 반영**
   - 패널별 총 응답 횟수 (`Status_sum_log`) → 로그 변환하여 정규화  
   - 응답한 사용자별 평균 획득 포인트를 구간화하여 피처 추가  

### 📏 3. 데이터 변환
- **수치형 피처:** StandardScaler 적용  
- **범주형 피처:** Ordinal Encoding 적용  

---

## Feature Selection
- SHAP(Shapley Additive Explanations) 라이브러리를 사용하여 **각 피처의 중요도를 분석**  
- 중요도가 낮은 피처를 제거하고, 성능이 향상되는 피처를 추가하여 최적의 Feature Selection 진행  

---

## Model Building & Evaluation

### 실험한 모델들
- 처음에는 **DecisionTree, RandomForest, Ridge, Lasso, BaggingClassifier** 등 다양한 모델을 테스트했지만,  
  결국 **LightGBM, CatBoost, XGBoost**가 가장 좋은 성능을 보였습니다.

### 🔄 교차검증을 활용한 OOF 앙상블
모델의 일반화 성능을 높이기 위해 **OOF(Out-of-Fold) 방식으로 5-Fold 교차검증을 적용**했습니다.

- **5개의 폴드를 생성하여 각각 학습**  
- **폴드별 예측값을 Soft Voting(예측값 평균) 방식으로 앙상블**  
- **임계값(`threshold`) 0.495 설정** (실험을 통해 도출한 최적임계값)  
- **최종적으로 Hard Voting (다수결 방식) 적용하여 최종 예측값 생성**  

### 🔀 추가적인 앙상블
마지막으로, **전체 피처를 변환하여 학습한 모델들을 보팅 앙상블**을 진행했습니다.

- **로그 변환 (`log1p`)**
- **제곱근 변환 (`sqrt`)**
- **MinMax 정규화 적용**  
  → 변환된 데이터를 기반으로 개별 모델을 학습하고, Soft Voting을 수행하여 최종 결과 도출  

---

## 📈 프로젝트 진행 과정
1️⃣ **EDA(탐색적 데이터 분석)**  
   - 데이터 분포, 결측치 처리, 피처 분포 확인 및 정리  

2️⃣ **Feature Engineering**  
   - 응답률, 설문 제목, 리워드, 시간 등 다양한 피처 생성  
   - SHAP을 활용한 Feature Selection  

3️⃣ **모델 학습 및 최적화**  
   - OOF 방식으로 5-Fold 학습 진행  
   - Soft Voting, Hard Voting 적용  
   - 데이터 변환 후 추가적인 앙상블 수행  

4️⃣ **결과 분석 & 제출**  
    - 1차적으로는 검증데이터 성능이 어떤지 확인하고
    - 그 후에 submission을 제출하여 Public 성능에 기반하여 분석

---

## 🎯 최종 정리
✔ **패널별, 설문별 응답률과 같은 핵심적인 피처를 생성**  
✔ **LightGBM, CatBoost, XGBoost 모델을 활용한 OOF 방식 5-Fold Soft Voting 적용**  
✔ **Hard Voting & 데이터 변환 후 추가적인 앙상블 기법을 적용하여 성능 최적화**  


📌 프로젝트 핵심 내용
이번 프로젝트에서 가장 중요한 과정은 다양한 피처를 실험하고 최적화하는 과정이었습니다.
특히, 데이터 양이 피처 개수에 비해 충분했기 때문에 피처 생성이 성능 향상의 핵심 요인이 되었습니다.

새로운 피처를 생성할 때마다, 이것이 과적합을 유발하는지 여부를 판단하기 위해 submission을 제출하고, 학습 데이터 성능과 Public 성능을 비교하며 검증했습니다.

결과적으로, 유의미한 피처를 선택하고, 적절한 데이터 변환 및 앙상블 기법을 활용하여 최적의 성능을 도출할 수 있었습니다. 
 
📢 **코드 및 상세 분석 과정은 [노트북 링크]에서 확인할 수 있습니다.**  
