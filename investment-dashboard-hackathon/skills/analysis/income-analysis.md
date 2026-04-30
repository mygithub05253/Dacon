# Income Analysis — Dividend & Income Return

> Stage 2 supplement.
> Input: Trade history with DIVIDEND records, current holdings.
> Output: Income metrics per stock + portfolio level.
> Referenced by: `basic-metrics` (total return split), `report/section-specs` (dividend section).
> Recommended model: sonnet

---

## 1. Return Decomposition

```yaml
return_decomposition:
  description: "총 수익률을 가격 수익과 배당 수익으로 분리"

  components:
    total_return:
      formula: "price_return + income_return"
    price_return:
      formula: "(current_price - avg_buy_price) / avg_buy_price"
      description: "주가 변동에 의한 수익률"
    income_return:
      formula: "total_dividends_received / invested_amount"
      description: "배당금에 의한 수익률"

  display:
    format: "stacked bar 또는 breakdown card"
    show_both: true
    combined_label: "총 수익률"
    price_label: "시세차익"
    income_label: "배당수익"
```

---

## 2. Dividend Metrics (Per Stock)

```yaml
dividend_metrics_per_stock:
  description: "종목별 배당 지표"

  metrics:
    dividend_yield:
      formula: "annual_dividend / current_price * 100"
      period: "trailing 12 months (TTM)"
      unit: "%"
      precision: 2
      short_history:
        condition: "데이터 기간 < 12개월"
        action: "비례 연환산 (annualize proportionally)"
        label: "추정치"
        message: "{months}개월 데이터 기반 추정 배당수익률입니다."

    payout_frequency:
      description: "배당 지급 주기 감지"
      categories:
        - "monthly"      # 매월
        - "quarterly"    # 분기
        - "semi_annual"  # 반기
        - "annual"       # 연간
        - "irregular"    # 불규칙
      detection: "배당 지급 날짜 간격의 중앙값으로 판별"

    dividend_growth_rate:
      description: "전년 대비 배당금 변화율"
      formula: "(dividend_year_n - dividend_year_n1) / dividend_year_n1 * 100"
      condition: "2년 이상 배당 데이터 필요"
      insufficient_data: "배당 성장률 산출에 2년 이상 데이터가 필요합니다."

    last_ex_dividend_date:
      description: "최근 배당락일"
      format: "YYYY-MM-DD"

    next_expected_dividend:
      description: "다음 예상 배당일 (패턴 감지 시)"
      method: "과거 배당 주기 기반 추정"
      label: "예상일 (확정 아님)"
```

---

## 3. Portfolio-Level Income Metrics

```yaml
portfolio_income_metrics:
  description: "포트폴리오 전체 배당 지표"

  metrics:
    portfolio_dividend_yield:
      formula: "sum(stock_yield_i * weight_i)"
      description: "비중 가중 평균 배당수익률"
      unit: "%"

    total_dividend_income:
      formula: "sum(all_dividend_records)"
      description: "수령한 배당금 총액"
      unit: "KRW 또는 USD (포트폴리오 기준 통화)"

    monthly_income_projection:
      description: "월별/분기별 예상 배당 수입"
      method: "과거 배당 패턴 기반 향후 12개월 추정"
      label: "추정치"
      display: "월별 막대 차트"

    income_coverage_ratio:
      formula: "total_dividend_income / total_invested_amount * 100"
      description: "투자금 대비 배당 수입 비율"
      unit: "%"
      precision: 2
```

---

## 4. Tax Impact on Dividends

```yaml
tax_impact:
  description: "배당소득세 영향 분석 (참조: tax-fee-impact.md)"

  rules:
    domestic_kr:
      description: "국내 배당"
      tax_rate: 0.154
      breakdown:
        income_tax: 0.14
        local_tax: 0.014
      note: "15.4% 원천징수 (소득세 14% + 지방소득세 1.4%)"

    foreign_us:
      description: "해외 배당 (미국)"
      tax_rate: 0.15
      note: "15% 원천징수 (한미 조세조약)"

    comprehensive_taxation:
      description: "금융소득종합과세"
      threshold: 20000000  # 2,000만원
      condition: "이자 + 배당 합산 연간 2,000만원 초과"
      max_rate: 0.495
      note: "초과분에 대해 최고 49.5% 과세 가능"

  calculation:
    formula: "after_tax_dividend = gross_dividend * (1 - tax_rate)"
    display_default: "세전 배당금"
    toggle_option: "추정 세후 배당금"

  disclaimer:
    message: "배당소득세는 개인 상황에 따라 달라질 수 있습니다."
    display: "세후 금액 옆에 항상 표시"
    style: "muted text"
```

---

## 5. Dividend Calendar

```yaml
dividend_calendar:
  description: "배당 일정 캘린더 뷰"

  views:
    monthly:
      description: "월별 배당 종목 및 금액 표시"
      format: "calendar grid 또는 timeline"
    upcoming:
      description: "향후 배당락일 하이라이트"
      lookahead_days: 90
    income_stream:
      description: "월별 예상 배당 수입 흐름"
      display: "bar chart (월별 예상 금액)"

  data_source:
    historical: "과거 배당 지급 기록"
    projected: "패턴 기반 추정 (irregular 제외)"
    label_projected: "예상"
```

---

## 6. Edge Cases

```yaml
edge_cases:
  no_dividend_records:
    condition: "배당 데이터가 전혀 없음"
    action: "Income Analysis 섹션 전체 비활성화"
    message: "배당 데이터가 없습니다."
    display: "빈 카드에 메시지만 표시"

  special_dividends:
    condition: "특별배당 (일회성)"
    action: "특별배당 플래그 표시"
    label: "특별배당"
    yield_calculation: "정기 배당수익률 산출에서 제외"
    total_income: "총 배당 수입에는 포함"

  stock_splits:
    condition: "주식 분할이 배당 이력에 영향"
    action: "분할 비율로 과거 배당금 조정"
    note: "분할 전 배당금을 분할 후 기준으로 환산"

  foreign_currency:
    condition: "외화 배당금"
    action: "배당 지급일 환율로 원화 환산 (참조: currency-rules.md)"
    display: "원화 환산액 + 원래 통화 금액 병기"

  reinvested_dividends:
    condition: "배당금 재투자 (DRIP)"
    action: "재투자 배당은 income으로 계산 후 추가 매수로도 반영"
```

---

## 7. Insights Integration

```yaml
insights_triggers:
  description: "insight/pattern-rules로 전달되는 트리거"

  high_yield:
    condition: "포트폴리오 배당수익률 > 5%"
    trigger: "high_yield"
    message: "포트폴리오 배당수익률이 {yield}%로 높은 수준입니다."

  dividend_cut:
    condition: "전년 대비 배당금 감소 > 20%"
    trigger: "dividend_cut"
    severity: "warning"
    message: "{ticker}의 배당금이 전년 대비 {pct}% 감소했습니다."

  dividend_concentration:
    condition: "단일 종목에서 전체 배당 수입의 60% 초과"
    trigger: "dividend_concentration"
    severity: "warning"
    message: "배당 수입의 {pct}%가 {ticker}에 집중되어 있습니다."

  consistent_growth:
    condition: "3년 연속 배당 증가"
    trigger: "dividend_growth_streak"
    message: "{ticker}이(가) {years}년 연속 배당을 증가시켰습니다."
```
