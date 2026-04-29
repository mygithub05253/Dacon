# Insight Rules — 인사이트 생성 규칙

> AlphaFolio AI 인사이트 계층의 규칙을 정의합니다.
> 이 문서를 참조하여 분석 결과에서 패턴을 감지하고 투자 인사이트를 자동 생성합니다.
>
> **파이프라인 위치**: Stage 3-B (analysis_rules.md 출력을 소비, visualization_rules.md와 병렬)
> **입력**: analysis_rules.md에서 계산된 지표 (return_rate, weight, mdd, sharpe_ratio 등)
> **출력**: 인사이트 카드 리스트 (type, severity, template 메시지)
> **하위 의존**: report_rules.md가 이 문서의 인사이트 카드를 참조하여 리포트에 배치합니다.

---

## 1. 인사이트 유형 정의

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

## 2. 패턴 감지 규칙

> **실행 순서**: 아래 규칙은 id 순서가 아닌, 각 규칙의 `data_required` 필드에 명시된 데이터 가용 여부에 따라 실행됩니다.
> 기본 지표(weight, return_rate, stock_count) 기반 규칙이 먼저, 고급 지표(mdd, sharpe_ratio, beta) 기반 규칙이 후순위로 실행됩니다.
> 각 규칙은 독립적이며, 하나의 규칙 실패가 다른 규칙에 영향을 주지 않습니다.

### 2.1 집중도 리스크
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

### 2.2 수익률 인사이트
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
    template: "⚠️ {stock_name}이(가) {return}% 손실 중입니다. 손절 기준이나 추가 매수 여부를 점검해보세요."
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

### 2.3 분산 투자 인사이트
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
    template: "💡 보유 종목이 {count}개로 많습니다. 관리 효율과 수수료를 고려하면 핵심 종목 15~20개 위주로 정리하는 것을 고려해보세요."
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

### 2.4 고급 지표 인사이트 (거래 내역 존재 시)
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
    template: "💡 {benchmark} 대비 {excess}%p 하회 중입니다. 패시브 전략(인덱스 ETF)과의 수익률 비교를 추천합니다."
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

### 2.5 특수 상황 인사이트
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

---

## 3. 인사이트 충돌 및 중복 해소 규칙

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

---

## 4. 인사이트 표시 규칙

### 4.1 우선순위 및 정렬
```yaml
display_rules:
  sort_by: "type priority ASC, severity DESC"
  max_display: 6
  always_show:
    - "overall_positive OR overall_negative"   # 전체 상태 항상 표시
    - "top_performer"                          # 항상 최고 수익 종목 표시
  collapse_after: 3
  collapse_label: "인사이트 {remaining}개 더보기"
  
  minimum_insights:
    description: "인사이트가 너무 적을 때"
    min_count: 2
    fallback_insights:
      - id: "portfolio_summary_fallback"
        condition: "항상 생성 가능"
        type: "info"
        template: "📊 총 {stock_count}개 종목, {total_valuation} 규모의 포트폴리오입니다."
      - id: "market_composition_fallback"
        condition: "마켓 데이터 존재"
        type: "info"
        template: "📊 {kr_pct}% 국내, {us_pct}% 해외 자산으로 구성되어 있습니다."
    message: "추가 인사이트를 위해 거래 내역 CSV도 업로드해보세요."

  no_insights:
    description: "어떤 규칙도 트리거되지 않는 극단적 경우"
    action: "fallback_insights 2개 강제 생성"
    note: "이 상황은 데이터 이상이 있을 때만 발생. 로그 기록."
```

### 4.2 인사이트 카드 포맷
```yaml
card_format:
  structure:
    - icon: "인사이트 유형 아이콘 (왼쪽)"
    - title: "인사이트 ID의 name (굵은 글씨)"
    - body: "template에서 생성된 메시지"
    - metadata: "관련 종목/섹터 태그 (선택적)"
  style:
    border_left: "4px solid {type.border_color}"
    background: "{type.bg_color}"
    padding: "16px"
    margin_bottom: "8px"
    border_radius: "8px"
    max_width: "100%"
  animation:
    entrance: "fade-in 0.3s"
    stagger: "0.1s per card"
```

---

## 5. 종합 요약 생성 규칙

```yaml
summary_generation:
  description: "모든 인사이트를 종합하여 2~4문장의 포트폴리오 요약을 생성"
  position: "인사이트 카드 영역 최상단"
  
  template_structure:
    sentence_1: "포트폴리오 기본 현황"
    sentence_2: "가장 중요한 인사이트 (danger/warning 중 1개)"
    sentence_3: "가장 긍정적인 인사이트 (positive 중 1개)"
    sentence_4: "행동 제안 (suggestion 중 1개, 있을 때만)"

  templates:
    balanced:
      condition: "danger 0개, warning 1개 이하"
      template: |
        총 {stock_count}개 종목, {total_valuation} 규모의 포트폴리오입니다. 
        전체 수익률은 {total_return}%이며, {positive_insight}. 
        {suggestion_if_any}
    
    cautious:
      condition: "warning 2개 이상 또는 danger 1개 이상"
      template: |
        총 {stock_count}개 종목, {total_valuation} 규모의 포트폴리오입니다. 
        {danger_or_warning_insight}. 
        다만 {positive_insight_if_any}. 전반적인 포트폴리오 점검을 권장합니다.
    
    minimal:
      condition: "데이터 부족으로 인사이트 2개 미만"
      template: |
        총 {stock_count}개 종목의 포트폴리오입니다. 
        더 상세한 인사이트를 위해 추가 데이터를 업로드해주세요.

  exception:
    all_data_na:
      action: "요약 대신 '데이터를 분석할 수 없습니다' 메시지"
    contradictory_insights:
      action: "요약에서는 severity가 높은 인사이트만 반영"
```

---

## 6. 확장 규칙

```yaml
extension_points:
  new_patterns:
    crypto:
      - id: "high_volatility_coin"
        condition: "24h_change > 15% OR 24h_change < -15%"
        type: "warning"
        template: "⚠️ {coin}이 24시간 내 {change}% 변동했습니다. 급변동에 주의하세요."
      - id: "stablecoin_ratio"
        condition: "stablecoin weight > 50%"
        type: "info"
        template: "📊 스테이블코인 비중이 {weight}%입니다. 관망 포지션이 큽니다."
      - id: "defi_exposure"
        condition: "any staking_status == true"
        type: "info"
        template: "📊 {count}개 자산이 스테이킹 중입니다. 예상 APY: {avg_apy}%"

    bond:
      - id: "maturity_approaching"
        condition: "days_to_maturity < 90"
        type: "warning"
        template: "⚠️ {bond}의 만기가 {days}일 남았습니다. 재투자 또는 상환 계획을 세우세요."
      - id: "duration_risk"
        condition: "portfolio_duration > 7"
        type: "warning"
        template: "⚠️ 포트폴리오 듀레이션이 {duration}년입니다. 금리 상승 시 큰 가격 하락이 예상됩니다."
      - id: "coupon_income"
        condition: "always (bond portfolio)"
        type: "info"
        template: "📊 연간 예상 이자 수익: {income}. 다음 이자 지급일: {next_date}"

    real_estate:
      - id: "low_occupancy"
        condition: "any occupancy_rate < 80%"
        type: "warning"
        template: "⚠️ {property}의 임대율이 {rate}%입니다. 공실 리스크를 점검하세요."
      - id: "nav_premium"
        condition: "price > nav * 1.1"
        type: "info"
        template: "📊 {reit}가 NAV 대비 {premium}% 프리미엄에 거래 중입니다."

  ai_enhancement:
    description: "LLM 연동 시 규칙 기반 인사이트를 자연어로 보강"
    mode: "optional"
    fallback: "LLM 미연동 시 규칙 기반 template만 사용 (성능 저하 없음)"
    prompt_template: |
      다음은 투자 포트폴리오의 자동 분석 결과입니다.
      
      포트폴리오 현황:
      - 총 평가금액: {total_valuation}
      - 총 수익률: {total_return}%
      - 보유 종목: {stock_count}개
      
      감지된 인사이트:
      {insight_cards_json}
      
      위 인사이트를 바탕으로 개인 투자자에게 도움이 되는 
      핵심 요약을 3문장 이내로 자연스럽게 작성해주세요.
      투자 권유가 아닌 정보 제공 목적임을 유의하세요.
    
    rate_limit: "분당 5회"
    timeout: "10초"
    api_failure:
      action: "규칙 기반 요약으로 fallback"
      message: "(AI 요약 대신 규칙 기반 요약을 표시합니다)"
```
