# Section Specs — 섹션별 상세 규칙

> Detailed specifications for every report section:
> header, KPI summary, insight highlight, portfolio composition, return analysis,
> risk analysis, market comparison, stock detail table, footer.

---

## 1. Header

```yaml
section: header
content:
  - portfolio_name:
      source: "사용자 입력"
      default: "내 포트폴리오"
      max_length: 30
      sanitize: "HTML 태그 제거, XSS 방지"
  - report_date:
      format: "YYYY-MM-DD HH:MM"
      timezone: "Asia/Seoul"
  - data_date:
      label: "데이터 기준일"
      source: "파싱된 데이터의 최신 날짜"
      fallback: "업로드 날짜 사용"
  - total_valuation:
      format: "currency_display"  # visualization_rules.md 참조
      font_size: "2.5rem"
      fallback_when_missing: "'평가금액 정보 없음' 텍스트 표시"
  - analysis_mode_badge:
      description: "현재 분석 모드를 뱃지로 표시"
      mapping:
        full_mode: { label: "전체 분석", color: "#639922" }
        standard_mode: { label: "표준 분석", color: "#378ADD" }
        minimal_mode: { label: "기본 분석", color: "#BA7517" }
        trade_history_mode: { label: "거래 내역 분석", color: "#7F77DD" }
  - data_quality_indicator:
      description: "데이터 품질 요약 표시 (파싱 단계에서 경고 있으면)"
      condition: "parsing_warnings > 0"
      template: "⚠️ 데이터 품질 알림 {warning_count}건"
      tooltip: "상세 내용은 하단 참고"

style:
  background: "linear-gradient(135deg, #1a1a2e 0%, #16213e 100%)"
  text_color: "#ffffff"
  alignment: "center"
  padding: "32px 24px"
  border_radius: "12px"
  margin_bottom: "24px"

exceptions:
  total_valuation_zero:
    condition: "total_valuation == 0"
    action: "표시하되 색상을 gray로 변경, '평가금액 0원' 명시"
  stale_data:
    condition: "data_date가 현재 기준 30일 이상 경과"
    action: "헤더에 '⏰ 데이터가 오래되었습니다 ({days}일 전)' 경고 배너 추가"
    banner_style:
      background: "rgba(186, 117, 23, 0.15)"
      color: "#BA7517"
      position: "헤더 하단"
```

---

## 2. KPI Summary Cards

```yaml
section: kpi_summary
layout:
  desktop: "4열 가로 배치"
  tablet: "2열 × 2행"
  mobile: "1열 수직 스택"

cards:
  # --- 항상 표시되는 카드 ---
  - metric: "total_return"
    label: "총 수익률"
    format: "percentage_with_sign"  # "+12.5%" / "-3.2%"
    color_rule: "positive=green, negative=red, zero=gray"
    fallback_when_missing:
      display: "- %"
      tooltip: "수익률 계산 불가 (현재가 또는 매수가 누락)"

  - metric: "total_profit_loss"
    label: "총 손익"
    format: "currency_display"  # "₩1,234,567" / "$12,345"
    color_rule: "positive=green, negative=red, zero=gray"
    fallback_when_missing:
      display: "- 원"
      tooltip: "손익 계산 불가"
    mixed_currency:
      action: "원화 환산 합계 표시"
      sub_label: "(환산 기준 {exchange_rate})"

  - metric: "stock_count"
    label: "보유 종목"
    format: "{count}개"
    always_available: true  # 파싱 성공하면 항상 존재

  - metric: "profitable_ratio"
    label: "수익 종목 비율"
    format: "{profitable}/{total} ({ratio}%)"
    formula: "profitable_count / total_count × 100"
    color_rule: "ratio >= 50 = green, ratio < 50 = red"
    fallback_when_missing:
      condition: "수익률 계산 불가 종목 존재"
      display: "{profitable}/{calculable} (일부 제외)"
      tooltip: "수익률 계산 불가 종목 {excluded}개 제외"

  kpi_tax_label:
    display: "(세전 기준)"
    position: "after return percentage value"
    style: "font-size: 11px, color: muted"
    # description: 세후 수익률과 혼동 방지

  # --- 조건부 카드 (고급 지표 존재 시 교체) ---
  conditional_cards:
    - condition: "sharpe_ratio 계산 가능"
      replace: "stock_count"  # stock_count → 인사이트 카드 하단으로 이동
      metric: "sharpe_ratio"
      label: "샤프 비율"
      format: "{value:.2f}"
      color_rule:
        value_gte_1: "green"
        value_between_0_1: "orange"
        value_lt_0: "red"

    - condition: "mdd 계산 가능"
      add_as_5th_card: true  # 기존 4개 유지 + 1개 추가
      metric: "mdd"
      label: "최대 낙폭"
      format: "{value}%"
      color_rule:
        value_gte_minus10: "green"
        value_between_minus10_minus30: "orange"
        value_lt_minus30: "red"

exceptions:
  all_metrics_missing:
    condition: "total_return, total_profit_loss 모두 계산 불가"
    action: "KPI 카드를 2개만 표시 (stock_count + profitable_ratio)"
    message: "매수가 또는 현재가 정보가 없어 수익률·손익을 계산할 수 없습니다."
  single_stock:
    adjustment: "profitable_ratio 카드 숨기고 해당 종목의 개별 수익률 카드로 교체"
  extreme_values:
    condition: "total_return > 500% 또는 < -90%"
    action: "정상 표시하되 '⚠️ 극단적 수치입니다. 데이터를 확인해주세요.' 안내 추가"
```

---

## 3. Insight Highlight

```yaml
section: insight_highlight
source: "insight_rules.md에서 생성된 인사이트 카드"

display:
  max_cards: 3
  selection_priority:
    1: "severity가 danger인 항목 우선"
    2: "그 다음 warning"
    3: "나머지 중 priority 값 기준 오름차순"
  layout: "수직 스택"
  card_style:
    border_left: "4px solid {type_color}"
    background: "{type_bg_color}"
    padding: "16px 20px"
    border_radius: "8px"
    margin_bottom: "12px"
  include_summary: true  # 종합 요약문 표시
  summary_position: "인사이트 카드 위"

exceptions:
  zero_insights:
    condition: "insight_rules.md에서 인사이트 0개 생성"
    action: "기본 안내 카드 1개 표시"
    card:
      type: "info"
      message: "현재 데이터에서 특별한 주의가 필요한 사항은 감지되지 않았습니다."
  too_many_danger:
    condition: "danger 인사이트가 3개 이상"
    action: "danger 3개만 표시 + '추가 위험 신호 {n}건이 더 있습니다' 링크"
    expand_behavior: "클릭 시 전체 인사이트 목록 표시"
  insight_generation_failed:
    condition: "insight_rules.md 처리 중 에러 발생"
    action: "에러 메시지 대신 안내 카드 표시"
    card:
      type: "info"
      message: "인사이트 분석 중 오류가 발생했습니다. 기본 분석 결과를 참고해 주세요."
  partial_insights:
    condition: "일부 규칙만 실행 성공"
    action: "성공한 인사이트만 표시 + 하단에 '일부 분석이 제한되었습니다' 작은 텍스트"
```

---

## 4. Portfolio Composition

```yaml
section: portfolio_composition
charts:
  - chart_ref: "portfolio_composition.donut"
    title: "종목별 비중"
    width: "50%"
    fallback_chain:
      1: { condition: "종목 20개 초과", action: "top 15 + 기타 묶음" }
      2: { condition: "차트 렌더링 실패", action: "수평 바 차트로 대체" }
      3: { condition: "대체 차트도 실패", action: "비중 테이블로 표시" }

  - chart_ref: "portfolio_composition.treemap"
    title: "섹터/마켓별 구성"
    width: "50%"
    condition: "섹터 정보가 2개 이상 존재"
    fallback_chain:
      1: { condition: "섹터 정보 없음", action: "마켓별 도넛 차트로 대체" }
      2: { condition: "마켓도 1개", action: "이 차트 슬롯을 종목 Top 5 카드로 교체" }
      3: { condition: "렌더링 실패", action: "섹터/마켓 비중 텍스트 목록" }

layout:
  desktop: "2열 나란히 (50% + 50%)"
  tablet: "2열 나란히 (50% + 50%)"
  mobile: "수직 스택"

description_auto: true
description_template: |
  포트폴리오는 {stock_count}개 종목으로 구성되며,
  {market_composition_text}.
  {top_stock_name}이(가) {top_stock_weight}%로 가장 높은 비중을 차지합니다.

description_exceptions:
  single_market:
    template: "포트폴리오는 {market_name} 시장의 {stock_count}개 종목으로 구성됩니다."
  multi_market:
    template: |
      포트폴리오는 {market_kr_pct}% 국내, {market_us_pct}% 해외로 구성되어 있으며,
      {top_sector}이(가) {top_sector_pct}%로 가장 높은 비중을 차지합니다.
  single_stock:
    template: "{stock_name} 단일 종목 포트폴리오입니다."
    chart_override: "도넛 차트 대신 종목 정보 카드 1개 표시"
  sector_unavailable:
    template: "포트폴리오는 {stock_count}개 종목으로 구성됩니다. (섹터 정보 미제공)"
```

---

## 5. Return Analysis

```yaml
section: return_analysis
charts:
  - chart_ref: "return_analysis.horizontal_bar"
    title: "종목별 수익률"
    width: "100%"
    min_data: 1
    fallback_chain:
      1: { condition: "종목 30개 초과", action: "Top 10 + Bottom 10만 표시 + '전체 보기' 토글" }
      2: { condition: "수익률 계산 불가 종목 존재", action: "계산 가능 종목만 표시 + 안내" }
      3: { condition: "모든 종목 수익률 계산 불가", action: "텍스트 안내 표시" }
      4: { condition: "차트 렌더링 실패", action: "수익률 텍스트 테이블" }

  - chart_ref: "return_analysis.bar_grouped"
    title: "섹터별 수익률"
    width: "100%"
    condition: "섹터 2개 이상"
    fallback_chain:
      1: { condition: "섹터 1개", action: "이 차트 숨김" }
      2: { condition: "렌더링 실패", action: "섹터별 수익률 텍스트 테이블" }

  - chart_ref: "distribution.histogram"
    title: "수익률 분포"
    width: "100%"
    condition: "종목 5개 이상"
    fallback_chain:
      1: { condition: "종목 4개 이하", action: "히스토그램 대신 각 종목 수익률 나열" }

layout: "수직 스택"
gap: "24px"

exceptions:
  all_returns_zero:
    condition: "모든 종목의 수익률이 0%"
    action: "차트 정상 표시 + 안내 메시지"
    message: "모든 종목의 수익률이 0%입니다. 매수가와 현재가가 동일하거나 데이터를 확인해 주세요."
  extreme_outlier:
    condition: "특정 종목 수익률이 ±500% 초과"
    action: "차트 축 자동 조정 + 해당 종목에 '⚠️' 마커"
  mixed_currency_returns:
    condition: "KRW/USD 종목 혼합"
    action: "각 종목의 원화 기준 수익률로 통일하여 비교"
    sub_note: "* 환율 변동 효과 포함"
```

---

## 6. Risk Analysis (optional)

```yaml
section: risk_analysis
condition: "analysis_mode in ['full_mode'] OR 고급 지표 1개 이상 계산 가능"

charts:
  - chart_ref: "time_series.line"
    title: "수익률 추이"
    width: "100%"
    min_data: "시계열 데이터 10일 이상"
    fallback_chain:
      1: { condition: "시계열 5~9일", action: "일별 막대 차트로 대체" }
      2: { condition: "시계열 4일 이하", action: "텍스트 요약 (시작점 → 끝점 수익률)" }
      3: { condition: "렌더링 실패", action: "기간별 수익률 표" }

  - chart_ref: "risk_return.scatter"
    title: "리스크 vs 수익률"
    width: "50%"
    min_data: "종목 3개 이상 + 변동성 계산 가능"
    fallback_chain:
      1: { condition: "종목 2개 이하", action: "텍스트 비교표로 대체" }
      2: { condition: "변동성 계산 불가", action: "이 차트 숨김" }

  - chart_ref: "time_series.mdd"
    title: "MDD 추이"
    width: "50%"
    min_data: "시계열 데이터 20일 이상"
    fallback_chain:
      1: { condition: "시계열 10~19일", action: "단순 MDD 값만 카드로 표시" }
      2: { condition: "시계열 9일 이하", action: "이 차트 숨김" }

additional_metrics:
  display: "카드 형태로 차트 상단에 배치"
  layout: "가로 나열 (반응형)"
  metrics:
    - id: "sharpe_ratio"
      label: "샤프 비율"
      format: "{value:.2f}"
      fallback: "N/A"
      tooltip: "무위험이자율 대비 초과 수익의 위험 조정 성과"
    - id: "mdd"
      label: "최대 낙폭"
      format: "{value:.1f}%"
      fallback: "N/A"
      tooltip: "고점 대비 최대 하락 폭"
    - id: "volatility"
      label: "변동성"
      format: "{value:.1f}%"
      fallback: "N/A"
      tooltip: "수익률의 연간화 표준편차"
    - id: "beta"
      label: "베타"
      format: "{value:.2f}"
      fallback: "N/A"
      tooltip: "벤치마크 대비 민감도"
    - id: "sortino_ratio"
      label: "소르티노 비율"
      format: "{value:.2f}"
      fallback: "N/A"
      tooltip: "하방 위험 대비 초과 수익 성과"
      condition: "sortino_ratio 계산 가능"

exceptions:
  partial_metrics:
    condition: "일부 고급 지표만 계산 가능"
    action: "계산 가능한 지표만 표시, 불가능한 지표는 'N/A' + 설명 tooltip"
  no_advanced_metrics:
    condition: "고급 지표 전부 계산 불가이나 섹션이 활성화됨"
    action: "섹션 전체를 안내 메시지로 대체"
    message: "리스크 분석에 필요한 시계열 데이터가 부족합니다. 일별 거래 내역을 포함한 데이터를 업로드하면 더 상세한 분석이 가능합니다."
  single_stock_risk:
    adjustment: "scatter 차트 숨김 (비교 대상 없음), MDD/변동성만 카드로 표시"
```

---

## 7. Market Comparison (optional)

```yaml
section: market_comparison
condition: "포트폴리오 내 market 유형이 2개 이상"

content:
  - chart_ref: "market_composition.stacked_bar"
    title: "마켓별 자산 배분"
    width: "100%"

  - comparison_table:
      title: "마켓별 성과 비교"
      columns: ["마켓", "종목 수", "투자금액", "평가금액", "수익률", "비중"]
      rows_from: "market 기준 집계"

  - description_auto: true
    template: |
      국내 시장에 {kr_pct}%, 해외 시장에 {us_pct}%를 투자하고 있으며,
      {better_market} 시장의 수익률({better_return}%)이 더 높습니다.

exceptions:
  three_or_more_markets:
    condition: "KR, US 외 추가 마켓 존재 (crypto, etc.)"
    action: "마켓별 행 추가, 차트 색상 자동 할당"
  one_market_dominant:
    condition: "한 마켓이 95% 이상"
    action: "비교보다는 단일 마켓 집중 안내 메시지"
    message: "투자 자산의 {pct}%가 {market} 시장에 집중되어 있습니다."
  currency_note:
    action: "비교 시 환산 기준 명시"
    template: "* 금액 비교는 원화({exchange_rate}) 기준입니다."
```

---

## 8. Stock Detail Table

```yaml
section: stock_detail_table

columns:
  # 항상 표시되는 컬럼
  - header: "종목명"
    field: "name"
    width: "auto"
    min_width: "120px"
    sticky: true  # 가로 스크롤 시 고정

  - header: "티커"
    field: "ticker"
    width: "80px"
    mobile_hide: true  # 모바일에서 숨김

  - header: "시장"
    field: "market"
    width: "50px"
    display: "뱃지 (KR=파란, US=초록)"
    mobile_hide: true

  - header: "수량"
    field: "quantity"
    align: "right"
    format: "integer_with_comma"
    fallback: "-"

  - header: "매수가"
    field: "avg_price"
    format: "currency"
    align: "right"
    fallback: "-"
    tooltip_when_fallback: "매수가 정보 없음"

  - header: "현재가"
    field: "current_price"
    format: "currency"
    align: "right"
    fallback: "-"
    tooltip_when_fallback: "현재가 정보 없음"

  - header: "수익률"
    field: "return_rate"
    format: "percentage_with_sign"
    color_rule: "positive=green, negative=red, zero=gray"
    align: "right"
    fallback: "-"

  - header: "평가금액"
    field: "valuation"
    format: "currency"
    align: "right"
    fallback: "-"

  - header: "비중"
    field: "weight"
    format: "percentage_1dp"  # "12.3%"
    align: "right"
    fallback: "-"

  # 조건부 컬럼 (고급 지표 존재 시)
  conditional_columns:
    - condition: "analysis_mode == 'full_mode'"
      columns:
        - header: "변동성"
          field: "volatility"
          format: "percentage_1dp"
          align: "right"
          fallback: "-"
        - header: "베타"
          field: "beta"
          format: "decimal_2dp"
          align: "right"
          fallback: "-"

sort:
  default: "valuation DESC"
  user_selectable: true
  options: ["valuation DESC", "return_rate DESC", "return_rate ASC", "weight DESC", "name ASC"]

style:
  stripe: true
  stripe_colors: ["#ffffff", "#f8f9fa"]
  highlight_top3:
    condition: "수익률 상위 3개"
    style: "좌측 border 3px green"
  highlight_bottom3:
    condition: "수익률 하위 3개"
    style: "좌측 border 3px red"
  row_height: "48px"
  font_size: "14px"
  header_style:
    background: "#f1f3f5"
    font_weight: "600"
    position: "sticky top"

pagination:
  enabled: true
  threshold: 20  # 20개 초과 시 페이지네이션 활성화
  page_size: 20
  show_all_option: true

search:
  enabled: true
  placeholder: "종목명 또는 티커 검색"
  fields: ["name", "ticker"]
  min_stocks_to_show: 10  # 10개 이상일 때만 검색 바 표시

exceptions:
  missing_columns:
    condition: "current_price 또는 avg_price 없음"
    action: "해당 컬럼은 '-'로 표시, 관련 파생 컬럼(수익률, 평가금액)도 '-'"
  duplicate_tickers:
    condition: "같은 티커가 여러 행 존재 (다른 계좌 등)"
    action: "각 행 표시 + 동일 티커 행에 같은 배경색 적용"
  too_many_stocks:
    condition: "종목 수 50개 초과"
    action: "기본 20개 표시 + 페이지네이션 + 경고"
    message: "종목이 {count}개로 많습니다. 페이지를 이동하거나 검색을 이용해 주세요."
  zero_weight_stocks:
    condition: "비중이 0% 또는 평가금액이 0"
    action: "테이블 하단에 별도 그룹으로 표시 + 연한 회색 텍스트"
    label: "평가금액 0원 종목 ({count}개)"
  mixed_currency:
    action: "통화 기호를 각 행에 명시 (₩ / $)"
    sort_note: "금액 정렬 시 원화 환산 기준"
```

---

## 9. Correlation & Diversification Analysis (Phase 3)

```yaml
# === Phase 3: New Report Sections ===

section: correlation_analysis
section_id: "correlation_analysis"
condition: "analysis_mode in ['full_mode', 'trade_history_mode'] AND stock_count >= 3"
order: "after risk_analysis (section 6)"

components:
  - chart_ref: "correlation_analysis.heatmap"
    title: "종목 간 상관관계"
    width: "100%"
    min_data: "종목 3개 이상 + 60일 이상 거래 이력"

  - diversification_ratio_card:
      title: "분산비율 (Diversification Ratio)"
      format: "{value:.2f}"
      interpretation:
        dr_gte_1_5: "양호한 분산투자"
        dr_between_1_2_1_5: "보통 수준의 분산"
        dr_lt_1_2: "분산 효과 제한적"

  - high_correlation_pairs_table:
      title: "높은 상관관계 종목 쌍"
      columns: ["종목 A", "종목 B", "상관계수"]
      max_rows: 5
      sort: "correlation DESC"

fallback:
  condition: "stock_count < 3 OR 거래 이력 < 60일"
  message: "분산투자 분석에는 3종목 이상과 60일 이상의 거래 이력이 필요합니다."
```

---

## 10. Dividend & Income Analysis (Phase 3)

```yaml
section: dividend_analysis
section_id: "dividend_analysis"
condition: "dividend records exist in parsed data"
order: "after correlation_analysis section"

components:
  - return_decomposition_card:
      title: "수익 구성 분해"
      display: "총 수익률 {total}% = 자본이익 {price}% + 배당수익 {income}%"

  - chart_ref: "dividend_analysis.monthly_bar"
    title: "월별 배당 수입"
    width: "100%"

  - dividend_calendar:
      title: "배당 일정"
      description: "향후 예정된 배당락일 표시"
      condition: "upcoming ex-dividend dates available"

  - tax_impact_note:
      title: "세전/세후 배당금 비교"
      display: "세전 {gross} → 세후 {net} (세율 {tax_rate}%)"

fallback:
  condition: "dividend records not found"
  message: "배당 데이터가 없어 배당 분석 섹션을 생략합니다."
```

---

## 11. Trading Activity Analysis (Phase 3)

```yaml
section: trading_activity
section_id: "trading_activity"
condition: "analysis_mode in ['trade_history_mode'] AND sell records exist"
order: "after dividend_analysis section"

components:
  - turnover_summary_card:
      title: "매매 활동 요약"
      metrics:
        - turnover_ratio: "연환산 회전율"
        - avg_holding_period: "평균 보유 기간"
        - trade_count: "총 거래 횟수"

  - chart_ref: "trading_activity.trend_line"
    title: "월별 회전율 추이"
    width: "100%"

  - fee_impact_card:
      title: "거래 비용 영향"
      display: "추정 연간 거래 비용: {annual_cost} (수익 대비 {cost_pct}%)"

  - churning_alert:
      condition: "turnover > 500% OR cost_to_return_ratio > 30%"
      type: "danger"
      message: "매매 빈도가 매우 높습니다. 거래 비용이 수익을 크게 잠식할 수 있습니다."

fallback:
  condition: "매도 내역 없음"
  message: "매도 내역이 없어 매매 활동 분석을 생략합니다."
```

---

## 12. Footer

```yaml
section: footer
content:
  - disclaimer:
      texts:
        - "본 리포트는 정보 제공 목적으로만 작성되었으며, 투자 권유가 아닙니다."
        - "과거 수익률이 미래 수익을 보장하지 않습니다."
        - "투자 판단의 책임은 투자자 본인에게 있습니다."
        - "세전 기준 수익률이며, 실제 수익은 세금 및 수수료에 따라 달라질 수 있습니다."
      display: "all texts joined with line break"
      style: "font-size: 11px, color: muted, border-top separator"
      required: true  # 절대 생략 불가

  - data_limitations:
      condition: "분석 모드가 full_mode가 아닐 때"
      text: "일부 분석이 데이터 제한으로 생략되었습니다. 더 정확한 분석을 위해 일별 거래 내역이 포함된 데이터를 업로드해 주세요."

  - parsing_warnings_summary:
      condition: "parsing 단계에서 경고 발생"
      text: "데이터 처리 중 {warning_count}건의 알림이 발생했습니다."
      detail_toggle: true  # 펼쳐서 상세 보기

  - estimated_values_notice:
      condition: "추정값이 사용된 경우"
      text: "* 일부 수치는 추정값을 포함합니다."

  - exchange_rate_notice:
      condition: "if mixed_currency portfolio"
      display: "환율 기준: {source} ({date} 기준, {rate})"
      # description: 환율 출처와 기준 시점을 명시하여 투명성 확보

  - generated_by:
      text: "AlphaFolio - AI 투자 대시보드"
      link: null  # 배포 URL은 추후 설정

  - timestamp:
      format: "YYYY-MM-DD HH:MM:SS KST"
      label: "리포트 생성 시각"

  - data_source:
      text: "사용자 업로드 데이터 기반"
      filename: "{original_filename}"

style:
  font_size: "12px"
  color: "#868e96"
  border_top: "1px solid #e9ecef"
  padding_top: "20px"
  margin_top: "40px"
  line_height: "1.8"
```
