# Display Rules — 인사이트 표시 규칙

> Priority/sorting, max display, card format/style, summary generation templates, and extension points including AI enhancement.

---

## 1. Priority & Sorting

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

---

## 2. Insight Card Format

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

## 3. Summary Generation

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

## 4. Extension Points

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
