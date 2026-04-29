# Report Flow — 리포트 구성 흐름

> Structure follows a top-to-bottom story arc.
> Sections activate/deactivate based on analysis mode.

---

## 1. Flow Definition

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
```

---

## 2. Mode-Section Map

```yaml
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
```

---

## 3. Section Render Failure (global fallback)

```yaml
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
