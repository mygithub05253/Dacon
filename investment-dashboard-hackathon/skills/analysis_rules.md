# Analysis Rules — 투자 지표 분석 규칙

> AlphaFolio 지표 계산 계층의 규칙을 정의합니다.
> 이 문서를 참조하여 정규화된 데이터에서 투자 지표를 계산합니다.
>
> **파이프라인 위치**: Stage 2 (parsing_rules.md 출력을 소비)
> **입력**: parsing_rules.md의 표준 스키마(standard_schema) DataFrame
> **출력**: 종목별·포트폴리오 지표가 부착된 DataFrame + 고급 지표 딕셔너리
> **하위 의존**: visualization_rules.md와 insight_rules.md가 이 문서의 지표명을 참조합니다.

---

## 1. 종목별 기본 지표

### 1.1 수익률 (Return)
```yaml
metric: return_rate
description: "개별 종목의 수익률"
formula: "(current_price - avg_price) / avg_price * 100"
unit: "%"
display_format: "+12.5%" or "-3.2%"
precision: 2  # 소수점 둘째 자리
color_rule:
  positive: "green"    # > 0
  negative: "red"      # < 0
  zero: "gray"         # == 0
exception:
  avg_price_zero:
    condition: "avg_price == 0"
    action: "return_rate = N/A, 해당 종목 수익률 관련 차트에서 제외"
    message: "'{name}' 매수가가 0이어서 수익률을 계산할 수 없습니다."
  current_price_missing:
    condition: "current_price가 NaN"
    action: "return_rate = N/A, 포트폴리오 전체 수익률 계산에서도 제외"
    ui_display: "현재가 없음"
  extreme_return:
    condition: "abs(return_rate) > 500"
    action: "경고 표시. 액면분할/병합/데이터 오류 가능성."
    message: "수익률이 {return}%입니다. 데이터를 확인해주세요."
    include_in_calculation: true  # 계산은 포함하되 시각적 경고
```

### 1.2 평가금액 (Valuation)
```yaml
metric: valuation
description: "현재 보유 가치"
formula: "current_price * quantity"
unit: "KRW or USD (currency 기준)"
display_format: "₩1,234,567" or "$12,345.67"
precision: 0  # 원화는 정수, 달러는 소수점 2자리
exception:
  current_price_missing:
    action: "avg_price * quantity 로 대체 (수익률 0% 가정)"
    flag: "estimated"
    display: "₩1,234,567 (추정)"
  quantity_zero:
    action: "valuation = 0, 비중 계산에서 제외"
    note: "매도 완료 종목. '과거 보유 종목' 섹션에 별도 표시."
  mixed_currency:
    description: "KRW + USD 혼합 포트폴리오"
    action: "각 통화별 소계 + 환산 총계 모두 계산"
    display: "원화: ₩XX / 달러: $XX / 합산: ₩XX (환율 적용)"
```

### 1.3 손익금액 (Profit/Loss)
```yaml
metric: profit_loss
description: "절대 수익/손실 금액"
formula: "(current_price - avg_price) * quantity"
unit: "KRW or USD"
display_format: "+₩500,000" or "-$1,234"
exception:
  any_price_missing:
    condition: "avg_price 또는 current_price가 NaN"
    action: "profit_loss = N/A"
  mixed_currency:
    action: "각 통화 단위로 별도 계산 후 환산 합산"
```

### 1.4 포트폴리오 비중 (Weight)
```yaml
metric: weight
description: "포트폴리오 내 비중"
formula: "valuation / total_valuation * 100"
unit: "%"
precision: 1
display_format: "23.4%"
threshold:
  concentration_warning: 30    # 30% 초과
  concentration_danger: 50     # 50% 초과
  min_meaningful: 0.1          # 0.1% 미만은 "0.1% 미만"으로 표시
exception:
  total_valuation_zero:
    condition: "전체 평가금액이 0"
    action: "모든 비중 = N/A. 수량 기반 균등 비중으로 대체 표시."
    message: "평가금액을 계산할 수 없어 수량 기준으로 비중을 표시합니다."
  single_stock:
    condition: "종목 1개"
    action: "weight = 100%. 비중 차트 비활성화."
  rounding_mismatch:
    description: "비중 합계가 반올림으로 100%와 차이"
    action: "가장 큰 비중 종목에 잔여분 배분 (Largest Remainder Method)"
```

---

## 2. 포트폴리오 전체 지표

### 2.1 총 수익률
```yaml
metric: total_return
description: "포트폴리오 가중평균 수익률"
formula: "sum(return_rate_i * weight_i / 100) for all stocks"
alternative: "(total_valuation - total_investment) / total_investment * 100"
unit: "%"
precision: 2
exception:
  partial_data:
    description: "일부 종목만 수익률 계산 가능"
    action: "계산 가능한 종목만으로 가중평균 (가중치 재조정)"
    message: "{total}개 중 {available}개 종목으로 수익률을 계산했습니다."
  total_investment_zero:
    action: "total_return = N/A"
  all_returns_na:
    action: "총 수익률 표시 불가"
    ui: "KPI 카드에 '—' 표시"
```

### 2.2 총 평가금액 / 투자금액 / 손익
```yaml
metrics:
  total_valuation:
    formula: "sum(valuation_i)"
    exception:
      mixed_currency: "각 통화별 합산 후 base_currency로 환산"
      some_missing: "계산 가능한 종목만 합산, '일부 추정' 태그"
  total_investment:
    formula: "sum(invested_amount_i)"  # invested_amount = avg_price * quantity (parsing_rules.md 참조)
    exception:
      mixed_currency: "매수 시점 환율 불명 → 현재 환율로 일괄 환산 + 경고"
  total_profit_loss:
    formula: "total_valuation - total_investment"
    exception:
      mixed_currency:
        action: "base_currency 환산 후 차액"
        warning: "환율 변동 수익/손실이 포함되어 있습니다."
```

### 2.3 종목 수 통계
```yaml
metric: stock_counts
items:
  total: "전체 종목 수"
  profitable: "수익 종목 수 (return > 0)"
  losing: "손실 종목 수 (return < 0)"
  breakeven: "보합 종목 수 (return == 0)"
  na_return: "수익률 미산출 종목 수"
  kr_stocks: "국내 종목 수"
  us_stocks: "해외 종목 수"
  other_stocks: "기타 시장 종목 수"
  zero_quantity: "보유량 0 종목 (매도 완료)"
display:
  format: "수익 {profitable} / 손실 {losing} / 보합 {breakeven}"
  win_rate: "profitable / (profitable + losing) * 100"
  win_rate_label: "승률"
```

---

## 3. 집계 분석

### 3.1 섹터별 분석
```yaml
aggregation: sector_analysis
group_by: "sector"
metrics:
  - sector_weight: "sum(weight) per sector"
  - sector_return: "weighted_avg(return) per sector"
  - sector_valuation: "sum(valuation) per sector"
  - stock_count: "count per sector"
  - sector_profit_loss: "sum(profit_loss) per sector"
sort_by: "sector_weight DESC"
exception:
  no_sector_data:
    condition: "모든 종목의 섹터가 '기타' 또는 NaN"
    action: "섹터 분석 섹션 전체를 비활성화"
    message: "섹터 정보가 부족하여 섹터별 분석을 제공할 수 없습니다."
  single_sector:
    condition: "섹터가 1개뿐"
    action: "섹터 차트 대신 '단일 섹터' 안내 텍스트 표시"
  unknown_sector_high:
    condition: "'기타' 섹터 비중 > 50%"
    action: "경고 표시"
    message: "섹터 분류가 안 된 종목이 많아 분석이 제한적입니다."
```

### 3.2 마켓별 분석
```yaml
aggregation: market_analysis
group_by: "market"
metrics:
  - market_weight: "sum(weight) per market"
  - market_return: "weighted_avg(return) per market"
  - market_valuation: "sum(valuation) per market"
  - currency_exposure: "통화별 비중"
sort_by: "market_weight DESC"
exception:
  single_market:
    condition: "모든 종목이 같은 시장"
    action: "마켓 비교 차트 비활성화. '국내 전용' 또는 '해외 전용' 레이블 표시."
  other_market_exists:
    condition: "KR/US 외의 시장 종목 존재"
    action: "'기타' 그룹으로 합산"
```

### 3.3 수익/손실 구간 분석
```yaml
aggregation: return_distribution
buckets:
  - label: "대손실 (< -20%)"
    range: [-inf, -20]
    color: "#A32D2D"
  - label: "소손실 (-20% ~ -5%)"
    range: [-20, -5]
    color: "#E24B4A"
  - label: "소폭 손실 (-5% ~ 0%)"
    range: [-5, 0]
    color: "#F09595"
  - label: "보합 (0%)"
    range: [0, 0]
    color: "#B4B2A9"
  - label: "소폭 수익 (0% ~ 5%)"
    range: [0, 5]
    color: "#C0DD97"
  - label: "소수익 (5% ~ 20%)"
    range: [5, 20]
    color: "#97C459"
  - label: "대수익 (> 20%)"
    range: [20, inf]
    color: "#27680A"
exception:
  all_same_bucket:
    condition: "모든 종목이 같은 구간"
    action: "구간 분포 차트 대신 '전 종목 {bucket_label}' 텍스트 표시"
  na_returns:
    condition: "수익률 N/A 종목 존재"
    action: "별도 '미분류' 구간으로 표시"
```

---

## 4. 고급 지표 (거래 내역 필요)

> 아래 지표는 거래 내역 데이터(일별 수익률 시계열)가 있을 때만 계산합니다.
> 거래 내역이 없으면 이 섹션은 건너뛰고, 기본 지표만 표시합니다.

### 4.0 일별 수익률 시계열 생성
```yaml
daily_returns_generation:
  description: "거래 내역에서 일별 포트폴리오 수익률 시계열을 생성하는 전처리 규칙"
  input: "거래 내역 DataFrame (date, ticker, quantity, price 등)"
  
  steps:
    step_1_daily_portfolio_value:
      description: "각 날짜별 전체 포트폴리오 평가금액 계산"
      formula: "sum(quantity_i * price_i) for all stocks on each date"
      missing_price_fill: "forward fill (마지막 유효 가격으로 채움)"
    
    step_2_daily_return:
      description: "일별 수익률 계산"
      formula: "(portfolio_value_t - portfolio_value_t-1) / portfolio_value_t-1"
      first_day: "수익률 = 0 (기준점)"
    
    step_3_cumulative_return:
      description: "누적 수익률 계산"
      formula: "(1 + daily_return).cumprod() - 1"
    
    step_4_running_max:
      description: "MDD 계산을 위한 누적 최고점 추적"
      formula: "cumulative_value.cummax()"

  output:
    columns: ["date", "daily_return", "cumulative_return", "portfolio_value", "running_max"]
    index: "date (datetime)"
  
  exception:
    gaps_in_dates:
      description: "거래일 사이 빈 날짜 존재 (주말/휴일)"
      action: "거래일 기준으로만 계산 (빈 날짜 무시)"
    single_day:
      action: "일별 수익률 시계열 생성 불가. 고급 지표 전체 비활성화."
    deposit_withdrawal:
      description: "입출금으로 인한 포트폴리오 가치 변동"
      action: "시간가중수익률(TWR) 방식으로 입출금 효과 제거"
      formula: "각 입출금 구간별 수익률을 기하평균으로 연결"
```

### 4.1 데이터 충분성 검사
```yaml
data_sufficiency:
  min_data_points: 20     # 최소 20일치 데이터 필요
  min_period_days: 30     # 최소 30일 기간
  rules:
    sharpe_ratio:
      min_points: 20
      reason: "표준편차 신뢰도를 위해 최소 20 데이터 포인트 필요"
    mdd:
      min_points: 5
      reason: "최대 낙폭은 5개 이상의 시점 필요"
    beta:
      min_points: 30
      reason: "공분산/분산 계산에 최소 30개 데이터 필요"
    volatility:
      min_points: 10
      reason: "표준편차 계산에 최소 10개 데이터 필요"
  insufficient_data:
    action: "해당 지표 비활성화 + 이유 표시"
    message: "데이터가 {actual}일치로 부족합니다. {metric} 계산에 최소 {required}일이 필요합니다."
    ui: "지표 카드에 자물쇠 아이콘 + 툴팁"
```

### 4.2 샤프 비율 (Sharpe Ratio)
```yaml
metric: sharpe_ratio
description: "위험 대비 초과수익 측정"
formula: "(annualized_return - risk_free_rate) / annualized_std"
parameters:
  risk_free_rate: 3.5     # 한국 기준금리 기반 (연율, %)
  period: "daily"
  annualize_factor: 252   # 거래일 기준 연환산
  annualized_return: "mean(daily_returns) * 252"
  annualized_std: "std(daily_returns) * sqrt(252)"
interpretation:
  excellent: { range: "> 2.0", label: "우수", color: "#27680A" }
  good: { range: "1.0 ~ 2.0", label: "양호", color: "#639922" }
  average: { range: "0.5 ~ 1.0", label: "보통", color: "#BA7517" }
  poor: { range: "0.0 ~ 0.5", label: "미흡", color: "#D85A30" }
  negative: { range: "< 0.0", label: "부진", color: "#E24B4A" }
exception:
  zero_std:
    condition: "annualized_std == 0 (모든 일별 수익률 동일)"
    action: "sharpe_ratio = N/A"
    message: "변동성이 0이어서 샤프 비율을 계산할 수 없습니다."
  very_short_period:
    condition: "데이터 기간 < 60일"
    action: "계산하되 '참고용' 표시"
    message: "단기 데이터 기반 추정치입니다."
  mixed_market_rate:
    condition: "국내+해외 혼합 포트폴리오"
    action: "국내 비중이 높으면 한국 기준금리, 해외 비중 높으면 미국 국채 수익률 사용"
    us_risk_free_rate: 4.5
```

### 4.3 MDD (Maximum Drawdown)
```yaml
metric: mdd
description: "최대 낙폭 — 고점 대비 최대 하락률"
formula: "min((cumulative_value - running_max) / running_max) * 100"
unit: "%"
display: "항상 음수 또는 0으로 표시 (예: -15.3%)"
additional_info:
  mdd_start_date: "고점 날짜"
  mdd_end_date: "저점 날짜"
  mdd_recovery_date: "회복 날짜 (있으면)"
  mdd_duration: "낙폭 기간 (거래일)"
interpretation:
  low_risk: { range: "MDD > -10%", label: "안정적", color: "#639922" }
  moderate: { range: "-20% < MDD <= -10%", label: "보통", color: "#BA7517" }
  high_risk: { range: "-30% < MDD <= -20%", label: "높음", color: "#D85A30" }
  extreme: { range: "MDD <= -30%", label: "매우 높음", color: "#E24B4A" }
exception:
  always_increasing:
    condition: "MDD == 0 (한 번도 하락한 적 없음)"
    action: "MDD = 0% 표시, '고점 대비 하락 없음' 메시지"
  single_day_data:
    action: "MDD = N/A"
```

### 4.4 변동성 (Volatility)
```yaml
metric: volatility
description: "수익률의 표준편차 (연환산)"
formula: "std(daily_returns) * sqrt(252)"
unit: "%"
precision: 2
interpretation:
  low: { range: "< 15%", label: "저변동" }
  moderate: { range: "15% ~ 25%", label: "보통" }
  high: { range: "25% ~ 40%", label: "고변동" }
  extreme: { range: "> 40%", label: "초고변동" }
exception:
  zero_volatility:
    condition: "모든 수익률이 동일"
    action: "0% 표시. 데이터 오류일 수 있으므로 경고."
  insufficient_data:
    condition: "데이터 포인트 < 10"
    action: "계산 가능하나 '참고용' 표시"
```

### 4.5 베타 (Beta)
```yaml
metric: beta
description: "시장 대비 민감도"
formula: "cov(stock_returns, market_returns) / var(market_returns)"
benchmark:
  KR: { default: "KOSPI", alternatives: ["KOSDAQ", "KOSPI200"] }
  US: { default: "S&P 500", alternatives: ["NASDAQ", "DOW"] }
  MIXED: "비중이 높은 시장의 벤치마크 사용"
interpretation:
  very_defensive: { range: "< 0.5", label: "매우 방어적" }
  defensive: { range: "0.5 ~ 0.8", label: "방어적" }
  neutral: { range: "0.8 ~ 1.2", label: "시장 중립" }
  aggressive: { range: "1.2 ~ 1.5", label: "공격적" }
  very_aggressive: { range: "> 1.5", label: "매우 공격적" }
  negative: { range: "< 0", label: "역상관", note: "시장과 반대로 움직임" }
exception:
  benchmark_unavailable:
    condition: "벤치마크 수익률 데이터를 가져올 수 없음"
    action: "베타 계산 불가 표시"
    fallback: "더미 벤치마크 데이터 사용 가능 옵션 제공"
  market_zero_variance:
    condition: "벤치마크 수익률 분산 == 0"
    action: "beta = N/A"
  different_periods:
    condition: "포트폴리오와 벤치마크의 기간이 다름"
    action: "겹치는 기간만 사용하여 계산. 기간 차이 경고."
```

### 4.6 추가 고급 지표
```yaml
additional_metrics:
  sortino_ratio:
    description: "하방 리스크 대비 초과수익 (하락 변동성만 고려)"
    formula: "(annualized_return - risk_free_rate) / downside_std"
    note: "일별 수익률 중 음수만 추출하여 표준편차 계산"
    exception:
      no_negative_returns: "downside_std = 0 → sortino = N/A"

  calmar_ratio:
    description: "MDD 대비 연환산 수익률"
    formula: "annualized_return / abs(mdd)"
    exception:
      mdd_zero: "calmar = N/A (낙폭 없음)"

  win_rate:
    description: "일별 수익률 기준 양수일 비율"
    formula: "count(daily_return > 0) / total_days * 100"
    unit: "%"

  avg_win_loss_ratio:
    description: "평균 수익일 수익률 / 평균 손실일 수익률 (절대값)"
    formula: "mean(positive_returns) / abs(mean(negative_returns))"
    exception:
      no_loss_days: "ratio = inf → '전일 수익' 표시"
      no_win_days: "ratio = 0 → '전일 손실' 표시"
```

---

## 5. 벤치마크 비교

```yaml
benchmarks:
  KR:
    - name: "KOSPI"
      description: "국내 대표 지수 (유가증권시장)"
      default: true
      ticker: "^KS11"
    - name: "KOSDAQ"
      description: "코스닥 지수"
      ticker: "^KQ11"
    - name: "KOSPI200"
      description: "대형주 200종 지수"
      ticker: "^KS200"
  US:
    - name: "S&P 500"
      description: "미국 대형주 500종 지수"
      default: true
      ticker: "^GSPC"
    - name: "NASDAQ"
      description: "나스닥 종합 지수"
      ticker: "^IXIC"
    - name: "DOW"
      description: "다우존스 산업평균"
      ticker: "^DJI"

  selection_rule:
    single_market: "해당 시장의 default 벤치마크"
    mixed_market: "비중 기준 주력 시장의 벤치마크 + 보조 시장 벤치마크 병기"

  comparison_metrics:
    - excess_return:
        formula: "portfolio_return - benchmark_return"
        label: "초과수익률"
    - tracking_error:
        formula: "std(daily_excess_return) * sqrt(252)"
        label: "추적 오차"
    - information_ratio:
        formula: "annualized_excess_return / tracking_error"
        label: "정보 비율"
        exception:
          tracking_error_zero: "information_ratio = N/A"

  exception:
    benchmark_data_missing:
      action: "벤치마크 비교 섹션 전체 비활성화"
      message: "벤치마크 데이터를 가져올 수 없어 비교 분석을 제공할 수 없습니다."
      fallback: "벤치마크 없이 절대 수익률만 표시"
    period_mismatch:
      description: "포트폴리오 기간과 벤치마크 기간 불일치"
      action: "겹치는 기간만 비교. 제외 기간 명시."
```

---

## 6. 분석 모드 자동 전환

```yaml
analysis_modes:
  description: "데이터 특성에 따라 분석 깊이를 자동 조절"

  full_mode:
    condition: "보유 현황 + 거래 내역 모두 존재, 종목 5개 이상, 30일+ 데이터"
    features: "모든 기본 지표 + 고급 지표 + 벤치마크 비교 + 전체 시각화"

  standard_mode:
    condition: "보유 현황만 존재, 종목 3개 이상"
    features: "기본 지표 + 집계 분석 + 기본 시각화"
    disabled: "시계열 차트, 샤프/MDD/베타, 벤치마크 비교"

  minimal_mode:
    condition: "보유 현황만 존재, 종목 1~2개"
    features: "종목별 수익률, 평가금액, 손익만 표시"
    disabled: "분산도 분석, 섹터 분석, 대부분의 차트"

  trade_history_mode:
    condition: "거래 내역만 존재 (보유 현황 없음)"
    action: "거래 내역에서 현재 보유 현황을 재계산"
    calculation: "매수 수량 합산 - 매도 수량 합산 = 현재 보유량"
    avg_price_calc: "가중평균 매수가 (FIFO 또는 이동평균)"

  mode_display:
    show_current_mode: true
    upgrade_suggestion: "더 상세한 분석을 위해 거래 내역도 업로드해보세요."
```

---

## 7. 확장 규칙

```yaml
extension_points:
  new_asset_metrics:
    crypto:
      additional_metrics:
        - market_cap_weight: "시가총액 기준 비중"
        - staking_yield: "스테이킹 수익률 (APY)"
        - impermanent_loss: "LP 비영구적 손실"
        - dominance: "전체 크립토 시장 대비 비중"
      modified_parameters:
        annualize_factor: 365  # 24/7 거래 → 365일
        risk_free_rate: 0      # 크립토 기준금리 없음

    bond:
      additional_metrics:
        - ytm: "만기수익률 (Yield to Maturity)"
        - duration: "듀레이션 (금리 민감도)"
        - modified_duration: "수정 듀레이션"
        - coupon_income: "이자 수익 (연간)"
        - accrued_interest: "경과 이자"
      modified_parameters:
        return_includes_coupon: true  # 쿠폰 수익 포함
        benchmark: "국고채 금리 곡선"

    real_estate:
      additional_metrics:
        - cap_rate: "자본환원율 (NOI / 매입가)"
        - nav_premium: "NAV 대비 프리미엄/할인율"
        - dividend_yield: "배당수익률"
        - occupancy_rate: "임대율"
      modified_parameters:
        annualize_factor: 12   # 월간 데이터 기준

    fund:
      additional_metrics:
        - alpha: "젠센의 알파 (초과수익)"
        - r_squared: "R² (벤치마크 설명력)"
        - expense_impact: "보수 차감 영향"
        - tracking_difference: "벤치마크 추적 차이"

  dividend_handling:
    description: "배당금 처리 규칙"
    modes:
      total_return: "배당 재투자 가정 (총수익률)"
      price_return: "배당 제외 (가격 수익률만)"
    default: "total_return"
    detection: "trade_type에 DIVIDEND가 있으면 배당 내역 포함"

  stock_split_handling:
    description: "액면분할/병합 처리"
    detection: "같은 종목의 가격이 전일 대비 40% 이상 급변"
    action: "경고 표시. 자동 보정은 하지 않음 (데이터 신뢰성 이슈)."
    message: "'{name}' 가격이 급변했습니다. 액면분할/병합이 있었다면 매수가를 조정해주세요."

  custom_benchmark:
    description: "사용자 정의 벤치마크 추가 가능"
    input_format: "CSV (date, return 컬럼)"
    validation: "날짜 형식 검증, 수익률 범위 검증 (-100~+500%)"
```
