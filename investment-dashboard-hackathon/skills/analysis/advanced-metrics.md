# Advanced Metrics — Time-Series & Risk Indicators

> Requires trade history data (daily return time series).
> If no trade history exists, skip this entire module and show basic metrics only.
> Source: `analysis_rules.md` Section 4 (4.0 - 4.6)

---

## 1. Daily Returns Generation (Preprocessing)

```yaml
daily_returns_generation:
  description: "거래 내역에서 일별 포트폴리오 수익률 시계열을 생성하는 전처리 규칙"
  input: "거래 내역 DataFrame (date, ticker, quantity, price 등)"

  steps:
    step_1_daily_portfolio_value:
      description: "각 날짜별 전체 포트폴리오 평가금액 계산"
      formula: "sum(quantity_i * price_i) for all stocks on each date"
      missing_price_fill: "forward fill (마지막 유효 가격으로 채움)"
      forward_fill_limit:
        max_days: 30  # 30일 초과 gap은 forward fill 금지
        action_on_exceed: "mark_as_delisted_or_suspended"
        display: "거래 정지 또는 상장 폐지 의심 — 최종 거래일: {last_trade_date}"
        # description: 상장폐지/거래정지 종목의 가격이 영구 유지되는 것 방지

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

  exceptions:
    gaps_in_dates:
      description: "거래일 사이 빈 날짜 존재 (주말/휴일)"
      action: "거래일 기준으로만 계산 (빈 날짜 무시)"
    single_day:
      action: "일별 수익률 시계열 생성 불가. 고급 지표 전체 비활성화."
    deposit_withdrawal:
      description: "입출금으로 인한 포트폴리오 가치 변동"
      action: "시간가중수익률(TWR) 방식으로 입출금 효과 제거"
      formula: "각 입출금 구간별 수익률을 기하평균으로 연결"
      twr_calculation:
        step_1: "Identify sub-periods by deposit/withdrawal events"
        step_2: "For each sub-period: HPR_i = (end_value - cash_flow) / start_value - 1"
        step_3: "Geometric linking: TWR = product(1 + HPR_i) - 1"
        step_4: "Annualize: TWR_annual = (1 + TWR)^(365/total_days) - 1"
        deposit_detection:
          column: "invested_amount changes between consecutive records"
          threshold: "change > 1% of previous invested_amount"
        fallback_if_no_history: "use simple return (current_value / invested_amount - 1)"
        # description: 입출금이 있는 포트폴리오의 정확한 수익률. 단순수익률은 입출금 시 왜곡됨
```

---

## 2. Data Sufficiency Check

```yaml
data_sufficiency:
  min_data_points: 20
  min_period_days: 30

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

---

## 3. Sharpe Ratio

```yaml
metric: sharpe_ratio
description: "위험 대비 초과수익 측정"
formula: "(annualized_return - risk_free_rate) / annualized_std"

parameters:
  risk_free_rate:
    source_priority:
      1: api_lookup  # 한국은행/FRED API에서 최신 기준금리 조회
      2: manual_override  # 사용자 지정값
      3: fallback_static  # API 실패 시 고정값
    fallback_values:
      KR: 0.035  # 최종 갱신: 2025-04-01
      US: 0.045  # 최종 갱신: 2025-04-01
    staleness_warning_days: 30
    staleness_message: "무위험수익률이 {days}일 전 기준입니다. Sharpe/Sortino 비율이 부정확할 수 있습니다."
    display_source: true  # 리포트에 출처 및 기준일 표시
    # description: 금리 변동 시 Sharpe/Sortino가 전부 틀어지므로 동적 조회 우선
  period: "daily"
  annualize_factor: 252     # trading days
  annualized_return: "mean(daily_returns) * 252"
  annualized_std: "std(daily_returns) * sqrt(252)"

interpretation:
  excellent: { range: "> 2.0", label: "우수", color: "#27680A" }
  good:      { range: "1.0 ~ 2.0", label: "양호", color: "#639922" }
  average:   { range: "0.5 ~ 1.0", label: "보통", color: "#BA7517" }
  poor:      { range: "0.0 ~ 0.5", label: "미흡", color: "#D85A30" }
  negative:  { range: "< 0.0", label: "부진", color: "#E24B4A" }

exceptions:
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

annualization_method:
  standard: geometric  # (1 + cumulative_return)^(252/trading_days) - 1
  arithmetic_only_for: "sharpe_numerator"  # Sharpe는 mean(daily) * 252 (학계 관행)
  note: "Calmar, Sortino의 annualized_return은 반드시 geometric 방식 사용"
  # description: 산술/기하 연환산 혼용 방지. 기간이 길수록 차이가 커짐
```

---

## 4. MDD (Maximum Drawdown)

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
  moderate:  { range: "-20% < MDD <= -10%", label: "보통", color: "#BA7517" }
  high_risk: { range: "-30% < MDD <= -20%", label: "높음", color: "#D85A30" }
  extreme:   { range: "MDD <= -30%", label: "매우 높음", color: "#E24B4A" }

exceptions:
  always_increasing:
    condition: "MDD == 0 (한 번도 하락한 적 없음)"
    action: "MDD = 0% 표시, '고점 대비 하락 없음' 메시지"
  single_day_data:
    action: "MDD = N/A"
```

---

## 5. Volatility

```yaml
metric: volatility
description: "수익률의 표준편차 (연환산)"
formula: "std(daily_returns) * sqrt(252)"
unit: "%"
precision: 2

interpretation:
  low:      { range: "< 15%", label: "저변동" }
  moderate: { range: "15% ~ 25%", label: "보통" }
  high:     { range: "25% ~ 40%", label: "고변동" }
  extreme:  { range: "> 40%", label: "초고변동" }

exceptions:
  zero_volatility:
    condition: "모든 수익률이 동일"
    action: "0% 표시. 데이터 오류일 수 있으므로 경고."
  insufficient_data:
    condition: "데이터 포인트 < 10"
    action: "계산 가능하나 '참고용' 표시"
```

---

## 6. Beta

```yaml
metric: beta
description: "시장 대비 민감도"
formula: "cov(stock_returns, market_returns) / var(market_returns)"

benchmark:
  KR: { default: "KOSPI", alternatives: ["KOSDAQ", "KOSPI200"] }
  US: { default: "S&P 500", alternatives: ["NASDAQ", "DOW"] }
  MIXED: "비중이 높은 시장의 벤치마크 사용"

interpretation:
  very_defensive:  { range: "< 0.5", label: "매우 방어적" }
  defensive:       { range: "0.5 ~ 0.8", label: "방어적" }
  neutral:         { range: "0.8 ~ 1.2", label: "시장 중립" }
  aggressive:      { range: "1.2 ~ 1.5", label: "공격적" }
  very_aggressive: { range: "> 1.5", label: "매우 공격적" }
  negative:        { range: "< 0", label: "역상관", note: "시장과 반대로 움직임" }

exceptions:
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

---

## 7. Additional Advanced Metrics

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
