# 초기 분석 계획 대비 실제 진행 내용과 변경 이유를 정리

---

## 전체 분석 구조 개요

```
PART 1 — 방법론 검증 (Las Vegas Restaurants 전체)
    FPI 개념 설계 및 검증
    임계거리 확정, 회귀분석, FPI 지도

PART 2 — 업종 특화 심화 분석 (패스트푸드 업종)
    PART 1의 한계(업종 이질성) 해결
    매개효과 분석, 텍스트 심화 분석
```

---

## PART 1 — Restaurants 전체 분석

### 분석 목적
Las Vegas 전체 음식점 데이터를 대상으로
FPI(프랜차이즈 경쟁압력 지수) 개념을 설계하고 검증한다.

### STEP별 진행 내용

#### STEP 1~2. 데이터 준비
- 분석 대상: Las Vegas Restaurants 업체 5,899개 / 리뷰 929,606개
- 프랜차이즈 판정: 브랜드명 10개 이상 등장 + 화이트리스트
  - 프랜차이즈 1,081개 / 독립 브랜드 4,818개

#### STEP 3~4. 텍스트 전처리 및 감성점수 산출 
- 전처리: 소문자화, 불용어 제거, Porter Stemmer 어간 추출
- TF-IDF 감성점수 산출 방식 (팀 자체 설계)
  - 긍정(4~5점) / 부정(1~2점) 코퍼스 각각 TF-IDF 상위 300개 추출
  - 교집합 제거 → 순수 긍정/부정 사전 확정
  - 감성점수 = (긍정단어 수 - 부정단어 수) / 전체 단어 수
- VADER 검증: 상관계수 0.724 ✅

#### STEP 5. FPI 산출 및 민감도 분석 
- **수정 사항**: 기존 FPI 계산에서 감성점수 음수로 인한 FPI 음수 문제 발견
  - Shift 정규화 적용: `sentiment_shifted = sentiment - min(sentiment)`
  - 음수 제거 + 상대적 간격 보존

**FPI 공식**

$$FPI = \sum \frac{(\text{별점} \times \ln(\text{리뷰수}+1) \times \text{감성점수}_{shifted})}{(\text{거리(km)}+k)^2}$$

- 반경별(100m/200m/300m/500m) 민감도 분석 수행
- **임계거리 확정: 300m** (별점 + 감성점수 두 종속변수 모두 p<0.05인 최소 반경)
- FPI 구간 분류
  - NP (무풍지대): fpi=0 → 977개 (20.3%)
  - LP (저압력): 중앙값 이하 → 1,921개 (39.9%)
  - HP (고압력): 중앙값 초과 → 1,920개 (39.9%)

#### STEP 6. 영향 분석 
- 분석 방법: OLS + HC3 Robust Standard Error
- 사전 필터링: is_open=1 (영업 중 3,017개)
- 통제변수: log(review_count), neighborhood 더미

**회귀분석 결과**

| 종속변수 | FPI 계수 | p-value | 유의미 |
|---|---|---|---|
| 별점 | -0.0119 | 0.033 | ✅ |
| 감성점수 | -0.0001 | 0.384 | ❌ |

**ANOVA 결과 (별점)**

| 구간 | 평균 별점 |
|---|---|
| NP (무풍지대) | 3.83 |
| LP (저압력) | 3.69 |
| HP (고압력) | 3.61 |

#### STEP 7~8. 텍스트 분석 및 생존 브랜드 분석
> **분석 한계 및 전환 배경**
>
> 전체 Restaurants 대상 텍스트 분석에서 두 가지 문제가 발생했다.
>
> 1. **업종 이질성**: 태국 음식, 스시, 스테이크, 버거 등 다양한 업종이 혼재하여
>    키워드가 업종 특성을 반영할 뿐 FPI 구간의 순수한 차이를 보기 어려웠다.
>
> 2. **카지노 상권 노이즈**: 생존/고전 브랜드 비교에서
>    hotel, casino, mgm 등 카지노 관련 키워드가 지속적으로 등장하여
>    음식/서비스 품질 차이를 가리는 문제가 있었다.
>
> 이를 해결하기 위해 **PART 2에서 패스트푸드 업종으로 범위를 좁혀
> 동일한 분석을 수행**하였다.

#### STEP 9. FPI 지도 
- 산출물: fpi_map.html (연속형), fpi_map_group.html (구간별), 업종별 5개
- Strip/Paradise 중심 고압력 집중, 외곽 저압력 분산 패턴 확인

---

## PART 2 — 패스트푸드 업종 심화 분석

### 분석 목적
PART 1의 업종 이질성 문제를 해결하고
동일 업종 내에서 FPI 효과와 생존 전략을 더 정교하게 분석한다.

### 업종 정의 근거
분석 범위는 패스트푸드(QSR, Quick Service Restaurant) 업종으로 제한하였다.
Yelp 데이터에서는 패스트푸드 관련 업종이 `Fast Food`, `Burgers`, `Sandwiches` 등으로 분리되어 있어, 동일한 소비 및 경쟁 메커니즘을 공유하는 업종을 포괄하기 위해 세 카테고리를 모두 포함하였다.

반면 Pizza, Chicken 등 일부 업종은 배달 중심 소비나 저녁·가족 단위 수요 비중이 높아 입지 및 경쟁 구조가 상이할 가능성이 있어 본 분석 범위에서 제외하였다.

### STEP별 진행 내용

#### STEP 1. 데이터 준비
- 필터링: Burgers + Sandwiches + Fast Food 카테고리 + Las Vegas
- 전체 1,579개 / 프랜차이즈 804개(50.9%) / 독립 775개
- 영업 중 독립 브랜드: 535개
- **주목**: restaurant(18.3%) 대비 프랜차이즈 비율 50.9%로 압도적으로 높음

#### STEP 2. 감성점수 산출
- PART 1과 동일한 방식 적용
- **VADER 상관계수: 0.792** (PART 1 0.724보다 높음)
  → 업종을 좁히면 리뷰 언어 일관성이 높아져 감성점수 신뢰도 향상

#### STEP 3. FPI 산출
- PART 1과 동일한 공식 + Shift 정규화 적용
- 임계거리 300m 재확인 (별점 p=0.015 ✅, 감성점수 전 반경 ❌)
  - PART 1과 달리 감성점수는 유의미하지 않음
  - 버거 업종 리뷰 언어가 경쟁 환경보다 개별 매장 경험에 더 민감하기 때문으로 해석
- FPI 구간: NP 183개(23.6%) / LP 296개(38.2%) / HP 296개(38.2%)

#### STEP 4. 회귀분석 (신규: 매개효과 분석 추가)
- PART 1 대비 **폐업여부(is_open) 분석 추가**
- 분석 설계

| 모델 | 종속변수 | 통제변수 |
|---|---|---|
| OLS + HC3 | 별점 | log(review_count), neighborhood 더미 |
| Logistic + HC3 | 폐업여부 | stars, log(review_count), neighborhood 더미 |

**모델 1 — FPI → 별점**

| 항목 | PART 1 | PART 2 |
|---|---|---|
| FPI 계수 | -0.012 | **-0.027** |
| p-value | 0.033 ✅ | 0.048 ✅ |
| R² | 0.0717 | 0.0825 |

업종을 좁히니 FPI 효과가 2.3배 강하게 나타남.

**모델 2 — FPI → 폐업여부 (Logistic)**
- FPI 직접 효과: p=0.401 ❌
- 별점(OR=0.741, p=0.005) ✅ / 리뷰수(OR=1.621, p=0.000) ✅

**매개효과 분석 (Bootstrap 5,000회)**

```
경로 a: FPI → 별점       coef=-0.027, p=0.048 ✅
경로 b: 별점 → 폐업      coef=-0.300, p=0.005 ✅
경로 c': FPI → 폐업 직접  p=0.401 ❌
간접효과 (a×b): 0.0081, 95% CI [0.0002, 0.0205] ✅
```

> **완전 매개 확인**: FPI는 별점을 매개로 폐업에 간접적으로 영향을 미친다.

#### STEP 5. 텍스트 분석 (PART 1 대비 방법론 대폭 확장)

| 방법 | PART 1 | PART 2 |
|---|---|---|
| TF-IDF 차이 | ✅ | ✅ |
| N-gram | ❌ | ✅ |
| Sentiment Intensity | ❌ | ✅ |
| LDA Topic Modeling | ❌ | ✅ |
| 시계열 분석 | ❌ | ✅ |
| 프랜차이즈 직접 언급 | ❌ | ✅ |
| 시기별 원인 파악 | ❌ | ✅ |

**5-2. FPI 구간별 키워드**

| 구간 | 고유 키워드 | 해석 |
|---|---|---|
| NP | pizza, sandwich, coffee, truck, vegan | 로컬 특색, 푸드트럭 |
| LP | burger, egg, bbq, breakfast, steak | 정통 음식 전문성 |
| HP | buffet, burger, line, wait, bacchanal | 뷔페+대기+카지노 |

**5-4. 생존 vs 고전 브랜드 키워드 (TF-IDF 차이)**

| 그룹 | 키워드 |
|---|---|
| 생존 | great, delici, amaz, flavor, dish |
| 고전 | buffet, line, wait, bad, manag |

**5-5. N-gram 분석**
- 생존: `crab leg`, `sweet potato fry`, `truffle parmesan fry`, `bachi burger`
  → 시그니처 메뉴 전문성
- 고전: `hot dog`, `white castle`, `wait minute`, `long line`
  → 특색 없는 저가 메뉴 + 대기 불만

**5-6. Sentiment Intensity**

| 지표 | 생존 브랜드 | 고전 브랜드 |
|---|---|---|
| 평균 VADER | **0.738** | 0.454 |
| 25%ile | **0.778** | 0.000 |
| Mann-Whitney p | **0.000 ✅** | - |

고전 브랜드 리뷰 25%가 중립(0) 이하. 감정 강도 자체가 다름.

**5-7. LDA Topic Modeling (토픽 5개)**

| 그룹 | 주요 토픽 |
|---|---|
| 생존 | 특색 메뉴 / 시그니처 버거 / 서비스 경험 / 전반적 만족 / 고급 뷔페 |
| 고전 | 단순 샌드위치 / 저가 버거(White Castle) / 핫도그 / 서비스 불만 / 뷔페 |

**5-8. 생존 브랜드 vs 프랜차이즈**
- 생존 차별화: homemade(90.6회/만), house made(29.4회/만), owner(180.8회/만)
- 프랜차이즈: locat(-0.167), fast, drive, manag
- Sentiment: 생존 0.738 vs 프랜차이즈 0.422

**5-9. 시계열 분석**

| | 생존 브랜드 | 고전 브랜드 |
|---|---|---|
| 추세 | 완만한 하락 (-0.078) | **급격한 하락 (-0.300)** |
| 변곡점 | 없음 (안정적) | 2006~2008 (초기 우위 후 급락) |

> "한때는 좋았는데" 패턴 확인.
> 고전 브랜드는 처음부터 나쁜 게 아니라 시간이 지나면서 품질 유지에 실패했다.

**5-10. 프랜차이즈 직접 언급 분석**

| 표현 | 생존 (만리뷰당) | 고전 (만리뷰당) |
|---|---|---|
| homemade | 90.6 | 52.1 |
| fresh ingredients | 32.7 | 9.0 |
| house made | 29.4 | 3.0 |
| owner | 180.8 | 95.2 |
| corporate | 17.1 | **61.1** |
| fast food chain | 5.5 | **33.1** |

**McDonald's 언급 맥락 (실제 리뷰 샘플링으로 확인)**
- 생존: "way better than McDonald's", "blows McDonald's out of the water"
- 고전: "McDonald's would have been better", "reminded me of plain McDonald's"

> McDonald's가 소비자의 **최소 기준선**으로 작동.
> 생존은 이 기준선을 넘었고, 고전은 넘지 못했다.

**5-11. 고전 브랜드 시기별 변화 (원인 파악)**

| 시기 | 평균 VADER | 주요 키워드 |
|---|---|---|
| 초기 (2005~2008) | 0.682 | buffet, crab leg, prime rib, crepe (차별화 메뉴) |
| 후기 (2014~2017) | 0.382 | white castle, employe, staff, frozen (저가화+직원 문제) |

실제 리뷰 확인:
> "this place has gone downhill from years ago. in the past it was really awesome... now it's cafeteria style"

고전 브랜드 하락 경로:
```
초기 차별화 메뉴 (뷔페·델리·씨푸드)
    ↓
리뷰 급증 (소비자 유입, 297개 → 6,582개)
    ↓
품질 유지 실패 (employe, staff, manag 불만)
    ↓
저가 메뉴로 전환 (white castle, frozen)
    ↓
"McDonald's보다 못하다" (기준선 미달)
    ↓
도태
```

**5-12. 생존 브랜드 전체 특성**
- 상위 20개 중 12개가 The Strip (Gordon Ramsay Burger, Eggslut, Bachi Burger 등)
- HP 구간 생존의 두 가지 경로 확인
  1. **셰프 브랜드 전략**: Gordon Ramsay, Bobby's Burger Palace
  2. **시그니처 메뉴 전략**: Bachi Burger, Eggslut, Dog Haus

#### STEP 6. FPI 지도
- 산출물: fpi_map_burger.html, fpi_map_burger_group.html, fpi_map_burger_survival.html
- 패스트푸드 업종은 팀플 대비 Strip 의존도가 더 높음
- 동일 HP 상권 안에서 생존/고전 브랜드 혼재 → 입지보다 브랜드 전략이 생존을 결정

---

## 최종 인사이트 요약

### 연구질문별 답변

| 연구질문 | 답변 |
|---|---|
| Q1. FPI → 별점 | FPI 높을수록 별점 낮아짐 (PART1: -0.012, PART2: -0.027) |
| Q2. 임계거리 | 300m (두 분석 모두 동일) |
| Q3. FPI → 폐업 | 직접 효과 없음. 별점을 완전 매개로 간접 영향 |
| Q4. 생존 전략 | 시그니처 메뉴 + 수제·오너십 + 장기적 품질 유지 |
| Q5. 고전 원인 | 초기 차별화 → 품질 유지 실패 → 저가화 → 도태 |

### PART 1 vs PART 2 비교

| 항목 | PART 1 | PART 2 |
|---|---|---|
| FPI 효과 | -0.012 | **-0.027 (2.3배)** |
| 감성점수 VADER 상관 | 0.724 | **0.792** |
| 매개효과 | 미분석 | **완전 매개 ✅** |
| 텍스트 분석 방법 | TF-IDF만 | **6가지 방법** |
| 인사이트 | 일반적 | **업종 특화, 구체적** |

> 업종을 좁힐수록 FPI 효과가 더 강하게 나타나고,
> 텍스트 분석에서 훨씬 구체적이고 처방적인 인사이트가 도출된다.

---

## 노트북 파일 구조

```
notebooks/
├── PART1_restaurants/
│   ├── 01_data_setup.ipynb       ← 데이터 준비 (내 담당)
│   ├── 02_dtm_tfidf.ipynb        ← DTM/TF-IDF  
│   ├── 03_sentiment.ipynb        ← 감성점수 
│   ├── 04_fpi.ipynb              ← FPI 산출 (타팀원->내가 보완)
│   ├── 05_regression.ipynb       ← 회귀분석 (타팀원->내가 보완)
│   ├── 06_text_analysis.ipynb    ← 텍스트 분석 (내 담당)
│   ├── 07_survival.ipynb         ← 생존 브랜드 (내 담당)
│   └── 08_fpi_map.ipynb          ← FPI 지도 (내 담당)
│
└── PART2_fastfood/               ← 업종 특화 심화 분석 (내 담당)
    ├── 01_data_prep.ipynb
    ├── 02_sentiment.ipynb
    ├── 03_fpi.ipynb
    ├── 04_regression.ipynb
    ├── 05_text_analysis.ipynb
    └── 06_fpi_map.ipynb
```