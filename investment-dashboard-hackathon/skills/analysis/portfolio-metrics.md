# Portfolio Metrics — Aggregate & Distribution Analysis

> Defines portfolio-level metrics, sector/market aggregations, and return distribution.
> Source: `analysis_rules.md` Sections 2 + 3

---

## 1. Total Return

```yaml
metric: total_return
description: "포트폴리오 가중평균 수익률"
formula: "sum(return_rate_i * weight_i / 100) for all stocks"
alternative: "(total_valuation - total_investment) / total_investment * 100"
unit: "%"
precision: 2

exceptions:
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

---

## 2. Total Valuation / Investment / Profit-Loss

```yaml
metrics:
  total_valuation:
    description: "전체 평가금액 합산"
    formula: "sum(valuation_i)"
    exceptions:
      mixed_currency: "각 통화별 합산 후 base_currency로 환산"
      some_missing: "계산 가능한 종목만 합산, '일부 추정' 태그"

  total_investment:
    description: "전체 투자금액 합산"
    formula: "sum(invested_amount_i)"  # invested_amount = avg_price * quantity
    exceptions:
      mixed_currency: "매수 시점 환율 불명 → 현재 환율로 일괄 환산 + 경고"

  total_profit_loss:
    description: "전체 평가손익"
    formula: "total_valuation - total_investment"
    exceptions:
      mixed_currency:
        action: "base_currency 환산 후 차액"
        warning: "환율 변동 수익/손실이 포함되어 있습니다."
```

---

## 3. Stock Counts

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

extreme_count_warning:
  max_reasonable: 200  # 200종목 초과 시 경고
  warning: "포트폴리오에 {count}개 종목이 있습니다. 데이터 중복이 아닌지 확인해주세요."

display:
  format: "수익 {profitable} / 손실 {losing} / 보합 {breakeven}"
  win_rate: "profitable / (profitable + losing) * 100"
  win_rate_label: "승률"
```

---

## 4. Sector Analysis

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

exceptions:
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

---

## 5. Market Analysis

```yaml
aggregation: market_analysis
group_by: "market"
metrics:
  - market_weight: "sum(weight) per market"
  - market_return: "weighted_avg(return) per market"
  - market_valuation: "sum(valuation) per market"
  - currency_exposure: "통화별 비중"
sort_by: "market_weight DESC"

exceptions:
  single_market:
    condition: "모든 종목이 같은 시장"
    action: "마켓 비교 차트 비활성화. '국내 전용' 또는 '해외 전용' 레이블 표시."
  other_market_exists:
    condition: "KR/US 외의 시장 종목 존재"
    action: "'기타' 그룹으로 합산"
```

---

## 6. Return Distribution Buckets

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

bucket_boundary_rule:
  convention: "left-inclusive, right-exclusive"  # [a, b)
  zero_bucket: "[-0.001, 0.001)"  # 사실상 0% 수익률 (부동소수점 오차 허용)
  example: "return = 0.0 → zero_bucket, return = 5.0 → [5, 10) bucket"
  # description: 경계값 종목이 구현자마다 다른 bucket에 배정되는 문제 방지

exceptions:
  all_same_bucket:
    condition: "모든 종목이 같은 구간"
    action: "구간 분포 차트 대신 '전 종목 {bucket_label}' 텍스트 표시"
  na_returns:
    condition: "수익률 N/A 종목 존재"
    action: "별도 '미분류' 구간으로 표시"
```
