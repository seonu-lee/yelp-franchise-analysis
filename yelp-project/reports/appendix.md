# 부록 (Appendix)
## 프랜차이즈 경쟁 압력 분석 — 보조 분석 및 추가 결과

---

## A. 데이터 파이프라인 상세

### A-1. 원본 데이터 기본 통계

| 항목 | 수치 |
|---|---|
| yelp_business.csv 전체 행 수 | 174,566개 |
| yelp_review.csv 전체 행 수 | 5,261,668개 |
| business 컬럼 수 | 13개 |
| review 컬럼 수 | 9개 |
| business 결측치 제거 행 | 1개 (위도/경도 결측) |
| review 결측치 | 0개 |
| business 평균 별점 | 3.63점 |
| business 평균 리뷰 수 | 30.14개 (중앙값: 8개, 최대: 7,361개) |

### A-2. 도시별 업체 수 전체 현황 (상위 20개)

| 순위 | 도시 | 주(State) | 업체 수 | 전체 리뷰 수 |
|---|---|---|---|---|
| 1 | Las Vegas | NV | 26,764 | 1,603,616 |
| 2 | Phoenix | AZ | 17,213 | 576,700 |
| 3 | Toronto | ON | 17,204 | 430,957 |
| 4 | Charlotte | NC | 8,553 | 237,308 |
| 5 | Scottsdale | AZ | 8,227 | 308,564 |
| 6 | Pittsburgh | PA | 6,354 | 179,515 |

### A-3. 업종별 프랜차이즈 비율 (Las Vegas, 전체 업체 기준)

| 업종 | 전체 | 프랜차이즈 | 비율 |
|---|---|---|---|
| Fast Food | 884 | 615 | 69.6% |
| Burgers | 531 | 225 | 42.4% |
| Sandwiches | 655 | 261 | 39.8% |
| Coffee & Tea | 505 | 180 | 35.6% |
| **Restaurants** | **5,899** | **925** | **15.7%** |

---

## B. 프랜차이즈 판정 상세

### B-1. 브랜드명 수동 병합 목록 (13건)

| 원본 브랜드명 | 병합 결과 |
|---|---|
| einstein bros bagels | einstein bros |
| arbys roast beef sandwich restaurants | arbys |
| little caesars pizza | little caesars |
| little ceasars pizza | little caesars |
| little ceasars | little caesars |
| dairy queen grill chill | dairy queen |
| dairy queen orange julius | dairy queen |
| dq grill chill | dairy queen |
| popeyes lousiana kitchen | popeyes louisiana kitchen |
| popeyes famous fried chicken | popeyes louisiana kitchen |
| carls jr restaurants | carls jr |
| jimmy johns gourmet sandwiches | jimmy johns |
| capriottis | capriottis sandwich shop |

### B-2. 화이트리스트 브랜드 목록 (12개, PART 1)

데이터 내 지점 수 10개 미만이지만 전국 체인으로 판단하여 수동 추가:

| 브랜드 | 데이터 내 지점 수 | 추가 근거 |
|---|---|---|
| Cafe Rio | 9 | 미국 멕시칸 패스트캐주얼 체인 |
| Raising Cane's | 9 | 미국 치킨 파이 체인 |
| Fatburger | 9 | 미국 캘리포니아 기반 버거 체인 |
| Teriyaki Madness | 9 | 미국 테리야키 체인 |
| Long John Silver's | 8 | 미국 해산물 패스트푸드 체인 |
| TGI Fridays | 8 | 미국 패밀리 레스토랑 체인 |
| Wienerschnitzel | 8 | 미국 핫도그 체인 |
| Applebee's | 8 | 미국 캐주얼 다이닝 체인 |
| Papa Murphy's | 8 | 미국 테이크앤베이크 피자 체인 |
| California Pizza Kitchen | 7 | 미국 피자 레스토랑 체인 |
| Chili's | 7 | 미국 텍스멕스 레스토랑 체인 |
| Checkers | 7 | 미국 드라이브스루 버거 체인 |

### B-3. PART 2 화이트리스트 브랜드 (35개 추가)

PART 2에서는 자동 판정(빈도 ≥10) 28개 + 화이트리스트 35개를 추가하여 총 프랜차이즈 브랜드 수를 확정했다. 주요 추가 브랜드: McDonald's 관련 변형명, Subway 변형명, 지역 체인 구분 등.

독립 브랜드 이중 검증: 5회 이상 등장 브랜드를 재확인하여 Tropical Smoothie Cafe(9개), Farmer Boys(7개) 등은 지역 체인으로 판단하여 독립 브랜드로 유지했다.

---

## C. 감성 분석 상세

### C-1. TF-IDF 감성 사전 구성 (PART 1)

**긍정 사전 (84개 예시)**:
`awesome`, `tasty`, `crispy`, `reasonable`, `fresh`, `delicious`, `friendly`, `authentic`, `flavorful`, `tender`

**부정 사전 (84개 예시)**:
`awful`, `cold`, `dirty`, `salty`, `rude`, `bland`, `greasy`, `stale`, `overpriced`, `disgusting`

**교집합 제거**: 300개 + 300개 = 600개 후보 중 양쪽 모두 등장한 216개 교집합 제거

### C-2. TF-IDF 감성 사전 구성 (PART 2 — 버거 특화)

**긍정 사전 (96개 예시)**:
`crispy`, `delicious`, `avocado`, `garlic`, `flavor`, `juicy`, `fresh`, `amazing`

**부정 사전 (96개 예시)**:
`poor`, `attitude`, `charge`, `employees`, `bland`, `cold`, `frozen`, `overcooked`

**교집합 제거**: 204개 제거 후 긍정 96개 + 부정 96개 확정

### C-3. DTM / TF-IDF 구축 파라미터

| 파라미터 | 값 | 선택 근거 |
|---|---|---|
| MIN_DF_RATE | 0.1 | 전체 브랜드의 10% 미만 등장 단어 제거 |
| MAX_DF_RATE | 0.9 | 전체 브랜드의 90% 초과 등장 단어 제거 |
| MIN_DOC_COUNT | 10 | 최소 10개 브랜드 이상 등장 |
| min_review_prop | 0.3 | 브랜드 내 리뷰 비율 30% 이상 |
| 최종 어휘 수 | 902개 | — |

min_review_prop = 0.1(2,288개) 대비 0.3(902개)이 노이즈 감소 효과 우수하여 채택.

---

## D. FPI 산출 상세

### D-1. Shift 정규화 수치

| 항목 | PART 1 | PART 2 |
|---|---|---|
| 감성점수 최솟값 | −0.1114 | −0.1109 |
| 정규화 후 범위 | 0.000 ~ 0.156 | 0.000 ~ 유사 |
| 정규화 전 음수 업체 수 | — | 594개 (73.9%) |

### D-2. 하버사인 거리 계산

```python
# 하버사인 공식
dlat = np.radians(lat2 - lat1)
dlon = np.radians(lon2 - lon1)
a = np.sin(dlat/2)**2 + np.cos(lat1)*np.cos(lat2)*np.sin(dlon/2)**2
distances_m = 2 * np.arcsin(np.sqrt(a)) * 6371000
```

- PART 1: 4,818 × 1,081 ≈ 521만 쌍
- PART 2: 775 × 804 = 623,100쌍

### D-3. FPI 커버리지 분석 (PART 2)

| 반경 | FPI > 0 업체 수 | 비율 |
|---|---|---|
| 100m | 293개 | 37.8% |
| 200m | 511개 | 65.9% |
| 300m | 592개 | 76.4% |
| 500m | 672개 | 86.7% |

300m에서 76.4%의 업체가 최소 1개 이상의 프랜차이즈를 반경 내에 보유한다.

---

## E. 회귀분석 상세

### E-1. OLS + HC3 전체 모델 요약 (PART 1 — 별점)

| 항목 | 값 |
|---|---|
| 모델 | OLS + HC3 Robust SE |
| 종속변수 | stars |
| 관측치 | 3,017개 (is_open=1 필터링) |
| FPI 계수 | −0.0119 |
| FPI p-value | 0.033 |
| log(review_count) 계수 | 양수, 유의미 |
| R² | 0.0717 |
| Adj R² | 0.0661 |
| F-stat | 12.86 |
| F p-value | < 0.001 |

### E-2. OLS + HC3 전체 모델 요약 (PART 1 — 감성점수)

| 항목 | 값 |
|---|---|
| FPI 계수 | −0.0001 |
| FPI p-value | 0.384 (비유의) |
| R² | 0.0632 |
| F p-value | < 0.001 |

### E-3. OLS + HC3 전체 모델 요약 (PART 2 — 별점)

| 항목 | 값 |
|---|---|
| 관측치 | 775개 (전체 독립 브랜드) |
| FPI 계수 | −0.0271 |
| FPI p-value | 0.048 |
| 95% CI | [−0.054, −0.000] |
| R² | 0.0825 |
| Adj R² | 0.0607 |
| F p-value | < 0.001 |

### E-4. Logistic 회귀 상세 (PART 2 — 폐업여부)

| 변수 | 계수 | OR | p-value |
|---|---|---|---|
| fpi_300m | −0.0342 | 0.966 | 0.401 |
| stars | −0.2997 | 0.741 | 0.005 |
| log(review_count) | +0.4828 | 1.621 | < 0.001 |
| Pseudo R² | 0.0843 | — | — |
| 관측치 | 775개 | — | — |

**별점 계수 방향 해석**: 종속변수가 is_open(1=영업 중)이므로 별점 계수 음수(−0.300)는 "별점 높을수록 영업 중 확률 낮아짐"을 의미한다. 이는 직관에 반하는 결과로, 데이터 수집 시점(2017년) 이전 폐업 브랜드와 현재 영업 중 브랜드의 횡단면 혼재로 발생한 구조적 아티팩트다.

### E-5. Neighborhood별 평균 별점 및 FPI (PART 2)

| Neighborhood | 업체 수 | 평균 별점 | 평균 FPI |
|---|---|---|---|
| The Strip | 138 | 3.37 | 3.34 |
| Southeast | 96 | 3.30 | 1.68 |
| Westside | 87 | 3.72 | 1.43 |
| Spring Valley | 77 | 3.62 | 2.65 |
| Eastside | 68 | 3.57 | 1.48 |
| Downtown | 66 | 3.79 | 0.86 |
| Southwest | 33 | 3.29 | 0.87 |

The Strip의 FPI(3.34)는 Downtown(0.86)의 약 4배로, 상권별 FPI 수준이 크게 다름을 확인했다.

---

## F. 텍스트 분석 상세

### F-1. PART 1 FPI 구간별 TF-IDF 상위 키워드 (전체)

**NP 구간 상위 10개**

| 키워드 | TF-IDF 차이값 |
|---|---|
| thai | 0.0399 |
| coffe | 0.0352 |
| pho | 0.0338 |
| buffet | 0.0317 |
| noodl | 0.0171 |
| sushi | 0.0165 |
| curry | 0.0148 |
| dim | 0.0130 |
| ramen | 0.0112 |
| dumplings | 0.0108 |

**HP 구간 상위 10개**

| 키워드 | TF-IDF 차이값 |
|---|---|
| burger | 0.0431 |
| drink | 0.0184 |
| chees | 0.0162 |
| sandwich | 0.0147 |
| waiter | 0.0093 |
| server | 0.0087 |
| experi | 0.0082 |
| atmospher | 0.0078 |
| bar | 0.0072 |
| cocktail | 0.0068 |

### F-2. PART 2 생존 vs 고전 TF-IDF 차이 상위 키워드

**생존 브랜드 (HP + stars ≥ 4.0)**

| 키워드(stem) | TF-IDF 차이값 |
|---|---|
| great | 0.0450 |
| delici | 0.0440 |
| amaz | 0.0380 |
| flavor | 0.0250 |
| oxtail | 0.0240 |
| worth | 0.0210 |
| perfect | 0.0190 |
| fresh | 0.0180 |
| tender | 0.0170 |
| recommend | 0.0160 |

**고전 브랜드 (HP + stars ≤ 3.0)**

| 키워드(stem) | TF-IDF 차이값 |
|---|---|
| dog | 0.1100 |
| crepe | 0.0660 |
| castl | 0.0570 |
| bad | 0.0330 |
| never | 0.0290 |
| wait | 0.0270 |
| minut | 0.0250 |
| manag | 0.0230 |
| employ | 0.0210 |
| worst | 0.0180 |

### F-3. LDA 토픽 모델링 상세 (PART 2, 토픽 수=5)

**생존 브랜드 5개 토픽**

| 토픽 번호 | 주요 단어 | 해석 |
|---|---|---|
| T1 | fri, waffle, sauc, egg, chicken | 특색 있는 조리 방법·메뉴 |
| T2 | earl, bachi, burger, spici, miso | 시그니처 버거 브랜드 |
| T3 | server, seat, tabl, staff, servic | 긍정적 서비스 경험 |
| T4 | buffet, crab, dessert, prime, rib | 고급 뷔페 경험 |
| T5 | fresh, ingredi, local, homemad, qualiti | 식재료 품질·수제 정체성 |

**고전 브랜드 5개 토픽**

| 토픽 번호 | 주요 단어 | 해석 |
|---|---|---|
| T1 | pastrami, beef, sandwich, deli, meat | 단순 샌드위치 |
| T2 | white, castl, slider, burger, small | 저가 슬라이더 버거 |
| T3 | dog, pink, chili, hot, frank | 핫도그 중심 메뉴 |
| T4 | wait, minut, manag, locat, employe | 서비스 불만·운영 문제 |
| T5 | buffet, line, open, close, hour | 뷔페 운영·대기 불만 |

### F-4. 고전 브랜드 시기별 키워드 변화 상세

| 시기 | 기간 | 평균 VADER | 대표 키워드 |
|---|---|---|---|
| 초기 | 2005~2008 | 0.682 | buffet, crab leg, prime rib, crepe, fresh, lobster |
| 중기 | 2009~2013 | 0.530 | decent, okay, average, disappoint, expect |
| 후기 | 2014~2017 | 0.382 | white castle, employee, staff, frozen, chain, corporate |

리뷰 수 변화: 초기 297개 → 후기 6,582개 (+2,116% 증가)

---

## G. 방법론적 선택과 대안 검토

### G-1. 감성 점수 방법론 비교

| 방법 | 특징 | 채택 여부 |
|---|---|---|
| TF-IDF 자체 사전 (채택) | 도메인 특화, VADER와 r=0.724~0.792 | ✅ 채택 |
| VADER | 교차 검증에 사용, 일반 목적 | 검증 도구로 활용 |
| TextBlob | 영어 일반 목적 | 미채택 (도메인 특화 부족) |
| BERT 기반 | 고성능, 고비용 | 미채택 (컴퓨팅 리소스) |

### G-2. 이분산 처리 방법 비교

| 방법 | 특징 | 선택 여부 |
|---|---|---|
| OLS + HC3 (채택) | 계수 유지, 표준오차만 보정 | ✅ 채택 |
| WLS | log_review를 통제변수로 투입한 상태에서 가중치로도 사용 시 이중 처리 모순 | ❌ 미채택 |
| Bootstrap SE | 계산 비용 높음 | 미채택 |

### G-3. 프랜차이즈 판정 기준 비교

| 기준 | 프랜차이즈 업체 수 | 장단점 |
|---|---|---|
| ≥3개 | 1,598개 | 소규모 다점포 오분류 위험 높음 |
| ≥5개 | 1,278개 | 여전히 오분류 가능성 존재 |
| **≥10개 (채택)** | **984개** | 명확한 프랜차이즈 안정적 포함 |
| ≥15개 | 815개 | 일부 실제 프랜차이즈 누락 위험 |
| ≥20개 | 746개 | 프랜차이즈 과소 추정 위험 |

---

## H. 시각화 결과 목록

### 생성된 보고서용 시각화 (figures/ 디렉토리)

| 파일명 | 내용 |
|---|---|
| fig1_city_comparison.png | 후보 도시 업체 수 및 평균 리뷰 수 비교 |
| fig2_fpi_formula.png | FPI 공식 구성 요소 다이어그램 |
| fig3_sensitivity.png | 임계거리 민감도 분석 (PART 1 & 2) |
| fig4_anova_results.png | FPI 구간별 평균 별점 비교 (PART 1 vs 2) |
| fig5_regression.png | OLS 회귀분석 계수 비교 |
| fig6_sentiment_comparison.png | VADER 감성 강도 비교 (3그룹) |
| fig7_franchise_mention.png | 수제·독립성 vs 체인 관련 표현 빈도 |
| fig8_temporal_trend.png | 시계열 감성 추세 (2005~2017) |
| fig9_decline_pathway.png | 고전 브랜드 하락 경로 재구성 |
| fig10_plan_vs_actual.png | 계획 대비 수행 결과 |

### 노트북 생성 시각화 (results/ 디렉토리)

| 파일명 | 내용 |
|---|---|
| step4_burger_mediation.png | 매개효과 분석 시도 결과 |
| step5_burger_fpi_keywords.png | FPI 구간별 키워드 (PART 2) |
| step5_burger_franchise_mention.png | 프랜차이즈 언급 분석 |
| step5_burger_lda.png | LDA 토픽 모델 |
| step5_burger_ngram.png | N-gram 분석 결과 |
| step5_burger_sentiment_intensity.png | VADER 감성 강도 분포 |
| step5_burger_survival_tfidf.png | 생존 vs 고전 TF-IDF 차이 |
| step5_burger_survivor_temporal.png | 생존 브랜드 시간적 감성 |
| step5_burger_temporal_change.png | 시계열 감성 변화 |
| step5_burger_timeseries.png | 연도별 VADER 추세 |
| step5_burger_vs_franchise.png | 생존 브랜드 vs 프랜차이즈 |
| fpi_map_burger.html | 연속형 FPI 인터랙티브 지도 |
| fpi_map_burger_group.html | FPI 구간별 인터랙티브 지도 |
| fpi_map_burger_survival.html | 생존/고전 브랜드 위치 지도 |

---

## I. 재현 가능성 (Reproducibility)

모든 분석은 `notebooks/` 디렉토리의 Jupyter 노트북으로 완전히 재현 가능하다.

| 노트북 | 산출물 |
|---|---|
| PART1/01_data_setup.ipynb | results/biz_target.csv, results/review_target.csv |
| PART1/02_dtm_tfidf.ipynb | DTM / TF-IDF 행렬 (메모리) |
| PART1/03_sentiment.ipynb | results/biz_sentiment.csv |
| PART1/04_FPI.ipynb | results/biz_indie_with_groups.csv |
| PART1/05_regression.ipynb | 회귀분석 결과 출력 |
| PART1/06_text_analysis.ipynb | TF-IDF 구간별 키워드 출력 |
| PART1/07_fpi_map.ipynb | results/fpi_map.html, fpi_map_group.html |
| PART2/01_data_prep.ipynb | results/biz_target_burger.csv, review_target_burger.csv |
| PART2/02_sentiment.ipynb | results/biz_sentiment_burger.csv |
| PART2/03_fpi.ipynb | results/biz_indie_with_groups_burger.csv |
| PART2/04_regression.ipynb | 회귀분석 결과 출력 |
| PART2/05_text_analysis.ipynb | results/step5_burger_*.png |
| PART2/06_fpi_map.ipynb | results/fpi_map_burger*.html |

---

*부록 작성일: 2026년 6월*
