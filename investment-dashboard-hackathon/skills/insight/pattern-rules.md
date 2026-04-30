# Pattern Rules — 패턴 감지 규칙

> Insight type definitions, ALL pattern detection rules, and conflict/dedup resolution.

---

## 1. Insight Type Definitions

```yaml
insight_types:
  - type: "danger"
    icon: "🔴"
    color: "#E24B4A"
    bg_color: "rgba(226, 75, 74, 0.06)"
    border_color: "#E24B4A"
    description: "즉각적인 점검이 필요한 위험 상황"
    priority: 0  # 가장 먼저 표시

  - type: "warning"
    icon: "⚠️"
    color: "#BA7517"
    bg_color: "rgba(186, 117, 23, 0.06)"
    border_color: "#BA7517"
    description: "리스크 또는 주의가 필요한 상황"
    priority: 1

  - type: "suggestion"
    icon: "💡"
    color: "#7F77DD"
    bg_color: "rgba(127, 119, 221, 0.06)"
    border_color: "#7F77DD"
    description: "포트폴리오 개선을 위한 제안"
    priority: 2

  - type: "positive"
    icon: "✅"
    color: "#639922"
    bg_color: "rgba(99, 153, 34, 0.06)"
    border_color: "#639922"
    description: "긍정적인 성과 또는 상태"
    priority: 3

  - type: "info"
    icon: "📊"
    color: "#378ADD"
    bg_color: "rgba(55, 138, 221, 0.06)"
    border_color: "#378ADD"
    description: "참고할 만한 정보성 인사이트"
    priority: 4
```

---

## 2. Pattern Detection Rules

> **Execution order**: Rules execute by `data_required` availability, not by id.
> Basic metrics (weight, return_rate, stock_count) first, advanced metrics (mdd, sharpe_ratio, beta) second.
> Each rule is independent — one failure does not affect others.

```yaml
insight_global_disclaimer: "모든 인사이트는 데이터 기반 참고 정보이며, 개인 투자 상황에 따라 다를 수 있습니다. 투자 권유가 아닙니다."
```

### 2.1 Concentration Risk

```yaml
rules:
  - id: "concentration_extreme"
    name: "극단적 단일 종목 집중"
    condition: "any stock weight > 50%"
    type: "danger"
    template: "🔴 {stock_name}에 자산의 절반 이상({weight}%)이 집중되어 있습니다. 해당 종목의 급락 시 포트폴리오 전체에 치명적인 영향을 줄 수 있습니다."
    severity: "high"
    data_required: ["weight"]
    exception:
      single_stock_portfolio:
        condition: "전체 종목이 1개"
        action: "이 규칙 스킵 (단일 종목 분석 모드에서는 의미 없음)"

  - id: "concentration_single_stock"
    name: "단일 종목 집중"
    condition: "any stock weight > 30% AND weight <= 50%"
    type: "warning"
    template: "⚠️ {stock_name} 비중이 {weight}%로 높습니다. 단일 종목 리스크에 노출되어 있으니 분산을 고려하세요."
    severity: "medium"
    data_required: ["weight"]
    suppress_if: "concentration_extreme 이미 트리거됨 (같은 종목)"

  - id: "concentration_sector"
    name: "섹터 집중"
    condition: "any sector weight > 50%"
    type: "warning"
    template: "📊 {sector} 섹터에 {weight}% 집중되어 있습니다. 산업 특정 이벤트(규제, 경기 변동)에 취약할 수 있으니 섹터 다각화를 검토하세요."
    severity: "medium"
    data_required: ["sector", "weight"]
    exception:
      no_sector_data:
        action: "이 규칙 스킵"
      single_sector:
        action: "표현을 '모든 종목이 {sector} 섹터에 속합니다'로 변경"

  - id: "concentration_top3"
    name: "상위 3종목 과집중"
    condition: "sum(top 3 stock weights) > 80%"
    type: "warning"
    template: "⚠️ 상위 3종목({stock1}, {stock2}, {stock3})이 전체의 {total_weight}%를 차지합니다. 소수 종목에 지나치게 의존하고 있습니다."
    severity: "medium"
    data_required: ["weight"]
    exception:
      total_stocks_less_than_4:
        action: "이 규칙 스킵 (종목이 3개 이하면 상위 3종목 = 전체)"
```

### 2.2 Return Insights

```yaml
  - id: "top_performer"
    name: "최고 수익 종목"
    condition: "always (종목 1개 이상이고 수익률 계산 가능)"
    type: "positive"
    template: "✅ 최고 수익 종목: {stock_name} (+{return}%). 포트폴리오 수익에 {contribution}%p 기여했습니다."
    severity: "low"
    data_required: ["return_rate", "weight"]
    exception:
      all_returns_na:
        action: "이 규칙 스킵"
      all_returns_negative:
        action: "표현 변경: '상대적 최소 손실 종목: {stock_name} ({return}%)'"

  - id: "worst_performer"
    name: "최대 손실 종목"
    condition: "min return_rate < -10%"
    type: "warning"
    template: "⚠️ {stock_name}이(가) {return}% 손실 중입니다. 해당 종목의 최근 실적을 확인해보시기 바랍니다. (본 정보는 투자 권유가 아닙니다.)"
    severity: "medium"
    data_required: ["return_rate"]
    exception:
      return_na:
        action: "스킵"
    escalation:
      condition: "min return_rate < -30%"
      upgrade_to: "danger"
      template: "🔴 {stock_name}이(가) {return}% 대폭 하락했습니다. 손실 확대 방지를 위한 점검이 필요합니다."

  - id: "overall_positive"
    name: "전체 수익 상태"
    condition: "total_return > 0"
    type: "positive"
    template: "✅ 포트폴리오 전체 수익률이 +{total_return}%입니다. 총 {profit_loss} 수익을 기록 중입니다."
    severity: "low"
    data_required: ["total_return", "total_profit_loss"]

  - id: "overall_negative"
    name: "전체 손실 상태"
    condition: "total_return < 0"
    type: "warning"
    template: "⚠️ 포트폴리오가 {total_return}% 손실 중입니다. 현재 {losing_count}개 종목이 손실 상태입니다."
    severity: "medium"
    data_required: ["total_return", "stock_counts"]
    escalation:
      condition: "total_return < -15%"
      upgrade_to: "danger"
      template: "🔴 포트폴리오가 {total_return}% 큰 손실 중입니다. 전체적인 포지션 재검토가 필요합니다."

  - id: "win_loss_ratio"
    name: "수익/손실 종목 비율"
    condition: "losing_count > profitable_count AND total_stocks >= 5"
    type: "info"
    template: "📊 수익 종목({profitable})보다 손실 종목({losing})이 더 많습니다. 종목 선정 기준을 재점검해보세요."
    severity: "low"
    data_required: ["stock_counts"]
```

### 2.3 Diversification Insights

```yaml
  - id: "market_imbalance"
    name: "시장 편중"
    condition: "KR weight > 85% OR US weight > 85%"
    type: "info"
    template: "📊 {dominant_market} 시장에 {weight}% 집중되어 있습니다. 글로벌 분산 투자를 통해 지역 리스크를 줄일 수 있습니다."
    severity: "low"
    data_required: ["market", "weight"]
    exception:
      single_market_only:
        condition: "데이터에 단일 시장만 존재"
        action: "표현 변경: '{market} 시장 전용 포트폴리오입니다. 해외 자산 추가로 분산 효과를 높일 수 있습니다.'"

  - id: "good_diversification"
    name: "양호한 분산"
    condition: "no stock weight > 20% AND sector count >= 3 AND stock_count >= 5"
    type: "positive"
    template: "✅ 종목과 섹터가 고르게 분산되어 있습니다. {sector_count}개 섹터에 걸쳐 안정적인 포트폴리오를 구성하고 있습니다."
    severity: "low"
    data_required: ["weight", "sector"]

  - id: "too_many_stocks"
    name: "과다 종목"
    condition: "stock_count > 25"
    type: "suggestion"
    template: "💡 종목 수가 많아 관리 복잡도가 높을 수 있습니다. 포트폴리오 구성을 점검해보시기 바랍니다."
    severity: "low"
    data_required: ["stock_counts"]

  - id: "too_few_stocks"
    name: "종목 부족"
    condition: "stock_count >= 2 AND stock_count <= 3"
    type: "warning"
    template: "⚠️ 보유 종목이 {count}개뿐입니다. 개별 종목 리스크가 크므로 최소 5~10개 종목으로 분산을 권장합니다."
    severity: "medium"
    data_required: ["stock_counts"]
    exception:
      single_stock:
        action: "별도의 단일 종목 모드 메시지 사용"
        message: "단일 종목 포트폴리오입니다. 분산 투자 관련 인사이트는 생략합니다."

  - id: "currency_risk"
    name: "환율 리스크"
    condition: "US weight >= 20% AND US weight <= 80%"
    type: "info"
    template: "📊 해외 자산 비중이 {us_weight}%입니다. 원/달러 환율 변동이 전체 수익에 영향을 줄 수 있습니다."
    severity: "low"
    data_required: ["market", "weight"]
```

### 2.4 Advanced Metric Insights (requires trade history)

```yaml
  - id: "high_mdd"
    name: "높은 MDD"
    condition: "mdd < -20%"
    type: "warning"
    template: "⚠️ 최대 낙폭(MDD)이 {mdd}%입니다({mdd_start} ~ {mdd_end}). 변동성이 큰 포트폴리오이므로 손절 기준이나 리밸런싱 주기를 점검하세요."
    severity: "medium"
    data_required: ["mdd"]
    escalation:
      condition: "mdd < -30%"
      upgrade_to: "danger"
      template: "🔴 최대 낙폭이 {mdd}%에 달합니다. 고위험 포트폴리오입니다."

  - id: "low_mdd"
    name: "낮은 MDD"
    condition: "mdd > -10% AND mdd is not N/A"
    type: "positive"
    template: "✅ 최대 낙폭이 {mdd}%로 안정적입니다. 리스크 관리가 잘 되고 있습니다."
    severity: "low"
    data_required: ["mdd"]

  - id: "good_sharpe"
    name: "양호한 샤프비율"
    condition: "sharpe_ratio > 1.0"
    type: "positive"
    template: "✅ 샤프비율이 {sharpe}로 양호합니다. 감수한 리스크 대비 효율적인 수익을 얻고 있습니다."
    severity: "low"
    data_required: ["sharpe_ratio"]

  - id: "poor_sharpe"
    name: "낮은 샤프비율"
    condition: "sharpe_ratio < 0.5 AND sharpe_ratio is not N/A"
    type: "suggestion"
    template: "💡 샤프비율이 {sharpe}로 낮습니다. 리스크 대비 수익 효율이 떨어지고 있으니 고변동 종목 비중 조절을 고려하세요."
    severity: "medium"
    data_required: ["sharpe_ratio"]

  - id: "high_volatility"
    name: "높은 변동성"
    condition: "volatility > 30%"
    type: "warning"
    template: "⚠️ 연환산 변동성이 {volatility}%로 높습니다. 급격한 가치 변동에 대비하세요."
    severity: "medium"
    data_required: ["volatility"]

  - id: "beat_benchmark"
    name: "벤치마크 초과"
    condition: "excess_return > 0"
    type: "positive"
    template: "✅ 포트폴리오가 {benchmark} 대비 +{excess}%p 초과 수익 중입니다. 시장을 이기고 있습니다."
    severity: "low"
    data_required: ["excess_return", "benchmark"]

  - id: "underperform_benchmark"
    name: "벤치마크 하회"
    condition: "excess_return < -5%"
    type: "suggestion"
    template: "💡 벤치마크 대비 수익률이 낮습니다. 포트폴리오 전략을 재검토해보시기 바랍니다."
    severity: "low"
    data_required: ["excess_return", "benchmark"]

  - id: "high_beta"
    name: "높은 베타"
    condition: "beta > 1.3"
    type: "info"
    template: "📊 포트폴리오 베타가 {beta}입니다. 시장 하락 시 포트폴리오가 더 크게 하락할 수 있습니다."
    severity: "low"
    data_required: ["beta"]
```

### 2.5 Special Situation Insights

```yaml
  - id: "zero_quantity_stocks"
    name: "매도 완료 종목 존재"
    condition: "any quantity == 0"
    type: "info"
    template: "📊 매도 완료된 종목이 {count}개 있습니다: {stock_names}. 현재 보유 종목만 분석에 반영했습니다."
    severity: "low"
    data_required: ["quantity"]

  - id: "stale_data"
    name: "오래된 데이터"
    condition: "data_date이 30일 이상 이전"
    type: "warning"
    template: "⚠️ 데이터 기준일이 {data_date}입니다. 현재 시세와 차이가 있을 수 있으니 최신 데이터로 업데이트하세요."
    severity: "medium"
    data_required: ["date"]

  - id: "estimated_values"
    name: "추정값 사용"
    condition: "any valuation flagged as 'estimated'"
    type: "info"
    template: "📊 {count}개 종목의 현재가가 없어 매수가 기준으로 추정했습니다. 실제 수익률과 차이가 있을 수 있습니다."
    severity: "low"

  - id: "partial_analysis"
    name: "부분 분석"
    condition: "analysis_mode == 'minimal_mode' OR analysis_mode == 'standard_mode'"
    type: "suggestion"
    template: "💡 현재 {mode} 모드로 분석 중입니다. {missing_data}를 추가로 업로드하면 더 상세한 분석이 가능합니다."
    severity: "low"
```

### 2.6 Special Situation Insights

(See section 2.5 above)

### 2.7 Correlation & Diversification Insights

```yaml
# === Phase 3: New Insight Patterns ===

  - id: "high_correlation_pair"
    name: "높은 상관관계 종목 쌍"
    condition: "correlation between any pair > 0.8"
    type: "warning"
    template: "⚠️ {stock_a}와 {stock_b}의 상관관계가 {corr:.1%}로 높습니다. 유사한 가격 움직임을 보일 수 있습니다."
    severity: "medium"
    data_required: ["correlation_matrix"]

  - id: "low_diversification"
    name: "낮은 분산투자 효과"
    condition: "diversification_ratio < 1.2"
    type: "warning"
    template: "⚠️ 분산투자 효과가 제한적입니다 (분산비율: {dr:.2f}). 포트폴리오 내 종목들의 상관관계가 높습니다."
    severity: "medium"
    data_required: ["diversification_ratio"]
```

### 2.8 Dividend & Income Insights

```yaml
  - id: "high_dividend_yield"
    name: "높은 배당수익률"
    condition: "portfolio_dividend_yield > 5%"
    type: "info"
    template: "📊 포트폴리오 배당수익률이 {yield:.1%}입니다."
    severity: "low"
    data_required: ["portfolio_dividend_yield"]

  - id: "dividend_cut_detected"
    name: "배당금 감소 감지"
    condition: "any stock YoY dividend decrease > 20%"
    type: "warning"
    template: "⚠️ {ticker}의 배당금이 전년 대비 {change:.1%} 감소했습니다."
    severity: "medium"
    data_required: ["dividend_history"]

  - id: "dividend_concentration"
    name: "배당 수입 집중"
    condition: "single stock contributes > 60% of total dividend income"
    type: "warning"
    template: "⚠️ 배당 수입의 {pct:.0%}가 {ticker}에 집중되어 있습니다."
    severity: "medium"
    data_required: ["dividend_amount"]
```

### 2.9 Trading Activity Insights

```yaml
  - id: "high_turnover"
    name: "높은 회전율"
    condition: "annualized turnover > 200%"
    type: "warning"
    template: "⚠️ 연환산 회전율이 {turnover:.0%}입니다. 거래 비용이 수익에 미치는 영향을 확인해보시기 바랍니다."
    severity: "medium"
    data_required: ["turnover_ratio"]

  - id: "churning_alert"
    name: "과매매 경고"
    condition: "turnover > 500% OR cost_to_return_ratio > 30%"
    type: "danger"
    template: "🔴 매매 빈도가 매우 높습니다. 추정 거래 비용이 수익의 {cost_pct:.0%}를 차지합니다."
    severity: "high"
    data_required: ["turnover_ratio", "cost_to_return_ratio"]

  - id: "short_holding_dominant"
    name: "단기 매매 비중 높음"
    condition: "more than 50% of trades have holding_period < 30 days"
    type: "info"
    template: "📊 거래의 {pct:.0%}가 30일 이내 단기 매매입니다. 보유 기간을 확인해보시기 바랍니다."
    severity: "low"
    data_required: ["holding_periods"]
```

---

## 3. Conflict & Dedup Resolution

```yaml
conflict_resolution:
  description: "여러 규칙이 동시에 트리거될 때 충돌을 해소하는 규칙"

  suppress_rules:
    - "concentration_extreme이 트리거되면 같은 종목에 대한 concentration_single_stock 억제"
    - "overall_negative + escalation이 트리거되면 overall_negative 기본 버전 억제"
    - "worst_performer + escalation이 트리거되면 worst_performer 기본 버전 억제"

  dedup_rules:
    - description: "같은 종목에 대한 중복 인사이트"
      action: "severity가 높은 것만 유지"
    - description: "같은 주제(분산도)에 대한 중복"
      action: "가장 구체적인 규칙만 유지 (concentration_top3 vs sector_concentration)"
    - description: "상반된 인사이트"
      examples:
        - "good_diversification + concentration_sector 동시 발생"
      action: "조건을 다시 검증. 모순이면 severity 높은 것 우선."

  priority_override:
    - "danger > warning > suggestion > positive > info"
    - "같은 type 내에서는 severity 기준 (high > medium > low)"
    - "같은 severity 내에서는 규칙 정의 순서 (위에 정의된 규칙이 우선)"
```
