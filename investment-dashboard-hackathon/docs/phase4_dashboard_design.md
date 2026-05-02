# Phase 4. 대시보드 UI 구성 설계

## 설계 원칙

1. **데이터가 주인공** — UI 장식을 최소화하고, 업로드 후 3초 안에 의미 있는 결과를 보여준다.
2. **점진적 공개** — 페이지별로 분석 깊이를 단계적으로 높인다. 개요 → 상세 → 심화.
3. **분석 모드 투명성** — 현재 어떤 모드로 분석되었는지, 왜 일부 기능이 비활성인지 사용자에게 명시한다.
4. **실패에 강한 UI** — 어떤 단계가 실패하더라도 빈 화면이 아닌, 안내 메시지와 대안을 보여준다.

---

## 1. 멀티페이지 구조

Streamlit의 `pages/` 디렉토리 기반 멀티페이지 기능을 사용합니다.

```
┌──────────────────────────────────────────────────────────────┐
│  🏠 홈         📊 포트폴리오       📈 성과분석                 │
│                ⚠️ 리스크          📋 종목상세       📄 리포트   │
├──────────┬───────────────────────────────────────────────────┤
│          │                                                   │
│  사이드바  │              각 페이지별 메인 콘텐츠                │
│  (공통)   │                                                   │
│          │                                                   │
└──────────┴───────────────────────────────────────────────────┘
```

### 페이지 구성

| 순서 | 페이지 | 파일명 | 핵심 역할 | 활성 조건 |
|------|--------|--------|-----------|----------|
| 1 | 🏠 홈 | `main.py` | 업로드 + KPI 개요 + 핵심 인사이트 | 항상 |
| 2 | 📊 포트폴리오 | `pages/1_📊_포트폴리오.py` | 종목/섹터/마켓별 구성 분석 | 데이터 업로드 후 |
| 3 | 📈 성과분석 | `pages/2_📈_성과분석.py` | 수익률 분석, 섹터별 비교, 벤치마크, 배당 소득, 매매 활동 | 데이터 업로드 후 |
| 4 | ⚠️ 리스크 | `pages/3_⚠️_리스크.py` | 고급 지표, MDD, 변동성, 시계열, 상관관계 히트맵 | full_mode 전용 |
| 5 | 📋 종목상세 | `pages/4_📋_종목상세.py` | 전체 종목 테이블, 검색, 필터 | 데이터 업로드 후 |
| 6 | 📄 리포트 | `pages/5_📄_리포트.py` | 통합 리포트 뷰 + PDF/CSV 내보내기 | 데이터 업로드 후 |

### 페이지 간 데이터 공유

```yaml
data_sharing:
  method: "st.session_state"
  description: |
    모든 페이지가 st.session_state를 통해 동일한 분석 결과를 참조합니다.
    홈에서 업로드 + 파이프라인 실행 → 결과를 session_state에 저장 →
    나머지 페이지는 session_state에서 데이터를 꺼내 화면만 렌더링.

  guard:
    description: "데이터 없이 하위 페이지 접근 시 처리"
    check: "if 'parsed_df' not in st.session_state"
    action: |
      st.warning("먼저 🏠 홈에서 데이터를 업로드해 주세요.")
      st.page_link("main.py", label="🏠 홈으로 이동", icon="🏠")
      st.stop()
```

### 분석 모드별 페이지 활성화

```yaml
mode_page_map:
  full_mode:
    active: ["홈", "포트폴리오", "성과분석", "리스크", "종목상세", "리포트"]

  standard_mode:
    active: ["홈", "포트폴리오", "성과분석", "종목상세", "리포트"]
    disabled:
      리스크: "시계열 데이터가 부족하여 리스크 분석이 제한됩니다."

  minimal_mode:
    active: ["홈", "포트폴리오", "종목상세", "리포트"]
    disabled:
      성과분석: "수익률 계산에 필요한 데이터가 부족합니다."
      리스크: "리스크 분석에 필요한 데이터가 부족합니다."

  trade_history_mode:
    active: ["홈", "성과분석", "종목상세", "리포트"]
    disabled:
      포트폴리오: "현재 보유 현황 데이터가 없어 구성 분석을 생략합니다."
      리스크: "리스크 분석에 필요한 데이터가 부족합니다."

  disabled_page_behavior:
    action: "사이드바 네비게이션에 비활성(회색) 표시"
    click_action: "st.info로 비활성 이유 안내 + 필요한 데이터 형식 가이드"
```

---

## 2. 공통 사이드바

모든 페이지에서 동일한 사이드바를 공유합니다. `components/sidebar.py`에서 렌더링.

```yaml
shared_sidebar:
  sections:
    # ─── 1. 데이터 업로드 (홈에서만 활성) ───
    - id: "upload"
      visibility: "홈 페이지에서만 표시"
      components:
        file_uploader:
          component: "st.file_uploader"
          label: "📂 투자 데이터 업로드"
          type: ["csv", "tsv", "xlsx", "xls", "txt"]
          accept_multiple_files: false
          help: "증권사에서 내려받은 CSV 파일을 올려주세요"

        sample_button:
          component: "st.button"
          label: "🎯 샘플 데이터로 시작"
          description: "국내 10종목 + 미국 ETF 5종목"

    # ─── 2. 데이터 현황 (업로드 후, 모든 페이지) ───
    - id: "data_status"
      visibility: "데이터 업로드 후, 모든 페이지"
      components:
        file_info:
          component: "st.markdown"
          template: |
            **📁 {filename}**
            {stock_count}종목 · {data_date} 기준
        mode_badge:
          component: "st.markdown (뱃지 HTML)"
          mapping:
            full_mode: "🟢 전체 분석"
            standard_mode: "🔵 표준 분석"
            minimal_mode: "🟡 기본 분석"
            trade_history_mode: "🟣 거래 내역 분석"
        change_file:
          component: "st.button"
          label: "🔄 다른 파일 업로드"
          behavior: "session_state 초기화 + 홈으로 이동"

    # ─── 3. 필터 (포트폴리오/성과/종목상세 페이지) ───
    - id: "filters"
      visibility: "포트폴리오, 성과분석, 종목상세 페이지에서만 표시"
      section_label: "🔍 필터"
      components:
        market_filter:
          component: "st.multiselect"
          label: "시장"
          options_from: "데이터 내 market 고유값"
          default: "전체 선택"
          disabled_when: "market 유형 1개뿐"
        sector_filter:
          component: "st.multiselect"
          label: "섹터"
          options_from: "데이터 내 sector 고유값"
          default: "전체 선택"
          hide_when: "섹터 정보 없음"
        return_filter:
          component: "st.select_slider"
          label: "수익률 범위"
          range: "min~max return_rate"
          default: "전체 범위"
          step: 5
        reset_button:
          component: "st.button"
          label: "🔄 필터 초기화"

    # ─── 4. 내보내기 (리포트 페이지에서만 강조, 나머지는 축소) ───
    - id: "export"
      visibility: "데이터 업로드 후, 모든 페이지"
      section_label: "📥 내보내기"
      components:
        pdf_button:
          component: "st.download_button"
          label: "📄 PDF 리포트"
        csv_button:
          component: "st.download_button"
          label: "📊 CSV 데이터"

  filter_sync:
    description: "필터 상태는 session_state에 저장 → 페이지 이동해도 유지"
    apply: "실시간 반영 (버튼 없이 즉시)"
    empty_result:
      message: "필터 조건에 맞는 종목이 없습니다."
      action: "필터 초기화 버튼 강조"
```

---

## 3. 페이지별 상세 설계

### 3.1 🏠 홈 (`main.py`)

진입점. 업로드 전/후 두 가지 상태를 관리합니다.

#### 상태 A: 업로드 전 (랜딩)

```yaml
state_idle:
  layout:
    - hero:
        content: |
          # AlphaFolio
          당신의 포트폴리오에서 알파를 찾아주는 AI 투자 대시보드
        style: "text-align: center; padding: 48px 0 32px"

    - upload_cta:
        component: "st.container (border=True)"
        icon: "📂"
        title: "투자 데이터를 업로드해 주세요"
        subtitle: "증권사에서 내려받은 CSV 파일, 또는 샘플 데이터로 체험"
        inner_components:
          - file_uploader: "st.file_uploader (메인 영역에도 배치 — 사이드바와 이중)"
          - sample_button: "st.button('🎯 샘플 데이터로 체험하기')"
        style: "border: 2px dashed; border-radius: 16px; padding: 40px; text-align: center"

    - features:
        component: "st.columns(3)"
        cards:
          - icon: "📊"
            title: "자동 시각화"
            desc: "종목/섹터/마켓별 차트 자동 생성"
          - icon: "🤖"
            title: "AI 인사이트"
            desc: "20개 이상 규칙으로 위험과 기회 감지"
          - icon: "🔄"
            title: "범용 파서"
            desc: "주요 증권사 CSV 자동 인식"

    - supported_formats:
        component: "st.expander('지원하는 파일 형식 보기')"
        content: "CSV, TSV, XLSX, XLS, TXT + 증권사 5개 프리셋"
```

#### 상태 B: 로딩 (분석 중)

```yaml
state_loading:
  component: "st.status('포트폴리오 분석 중...', expanded=True)"
  steps:
    - "📄 파일 읽는 중... (인코딩 감지, 컬럼 매핑)"
    - "🔧 데이터 정규화 중... (표준 스키마 변환)"
    - "📈 지표 계산 중... (수익률, 비중, 리스크)"
    - "📊 차트 생성 중... (데이터 맞춤 차트 선택)"
    - "🤖 인사이트 생성 중... (패턴 감지, 제안 생성)"
  timeout: 30
  timeout_message: "분석 시간이 초과되었습니다. 파일 크기를 확인해 주세요."
```

#### 상태 C: 개요 대시보드 (업로드 완료)

```yaml
state_overview:
  description: "한눈에 보는 포트폴리오 요약. 상세 분석은 각 페이지로 유도."

  sections:
    # ─── 헤더 배너 ───
    - id: "header_banner"
      component: "st.markdown(unsafe_allow_html=True)"
      content: |
        <div style='background: linear-gradient(135deg, #1a1a2e, #16213e);
                    padding: 28px; border-radius: 12px; color: white; text-align: center;'>
          <p style='margin: 0; font-size: 0.9rem; opacity: 0.8;'>{portfolio_name}</p>
          <h1 style='margin: 8px 0; font-size: 2.2rem;'>{total_valuation}</h1>
          <p style='margin: 0; font-size: 0.85rem; opacity: 0.7;'>
            {report_date} 기준 · {analysis_mode_badge}
          </p>
        </div>
      stale_warning:
        condition: "데이터 30일 이상 경과"
        component: "st.warning('⏰ 데이터가 {days}일 전 기준입니다.')"

    # ─── KPI 카드 ───
    - id: "kpi_cards"
      layout: "st.columns(4)"
      cards:
        - metric: "total_return"
          label: "총 수익률 (세전 기준)"
          format: "percentage_with_sign"
          color: "green/red"
          note: "세전/세후 토글 버튼 제공 (tax-fee-impact.md)"
        - metric: "total_profit_loss"
          label: "총 손익 (세전 기준)"
          format: "currency_display"
          color: "green/red"
        - metric: "stock_count"
          label: "보유 종목"
          format: "{count}개"
        - metric: "profitable_ratio"
          label: "수익 종목"
          format: "{profitable}/{total} ({ratio}%)"
      conditional:
        sharpe_available: "stock_count 카드를 sharpe_ratio로 교체"
        mdd_available: "5번째 카드로 MDD 추가"
      exceptions:
        all_missing: "stock_count + '데이터 부족' 안내만 표시"
        single_stock: "profitable_ratio 대신 해당 종목 수익률"

    # ─── 핵심 인사이트 ───
    - id: "insight_highlight"
      summary:
        component: "st.markdown"
        content: "종합 요약문 (insight_rules.md 결과)"
      cards:
        method: "st.markdown + 커스텀 HTML"
        max_display: 3
        style: "border-left: 4px solid {type_color}; background: {type_bg}"
      expand:
        condition: "인사이트 4개 이상"
        component: "st.expander('인사이트 {n}개 더 보기')"
      zero_insights:
        component: "st.info('현재 데이터에서 특별한 주의 사항은 감지되지 않았습니다.')"

    # ─── 미니 차트 (Quick Glance) ───
    - id: "quick_charts"
      description: "홈에서는 핵심 차트 2개만 작게 보여주고, 상세 페이지로 유도"
      layout: "st.columns(2)"
      left:
        chart: "도넛 차트 (종목별 비중) — 간소화 버전"
        component: "st.plotly_chart(use_container_width=True, config={displayModeBar: false})"
        caption: "st.page_link('📊 포트폴리오 상세 보기 →')"
      right:
        chart: "수익률 Top 5 / Bottom 5 수평 바"
        component: "st.plotly_chart(use_container_width=True)"
        caption: "st.page_link('📈 성과분석 상세 보기 →')"

    # ─── 페이지 네비게이션 카드 ───
    - id: "nav_cards"
      description: "각 하위 페이지로 연결되는 카드 — 사용자가 다음 행동을 알 수 있게"
      layout: "st.columns(4)"
      cards:
        - icon: "📊"
          title: "포트폴리오"
          desc: "종목/섹터/마켓 구성"
          link: "pages/1_📊_포트폴리오.py"
          disabled_when: "trade_history_mode"
        - icon: "📈"
          title: "성과분석"
          desc: "수익률 분석 · 비교"
          link: "pages/2_📈_성과분석.py"
          disabled_when: "minimal_mode"
        - icon: "⚠️"
          title: "리스크"
          desc: "MDD · 변동성 · 베타"
          link: "pages/3_⚠️_리스크.py"
          disabled_when: "not full_mode"
        - icon: "📋"
          title: "종목상세"
          desc: "전체 종목 테이블"
          link: "pages/4_📋_종목상세.py"
```

#### 상태 D: 에러 (파싱 실패)

```yaml
state_error:
  display:
    - component: "st.error('업로드된 파일에서 투자 데이터를 인식할 수 없습니다.')"
    - guidance:
        component: "st.markdown"
        content: "파일 형식, 필수 컬럼, 인코딩 확인 가이드"
    - detail:
        component: "st.expander('상세 오류 정보')"
    - sample:
        component: "st.download_button('📥 샘플 CSV 다운로드')"
    - retry:
        component: "st.button('🔄 다시 시도')"
```

---

### 3.2 📊 포트폴리오 (`pages/1_📊_포트폴리오.py`)

포트폴리오 구성과 자산 배분을 시각적으로 분석합니다.

```yaml
page_portfolio:
  title: "📊 포트폴리오 구성"
  subtitle: "종목, 섹터, 마켓별 자산 배분을 확인합니다."
  condition: "analysis_mode not in ['trade_history_mode']"

  sections:
    # ─── 1. 종목별 비중 ───
    - id: "stock_allocation"
      title: "종목별 비중"
      layout: "st.columns([3, 2])"
      left:
        chart: "도넛 차트"
        component: "st.plotly_chart"
        config:
          hole: 0.45
          top_n: 10
          others_threshold: 2.0
          center_text: "총 평가금액"
        fallback:
          single_stock: "단일 종목 카드"
          too_many: "상위 15 + 기타"
          render_fail: "비중 테이블"
      right:
        content: "비중 상위 종목 목록 (순위, 종목명, 비중%, 평가금액)"
        component: "st.dataframe"
        max_rows: 10
        highlight: "1위 종목 강조"

    # ─── 2. 섹터별 구성 ───
    - id: "sector_allocation"
      title: "섹터별 구성"
      condition: "섹터 정보 2개 이상"
      chart: "트리맵 (market > sector > stock)"
      component: "st.plotly_chart"
      config:
        color_by: "return_rate"
        color_scale: "RdYlGn"
      fallback:
        no_sector: "마켓별 도넛"
        single_market: "sector > stock 2단계"
        render_fail: "섹터별 비중 테이블"
      hidden_when: "섹터 정보 0건"

    # ─── 3. 마켓별 비교 ───
    - id: "market_comparison"
      title: "마켓별 비교"
      condition: "market 유형 2개 이상"
      layout: "st.columns([1, 1])"
      left:
        chart: "마켓별 자산 배분 (stacked_bar 또는 도넛)"
        component: "st.plotly_chart"
      right:
        content: "마켓별 성과 비교 테이블 (종목 수, 투자금액, 평가금액, 수익률, 비중)"
        component: "st.dataframe"
      description:
        component: "st.caption"
        template: "국내 {kr_pct}%, 해외 {us_pct}% 배분. {better_market} 시장의 수익률이 더 높습니다."
      hidden_when: "market 1개만"

    # ─── 4. 구성 요약 ───
    - id: "composition_summary"
      component: "st.markdown"
      content: "자동 생성 텍스트 요약 (report_rules.md description_template)"

  exceptions:
    single_stock:
      override: "전체 페이지를 단일 종목 상세 카드로 교체"
      content: "종목 정보 + 수익률 + 매수가/현재가 비교"
```

---

### 3.3 📈 성과분석 (`pages/2_📈_성과분석.py`)

수익률 기반 성과를 다각도로 분석합니다.

```yaml
page_performance:
  title: "📈 성과분석"
  subtitle: "종목별, 섹터별 수익률을 분석하고 비교합니다."
  condition: "analysis_mode not in ['minimal_mode']"

  sections:
    # ─── 1. 수익률 개요 카드 ───
    - id: "return_kpi"
      layout: "st.columns(4)"
      cards:
        - label: "최고 수익"
          value: "{best_stock_name} +{return}%"
          color: "green"
        - label: "최대 손실"
          value: "{worst_stock_name} {return}%"
          color: "red"
        - label: "평균 수익률"
          value: "{avg_return}%"
          color: "green/red"
        - label: "수익/손실 비율"
          value: "{win_count}:{loss_count}"

    # ─── 2. 종목별 수익률 ───
    - id: "stock_returns"
      title: "종목별 수익률"
      chart: "수평 바 차트 (정렬, 양/음 색상 분리)"
      component: "st.plotly_chart"
      config:
        sort: "descending"
        color_positive: "#639922"
        color_negative: "#E24B4A"
        zero_line: true
        max_bars: 20
      toggle:
        condition: "종목 20개 초과"
        component: "st.checkbox('전체 종목 보기')"
        default: false
        behavior: "top10+bottom10 ↔ 전체"
      fallback:
        all_na: "텍스트 안내"
        render_fail: "수익률 테이블"

    # ─── 3. 섹터별 수익률 ───
    - id: "sector_returns"
      title: "섹터별 수익률 비교"
      condition: "섹터 2개 이상"
      chart: "그룹 바 차트 (섹터별 평균 수익률)"
      component: "st.plotly_chart"
      additional: "각 섹터 내 종목 수, 평균/최대/최소 수익률 표시"
      hidden_when: "섹터 1개 이하"

    # ─── 4. 수익률 분포 ───
    - id: "return_distribution"
      title: "수익률 분포"
      condition: "종목 5개 이상"
      chart: "히스토그램 (7개 구간, 가우시안 커브 오버레이)"
      component: "st.plotly_chart"
      buckets: ["< -30%", "-30~-15%", "-15~0%", "0~15%", "15~30%", "30~50%", "> 50%"]
      fallback:
        few_stocks: "종목 4개 이하 → 각 종목 수익률 나열"

    # ─── 5. 수익/손실 기여 ───
    - id: "profit_contribution"
      title: "수익 기여도"
      condition: "종목 3개 이상"
      chart: "워터폴 차트 (각 종목의 총 손익 기여)"
      component: "st.plotly_chart"
      description: "어떤 종목이 포트폴리오 성과를 끌어올리고/끌어내리는지"
      fallback:
        render_fail: "기여도 테이블"

    # ─── 6. 배당 소득 분석 ───
    - id: "dividend_income"
      title: "배당 소득 분석"
      condition: "배당 정보 컬럼 존재 시"
      layout: "st.columns([3, 2])"
      left:
        chart: "월별 배당 수령 누적 막대 차트 (ticker별 스택)"
        component: "st.plotly_chart"
        config:
          x_axis: "월 (YYYY-MM)"
          stack_by: "ticker"
          y_label: "배당 수령액 (원)"
          bar_mode: "stack"
        fallback:
          no_dividend: "st.info('배당 내역이 확인되지 않았습니다.')"
          render_fail: "배당 테이블"
      right:
        content: "수익 분해 요약"
        metrics:
          - label: "자본이득"
            value: "{capital_gain_pct:.1f}%"
          - label: "배당소득"
            value: "{dividend_pct:.1f}%"
          - label: "포트폴리오 배당 수익률"
            value: "{dividend_yield:.2f}%"
        chart: "파이 차트 (자본이득 vs 배당소득 비율)"
        component: "st.plotly_chart"
      hidden_when: "배당 정보 없음"

    # ─── 7. 매매 회전율 ───
    - id: "turnover_analysis"
      title: "매매 활동 분석"
      condition: "거래 내역 존재 시"
      layout: "st.columns([3, 2])"
      left:
        chart: "포트폴리오 회전율 추이 라인 차트"
        component: "st.plotly_chart"
        config:
          x_axis: "분기"
          y_axis: "회전율 (%)"
          reference_lines:
            - value: 200
              label: "200% 경계 (잦은 매매)"
              color: "orange"
              dash: "dash"
            - value: 500
              label: "500% 경계 (과도한 단기 매매)"
              color: "red"
              dash: "dot"
        fallback:
          no_history: "섹션 숨김"
          render_fail: "회전율 수치 카드"
      right:
        content: "매매 활동 요약 카드"
        metrics:
          - label: "연간 회전율"
            value: "{annual_turnover:.0f}%"
            color_rule: "<200 green, 200~500 orange, >500 red"
          - label: "평균 보유기간"
            value: "{avg_holding_days:.0f}일"
          - label: "단타 비중"
            value: "{short_trade_pct:.1f}% (30일 미만)"
      hidden_when: "거래 내역 없음"

  exceptions:
    all_returns_zero:
      message: "모든 종목의 수익률이 0%입니다. 데이터를 확인해 주세요."
    extreme_outlier:
      action: "축 자동 조정 + 이상치 마커(⚠️)"
    mixed_currency:
      note: "* 원화 기준 수익률로 통일 (환율 변동 효과 포함)"
```

---

### 3.4 ⚠️ 리스크 (`pages/3_⚠️_리스크.py`)

고급 리스크 지표와 시계열 분석. full_mode 전용.

```yaml
page_risk:
  title: "⚠️ 리스크 분석"
  subtitle: "포트폴리오의 변동성, 낙폭, 위험 조정 성과를 분석합니다."
  condition: "analysis_mode == 'full_mode' OR 고급 지표 1개 이상"

  sections:
    # ─── 1. 리스크 지표 카드 ───
    - id: "risk_metrics"
      layout: "st.columns(동적 — 계산 가능한 지표 수)"
      metrics:
        - id: "sharpe_ratio"
          label: "샤프 비율"
          format: "{:.2f}"
          color_rule: ">=1 green, 0~1 orange, <0 red"
          tooltip: "무위험이자율 대비 초과 수익의 위험 조정 성과"
        - id: "mdd"
          label: "최대 낙폭 (MDD)"
          format: "{:.1f}%"
          color_rule: ">=-10 green, -10~-30 orange, <-30 red"
        - id: "volatility"
          label: "변동성"
          format: "{:.1f}%"
          tooltip: "수익률의 연간화 표준편차"
        - id: "beta"
          label: "베타"
          format: "{:.2f}"
          tooltip: "벤치마크 대비 민감도"
        - id: "sortino_ratio"
          label: "소르티노"
          format: "{:.2f}"
          condition: "계산 가능 시"
      fallback_value: "N/A"

    # ─── 2. 수익률 추이 ───
    - id: "return_timeseries"
      title: "수익률 추이"
      chart: "라인 차트 (일별/주별 수익률)"
      component: "st.plotly_chart"
      min_data: "시계열 10일 이상"
      features:
        benchmark_overlay: "벤치마크(KOSPI/S&P 500) 비교선 토글"
        period_selector: "st.radio(['1M', '3M', '6M', '1Y', '전체'])"
      fallback:
        short_data: "5~9일 → 일별 막대 차트"
        too_short: "4일 이하 → 텍스트 요약"

    # ─── 3. 리스크 vs 수익률 ───
    - id: "risk_return_scatter"
      title: "리스크 vs 수익률"
      layout: "st.columns([1, 1])"
      left:
        chart: "산점도 (x=변동성, y=수익률, size=비중)"
        component: "st.plotly_chart"
        min_data: "종목 3개 이상 + 변동성 계산 가능"
        config:
          quadrant_lines: true  # 평균 기준 4분면 표시
          labels: "종목명"
        fallback:
          few_stocks: "텍스트 비교표"
      right:
        chart: "MDD 추이 (area chart)"
        component: "st.plotly_chart"
        min_data: "시계열 20일 이상"
        fallback:
          short_data: "MDD 값만 카드로 표시"

    # ─── 4. 벤치마크 비교 ───
    - id: "benchmark_comparison"
      title: "벤치마크 대비 성과"
      condition: "벤치마크 데이터 사용 가능"
      content:
        - "vs KOSPI (국내 비중 높을 때)"
        - "vs S&P 500 (미국 비중 높을 때)"
        - "vs 60:40 포트폴리오 (혼합)"
      auto_selection_badge:
        component: "st.caption"
        template: "벤치마크: {benchmark_name} (자동 선택) — 변경 ▼"
        allow_override: true
      chart: "라인 차트 (포트폴리오 vs 벤치마크 누적 수익률)"
      component: "st.plotly_chart"

    # ─── 5. 상관관계 히트맵 ───
    - id: "correlation_heatmap"
      title: "종목 간 상관관계"
      condition: "종목 3개 이상 + 일별 수익률 시계열 존재"
      layout: "st.columns([3, 2])"
      left:
        chart: "히트맵 (Heatmap)"
        component: "st.plotly_chart"
        config:
          color_scale: "RdYlGn_r"  # 고상관(빨강) → 저상관(초록)
          annot: true               # 셀에 수치 표시
          fmt: ".2f"
          vmin: -1.0
          vmax: 1.0
        fallback:
          too_few_stocks: "종목 2개 이하 → 섹션 숨김"
          render_fail: "상관계수 테이블"
      right:
        content: "분산화 요약 카드"
        metrics:
          - label: "분산화 비율"
            value: "{diversification_ratio:.2f}"
            color_rule: ">=1.5 green, 1~1.5 orange, <1 red"
          - label: "평균 상관계수"
            value: "{avg_correlation:.2f}"
            color_rule: "<0.4 green, 0.4~0.7 orange, >0.7 red"
        interpretation:
          component: "st.caption"
          template: |
            분산화 비율 {ratio}은 {level} 수준입니다.
            평균 상관계수 {avg_corr}로 포트폴리오 종목들이 {direction} 움직이는 경향이 있습니다.

  disabled_page:
    component: "st.info"
    message: |
      리스크 분석은 **전체 분석 모드**에서만 제공됩니다.
      일별 거래 내역이 포함된 데이터를 업로드하면 더 상세한 분석이 가능합니다.
    show_required_data: true
```

---

### 3.5 📋 종목상세 (`pages/4_📋_종목상세.py`)

전체 종목의 상세 데이터를 테이블로 보여주고, 검색·정렬·필터 기능을 제공합니다.

```yaml
page_stock_detail:
  title: "📋 종목 상세"
  subtitle: "보유 종목의 상세 정보를 확인하고 검색할 수 있습니다."

  sections:
    # ─── 1. 검색/정렬 바 ───
    - id: "search_bar"
      layout: "st.columns([3, 1])"
      left:
        component: "st.text_input"
        label: "🔍 종목 검색"
        placeholder: "종목명 또는 티커를 입력하세요"
        condition: "종목 10개 이상일 때만 표시"
      right:
        component: "st.selectbox"
        label: "정렬"
        options:
          - "평가금액 ↓"
          - "수익률 ↓"
          - "수익률 ↑"
          - "비중 ↓"
          - "종목명 (가나다)"

    # ─── 2. 요약 통계 ───
    - id: "table_summary"
      layout: "st.columns(3)"
      stats:
        - label: "전체 {total}종목"
          sub: "수익 {profitable} · 손실 {losing} · 보합 {neutral}"
        - label: "최고 수익률"
          sub: "{best_name} +{best_return}%"
        - label: "최대 손실률"
          sub: "{worst_name} {worst_return}%"

    # ─── 3. 종목 테이블 ───
    - id: "stock_table"
      component: "st.dataframe"
      params:
        use_container_width: true
        hide_index: true
        height: 500
        column_config:
          수익률:
            type: "st.column_config.NumberColumn"
            format: "%.2f%%"
          평가금액:
            type: "st.column_config.NumberColumn"
            format: "₩%,.0f"
          비중:
            type: "st.column_config.ProgressColumn"
            min_value: 0
            max_value: 100
            format: "%.1f%%"
          시장:
            type: "st.column_config.TextColumn"
            width: "small"

      columns:
        always: ["종목명", "티커", "시장", "수량", "매수가", "현재가", "수익률", "평가금액", "비중"]
        conditional_full_mode: ["변동성", "베타"]

      styling:
        stripe: true
        top3_highlight: "수익률 상위 3개 — 좌측 green border"
        bottom3_highlight: "수익률 하위 3개 — 좌측 red border"
        color_coding:
          수익률_positive: "background: rgba(99, 153, 34, 0.1)"
          수익률_negative: "background: rgba(226, 75, 74, 0.1)"

    # ─── 4. 비중 0원 종목 (별도 그룹) ───
    - id: "zero_weight_section"
      condition: "평가금액 0원 종목 존재"
      component: "st.expander('평가금액 0원 종목 ({count}개)')"
      content: "별도 테이블, 회색 텍스트"

  exceptions:
    missing_columns:
      action: "해당 열 '-'로 표시"
    too_many_stocks:
      condition: "50개 초과"
      message: "종목이 {count}개로 많습니다. 검색을 이용해 주세요."
    mixed_currency:
      action: "통화 기호를 각 행에 명시 (₩/$)"
```

---

### 3.6 📄 리포트 (`pages/5_📄_리포트.py`)

모든 분석 결과를 하나의 스크롤 뷰로 통합하고, PDF/CSV 내보내기를 제공합니다.

```yaml
page_report:
  title: "📄 리포트"
  subtitle: "분석 결과를 통합 리포트로 확인하고 내보내기할 수 있습니다."
  description: |
    이 페이지는 report_rules.md의 섹션 순서를 따라 전체 분석 결과를
    인쇄/PDF 친화적인 단일 스크롤 뷰로 구성합니다.
    다른 페이지의 인터랙티브 차트와 달리, 여기서는 정적 요약 위주로 배치합니다.

  sections:
    # report_rules.md의 섹션을 순서대로 렌더링
    - "리포트 헤더 (포트폴리오명, 총 평가금액 (세전 기준), 기준일)"
    - "KPI 카드 (4~5개 핵심 지표, 수익률·손익에 '세전 기준' 라벨)"
    - "인사이트 요약 (종합 요약문 + 인사이트 카드 최대 5개)"
    - "포트폴리오 구성 (도넛 + 트리맵, 자동 설명)"
    - "수익률 분석 (종목별 바 + 섹터별 바)"
    - "리스크 분석 (full_mode 시 고급 지표 카드 + 차트)"
    - "상관관계 분석 (히트맵 + 분산화 비율, 종목 3개 이상 + 시계열 조건)"
    - "배당/소득 분석 (월별 배당 막대 + 수익 분해 파이, 배당 정보 조건)"
    - "매매 활동 분석 (회전율 추이 + 단타 비중, 거래 내역 조건)"
    - "마켓 비교 (multi-market 시 비교 테이블)"
    - "종목 상세 테이블 (전체 종목)"
    - "푸터 (면책 4개 항목 + 환율 출처 표기 + 생성 시각)"

  export:
    position: "페이지 최상단 + 최하단 양쪽에 내보내기 버튼 배치"
    
    pdf:
      component: "st.download_button"
      label: "📄 PDF 리포트 다운로드"
      library: "fpdf2 (fallback: reportlab)"
      config:
        page_size: "A4"
        orientation: "portrait"
        korean_font: "NanumGothic"
        charts_as_image: "plotly.io.to_image()"
      exceptions:
        generation_timeout: "30초 초과 → 에러 메시지"
        font_missing: "한글 폰트 미설치 경고"
        generation_fail: "CSV 대안 + 브라우저 인쇄(Ctrl+P) 안내"
        large_portfolio: "100종목+ → 생성 전 확인"

    csv:
      component: "st.download_button"
      label: "📊 CSV 데이터 다운로드"
      files:
        - "portfolio_summary.csv (KPI)"
        - "stock_detail.csv (종목 상세)"
        - "insights.csv (인사이트 목록)"
      encoding: "UTF-8-BOM (Excel 호환)"

  footer:
    component: "st.markdown(unsafe_allow_html=True)"
    disclaimers:
      - "본 리포트는 투자 참고용 정보이며, 투자 권유 또는 금융 상품 추천이 아닙니다."
      - "모든 수익률 및 손익 수치는 세전 기준이며, 실제 세후 수익은 개인 세율에 따라 다를 수 있습니다."
      - "과거의 수익률은 미래의 성과를 보장하지 않습니다. 투자 원금 손실이 발생할 수 있습니다."
      - "해외 자산의 원화 환산 금액은 기준일 환율을 적용하며, 환율 변동에 따라 실제 금액과 차이가 생길 수 있습니다."
    exchange_rate_notice:
      component: "st.caption"
      template: |
        환율 정보: {fx_rate} 원/USD ({fx_date} 기준) · 출처: {fx_source}
        ※ 실시간 환율은 서비스 조건에 따라 지연될 수 있습니다.
    generation_timestamp:
      component: "st.caption"
      template: "리포트 생성: {generated_at} · AlphaFolio v{version}"

  mode_adaptations:
    standard_mode: "리스크 섹션 → '데이터 부족' 안내 텍스트"
    minimal_mode: "수익률 + 리스크 섹션 생략"
    trade_history_mode: "포트폴리오 구성 생략"
    no_correlation: "상관관계 분석 섹션 → 숨김"
    no_dividend: "배당/소득 분석 섹션 → 숨김"
    no_trade_history: "매매 활동 분석 섹션 → 숨김"
```

---

## 4. 인터랙션 규칙

### 4.1 필터 ↔ 페이지 연동

```yaml
filter_sync:
  scope: "포트폴리오, 성과분석, 종목상세 페이지에서 동작"
  behavior: "필터 변경 시 해당 페이지의 모든 차트/테이블 실시간 갱신"
  session_persistence: "페이지 이동해도 필터 상태 유지"

  affected_per_page:
    포트폴리오: "도넛, 트리맵, 마켓 비교 — 필터된 종목만"
    성과분석: "수익률 바, 섹터별 비교, 분포 — 필터된 종목만"
    종목상세: "테이블 행 필터링"

  not_affected:
    - "홈의 KPI 카드 (전체 기준 표시, '필터 적용됨' 표시 없음)"
    - "인사이트 (항상 전체 포트폴리오 기준)"
    - "리포트 (항상 전체 포트폴리오 기준)"

  empty_result:
    action: "빈 차트/테이블 대신 안내 메시지 + 초기화 버튼"
```

### 4.2 차트 인터랙션

```yaml
chart_interactions:
  hover: "Plotly 기본 tooltip (종목명, 값, 비중)"
  click:
    donut: "클릭한 종목 하이라이트 + session_state에 저장"
    bar: "종목 상세 expander 열기"
    scatter: "tooltip 고정"
  zoom:
    line_chart: "드래그 줌 + 더블클릭 리셋"
  cross_page:
    description: "차트 클릭 → 종목상세 페이지 해당 행으로 연결 (선택적 구현)"
```

---

## 5. 세션 상태 관리

```yaml
session_state:
  # ─── 데이터 계층 ───
  raw_df: "pd.DataFrame | None — 업로드 원본"
  parsed_df: "pd.DataFrame | None — 표준 스키마 변환 완료"
  analysis_result: "dict | None — 지표 계산 결과"
  insights: "list[dict] | None — 인사이트 목록"
  analysis_mode: "str — full/standard/minimal/trade_history"

  # ─── UI 상태 ───
  filters:
    market: "list[str] — 선택된 시장"
    sector: "list[str] — 선택된 섹터"
    return_range: "tuple[float, float] — 수익률 범위"
    sort_by: "str — 정렬 기준"
  show_all_stocks: "bool — 수익률 차트 전체 보기 토글"

  # ─── 메타 ───
  file_hash: "str — MD5 해시 (캐싱용)"
  filename: "str — 원본 파일명"
  processing_time: "float — 처리 시간(초)"
  parsing_warnings: "list[str] — 파싱 경고"

  lifecycle:
    on_new_upload: "file_hash 외 모든 키 초기화 → 파이프라인 재실행"
    on_filter_change: "차트/테이블만 재렌더링 (분석 결과 유지)"
```

---

## 6. 성능 최적화

```yaml
performance:
  caching:
    - target: "파싱 결과"
      decorator: "@st.cache_data"
      key: "file_hash"
    - target: "지표 계산 결과"
      decorator: "@st.cache_data"
      key: "file_hash + analysis_mode"
    - target: "Plotly figure"
      decorator: "@st.cache_data"
      key: "chart_id + data_hash + filter_state"

  file_guard:
    max_upload_mb: 10
    warning_threshold_mb: 5
    messages:
      over_limit: "파일이 너무 큽니다 ({size}MB). 10MB 이하만 지원합니다."
      large_warning: "파일이 커서 분석에 시간이 걸릴 수 있습니다."

  streamlit_config:
    layout: "wide"
    page_icon: "📊"
    page_title: "AlphaFolio"
```

---

## 7. 파일 구조 (구현용)

```
app/
├── main.py                        # 🏠 홈 (업로드 + 개요)
├── pages/
│   ├── 1_📊_포트폴리오.py          # 구성 분석
│   ├── 2_📈_성과분석.py            # 수익률 분석
│   ├── 3_⚠️_리스크.py             # 리스크 분석
│   ├── 4_📋_종목상세.py            # 종목 테이블
│   └── 5_📄_리포트.py             # 통합 리포트 + 내보내기
├── components/
│   ├── sidebar.py                 # 공통 사이드바
│   ├── header.py                  # 헤더 배너
│   ├── kpi_cards.py               # KPI 카드 렌더링
│   ├── insight_cards.py           # 인사이트 카드
│   ├── chart_renderer.py          # 차트 생성 + fallback
│   ├── stock_table.py             # 테이블 렌더링
│   ├── nav_cards.py               # 페이지 네비게이션 카드
│   └── page_guard.py              # 데이터 없을 때 가드
├── pipeline/
│   ├── parser.py                  # parsing_rules.md 참조
│   ├── analyzer.py                # analysis_rules.md 참조
│   ├── visualizer.py              # visualization_rules.md 참조
│   ├── insight_engine.py          # insight_rules.md 참조
│   └── reporter.py                # report_rules.md 참조
├── utils/
│   ├── formatting.py              # 숫자 포맷 (억/만, M/K)
│   ├── colors.py                  # 색상 체계
│   ├── export.py                  # PDF/CSV 내보내기
│   └── session.py                 # session_state 헬퍼
├── data/
│   ├── sample_kr.csv              # 국내 주식 샘플
│   └── sample_us.csv              # 미국 ETF 샘플
├── skills/                        # Skills.md (읽기 전용)
│   ├── parsing_rules.md
│   ├── analysis_rules.md
│   ├── visualization_rules.md
│   ├── insight_rules.md
│   └── report_rules.md
└── .streamlit/
    └── config.toml                # Streamlit 설정
```

---

## 8. 페이지 흐름도

```
[첫 방문] → 🏠 홈 (랜딩)
                │
    ┌───────────┼───────────┐
    │           │           │
  샘플 클릭   파일 업로드   파싱 실패
    │           │           │
    └─────┬─────┘       [에러 화면]
          │              안내 + 재시도
    [파이프라인 실행]
    [분석 모드 판별]
          │
    🏠 홈 (개요 대시보드)
    ┌─────┼──────┬──────┬──────┐
    │     │      │      │      │
    ▼     ▼      ▼      ▼      ▼
   📊    📈     ⚠️     📋     📄
 포트폴리오 성과분석  리스크  종목상세  리포트
 구성·배분 수익률   MDD    테이블  통합뷰
          비교   변동성   검색   PDF
                 베타          CSV

  ◄──── 사이드바 필터 공유 ────►
  ◄──── session_state 공유 ────►
```

---

## 9. 평가 기준 대응

| 평가 항목 (배점) | 멀티페이지 대응 전략 |
|-----------------|---------------------|
| 대시보드 자동 생성 — 자동 분석 (25점 中) | 홈에서 업로드 즉시 파이프라인 실행, 5개 페이지 자동 구성 |
| 대시보드 자동 생성 — 시각화 적절성 | 페이지별 목적에 맞는 차트 배치, fallback chain |
| 대시보드 자동 생성 — UI 완성도 | 페이지 분리로 깔끔한 구조, 네비게이션, 반응형 |
| 범용성 (25점) | 분석 모드별 페이지 활성화/비활성화 자동 처리 |
| 바이브코딩 활용 (15점) | 페이지-파이프라인-Skills.md 3층 대응 구조 |
| 실용성 및 창의성 (10점) | 리포트 페이지에서 원클릭 PDF 내보내기, 페이지 간 일관된 필터 |
