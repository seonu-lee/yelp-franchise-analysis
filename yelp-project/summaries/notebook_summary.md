# Yelp 프랜차이즈 경쟁압력 분석 — 노트북 요약

> **분석 개요**: Yelp 공개 데이터셋을 활용해 Las Vegas 지역 독립 레스토랑이 프랜차이즈로부터 받는 경쟁 압력(FPI, Franchise Pressure Index)을 정량화하고, 이것이 별점·감성·폐업에 미치는 영향을 규명한다.  
> **전체 흐름**: PART1(전체 Restaurants 업종 탐색) → PART2(버거·샌드위치·패스트푸드로 범위를 좁혀 심화 분석)

---

## 전체 파이프라인 한눈에 보기

```
[원본 데이터]
yelp_business.csv (174,567개)  +  yelp_review.csv (5,261,668개)
        │
        ▼ ─────────────────────── PART 1 ───────────────────────
[01_data_setup]  업체·리뷰 필터링 → 프랜차이즈 판정
        ↓ biz_target.csv (5,899개) / review_target.csv (929,606개)
[02_dtm_tfidf]   텍스트 전처리 → DTM → TF-IDF 행렬
        ↓ per-brand TF-IDF (4,228 × 910)
[03_sentiment]   감성 사전 구축 → 리뷰별 감성점수 → VADER 교차검증
        ↓ biz_sentiment.csv (5,899개 × 18열)
[04_FPI]         Shift 정규화 → 하버사인 거리 → FPI → 구간 분류(NP/LP/HP)
        ↓ biz_indie_with_groups.csv (4,818개)
[05_regression]  OLS + HC3 다중회귀 / ANOVA + Tukey HSD
[06_text_analysis]  TF-IDF 차이 → 구간별 키워드 → PCA Biplot
[07_fpi_map]     Plotly 인터랙티브 지도 (연속형 / 구간별 / 업종별)
        │
        ▼ ─────────────────────── PART 2 ───────────────────────
[01_data_prep]   업종 재필터링(버거·샌드·패스트푸드) → 프랜차이즈 재판정
        ↓ biz_target_burger.csv (1,579개) / review_target_burger.csv (178,094개)
[02_sentiment]   도메인 특화 감성 사전 재구축 → VADER 교차검증
        ↓ biz_sentiment_burger.csv (1,579개 × 16열)
[03_fpi]         반경별 FPI → 민감도 분석 → NP/LP/HP 구간 분류
        ↓ biz_indie_with_groups_burger.csv (775개)
[04_regression]  OLS 다중회귀 + Logistic 회귀 + 매개효과 분석 시도 (횡단면 데이터 한계로 성립하지 않음)
[05_text_analysis]  TF-IDF / N-gram / VADER Intensity / LDA / 시계열 감성
[06_fpi_map]     연속형·구간별·생존/고전 브랜드 위치 인터랙티브 지도
```

---

# PART 1: 전체 Restaurants 분석

---

## `01_data_setup.ipynb`

### 1. 파일명 및 목적
Yelp 데이터셋을 로드하고 분석 대상 지역·업종을 선정하며 프랜차이즈 판정 기준을 확립하는 전처리 기반 파일. STEP 0(데이터 로드) → STEP 1(지역/업종 선정) → STEP 2(프랜차이즈 판정) 3단계 구성.

### 2. 주요 데이터셋

| 파일 | 행 수 | 주요 컬럼 |
|---|---|---|
| `yelp_business.csv` | 174,567행 | `business_id`, `name`, `city`, `state`, `latitude`, `longitude`, `stars`, `review_count`, `categories` |
| `yelp_review.csv` | 5,261,668행 | `review_id`, `business_id`, `stars`, `date`, `text`, `useful`, `funny`, `cool` |

**파생 변수**: `name_clean` (소문자화+특수문자 제거), `is_franchise` (bool)

### 3. 핵심 분석 단계

**STEP 1 — 지역/업종 선정**

3개 후보 도시 비교 후 **Las Vegas + Restaurants** 선택:

| 기준 | Las Vegas | Phoenix | Charlotte |
|---|---|---|---|
| 전체 업체 수 | **5,899개** | 3,647개 | 2,460개 |
| 평균 리뷰 수 | **158개** | 91개 | 64개 |
| 전체 리뷰 수 | **1,603,616개** | 576,700개 | 237,308개 |

업종 선택 근거: Fast Food(69.6% 프랜차이즈)·Burgers(42.4%)는 독립 브랜드 수 부족 → **Restaurants** 채택 (프랜차이즈 925개 + 독립 4,974개)

**STEP 2 — 프랜차이즈 판정 (임계값 민감도 분석)**

```python
def clean_name(name):
    name = name.lower().strip()
    name = re.sub(r'[^a-z0-9\s]', '', name)
    return re.sub(r'\s+', ' ', name).strip()
```

| 임계값 | 프랜차이즈 브랜드 수 | 비율 |
|---|---|---|
| 3회 | 181개 | 27.1% |
| 5회 | 85개 | 21.7% |
| **10회 (채택)** | **38개** | **16.7%** |
| 20회 | 20개 | 12.6% |

수동 보정: 브랜드명 병합(13개 매핑) + 화이트리스트 추가(12개 전국 체인)

### 4. 주요 결과 및 결론

| 구분 | 업체 수 | 비율 |
|---|---|---|
| 프랜차이즈 | 1,081개 | 18.3% |
| 독립 브랜드 | 4,818개 | 81.7% |
| 분석 대상 리뷰 | 929,606개 | — |

| | 평균 별점 | 평균 리뷰 수 |
|---|---|---|
| 독립 브랜드 | 3.63 | 183개 |
| 프랜차이즈 | 2.71 | 42개 |

프랜차이즈 낮은 별점은 Yelp 사용자의 독립 레스토랑 선호 편향, 관광객 높은 기대치 등 복합 요인으로 해석.  
**출력**: `results/biz_target.csv`, `results/review_target.csv`

### 5. 개선 아이디어
- 브랜드명 병합에 퍼지 매칭(FuzzyWuzzy) 자동화로 수동 보정 축소
- 화이트리스트를 FTC FDD 등 공식 프랜차이즈 DB와 연동
- Phoenix·Charlotte도 비교군으로 병렬 분석하여 일반화 검토

---

## `02_dtm_tfidf.ipynb`

### 1. 파일명 및 목적
리뷰 텍스트를 전처리하고 브랜드별 DTM(Document-Term Matrix)과 TF-IDF 행렬을 구축하는 NLP 파이프라인. STEP 3-1(클리닝) → STEP 3-4(시각화) 4단계.

### 2. 주요 데이터셋

| 입력 | 내용 |
|---|---|
| `biz_target.csv` | Las Vegas Restaurants 5,899개 |
| `review_target.csv` | 리뷰 929,606개 |

**핵심 파라미터**: `MIN_DF_RATE=0.1`, `MAX_DF_RATE=0.9`, `MIN_DOC_COUNT=10`, `min_review_prop=0.3`

### 3. 핵심 분석 단계

**STEP 3-1 — 브랜드별 리뷰 Pooling + 텍스트 클리닝**

```python
def clean_text(text):
    text = text.lower()
    text = re.sub(r'\d+', '', text)
    text = re.sub(r'[^\w\s]', '', text)
    return tokenize_filter_stem(text)  # 불용어 제거 + Porter Stemmer

brand_reviews = reviews.groupby(name_col).agg(
    text=('text', ' '.join), ...  # 브랜드별 리뷰 전체 합산
)
```

**STEP 3-2 — DTM 생성 (4단계 필터링)**

| 단계 | 처리 | 최종 단어 수 (prop=0.3) |
|---|---|---|
| df_rate 필터 | 0.1~0.9 범위 밖 제거 | — |
| review_prop 필터 | 브랜드 내 리뷰 비율 0.3 미달 제거 | — |
| 최종 | min_doc=10 + 수동 불용어 제거 | **902개** |

`min_review_prop=0.1`(2,288개) 대비 `0.3`(902개)이 노이즈 감소 효과 우수 → 0.3 채택

**STEP 3-3 — TF-IDF 변환 (norm=None 버전 + L2 정규화 버전 생성)**

**STEP 3-4 — 시각화 3종**: 파라미터 비교 바차트 / 워드클라우드 / 히트맵 (상위 15개 브랜드 × 30개 단어)

### 4. 주요 결과 및 결론
- 최종 TF-IDF 행렬: **4,228개 브랜드 × 910열** (메타 8열 + 단어 902개), 결측치 0개
- 브랜드별 리뷰 pooling으로 희소성 문제 해결 및 안정적 TF-IDF 산출
- 히트맵에서 Bacchanal Buffet→buffet, Tacos El Gordo→taco 등 브랜드별 특징 단어 확인

### 5. 개선 아이디어
- Porter Stemmer → WordNetLemmatizer로 교체 시 가독성 향상
- n-gram(2-gram) 포함 시 `good food`, `bad service` 같은 복합 표현 포착 가능
- `min_review_prop`을 브랜드 리뷰 수에 비례한 동적 설정으로 소규모 브랜드 과필터 방지

---

## `03_sentiment.ipynb`

### 1. 파일명 및 목적
별점을 정답지(ground truth)로 TF-IDF 기반 긍정/부정 감성 사전을 구축하고, 리뷰별·업체별 감성점수를 산출한 뒤 VADER로 교차 검증. STEP 4-1 ~ 4-6 총 6단계.

### 2. 주요 데이터셋

| 입력 | 내용 |
|---|---|
| `review_target.csv` | 리뷰 929,606개 (별점 포함) |
| `biz_target.csv` | 업체 5,899개 (`is_franchise` 포함) |

**파생 변수**: `sentiment_score` (리뷰 단위), `tfidf_sentiment` (업체 평균), `vader_sentiment` (VADER 업체 평균)

### 3. 핵심 분석 단계

**STEP 4-1 — 긍정/부정 코퍼스 구성 (3점 의도적 제외)**

| 별점 | 분류 | 리뷰 수 |
|---|---|---|
| 4~5점 | 긍정 코퍼스 | 614,312개 |
| **3점** | **제외** (사전 오염 방지) | 125,290개 |
| 1~2점 | 부정 코퍼스 | 190,004개 |

**STEP 4-2 — TF-IDF 감성 사전 구축**

```python
vectorizer = TfidfVectorizer(max_features=50000, stop_words='english')
tfidf_matrix = vectorizer.fit_transform([pos_corpus, neg_corpus])
# 각 코퍼스 TOP 300 추출 → 교집합(216개) 제거
pos_lexicon = set(pos_top.index) - overlap  # 84개
neg_lexicon = set(neg_top.index) - overlap  # 84개
```

긍정 사전 예시: `awesome`, `tasty`, `crispy`, `reasonable` / 부정: `awful`, `cold`, `dirty`, `salty`

**STEP 4-3 — 리뷰별 감성점수**

$$\text{sentiment\_score} = \frac{\text{긍정단어 수} - \text{부정단어 수}}{\text{전체 단어 수}}$$

**STEP 4-5 — VADER 교차 검증**

```python
corr, pvalue = pearsonr(valid['tfidf_sentiment'], valid['vader_sentiment'])
# 결과: corr = 0.724, p ≈ 0.000  (기준: 0.6 이상 → 통과 ✅)
```

### 4. 주요 결과 및 결론

| 지표 | 값 |
|---|---|
| 긍정 사전 크기 | 84개 (교집합 216개 제거 후) |
| 부정 사전 크기 | 84개 |
| 업체 평균 감성점수 범위 | -0.111 ~ +0.089 |
| VADER 상관계수 | **0.724** (p < 0.001) |

TF-IDF 기반 자체 사전이 VADER와 높은 상관성 → 신뢰도 확보. 독립 브랜드가 프랜차이즈보다 감성점수 높음.  
**출력**: `biz_sentiment.csv` (5,899행, 18열)

### 5. 개선 아이디어
- `TOP_N_WORDS`를 100~500 범위 그리드 서치로 최적 사전 크기 탐색
- 사전 단어에도 동일 스테밍 적용하여 매칭률 개선
- TextBlob, BERT 기반 감성 분석과 다중 교차 검증

---

## `04_FPI.ipynb`

### 1. 파일명 및 목적
FPI(Franchise Pressure Index)를 산출하고 반경 민감도 분석으로 최적 임계거리를 도출하며, 독립 브랜드를 NP/LP/HP 3구간으로 분류. STEP 5-0 ~ 5-5 총 6단계.

### 2. 주요 데이터셋

| 입력 | 내용 |
|---|---|
| `biz_sentiment.csv` | 업체 5,899개 (`tfidf_sentiment`, 좌표 포함) |

**핵심 파라미터**: `radii = [100, 200, 300, 500]` (미터), `k = 1` (km, 거리 보정 상수)

**파생 변수**: `fpi_{r}m` (반경별 FPI), `fpi_group` (NP/LP/HP), `sentiment_shifted` (Shift 정규화 감성점수)

### 3. 핵심 분석 단계

**STEP 5-0 — Shift 정규화** (음수 감성점수 → FPI 부호 왜곡 방지)

```python
sentiment_min = fran_df['tfidf_sentiment'].min()  # = -0.1114
fran_df['sentiment_shifted'] = fran_df['tfidf_sentiment'] - sentiment_min
# 정규화 후: min=0.0000, max=0.1561 (상대적 간격 완전 보존)
```

**STEP 5-1 — 하버사인 거리 행렬 (4,818 × 1,081 = 약 521만 쌍)**

```python
a = np.sin(dlat/2)**2 + np.cos(indie_lat) * np.cos(fran_lat) * np.sin(dlon/2)**2
distances_m = 2 * np.arcsin(np.sqrt(a)) * 6371000
```

**STEP 5-2 — FPI 공식**

$$FPI_i = \sum_{j \in N(i,r)} \frac{\text{stars}_j \times \ln(\text{review\_count}_j+1) \times \text{sentiment\_shifted}_j}{(d_{ij}[\text{km}] + k)^2}$$

- `stars`: 프랜차이즈 품질 가중치 / `ln(리뷰수+1)`: 영향력 규모 / `sentiment_shifted`: 소비자 반응 / `거리² 역수`: 거리 감쇠

**STEP 5-4 — 민감도 분석 (반경별 OLS 단순회귀)**

| 반경 | 별점 p | 감성점수 p | 채택 여부 |
|---|---|---|---|
| 100m | 0.0001 ✅ | 0.2672 ❌ | — |
| 200m | 0.0002 ✅ | 0.5950 ❌ | — |
| **300m** | **0.0000 ✅** | **0.0207 ✅** | **채택** |
| 500m | 0.0000 ✅ | 0.0017 ✅ | — |

→ 별점+감성점수 **두 지표 모두 p<0.05인 최소 반경 = 300m 확정**

**STEP 5-5 — NP/LP/HP 분류** (FPI 중앙값 2.0791 기준)

### 4. 주요 결과 및 결론

| 그룹 | 업체 수 | 비율 |
|---|---|---|
| NP (No Pressure, FPI=0) | 977개 | 20.3% |
| LP (Low Pressure, FPI≤2.0791) | 1,921개 | 39.9% |
| HP (High Pressure, FPI>2.0791) | 1,920개 | 39.8% |

FPI가 높을수록 독립 브랜드 별점이 낮아짐 (계수 -0.02~-0.04, 모든 반경 p<0.001). 300m 이상부터 감성점수도 유의미하게 영향.  
**출력**: `biz_sentiment_with_fpi.csv` (5,899행), `biz_indie_with_groups.csv` (4,818개)

### 5. 개선 아이디어
- `k` 파라미터를 0.5, 2.0 등으로 변경하며 결과 안정성 검토
- 반경을 50m 단위 연속 탐색으로 임계거리 정밀화
- Shift 외 Percentile Rank 정규화 병행 적용 후 비교

---

## `05_regression.ipynb`

### 1. 파일명 및 목적
FPI가 독립 브랜드의 별점과 감성점수에 미치는 영향을 다중회귀(OLS + HC3)와 ANOVA로 통계 검증. 영업 중인 독립 브랜드 3,017개 대상.

### 2. 주요 데이터셋

| 입력 | 내용 |
|---|---|
| `biz_indie_with_groups.csv` | 독립 브랜드 4,818개 |

**분석 대상**: `is_open=1` 3,017개 / 독립변수: `fpi_300m` / 통제변수: `log(review_count+1)`, neighborhood 더미 16개

### 3. 핵심 분석 단계

**STEP 6-0 — 통제변수 선정**

| 변수 | 별점 상관 | 감성 상관 | FPI 상관 | 채택 |
|---|---|---|---|---|
| log(review_count) | 0.1585 | 0.1758 | 0.0741 | ✅ |
| neighborhood 더미 | 별점 최대 0.44점 차이 | — | FPI 10배 차이 | ✅ |
| is_open | 0.1028 | 0.0649 | -0.013 | ❌ (사전 필터링) |

**STEP 6-1 — OLS + HC3 (이분산 강건 표준오차)**

```python
model_stars = sm.OLS(indie_df['stars'], X).fit(cov_type='HC3')
model_sent  = sm.OLS(indie_df['tfidf_sentiment'], X).fit(cov_type='HC3')
```

WLS 미사용 이유: log_review를 통제변수로 투입한 상태에서 WLS 가중치로도 사용 시 이중 처리 모순.

**STEP 6-3 — ANOVA + Tukey HSD 사후검정**

```python
f_stat, p_val = stats.f_oneway(*groups)
tukey = pairwise_tukeyhsd(valid[col], valid['fpi_group'], alpha=0.05)
```

### 4. 주요 결과 및 결론

**OLS + HC3 결과**

| 종속변수 | FPI 계수 | p-value | 결론 |
|---|---|---|---|
| 별점 | **-0.0119** | **0.033 ✅** | FPI 1 증가 시 별점 0.012점 유의미 감소 |
| 감성점수 | -0.0001 | 0.384 ❌ | 통제 후 유의미 영향 없음 |

별점 모델 R²=0.0717, F-stat p=0.000

**ANOVA 결과 (별점: F=19.72, p=0.000)**

| 구간 | 평균 별점 | Tukey 결과 |
|---|---|---|
| NP | 3.83 | HP vs NP: p=0.000 ✅ |
| LP | 3.69 | HP vs LP: p=0.010 ✅ |
| HP | 3.61 | LP vs NP: p=0.000 ✅ |

**핵심 결론**: FPI는 소비자의 **정량적 평가(별점)** 에는 영향을 미치나, **텍스트 감성**까지 직접 견인하지 못함.

### 5. 개선 아이디어
- 현재 R²=0.072로 낮음 → 음식 가격대, 업종 클러스터링 등 추가 통제변수 투입
- 업종별 분리 회귀로 FPI 효과 크기 비교
- Moran's I 검정으로 공간적 자기상관 보정
- 분절 회귀(piecewise regression)로 FPI 비선형 효과 탐색

---

## `06_text_analysis.ipynb`

### 1. 파일명 및 목적
FPI 구간(NP/LP/HP)별 소비자 언어 패턴을 TF-IDF 차이 방식으로 비교하고, PCA Biplot으로 공간적 시각화. 분석 대상: 영업 중 독립 브랜드 리뷰 751,890개.

### 2. 주요 데이터셋

| 입력 | 내용 |
|---|---|
| `review_target.csv` | 929,606개 리뷰 |
| `biz_indie_with_groups.csv` | FPI 구간 정보 |

구간별 리뷰: NP 108,443개 / LP 300,586개 / HP 342,861개

### 3. 핵심 분석 단계

**STEP 7-2 — 텍스트 전처리**

소문자화 → 숫자·구두점 제거 → NLTK 불용어+수동 불용어(73개) 제거 → Porter Stemmer

**STEP 7-4 — 구간별 고유 키워드 추출 (TF-IDF 차이 방식)**

유효 단어 기준: 전체 500회 이상 + 순수 알파벳 3자 이상 → **4,236개**

```python
# 특정 구간 TF-IDF - 나머지 구간 평균 TF-IDF
diff = (target_score - other_score).sort_values(ascending=False)
```

**구간별 고유 키워드 상위 (차이값 기준)**

| NP (무풍지대) | LP (저압력) | HP (고압력) |
|---|---|---|
| thai (0.0399) | pizza (0.0228) | burger (0.0431) |
| coffe (0.0352) | steak (0.0205) | drink (0.0184) |
| pho (0.0338) | sushi (0.0165) | chees (0.0162) |
| buffet (0.0317) | ramen (0.0137) | sandwich (0.0147) |
| noodl (0.0171) | korean (0.0078) | waiter (0.0093) |

**STEP 7-6 — PCA Biplot**: 3,017개 브랜드 × 500단어 TF-IDF → PC1 6.3%, PC2 4.7% (누적 11.0%)  
업종 이질성이 FPI 구간 차이보다 분산을 더 크게 설명 → 보조 시각화로만 활용.

### 4. 주요 결과 및 결론

| 구간 | 언어 패턴 | 해석 |
|---|---|---|
| NP | thai, pho, buffet, noodle, curry | 에스닉 전문 요리 특화 |
| LP | pizza, sushi, ramen, steak, korean | 정통 외국 요리 전문성 |
| HP | waiter, server, experience, burger | 서비스·경험 차별화 지향 |

**핵심 인사이트**: FPI가 높아질수록 메뉴가 프랜차이즈와 유사해지고, 소비자는 서비스 경험으로 독립 브랜드를 평가하는 경향이 강해짐.  
분석 한계: 전체 Restaurants 업종 이질성·카지노 상권 노이즈 → PART2에서 업종 축소로 해결.

### 5. 개선 아이디어
- bigram/trigram 추가로 `"great service"`, `"friendly staff"` 등 구문 단위 분석
- 긍정/부정 리뷰 분리 후 구간별 부정 키워드 차이 분석
- UMAP으로 업종 이질성 노이즈 제거 후 FPI 구간 패턴 재시각화

---

## `07_fpi_map.ipynb`

### 1. 파일명 및 목적
FPI 분석 결과를 Plotly + OpenStreetMap 기반 인터랙티브 지도로 시각화. 창업자·상권 컨설턴트를 위한 실무 도구 제공.

### 2. 주요 데이터셋

| 입력 | 내용 |
|---|---|
| `biz_indie_with_groups.csv` | 영업 중 독립 브랜드 3,017개 |

**핵심 변수**: `latitude`, `longitude`, `fpi_300m`, `fpi_group`, `review_count`, `neighborhood`, `categories`

### 3. 핵심 분석 단계

**STEP 9-1 — 연속형 FPI 지도** (`RdYlGn_r` 컬러맵, 마커 크기=리뷰수)

```python
fig = px.scatter_mapbox(
    indie_map, lat='latitude', lon='longitude',
    color='fpi_300m', color_continuous_scale='RdYlGn_r',
    size='review_count', size_max=20, zoom=11
)
fig.write_html(f"{PATH_to_save}/fpi_map.html")
```

**STEP 9-2 — FPI 구간별 범주형 지도** (NP=파랑, LP=주황, HP=빨강)

**STEP 9-3 — 업종별 FPI 구간 지도 (5개 업종)**

| 업종 | 선정 이유 | 업체 수 |
|---|---|---|
| Pizza | HP 구간 고유 키워드, 프랜차이즈 경쟁 뚜렷 | 322개 |
| Mexican | NP/LP 에스닉 전문점 대표 | 395개 |
| Steakhouses | LP 구간 정통 요리 대표 | 146개 |
| Sushi Bars | 생존 키워드 sushi, roll과 직결 | 146개 |
| Burgers | HP 구간 고유 키워드 1위 | 180개 |

### 4. 주요 결과 및 결론

**공간적 패턴**

| 상권 유형 | FPI 수준 | 추천 전략 |
|---|---|---|
| Strip/Paradise (관광축) | 고압력 (4↑) | 경험형 컨셉, SNS 바이럴, 관광객 특화 |
| 주요 대로변 | 중간 (1~3) | 정통 요리 전문성, 서비스 차별화 |
| 외곽 주거지 | 저압력~무풍 (0~1) | 단골 기반, 커뮤니티형, 에스닉 전문점 |

**업종별 경쟁 강도**: Burgers (가장 어려움) > Steakhouses > Pizza > Sushi Bars > Mexican (가장 유리)

**출력**: `fpi_map.html`, `fpi_map_group.html`, 업종별 5개 HTML

### 5. 개선 아이디어
- KDE 기반 히트맵 레이어 추가로 밀집 패턴 직관적 표현
- 리뷰 작성 연도별 FPI 변화 애니메이션 (시계열 시각화)
- Plotly Dash dropdown으로 업종 필터 통합 대시보드 구현

---

# PART 2: 버거·샌드위치·패스트푸드 심화 분석

> **PART2의 목표**: PART1에서 발생한 업종 이질성 문제를 해결하기 위해 분석 범위를 `Burgers + Sandwiches + Fast Food` 3개 카테고리로 좁혀 FPI 효과를 재검증하고 생존 전략을 규명.

---

## `01_data_prep.ipynb`

### 1. 파일명 및 목적
버거·샌드위치·패스트푸드 업종에 특화된 데이터 준비. 업종 이질성 문제를 해결하기 위해 분석 범위를 축소하고 프랜차이즈/독립 브랜드 판정 기준을 재수립.

### 2. 주요 데이터셋

| 데이터 | 크기 | 주요 변수 |
|---|---|---|
| `yelp_business.csv` (원본) | 174,567행 | `business_id`, `name`, `city`, `categories`, `is_open`, `stars`, `review_count`, `latitude`, `longitude` |
| `biz_target_burger.csv` (산출) | 1,579행 | 위 변수 + `is_franchise` |
| `review_target_burger.csv` (산출) | 178,094행 | 원본 리뷰 구조 동일 |

**파생 변수**: `is_franchise` = 브랜드명 등장 횟수 ≥10 또는 수동 화이트리스트 포함 여부

### 3. 핵심 분석 단계

**STEP 1-1 — 업체 필터링**

```python
target_cats = ['Burgers', 'Sandwiches', 'Fast Food']
mask = (business_raw['city'] == 'Las Vegas') & (
    business_raw['categories'].apply(
        lambda x: any(cat in str(x) for cat in target_cats) if pd.notna(x) else False
    )
)
```

결과: 1,579개 (Burgers 531 / Sandwiches 655 / Fast Food 884, 중복 포함)

**STEP 1-2 — 프랜차이즈 판정 (2단계)**

1단계: 등장 빈도 ≥10 자동 분류 (28개 브랜드)  
2단계: 수동 화이트리스트 35개 브랜드 추가

```python
franchise_names = franchise_by_count | whitelist
biz_lv['is_franchise'] = biz_lv['name'].isin(franchise_names)
```

독립 브랜드 이중 검증: 5회 이상 등장 브랜드 재확인 → Tropical Smoothie Cafe(9), Farmer Boys(7) 등은 지역 체인으로 판단하여 독립 유지.

### 4. 주요 결과 및 결론

| 구분 | 수치 |
|---|---|
| 프랜차이즈 | 804개 (50.9%) |
| 독립 브랜드 전체 | 775개 (49.1%) |
| 독립 브랜드 중 영업 중 | 535개 |
| 분석 대상 리뷰 수 | 178,094개 |

PART1(전체 Restaurants 4,818개 독립 브랜드) 대비 업종을 좁혀 동질적 경쟁 환경 확보.

### 5. 개선 아이디어
- 화이트리스트 동적화: 외부 FTC Franchise Disclosure DB 연동
- 지역 체인(Tropical Smoothie Cafe 등)을 별도 세그먼트로 분리 분석
- 폐업 업체 비율 30.9% 탐색적 분석 추가

---

## `02_sentiment.ipynb`

### 1. 파일명 및 목적
버거·패스트푸드 도메인에 특화된 TF-IDF 기반 감성 사전 재구축 및 VADER 교차 검증.

### 2. 주요 데이터셋

| 데이터 | 크기 | 주요 변수 |
|---|---|---|
| `review_target_burger.csv` | 178,094행 | `business_id`, `text`, `stars` |
| `biz_sentiment_burger.csv` (산출) | 1,579행 × 16열 | 업체 정보 + `tfidf_sentiment`, `vader_sentiment` |

### 3. 핵심 분석 단계

**STEP 2-2 — 텍스트 전처리**

NLTK 불용어 + 커스텀 불용어 73개 추가 + Porter Stemmer + 길이 2 이하 토큰 제거

**STEP 2-3 — 긍정/부정 코퍼스 구성 (3점 제외)**

| 별점 | 분류 | 리뷰 수 |
|---|---|---|
| 4~5점 | 긍정 | 112,419개 |
| **3점** | **제외** | 24,819개 |
| 1~2점 | 부정 | 40,856개 |

**STEP 2-4 — TF-IDF 감성 사전 구축**

```python
pos_top = pos_series.nlargest(300)
neg_top = neg_series.nlargest(300)
overlap = set(pos_top.index) & set(neg_top.index)  # 204개
pos_lexicon = set(pos_top.index) - overlap  # 96개
neg_lexicon = set(neg_top.index) - overlap  # 96개
```

긍정 96개 (crispy, delicious, avocado, garlic...) / 부정 96개 (poor, attitude, charge, employees...)

**STEP 2-7 — VADER 교차 검증**

```python
corr, pval = pearsonr(valid['tfidf_sentiment'], valid['vader_sentiment'])
# corr = 0.7924, p = 0.0000  (PART1 0.724보다 높음)
```

### 4. 주요 결과 및 결론

| 지표 | 값 |
|---|---|
| 업체별 tfidf_sentiment 평균 | -0.0002 |
| TF-IDF vs VADER 상관계수 | **0.7924 ✅** |

업종 특화 사전이 PART1(0.724)보다 높은 상관계수 달성. 도메인 특화 전처리의 효과 확인.

### 5. 개선 아이디어
- 교집합 204개 단순 제거 대신 TF-IDF 점수 차이 기준 가중 할당
- 리뷰 길이 최소 필터링 (5단어 미만 제외)으로 감성점수 분산 안정화

---

## `03_fpi.ipynb`

### 1. 파일명 및 목적
버거·패스트푸드 업종 독립 브랜드의 FPI를 산출하고 민감도 분석으로 임계거리 확정. 775개 독립 브랜드 대상.

### 2. 주요 데이터셋

| 데이터 | 크기 | 주요 변수 |
|---|---|---|
| `biz_sentiment_burger.csv` | 1,579행 × 16열 | `business_id`, `stars`, `review_count`, `tfidf_sentiment`, 좌표, `is_franchise` |
| `biz_indie_with_groups_burger.csv` (산출) | 775행 × 21열 | 위 변수 + `fpi_{r}m`, `fpi_group`, `weighted_score` |

### 3. 핵심 분석 단계

**STEP 3-0 — Shift 정규화**: 프랜차이즈 감성점수 음수값 594개(73.9%) → 최솟값 -0.1109 차감으로 전체 양수 변환

**STEP 3-1 — 하버사인 거리 행렬 (775 × 804 = 623,100쌍)**

**STEP 3-2 — 반경별 FPI 산출 및 커버리지 확인**

| 반경 | FPI>0 업체 수 | 비율 |
|---|---|---|
| 100m | 293개 | 37.8% |
| 200m | 511개 | 65.9% |
| 300m | 592개 | 76.4% |
| 500m | 672개 | 86.7% |

**STEP 3-3 — 민감도 분석**

| 반경 | 별점 p | 감성 p | 복합지수 p | 폐업 p |
|---|---|---|---|---|
| 100m | 0.1226 ❌ | 0.1265 ❌ | 0.8741 ❌ | 0.7870 ❌ |
| **300m** | **0.0147 ✅** | 0.1395 ❌ | 0.7495 ❌ | 0.2272 ❌ |
| 500m | 0.0014 ✅ | 0.1914 ❌ | 0.6812 ❌ | 0.1842 ❌ |

→ 별점 기준 p<0.05 만족하는 최소 반경 **300m 확정**

**STEP 3-4 — NP/LP/HP 분류** (FPI 중앙값 1.9327 기준)

### 4. 주요 결과 및 결론

| 구간 | 업체 수 | 비율 | 평균 별점 |
|---|---|---|---|
| NP | 183개 | 23.6% | **3.70** |
| LP | 296개 | 38.2% | 3.53 |
| HP | 296개 | 38.2% | **3.46** |

**NP → LP → HP 단조 감소 패턴 확인** (PART1 결과 방향 일치).  
PART1 대비 차이: 감성점수는 300m에서 p=0.1395(비유의) → 패스트푸드 리뷰는 경쟁 환경보다 개별 매장 경험(속도, 가격, 직원)에 더 민감.

### 5. 개선 아이디어
- FPI 분자 구성 요소(별점/로그리뷰수/감성) 분산 분해로 기여도 분석
- 50m 단위 연속 탐색으로 임계거리 정밀화

---

## `04_regression.ipynb`

### 1. 파일명 및 목적
FPI가 별점과 폐업여부에 미치는 영향을 검증하고, 매개효과 경로(FPI → 별점 → 폐업, FPI → 리뷰수 → 폐업) 성립 여부를 검토. 두 경로 모두 횡단면 데이터 구조적 한계로 성립하지 않음.

### 2. 주요 데이터셋

| 항목 | 내용 |
|---|---|
| 입력 파일 | `biz_indie_with_groups_burger.csv` |
| 총 관측치 | 775개 (영업 중 535 / 폐업 240) |
| 종속변수 1 | `stars` (OLS) |
| 종속변수 2 | `is_open` (Logistic) |
| 통제변수 | `log_review`, neighborhood 더미 16개, 로지스틱 모델에만 `stars` 추가 |

### 3. 핵심 분석 단계

**STEP 4-2 — OLS + HC3 (FPI → 별점)**

```python
model_stars = sm.OLS(y_stars, X_stars).fit(cov_type='HC3')
```

**STEP 4-3 — Logistic + HC3 (FPI → 폐업여부)**

별점·리뷰수·상권 통제 후 FPI의 순수 직접효과 추출

**매개효과 분석 시도**

FPI → 별점 → 폐업, FPI → 리뷰수 → 폐업 두 가지 경로를 검토했으나 모두 성립하지 않았다.

| 매개 경로 | 경로 a | 경로 b | 성립 여부 |
|---|---|---|---|
| FPI → 별점 → 폐업 | p=0.048 ✅ | 방향 불일치 ❌ | ❌ |
| FPI → 리뷰수 → 폐업 | p=0.413 ❌ | p=0.000 ✅ | ❌ |

별점 계수(coef=-0.300, OR=0.741)가 음수로 나타나 "별점↑ → 영업중확률↓"라는 직관과 반대 결과였다. is_open이 2017년 기준 스냅샷이므로 현재 시점의 별점이 폐업의 선행 지표로 작동하지 않는 횡단면 데이터의 구조적 한계에서 비롯된다.

### 4. 주요 결과 및 결론

**OLS (FPI → 별점)**

| 항목 | 값 |
|---|---|
| FPI 계수 | **-0.0271 (p=0.048 ✅)** |
| R² / Adj R² | 0.0825 / 0.0607 |
| PART1 대비 | 계수 약 **2.3배** 강한 효과 (-0.012 → -0.027) |

**Logistic (FPI → 폐업)**

| 변수 | 계수 | Odds Ratio | p-value |
|---|---|---|---|
| FPI_300m | -0.0342 | 0.966 | 0.401 ❌ |
| 별점 | -0.300 | **0.741** | **0.005 ✅** |
| log(리뷰수) | +0.483 | **1.621** | **<0.001 ✅** |

**매개효과 분석 결과: 성립하지 않음**

| 항목 | 결과 |
|---|---|
| FPI → 별점 (경로 a) | coef=-0.0271, p=0.048 ✅ |
| 별점 → 폐업 (경로 b) | coef=-0.2997, OR=0.741, p=0.005 (방향 불일치 ❌) |
| FPI → 리뷰수 (경로 a') | coef=0.0215, p=0.413 ❌ |
| **종합** | **두 매개 경로 모두 성립하지 않음** |

폐업 브랜드 평균 별점(3.60) > 영업 중 브랜드(3.52): 데이터 자체에서 별점-폐업 선형 관계 미성립. 횡단면 데이터 한계로 인과 경로 검증 불가.

### 5. 개선 아이디어
- 패널 데이터 구축 후 Cox 비례위험 모델로 시간 의존적 폐업 예측 모델 구축
- 공간 회귀 모델(Spatial Lag Model)로 공간적 자기상관 보정
- Moran's I 검정으로 공간 독립성 가정 검증

---

## `05_text_analysis.ipynb`

### 1. 파일명 및 목적
FPI 구간별 언어 패턴 비교 및 **HP 구간 내 생존/고전 브랜드 차별화 요인** 규명. TF-IDF 차이 + N-gram + VADER Intensity + LDA Topic Modeling + 시계열 감성 변화 총 12개 STEP 심층 분석.

### 2. 주요 데이터셋

| 데이터 | 크기 | 내용 |
|---|---|---|
| `biz_indie_with_groups_burger.csv` | 775개 | FPI 구간 정보 |
| `review_burger_cleaned.csv` | 142,507개 | 전처리 완료 리뷰 |
| `review_target_burger.csv` | 178,094개 | 날짜 포함 전체 리뷰 |
| `biz_target_burger.csv` | 1,579개 | 프랜차이즈 포함 전체 업체 |

| 세그먼트 | 기준 | 업체/리뷰 수 |
|---|---|---|
| 생존 브랜드 | HP + 영업 중 + 별점≥4.0 + 리뷰≥10개 | 63개 / 30,913개 |
| 고전 브랜드 | HP + 영업 중 + 별점≤3.0 + 리뷰≥10개 | 65개 / 9,982개 |
| 프랜차이즈 | — | 리뷰 35,587개 |

### 3. 핵심 분석 단계

**STEP 5-4 — 생존 vs 고전 TF-IDF 차이**

- 생존 고유: `great(0.045), delici(0.044), amaz(0.038), flavor(0.025), oxtail(0.024)`
- 고전 고유: `dog(0.110), crepe(0.066), castl(0.057), bad(0.033), never(0.029)`

**STEP 5-5 — N-gram 분석 (Bigram/Trigram)**

```python
vec = CountVectorizer(ngram_range=(n, n), max_features=10000, stop_words='english')
```

| 그룹 | 대표 Bigram | 대표 Trigram |
|---|---|---|
| 생존 | crab leg, highli recommend, chicken waffl | sweet potato fri, oxtail chili chees, truffl parmesan fri |
| 고전 | hot dog, white castl, wait minut | — |

시그니처 메뉴의 복합 표현이 생존 브랜드에서 선명하게 드러남.

**STEP 5-6 — VADER Sentiment Intensity + Mann-Whitney U 검정**

| 지표 | 생존 브랜드 | 고전 브랜드 | p-value |
|---|---|---|---|
| 평균 compound | **0.738** | 0.454 | **0.000 ✅** |
| 25%ile | **0.778** | 0.000 | — |

고전 브랜드 하위 25%가 중립(0) 수준 → 단순 긍정 비율이 아니라 **감정 강도** 자체가 다름.

**STEP 5-7 — LDA Topic Modeling (토픽 수=5)**

| 그룹 | 주요 토픽 구성 |
|---|---|
| 생존 | 특색 메뉴(fri/waffl/sauc), 시그니처 버거(earl/bachi), **서비스 경험(server/seat/table)**, 고급 뷔페(buffet/crab/dessert) |
| 고전 | 단순 샌드위치(pastrami/beef), 저가 버거(white/castl/slider), 핫도그(dog/pink/chili), **서비스 불만(wait/minut/manager/location)** |

핵심 차이: 생존 브랜드 서비스 토픽=`server, seat, tabl` (경험) vs 고전=`wait, minut, manag` (불만)

**STEP 5-8 — 생존 브랜드 vs 프랜차이즈 차별화**

| 구분 | 고유 키워드 | 감성 평균 |
|---|---|---|
| 생존 브랜드 | buffet(0.191), worth(0.045), delici(0.043) | **0.738** |
| 프랜차이즈 | locat(-0.149), fast(-0.081), innout(-0.045), mcdonald(-0.040) | 0.422 |

McDonald's가 소비자의 **최소 기준선**으로 작동: 생존="better than McDonald's" / 고전="McDonald's would have been better"

**STEP 5-9 — 시계열 감성 변화 (2005~2017)**

| 그룹 | VADER 변화 |
|---|---|
| 생존 브랜드 | 0.85 → 0.70 (완만한 하락, -0.15) |
| 고전 브랜드 | 0.75 → 0.35 (급격한 하락, **-0.40**) |

2006~2008년: 고전 브랜드가 일부 구간에서 생존 브랜드 상회 → "한때는 좋았는데" 패턴 데이터 확인

**STEP 5-10 — 프랜차이즈 직접 언급 분석 (만 리뷰당 횟수)**

| 표현 | 생존 | 고전 |
|---|---|---|
| homemade | **90.6** | 52.1 |
| fresh ingredients | **32.7** | 9.0 |
| house made | **29.4** | 3.0 |
| family owned | **13.9** | 2.0 |
| corporate | 17.1 | **61.1** |
| fast food chain | 5.5 | **33.1** |

### 4. 주요 결과 및 결론

5개 분석 방법이 모두 같은 방향을 지시:

| 분석 방법 | 생존 브랜드 핵심 특성 |
|---|---|
| TF-IDF 차이 | 음식 품질 형용사 (delici, amaz, flavor, perfect) |
| N-gram | 시그니처 메뉴 복합어 (crab leg, truffle fry, spicy miso burger) |
| VADER Intensity | 평균 0.738 (고전 0.454 대비 +62% 강한 긍정) |
| LDA Topic Modeling | 음식 전문성 토픽 vs 고전의 운영 불만 토픽 |
| 프랜차이즈 언급 | homemade/house made/family owned 압도적 높음 |
| 시계열 | 완만 하락 vs 고전의 급격 하락 |

HP 구간 생존 경로: ① 유명 셰프 브랜드 파워, ② 독창적 시그니처 메뉴 컨셉

### 5. 개선 아이디어
- **ABSA(Aspect-Based Sentiment Analysis)**: 음식/서비스/가격/분위기 차원 분리 분석
- **BERTopic / word2vec**: 임베딩 기반 토픽 모델로 더 의미론적 군집 도출
- 고전 브랜드 감성 급락 시점 리뷰 군집화로 "무엇이 하락을 촉발했는가" 정량화

---

## `06_fpi_map.ipynb`

### 1. 파일명 및 목적
버거·패스트푸드 독립 브랜드 FPI 분포를 인터랙티브 지도로 시각화. **생존/고전 브랜드 위치 지도**를 추가하여 PART1보다 한 단계 심화된 실무 인사이트 제공.

### 2. 주요 데이터셋

| 항목 | 내용 |
|---|---|
| 입력 파일 | `biz_indie_with_groups_burger.csv` |
| 필터 조건 | `is_open==1` 영업 중 535개 |

FPI 분포: 평균 1.89, 중앙값 1.27, 최대 14.89, 90th percentile 4.70

### 3. 핵심 분석 단계

**STEP 6-1 — 연속형 FPI 지도 (90th percentile 기준 색상 스케일)**

```python
fig = px.scatter_mapbox(
    indie_map, lat='latitude', lon='longitude',
    color='fpi_300m', color_continuous_scale='RdYlGn_r',
    range_color=[0, indie_map['fpi_300m'].quantile(0.90)],  # 극단값 왜곡 방지
    size='review_count', size_max=20, zoom=11
)
fig.write_html(f"{PATH_to_save}/fpi_map_burger.html")
```

**STEP 6-2 — FPI 구간별 지도** (NP=파랑, LP=주황, HP=빨강)

**STEP 6-3 — 생존/고전 브랜드 위치 지도**

```python
survivor_map = hp_all[(hp_all['stars'] >= 4.0) & (hp_all['review_count'] >= 10)]  # 63개
struggle_map = hp_all[(hp_all['stars'] <= 3.0) & (hp_all['review_count'] >= 10)]  # 65개
```

### 4. 주요 결과 및 결론

**PART1과의 비교**

| 항목 | PART1 (전체 Restaurants) | PART2 (버거+패스트푸드) |
|---|---|---|
| 분석 업체 수 | 4,818개 | 535개 |
| HP 패턴 | Strip + 교통축 선형 | Strip 중심 **집중** (선형 약화) |
| Strip 의존도 | 보통 | **높음** (관광객 유동인구 민감) |

**핵심 인사이트**: 동일 HP 구역(Strip) 내에서 생존/고전 브랜드가 혼재 → **입지보다 브랜드 전략(시그니처 메뉴, 수제·오너십)** 이 생존을 결정.

**출력**: `fpi_map_burger.html` / `fpi_map_burger_group.html` / `fpi_map_burger_survival.html`

### 5. 개선 아이디어
- KDE 히트맵 레이어 추가
- 폐업 업체 레이어 추가 ("어느 상권에서 폐업 집중?" 공간 패턴)
- 반경 toggle 레이어로 300m/500m/1km 비교 시각화

---

## 분석 전체 결론 요약

| 검증 항목 | PART1 (전체 Restaurants) | PART2 (버거·패스트푸드) |
|---|---|---|
| FPI → 별점 | β=-0.0119, p=0.033 ✅ | **β=-0.0271, p=0.048 ✅** (효과 2.3배) |
| FPI → 감성점수 | p=0.384 ❌ (통제 후) | p=0.1395 ❌ |
| FPI → 폐업 직접효과 | — | p=0.401 ❌ |
| FPI → 별점 → 폐업 매개효과 | — | **❌ 성립하지 않음 (횡단면 데이터 한계)** |
| 임계거리 | 300m | 300m |
| VADER 검증 상관계수 | 0.724 | **0.792** |

**전략적 시사점**

1. **FPI는 별점에 실재하는 부(-)의 영향**: 업종을 좁힐수록 신호가 강해짐 (PART1→PART2 계수 2.3배 증가)
2. **FPI의 폐업 직접 효과 없음(p=0.401)**: 매개효과 경로도 횡단면 데이터 구조적 한계로 성립하지 않음. 폐업의 핵심 변수는 리뷰수(OR=1.621)
3. **HP 구간 생존 브랜드의 전략**: 시그니처 메뉴 + homemade/house made 강조 + 강한 긍정 감정 강도
4. **입지보다 전략이 결정적**: 동일 Strip 상권 내에서 생존/고전 브랜드 혼재 → 위치 자체가 운명을 결정하지 않음
5. **에스닉·전문 요리가 경쟁압력 회피 전략**: NP/LP 구간의 thai, pho, sushi 등 전문화된 독립 브랜드가 생존에 유리
