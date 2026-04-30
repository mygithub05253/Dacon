# Basic Metrics — Per-Stock Indicators

> Defines calculation rules for individual stock metrics.
> Source: `analysis_rules.md` Section 1

---

## 1. Return Rate

```yaml
metric: return_rate
description: "개별 종목의 수익률"
formula: "(current_price - avg_price) / avg_price * 100"
unit: "%"
display_format: "+12.5% or -3.2%"
precision: 2

color_rule:
  positive: "green"    # > 0
  negative: "red"      # < 0
  zero: "gray"         # == 0

input_validation:
  quantity_check:
    rule: "if quantity < 0, flag as potential short position"
    action: "exclude from standard metrics, add to short_positions separate analysis"
    warning: "음수 수량 종목이 감지되었습니다 — 공매도 포지션으로 별도 처리됩니다."
  extreme_valuation_check:
    rule: "if single stock weight > 99% OR valuation > 100x median"
    action: "flag_warning, still calculate but mark as outlier"
    warning: "비정상적으로 큰 평가금액이 감지되었습니다. 데이터를 확인해주세요."
  # description: 음수 수량/극단값이 전체 포트폴리오 지표를 왜곡하는 것 방지

exceptions:
  avg_price_zero:
    condition: "avg_price == 0"
    action: "return_rate = N/A, 해당 종목 수익률 관련 차트에서 제외"
    message: "'{name}' 매수가가 0이어서 수익률을 계산할 수 없습니다."
  current_price_missing:
    condition: "current_price is NaN"
    action: "return_rate = N/A, 포트폴리오 전체 수익률 계산에서도 제외"
    ui_display: "현재가 없음"
  extreme_return:
    condition: "abs(return_rate) > 500"
    action: "경고 표시. 액면분할/병합/데이터 오류 가능성."
    message: "수익률이 {return}%입니다. 데이터를 확인해주세요."
    include_in_calculation: true
```

---

## 2. Valuation

```yaml
metric: valuation
description: "현재 보유 가치"
formula: "current_price * quantity"
unit: "KRW or USD (currency-based)"
display_format: "₩1,234,567 or $12,345.67"
precision: 0  # KRW integer, USD 2 decimals

exceptions:
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

---

## 3. Profit / Loss

```yaml
metric: profit_loss
description: "절대 수익/손실 금액"
formula: "(current_price - avg_price) * quantity"
unit: "KRW or USD"
display_format: "+₩500,000 or -$1,234"

exceptions:
  any_price_missing:
    condition: "avg_price 또는 current_price가 NaN"
    action: "profit_loss = N/A"
  mixed_currency:
    action: "각 통화 단위로 별도 계산 후 환산 합산"
```

---

## 4. Portfolio Weight

```yaml
metric: weight
description: "포트폴리오 내 비중"
formula: "valuation / total_valuation * 100"
unit: "%"
precision: 1
display_format: "23.4%"

thresholds:
  concentration_warning: 30    # > 30%
  concentration_danger: 50     # > 50%
  min_meaningful: 0.1          # < 0.1% shown as "0.1% 미만"

exceptions:
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
