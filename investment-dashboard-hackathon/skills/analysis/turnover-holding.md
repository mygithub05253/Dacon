# Turnover & Holding Period Analysis

> Stage 2 supplement (requires trade_history mode).
> Input: Full trade records (BUY/SELL with dates and amounts).
> Output: Turnover ratio, avg holding period, churning flags.
> Referenced by: `insight/pattern-rules` (과매매 alerts), `tax-fee-impact` (fee estimation).
> Recommended model: sonnet

---

## 1. Turnover Ratio

```yaml
turnover_ratio:
  description: "포트폴리오 회전율 — 매매 빈도 정량화"
  formula: "total_sold_value / avg_portfolio_value * 100"
  unit: "%"
  precision: 1

  parameters:
    total_sold_value: "분석 기간 내 총 매도 금액"
    avg_portfolio_value: "월말 포트폴리오 평가금액의 평균"
    annualization: "available_period 기준으로 연환산"
    annualize_formula: "turnover_raw * (365 / period_days)"

  interpretation:
    low:
      range: "< 50%"
      label: "낮은 회전율 (장기투자형)"
      color: "#639922"
    moderate:
      range: "50% ~ 200%"
      label: "보통 회전율"
      color: "#BA7517"
    high:
      range: "200% ~ 500%"
      label: "높은 회전율 — 수수료 영향 확인 필요"
      color: "#D85A30"
    excessive:
      range: "> 500%"
      label: "과도한 회전율 — 과매매 의심"
      color: "#E24B4A"

  display: "percentage + 해석 텍스트 + 추정 수수료 영향"
```

---

## 2. Fee Impact Estimation

```yaml
fee_impact:
  description: "매매 비용이 수익에 미치는 영향 추정"

  components:
    commission:
      formula: "turnover * avg_commission_rate"
      description: "연간 추정 수수료"
      avg_commission_rate:
        online_broker: 0.00015  # 0.015% (일반 온라인 증권사)
        note: "증권사별로 상이 — 사용자 입력 우선"

    transaction_tax:
      description: "증권거래세 (매도 시 부과)"
      kospi:
        rate: 0.0018
        breakdown: "증권거래세 0.03% + 농특세 0.15%"
        note: "2025년 기준"
      kosdaq:
        rate: 0.0020
        breakdown: "증권거래세 0.18% + 농특세 0.02%"
        note: "2025년 기준"
      formula: "sold_value * tax_rate"

    total_trading_cost:
      formula: "commission + transaction_tax"

  cost_to_return_ratio:
    formula: "total_trading_cost / total_return * 100"
    unit: "%"
    display: "매매 비용이 수익의 {pct}%를 차지합니다."
    exception:
      negative_return:
        condition: "total_return <= 0"
        action: "비율 대신 절대 금액만 표시"
        message: "수익이 음수이므로 비용/수익 비율을 산출할 수 없습니다. 총 매매비용: {cost}원"
```

---

## 3. Holding Period Analysis

```yaml
holding_period:
  description: "종목별 보유 기간 분석"

  per_stock:
    formula: "sell_date - buy_date (calendar days)"
    matching: "FIFO (선입선출) — 가장 오래된 매수부터 매도와 매칭"
    current_holdings: "today - buy_date"

  portfolio_level:
    avg_holding_period:
      formula: "mean(all_holding_periods)"
      unit: "일"
    weighted_avg:
      formula: "sum(holding_period_i * amount_i) / sum(amount_i)"
      description: "금액 가중 평균 보유기간"

  distribution:
    buckets:
      - { range: "< 30일", label: "1개월 미만" }
      - { range: "30 ~ 90일", label: "1~3개월" }
      - { range: "90 ~ 180일", label: "3~6개월" }
      - { range: "180 ~ 365일", label: "6~12개월" }
      - { range: "> 365일", label: "1년 초과" }
    display: "histogram 또는 pie chart"

  tax_implication:
    description: "장기보유 여부 표시"
    long_term_threshold: 365  # 1년
    note: "대주주의 경우 장기보유특별공제 참조 (참조: tax-fee-impact.md)"
    label_long: "장기보유"
    label_short: "단기보유"
```

---

## 4. Churning Detection

```yaml
churning_detection:
  description: "과매매 감지 — 비정상적으로 높은 매매 빈도 경고"

  flag_conditions:
    condition_1:
      description: "연환산 회전율 > 500%"
      metric: "turnover_ratio_annualized > 500"
    condition_2:
      description: "동일 종목을 7일 이내에 매수-매도 반복 3회 이상"
      metric: "same_stock_round_trip_within_7_days >= 3"
    condition_3:
      description: "매매 비용이 수익의 30% 초과"
      metric: "cost_to_return_ratio > 30"

  severity:
    warning:
      condition: "1개 조건 충족"
      display: "노란색 경고 배너"
    alert:
      condition: "2개 이상 조건 충족"
      display: "빨간색 경고 배너"

  message: "매매 빈도가 높습니다. 거래 비용이 수익에 미치는 영향을 확인하세요."

  disclaimer:
    note: "투자 조언이 아닙니다 — 중립적 정보 제공 목적 (참조: safety-disclaimer.md)"
    display: "경고 메시지 하단에 항상 표시"
```

---

## 5. Edge Cases

```yaml
edge_cases:
  no_sell_records:
    condition: "매도 내역이 전혀 없음"
    action: "회전율 산출 불가, 보유기간만 표시"
    message: "매도 내역이 없어 회전율을 산출할 수 없습니다."
    display: "현재 보유 종목의 보유기간만 표시"

  short_history:
    condition: "분석 기간 < 3개월"
    action: "계산하되 경고 표시"
    message: "분석 기간이 짧아 회전율 연환산이 부정확할 수 있습니다."
    label: "참고용"

  transfer_in:
    condition: "타사 대체입고 (transfer-in)"
    action: "회전율 계산에서 제외 (실제 매매가 아님)"
    holding_period: "입고일 기준으로 보유기간 계산"
    note: "원래 매수일이 불명확하므로 입고일 사용"

  stock_splits_mergers:
    condition: "주식 분할/합병"
    action: "FIFO 매칭 전에 수량 조정"
    formula: "adjusted_quantity = original_quantity * split_ratio"

  partial_sells:
    condition: "일부 수량만 매도"
    action: "FIFO 기준으로 부분 수량 매칭"
    example: "100주 매수 → 30주 매도: 70주는 계속 보유 중으로 처리"

  day_trading:
    condition: "당일 매수-매도 (holding period = 0)"
    action: "보유기간 0일로 기록, 회전율에는 포함"
    label: "데이트레이딩"
```

---

## 6. Display Rules

```yaml
display_rules:
  description: "UI 표시 규칙"

  default_view:
    type: "summary card"
    content:
      - "연환산 회전율: {turnover}%"
      - "평균 보유기간: {avg_days}일"
      - "과매매 경고: {churning_status}"
    layout: "compact card with key metrics"

  detailed_view:
    monthly_trend:
      chart_type: "line chart"
      x_axis: "월"
      y_axis: "월별 회전율 (%)"
      description: "월별 회전율 추이"
    fee_breakdown:
      chart_type: "stacked bar"
      components: ["수수료", "거래세"]
      description: "월별 매매 비용 구성"

  per_stock_table:
    columns:
      - { key: "ticker", label: "종목" }
      - { key: "buy_date", label: "매수일" }
      - { key: "holding_days", label: "보유일수" }
      - { key: "trade_count", label: "매매 횟수" }
      - { key: "status", label: "상태", values: ["보유 중", "매도 완료"] }
    sorting: "default by holding_days descending"
    pagination: "20 rows per page"
```
