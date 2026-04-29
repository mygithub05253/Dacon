# Report Rules — 리포트 생성 규칙

> AlphaFolio 리포트 출력 계층의 규칙을 정의합니다.
> 이 문서를 참조하여 분석 결과를 구조화된 리포트로 자동 구성합니다.
> 다른 Skills.md 문서의 출력을 소비하는 최종 계층으로, 누락·부분 데이터·렌더링 실패에 대한 폴백을 포함합니다.
>
> **파이프라인 위치**: Stage 4 (최종 출력 계층)
> **입력**: visualization_rules.md의 차트 객체 + insight_rules.md의 인사이트 카드 + analysis_rules.md의 지표
> **출력**: Streamlit 대시보드 페이지 / PDF 리포트 / CSV 데이터
> **상위 의존**: parsing_rules.md → analysis_rules.md → visualization_rules.md + insight_rules.md → 이 문서

---

## 1. 리포트 구성 흐름

```yaml
report_flow:
  description: "리포트는 위에서 아래로 읽히는 스토리 구조를 따름"
  story_arc: "요약 → 구성 → 성과 → 리스크 → 상세 → 마무리"

  sections:
    - order: 1
      id: "header"
      name: "리포트 헤더"
      required: true
      skip_condition: null  # 절대 스킵 불가
      data_source: "메타데이터 (파일명, 업로드 시간)"

    - order: 2
      id: "kpi_summary"
      name: "KPI 요약 카드"
      required: true
      skip_condition: null
      data_source: "analysis_rules.md 기본 지표"

    - order: 3
      id: "insight_highlight"
      name: "핵심 인사이트"
      required: true
      skip_condition: null
      data_source: "insight_rules.md 생성 결과"
      fallback: "인사이트 0개 시 기본 안내 메시지 표시"

    - order: 4
      id: "portfolio_composition"
      name: "포트폴리오 구성"
      required: true
      skip_condition: null
      data_source: "visualization_rules.md 포트폴리오 구성 차트"

    - order: 5
      id: "return_analysis"
      name: "수익률 분석"
      required: true
      skip_condition: null
      data_source: "analysis_rules.md 수익률 지표 + visualization_rules.md 차트"

    - order: 6
      id: "risk_analysis"
      name: "리스크 분석"
      required: false
      skip_condition: "analysis_mode in ['minimal_mode', 'trade_history_mode'] AND 고급 지표 없음"
      data_source: "analysis_rules.md 고급 지표"
      skip_message: "거래 내역 데이터가 충분하지 않아 리스크 분석을 표시하지 않습니다."

    - order: 7
      id: "market_comparison"
      name: "마켓 비교"
      required: false
      skip_condition: "market 유형이 1개만 존재"
      data_source: "analysis_rules.md 마켓별 집계"
      skip_message: null  # 조용히 스킵

    - order: 8
      id: "stock_detail_table"
      name: "종목 상세 테이블"
      required: true
      skip_condition: null
      data_source: "표준 스키마 + 계산된 지표"

    - order: 9
      id: "footer"
      name: "리포트 푸터"
      required: true
      skip_condition: null
      data_source: "메타데이터"

  # 분석 모드별 섹션 활성화 매핑
  mode_section_map:
    full_mode:
      active: ["header", "kpi_summary", "insight_highlight", "portfolio_composition",
               "return_analysis", "risk_analysis", "market_comparison", "stock_detail_table", "footer"]
    standard_mode:
      active: ["header", "kpi_summary", "insight_highlight", "portfolio_composition",
               "return_analysis", "stock_detail_table", "footer"]
      disabled_message:
        risk_analysis: "시계열 데이터 부족으로 리스크 분석이 제한됩니다."
    minimal_mode:
      active: ["header", "kpi_summary", "insight_highlight", "portfolio_composition",
               "stock_detail_table", "footer"]
      disabled_message:
        return_analysis: "수익률 계산에 필요한 데이터가 부족합니다."
        risk_analysis: "리스크 분석에 필요한 데이터가 부족합니다."
    trade_history_mode:
      active: ["header", "kpi_summary", "insight_highlight", "return_analysis",
               "stock_detail_table", "footer"]
      disabled_message:
        portfolio_composition: "현재 보유 현황 데이터가 없어 구성 분석을 생략합니다."

  # 섹션 렌더링 실패 시 전역 폴백
  section_render_failure:
    strategy: "skip_with_notice"
    notice_template: "'{section_name}' 섹션을 생성할 수 없습니다. (원인: {error_summary})"
    notice_style:
      background: "#f8f9fa"
      border: "1px dashed #dee2e6"
      color: "#868e96"
      padding: "16px"
    max_failed_sections: 3  # 4개 이상 실패 시 아래 참조
    too_many_failures:
      action: "emergency_report"
      description: "required 섹션 4개 이상 실패 시 최소 리포트 모드"
      content: ["header (에러 포함)", "stock_detail_table (원본 데이터 표시)", "footer"]
      message: "분석 중 여러 오류가 발생하여 기본 데이터만 표시합니다. 데이터 형식을 확인해 주세요."
```

---

## 2. 섹션별 상세 규칙

### 2.1 리포트 헤더

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

### 2.2 KPI 요약 카드

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

### 2.3 핵심 인사이트

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

### 2.4 포트폴리오 구성

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

### 2.5 수익률 분석

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

### 2.6 리스크 분석 (선택)

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

### 2.7 마켓 비교 (선택)

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

### 2.8 종목 상세 테이블

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

### 2.9 리포트 푸터

```yaml
section: footer
content:
  - disclaimer:
      text: "본 리포트는 정보 제공 목적으로 생성되었으며, 투자 권유가 아닙니다."
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
      condition: "mixed_currency == true"
      text: "* 환율 기준: {exchange_rate_source}, {exchange_rate_date}"

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

---

## 3. 내보내기 규칙

### 3.1 대시보드 뷰 (기본)

```yaml
export_dashboard:
  format: "Streamlit 페이지"
  features:
    interactive_charts: true
    filters: true
    real_time_update: true
    sidebar_navigation: true
  description: "기본 출력 형태. 인터랙티브 차트와 필터 포함."

  performance:
    lazy_loading: true
    description: "risk_analysis 등 하단 섹션은 스크롤 시 렌더링"
    chart_cache:
      enabled: true
      ttl: 300  # 5분간 캐시
      invalidate_on: "데이터 재업로드"

  error_handling:
    streamlit_crash:
      action: "st.error()로 사용자 친화적 메시지 표시"
      message: "리포트 생성 중 오류가 발생했습니다. 페이지를 새로고침해 주세요."
      log: "에러 스택 트레이스를 콘솔에 기록"
    session_timeout:
      action: "데이터 재업로드 유도"
      message: "세션이 만료되었습니다. 데이터를 다시 업로드해 주세요."
```

### 3.2 PDF 내보내기 (확장)

```yaml
export_pdf:
  format: "PDF"
  method: "Streamlit 다운로드 버튼 + 서버사이드 PDF 생성"
  library_priority:
    1: "fpdf2"
    2: "reportlab"  # fpdf2 실패 시 폴백
  
  layout:
    page_size: "A4"
    orientation: "portrait"
    margin:
      top: "20mm"
      bottom: "20mm"
      left: "15mm"
      right: "15mm"
    header_on_every_page: true
    footer_on_every_page: true
    page_numbers: true

  content_rules:
    charts:
      conversion: "plotly.io.to_image(format='png', width=800, height=400, scale=2)"
      fallback_chain:
        1: { condition: "to_image 실패 (kaleido 미설치)", action: "matplotlib로 재생성" }
        2: { condition: "matplotlib도 실패", action: "차트 위치에 텍스트 요약 삽입" }
        3: { condition: "모든 차트 변환 실패", action: "텍스트 전용 PDF" }
      dpi: 150
      max_width: "180mm"
    tables:
      style: "그리드 라인 포함, stripe 배경"
      overflow: "가로 넘침 시 폰트 축소 (최소 8pt) → 그래도 넘치면 열 분할"
      max_rows_per_page: 30
    fonts:
      korean: "NanumGothic 또는 시스템 한글 폰트"
      fallback: "DejaVu Sans (한글 깨짐 시)"
      font_detection:
        action: "PDF 생성 전 한글 폰트 가용 여부 확인"
        missing_action: "st.warning('한글 폰트 미설치로 일부 텍스트가 깨질 수 있습니다')"
    colors:
      positive: "#22B14C"  # 인쇄 친화 녹색
      negative: "#E24B4A"  # 인쇄 친화 적색
      text: "#212529"
      background: "#ffffff"
      note: "화면 색상과 다름 — 인쇄/PDF 가독성 최적화"

  filename_template: "AlphaFolio_Report_{YYYYMMDD}_{HHmm}.pdf"
  max_file_size: "10MB"
  
  exceptions:
    generation_timeout:
      threshold: "30초"
      action: "생성 취소 + 에러 메시지"
      message: "PDF 생성에 시간이 너무 오래 걸립니다. 종목 수를 줄이거나 다시 시도해 주세요."
    generation_failure:
      action: "에러 로그 기록 + 사용자 안내"
      message: "PDF 생성에 실패했습니다. 대시보드에서 직접 확인하시거나 브라우저 인쇄 기능(Ctrl+P)을 이용해 주세요."
      fallback_offer: "CSV 다운로드 대안 제공"
    large_portfolio:
      condition: "종목 수 100개 초과"
      action: "PDF 생성 전 경고"
      message: "종목이 {count}개로 많아 PDF가 여러 페이지에 걸칠 수 있습니다. 생성하시겠습니까?"
    empty_sections:
      action: "빈 섹션은 PDF에서 완전히 제거 (빈 공간 남기지 않음)"
```

### 3.3 CSV 내보내기 (폴백 + 보조)

```yaml
export_csv:
  format: "CSV"
  description: "PDF 실패 시 폴백 또는 데이터 분석용 원시 데이터 내보내기"
  
  files:
    - name: "portfolio_summary.csv"
      content: "KPI 요약 (지표명, 값)"
      encoding: "UTF-8-BOM"  # Excel 호환

    - name: "stock_detail.csv"
      content: "종목 상세 테이블 전체 데이터"
      encoding: "UTF-8-BOM"

    - name: "insights.csv"
      content: "인사이트 목록 (유형, 심각도, 메시지)"
      encoding: "UTF-8-BOM"
      condition: "인사이트 1개 이상"

  packaging:
    single_file: "stock_detail.csv만 단독 다운로드 가능"
    bundle: "전체를 ZIP으로 묶어 다운로드"
    zip_filename: "AlphaFolio_Data_{YYYYMMDD}.zip"

  exceptions:
    encoding_issue:
      action: "CP949 인코딩 옵션도 제공 (한국 Excel 사용자용)"
    empty_data:
      action: "빈 CSV 대신 에러 메시지"
      message: "내보낼 데이터가 없습니다."
```

---

## 4. 반응형 레이아웃 규칙

```yaml
responsive_layout:
  breakpoints:
    desktop: "width >= 1024px"
    tablet: "768px <= width < 1024px"
    mobile: "width < 768px"

  adaptations:
    desktop:
      sidebar: "visible"
      chart_layout: "2열 가능"
      table_scroll: "horizontal if needed"

    tablet:
      sidebar: "collapsible"
      chart_layout: "2열 유지, 최소 너비 320px"
      table_scroll: "horizontal"
      font_scale: 0.95

    mobile:
      sidebar: "hidden (햄버거 메뉴)"
      chart_layout: "1열 풀 너비"
      table_columns_hide: ["ticker", "market", "beta", "volatility"]
      table_scroll: "horizontal with indicator"
      font_scale: 0.9
      kpi_cards: "2열 × 2행"
      chart_height_scale: 0.8  # 높이 20% 축소

  streamlit_specific:
    wide_mode: true
    description: "st.set_page_config(layout='wide')"
    container_max_width: "1200px"
    column_gap: "1rem"
```

---

## 5. 접근성 규칙

```yaml
accessibility:
  color:
    contrast_ratio_min: 4.5  # WCAG AA 기준
    colorblind_safe: true
    description: "모든 색상 코딩에 아이콘/패턴을 함께 사용"
    examples:
      positive: "녹색 + ▲ 아이콘 + '+' 부호"
      negative: "적색 + ▼ 아이콘 + '-' 부호"
      neutral: "회색 + '—' 기호"

  screen_reader:
    chart_alt_text: true
    description: "모든 차트에 aria-label 또는 alt 속성 포함"
    template: "{chart_title}: {summary_text}"

  keyboard:
    tab_navigation: true
    sort_toggle: "테이블 헤더 클릭/Enter로 정렬 변경"

  text:
    min_font_size: "12px"
    line_height: "1.5"
    language: "ko"
```

---

## 6. 확장 규칙

```yaml
extension_points:
  custom_sections:
    description: "자산군별 추가 섹션"
    how_to_add: |
      1. 섹션 정의를 report_flow.sections에 추가
      2. condition과 data_source 명시
      3. fallback_chain 포함 (최소 텍스트 폴백 1개)
      4. mode_section_map에 해당 모드 매핑 추가

    crypto:
      - section: "defi_positions"
        name: "DeFi 포지션 현황"
        order: "risk_analysis 다음"
        condition: "DeFi 프로토콜 데이터 존재"
        charts: ["protocol_allocation.donut", "yield_comparison.bar"]

      - section: "stablecoin_ratio"
        name: "스테이블코인 비율"
        order: "portfolio_composition 내 하위 섹션"

    bond:
      - section: "maturity_schedule"
        name: "만기 일정"
        order: "risk_analysis 다음"
        charts: ["maturity_timeline.gantt"]

      - section: "coupon_calendar"
        name: "이자 수령 일정"
        order: "maturity_schedule 다음"
        charts: ["coupon_monthly.bar"]

      - section: "duration_analysis"
        name: "듀레이션 분석"
        condition: "채권 3종목 이상"

    real_estate:
      - section: "occupancy_dashboard"
        name: "입주율 현황"
      - section: "nav_premium_discount"
        name: "NAV 프리미엄/디스카운트"

  export_formats:
    description: "추가 내보내기 형식"
    future:
      - format: "Excel (.xlsx)"
        complexity: "medium"
        description: "시트별 섹션 분리 (요약, 상세, 인사이트)"
      - format: "이메일 전송"
        complexity: "high"
        description: "PDF 첨부 + 요약 본문"
      - format: "Slack 공유"
        complexity: "medium"
        description: "KPI 요약 + 인사이트를 Slack 메시지로"
      - format: "Notion 페이지"
        complexity: "high"
        description: "Notion API로 구조화된 페이지 생성"

  theming:
    description: "리포트 테마 커스터마이징"
    customizable:
      - "primary_color"
      - "font_family"
      - "header_background"
      - "card_border_radius"
    preset_themes:
      - name: "Default (Dark Header)"
        primary: "#1a1a2e"
      - name: "Light"
        primary: "#ffffff"
      - name: "Corporate Blue"
        primary: "#0052CC"
```

---

## 7. 에러 상태 전체 매핑

```yaml
error_state_map:
  description: "리포트 생성 과정에서 발생 가능한 모든 에러와 대응"

  pipeline_errors:
    - stage: "parsing"
      error: "파싱 완전 실패 (유효 데이터 0건)"
      report_action: "리포트 생성 불가 페이지 표시"
      message: "업로드된 파일에서 투자 데이터를 인식할 수 없습니다. 지원하는 형식을 확인해 주세요."
      show: ["파일 형식 가이드 링크", "샘플 파일 다운로드 버튼"]

    - stage: "parsing"
      error: "부분 파싱 성공 (일부 행 오류)"
      report_action: "성공 데이터로 리포트 생성 + 경고"
      message: "전체 {total}건 중 {failed}건의 데이터를 처리하지 못했습니다."
      footer_note: true

    - stage: "analysis"
      error: "지표 계산 일부 실패"
      report_action: "계산 성공 지표로 리포트 생성 + 해당 섹션 부분 표시"
      missing_metric_display: "N/A"

    - stage: "analysis"
      error: "지표 계산 전체 실패"
      report_action: "원시 데이터 테이블만 표시"
      sections_available: ["header", "stock_detail_table (원본 데이터)", "footer"]
      message: "분석에 필요한 데이터가 부족합니다. 원본 데이터를 확인해 주세요."

    - stage: "visualization"
      error: "특정 차트 렌더링 실패"
      report_action: "해당 차트의 fallback_chain 실행"
      final_fallback: "텍스트 요약 또는 숨김"

    - stage: "visualization"
      error: "전체 차트 렌더링 실패"
      report_action: "텍스트 전용 리포트 모드"
      message: "차트 생성에 실패했습니다. 텍스트 기반 분석 결과를 표시합니다."

    - stage: "insight"
      error: "인사이트 생성 실패"
      report_action: "인사이트 섹션 스킵, 나머지 정상 표시"
      notice: "인사이트 분석을 수행할 수 없었습니다."

    - stage: "report"
      error: "PDF 생성 실패"
      report_action: "대시보드 정상 + PDF 버튼에 에러 표시"
      alternative: "CSV 다운로드 또는 브라우저 인쇄(Ctrl+P) 안내"

    - stage: "report"
      error: "메모리 부족 (대용량 데이터)"
      report_action: "데이터 샘플링 후 재시도"
      threshold: "종목 500개 초과"
      sampling: "평가금액 상위 100개로 축소"
      message: "데이터가 너무 많아 상위 100개 종목으로 분석합니다."

  graceful_degradation_levels:
    level_1:
      name: "완전 리포트"
      condition: "모든 파이프라인 정상"
      sections: "전체"

    level_2:
      name: "부분 리포트"
      condition: "일부 지표/차트 누락"
      sections: "사용 가능한 섹션만 + 누락 안내"

    level_3:
      name: "최소 리포트"
      condition: "분석 대부분 실패"
      sections: ["header", "kpi_summary (가능한 것만)", "stock_detail_table", "footer"]
      message: "제한된 데이터로 기본 리포트를 생성했습니다."

    level_4:
      name: "원시 데이터"
      condition: "분석 전체 실패"
      sections: ["header (에러 표시)", "stock_detail_table (원본)", "footer"]
      message: "분석에 실패하여 원본 데이터만 표시합니다."

    level_5:
      name: "에러 페이지"
      condition: "파싱부터 실패"
      sections: ["에러 메시지", "파일 형식 안내", "샘플 다운로드"]
```
