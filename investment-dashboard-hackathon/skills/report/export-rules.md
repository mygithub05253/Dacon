# Export Rules — 내보내기 규칙

> Dashboard view, PDF export, CSV export, responsive layout, and accessibility.

---

## 1. Dashboard View (default)

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

---

## 2. PDF Export (extension)

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

---

## 3. CSV Export (fallback + auxiliary)

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

## 4. Responsive Layout

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

## 5. Accessibility

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
