# Visualization Rules — 시각화 규칙

> AlphaFolio 시각화 계층의 규칙을 정의합니다.
> 이 문서를 참조하여 데이터 특성에 맞는 차트를 자동 선택하고 렌더링합니다.
>
> **파이프라인 위치**: Stage 3-A (analysis_rules.md 출력을 소비, insight_rules.md와 병렬)
> **입력**: analysis_rules.md에서 계산된 지표 (return_rate, weight, volatility 등)
> **출력**: 차트 객체 (Plotly Figure) + 레이아웃 메타데이터
> **하위 의존**: report_rules.md가 이 문서의 chart id (`portfolio_composition`, `return_analysis` 등)를 참조합니다.

---

## 1. 차트 자동 선택 규칙

> 조건을 위에서 아래로 순서대로 평가하여 첫 번째 매칭되는 차트를 선택합니다.
> 각 차트에는 `min_data` 조건이 있으며, 충족하지 못하면 다음 후보(fallback)로 넘어갑니다.
>
> **참조 규칙**: report_rules.md에서 차트를 참조할 때 `{id}.{type}` 형태를 사용합니다.
> 예: `portfolio_composition.donut`, `return_analysis.horizontal_bar`, `time_series.line`, `distribution.histogram`

### 1.1 포트폴리오 구성 시각화
```yaml
chart_rules:
  - id: "portfolio_composition"
    name: "포트폴리오 구성"
    condition: "weight 데이터 존재"
    priority: 1
    charts:
      - type: "donut"
        title: "종목별 비중"
        data: "weight per stock"
        min_data: { stocks: 2 }
        config:
          hole: 0.45
          sort: "descending"
          top_n: 10
          others_threshold: 2.0   # 2% 미만은 '기타'로 합산
          show_percentage: true
          center_text: "총 평가금액"
        exception:
          single_stock:
            action: "도넛 대신 단일 KPI 카드로 대체"
            display: "100% — {stock_name}"
          all_equal_weight:
            action: "도넛 렌더링하되 '균등 배분' 레이블 추가"
          too_many_stocks:
            condition: "종목 > 30"
            action: "상위 10개 + '기타(N종목)' 그룹"
            others_label: "기타 ({count}종목, {weight}%)"

      - type: "treemap"
        title: "마켓/섹터별 비중"
        data: "hierarchical: market > sector > stock"
        min_data: { stocks: 3, sectors: 2 }
        config:
          color_by: "return_rate"
          color_scale: "RdYlGn"
          show_values: true
          min_box_size: 1.5     # 1.5% 미만은 보이지 않을 수 있으므로
        fallback:
          condition: "섹터 1개뿐이거나 종목 < 3"
          replace_with: "horizontal_bar (비중 기준)"
        exception:
          no_sector_data:
            action: "market > stock 2단계 트리맵으로 단순화"
          single_market:
            action: "sector > stock 2단계 트리맵으로 변경"
          all_returns_na:
            action: "color_by를 weight로 변경 (수익률 색상 불가)"
```

### 1.2 수익률 분석 시각화
```yaml
  - id: "return_analysis"
    name: "수익률 분석"
    condition: "return_rate 데이터 존재 (1개 이상 종목에서 계산 가능)"
    priority: 2
    charts:
      - type: "horizontal_bar"
        title: "종목별 수익률"
        data: "return_rate per stock"
        min_data: { stocks: 1 }
        config:
          sort: "descending"
          color_rule:
            positive: "#639922"
            negative: "#E24B4A"
            na: "#B4B2A9"
          show_value_labels: true
          max_bars: 20      # 20개 초과 시 상위/하위 10개씩
          zero_line: true    # 0% 기준선 표시
        exception:
          too_many_stocks:
            condition: "종목 > 20"
            action: "상위 10 + 하위 10만 표시, 중간 생략"
            toggle: "전체 보기 버튼 제공"
          all_positive:
            action: "정상 렌더링. 색상 전부 green 계열."
          all_negative:
            action: "정상 렌더링. 색상 전부 red 계열."
          all_zero:
            action: "차트 렌더링하되 '수익률 변동 없음' 메시지 추가"
          extreme_outlier:
            condition: "한 종목 수익률이 나머지 평균의 10배 이상"
            action: "축 범위 자동 조정. 이상치 종목에 '📌' 표시."
          some_na:
            action: "N/A 종목은 회색 바로 표시, 하단에 '수익률 미산출' 범례"

      - type: "bar_grouped"
        title: "섹터별 수익률"
        data: "sector_return per sector"
        min_data: { sectors: 2 }
        config:
          sort: "descending"
          show_benchmark: true
        fallback:
          condition: "섹터 1개뿐"
          replace_with: "생략 (섹터 차트 비활성화)"
```

### 1.3 시계열 시각화 (거래 내역 존재 시)
```yaml
  - id: "time_series"
    name: "기간별 추이"
    condition: "date 컬럼 존재 + 2개 이상 시점 + 시계열 데이터 구성 가능"
    priority: 3
    charts:
      - type: "line"
        title: "포트폴리오 수익률 추이"
        data: "cumulative_return over time"
        min_data: { data_points: 5 }
        config:
          show_benchmark: true
          benchmark_label: "KOSPI / S&P 500"
          period_selector: [1W, 1M, 3M, 6M, 1Y, ALL]
          fill_area: true
          fill_opacity: 0.1
          show_tooltip: true
          x_axis_format: "auto"  # 기간에 따라 자동 (일별/주별/월별)
        exception:
          very_short_period:
            condition: "전체 기간 < 7일"
            action: "period_selector 비활성화, ALL만 표시"
          gaps_in_data:
            condition: "날짜가 비연속적 (주말/휴일 제외하고도 빈 날짜 존재)"
            action: "빈 날짜를 이전 값으로 forward fill"
            warning: "일부 거래일 데이터가 누락되어 보간했습니다."
          benchmark_missing:
            action: "포트폴리오 라인만 표시. 벤치마크 범례 제거."

      - type: "line"
        title: "MDD 추이"
        data: "drawdown over time"
        min_data: { data_points: 10 }
        config:
          color: "#E24B4A"
          fill_area: true
          fill_opacity: 0.2
          highlight_max_dd: true   # 최대 낙폭 구간 강조
          y_axis_max: 0            # 항상 0 이하 표시
        fallback:
          condition: "데이터 포인트 < 10"
          replace_with: "MDD 값만 KPI 카드로 표시"
```

### 1.4 리스크/수익 시각화
```yaml
  - id: "risk_return"
    name: "리스크 대비 수익"
    condition: "volatility + return_rate 모두 계산 가능 + 종목 3개 이상"
    priority: 4
    charts:
      - type: "scatter"
        title: "리스크 vs 수익률"
        data: "x=volatility, y=return_rate, size=weight"
        min_data: { stocks: 3 }
        config:
          x_label: "변동성 (%)"
          y_label: "수익률 (%)"
          color_by: "market"
          bubble_size_range: [8, 40]  # 최소/최대 버블 크기
          show_quadrant_labels: true
          quadrant_origin: "median"   # 중앙값 기준 4분면
          quadrants:
            top_left: { label: "저위험 고수익", color: "#E1F5EE" }
            top_right: { label: "고위험 고수익", color: "#FAEEDA" }
            bottom_left: { label: "저위험 저수익", color: "#F1EFE8" }
            bottom_right: { label: "고위험 저수익", color: "#FCEBEB" }
          show_annotations: true      # 각 점에 종목명 레이블
        exception:
          two_stocks:
            action: "산점도 대신 비교 테이블로 대체"
          all_same_volatility:
            action: "x축 범위 강제 확장. 경고: '변동성 차이 미미'"
          extreme_outlier:
            condition: "한 종목 변동성이 다른 종목의 5배 이상"
            action: "로그 스케일 전환 옵션 제공"
        fallback:
          condition: "종목 < 3 또는 변동성 데이터 없음"
          replace_with: "수익률 막대 차트 (기존 return_analysis 재활용)"
```

### 1.5 분포 시각화
```yaml
  - id: "distribution"
    name: "수익률 분포"
    condition: "종목 수 >= 5"
    priority: 5
    charts:
      - type: "histogram"
        title: "수익률 분포"
        data: "return_rate distribution"
        min_data: { stocks: 5 }
        config:
          bins: "auto"       # Sturges' rule: ceil(1 + 3.322 * log10(n))
          color: "#378ADD"
          show_kde: true      # 커널 밀도 추정 곡선
          vertical_lines:
            - value: 0
              label: "손익분기"
              style: "dashed"
              color: "#888780"
            - value: "mean"
              label: "평균"
              style: "solid"
              color: "#D85A30"
            - value: "median"
              label: "중앙값"
              style: "dotted"
              color: "#7F77DD"
        exception:
          bimodal_distribution:
            detection: "2개 이상 봉우리 감지"
            action: "정상 렌더링 + '수익률이 양극화되어 있습니다' 인사이트 트리거"
          extreme_skew:
            condition: "skewness > 2 또는 < -2"
            action: "로그 스케일 옵션 제공"
        fallback:
          condition: "종목 < 5"
          replace_with: "구간별 막대 차트 (return_distribution buckets)"
```

### 1.6 통화/마켓 비중 시각화
```yaml
  - id: "market_composition"
    name: "시장별 구성"
    condition: "2개 이상 마켓 존재"
    priority: 6
    charts:
      - type: "stacked_bar"
        title: "시장별/통화별 비중"
        data: "market_weight, currency_exposure"
        min_data: { markets: 2 }
        config:
          orientation: "horizontal"
          show_values: true
          color_map:
            KR: "#378ADD"
            US: "#D85A30"
            OTHER: "#888780"
    fallback:
      condition: "단일 시장"
      replace_with: "생략"
```

### 1.7 종목 상세 테이블
```yaml
  - id: "stock_detail_table"
    name: "종목 상세"
    condition: "항상 (파싱 성공 시)"
    priority: 7
    charts:
      - type: "data_table"
        title: "종목 상세 현황"
        data: "전체 종목 데이터 (표준 스키마 + 계산 지표)"
        min_data: { stocks: 1 }
        config:
          default_sort: "valuation DESC"
          sortable_columns: ["valuation", "return_rate", "weight", "name"]
          highlight:
            top3_return: { style: "left-border green 3px" }
            bottom3_return: { style: "left-border red 3px" }
          stripe: true
          pagination:
            enabled: true
            threshold: 20
            page_size: 20
          search:
            enabled: true
            min_stocks: 10
            fields: ["name", "ticker"]
        exception:
          too_many_stocks:
            condition: "종목 > 50"
            action: "기본 20개 표시 + 페이지네이션"
          zero_weight_stocks:
            action: "테이블 하단에 별도 그룹으로 분리 표시"
```

---

## 2. 차트 fallback 체인

> 차트가 렌더링 불가능할 때의 대체 순서

```yaml
fallback_chain:
  description: "차트 렌더링 실패 시 순차 대체"
  
  level_1_degrade:
    description: "같은 유형의 단순 버전"
    examples:
      - "트리맵 → 수평 막대 차트"
      - "산점도 → 비교 테이블"
      - "히스토그램 → 구간별 막대"
  
  level_2_replace:
    description: "다른 유형의 차트로 교체"
    examples:
      - "라인 차트 (데이터 부족) → KPI 카드"
      - "도넛 (종목 1개) → 단일 수치 카드"
  
  level_3_text:
    description: "차트 대신 텍스트로 표시"
    format: "안내 메시지 카드 (아이콘 + 설명)"
    template: "📊 {chart_name}을(를) 표시하기에 데이터가 부족합니다. {suggestion}"
  
  level_4_hide:
    description: "해당 영역 완전 숨김"
    condition: "관련 데이터가 전혀 없을 때"
    note: "빈 영역이 남지 않도록 레이아웃 자동 조정"

  render_error:
    description: "차트 라이브러리 렌더링 자체 실패 (Plotly 에러 등)"
    action: "에러 로그 기록 + level_3_text fallback"
    message: "차트를 표시하는 중 오류가 발생했습니다."
    retry: false  # 같은 데이터로 재시도하지 않음
```

---

## 3. 대시보드 레이아웃 규칙

### 3.1 KPI 카드 영역 (최상단)
```yaml
kpi_cards:
  layout: "horizontal_row"
  responsive:
    desktop: "4열"
    tablet: "2열 × 2행"
    mobile: "1열 × 4행"
  items:
    - metric: "total_valuation"
      label: "총 평가금액"
      format: "currency"
      icon: "💰"
      always_show: true
    - metric: "total_return"
      label: "총 수익률"
      format: "percentage_with_sign"
      color_rule: "positive=green, negative=red"
      icon: "📈"
      always_show: true
    - metric: "total_profit_loss"
      label: "총 손익"
      format: "currency_with_sign"
      color_rule: "positive=green, negative=red"
      icon: "💵"
      always_show: true
    - metric: "stock_count"
      label: "보유 종목"
      format: "integer"
      suffix: "개"
      icon: "📊"
      always_show: true
  
  conditional_cards:
    - metric: "sharpe_ratio"
      label: "샤프 비율"
      condition: "고급 지표 계산 가능 시"
      format: "float_2dp"
      replace: "stock_count"  # 기존 4번째 카드를 대체
    - metric: "win_rate"
      label: "승률"
      condition: "거래 내역 존재 시"
      format: "percentage"
  
  exception:
    all_na:
      action: "값 대신 '—' 표시"
    mixed_currency:
      action: "총 평가금액에 '(환산)' 라벨 추가"
```

### 3.2 차트 배치 규칙
```yaml
chart_layout:
  strategy: "adaptive_grid"
  rules:
    - section: "KPI 카드"
      position: "최상단"
      always_show: true
    - section: "포트폴리오 구성"
      position: "KPI 바로 아래"
      layout: "2열 (도넛 + 트리맵)"
      condition: "종목 2개 이상"
      single_stock_fallback: "단일 종목 상세 카드"
    - section: "수익률 분석"
      position: "그 아래"
      layout: "1열 전체 폭"
      condition: "return_rate 1개 이상 계산 가능"
    - section: "시계열"
      position: "그 아래"
      layout: "1열 전체 폭"
      condition: "거래 내역 존재 + 시점 5개 이상"
    - section: "리스크/수익"
      position: "그 아래"
      layout: "1열 전체 폭"
      condition: "고급 지표 존재 + 종목 3개 이상"
    - section: "수익률 분포"
      position: "그 아래"
      layout: "1열"
      condition: "종목 5개 이상"
    - section: "종목 상세 테이블"
      position: "최하단"
      always_show: true

  empty_state:
    description: "차트가 하나도 표시되지 못하는 극단적 경우"
    action: "종목 상세 테이블만 표시 + 데이터 보강 안내"
    message: "더 다양한 분석을 보려면 거래 내역 CSV도 업로드해보세요."

  section_separator:
    type: "subtle divider (st.divider)"
    spacing: "24px"
```

---

## 4. 색상 규칙

### 4.1 수익/손실 색상
```yaml
color_palette:
  profit_loss:
    strong_positive: "#27680A"
    positive: "#639922"
    slight_positive: "#97C459"
    neutral: "#888780"
    slight_negative: "#F09595"
    negative: "#E24B4A"
    strong_negative: "#A32D2D"

  profit_loss_background:
    positive_bg: "rgba(99, 153, 34, 0.08)"
    negative_bg: "rgba(226, 75, 74, 0.08)"
    neutral_bg: "rgba(136, 135, 128, 0.05)"

  sequential:
    primary: ["#E6F1FB", "#85B7EB", "#378ADD", "#185FA5", "#042C53"]

  categorical:
    markets:
      KR: "#378ADD"
      US: "#D85A30"
      OTHER: "#888780"
    sectors:
      - "#7F77DD"
      - "#1D9E75"
      - "#D85A30"
      - "#378ADD"
      - "#BA7517"
      - "#D4537E"
      - "#639922"
      - "#888780"
      - "#E24B4A"
      - "#534AB7"
    sector_overflow:
      description: "섹터 > 10개일 때"
      action: "11번째부터 gray 계열 변형 (#6E6D68, #5A5955, ...)"
```

### 4.2 차트 공통 스타일
```yaml
chart_style:
  font_family: "Pretendard, -apple-system, sans-serif"
  background: "transparent"
  grid:
    show: true
    color: "rgba(0,0,0,0.06)"
    style: "dashed"
  hover:
    enabled: true
    mode: "closest"
    show_all_values: true
  legend:
    position: "top"
    orientation: "horizontal"
    max_items: 8
    overflow: "'외 N개' 접기"
  animation:
    enabled: true
    duration: 500
    easing: "cubic-in-out"
  responsive:
    auto_resize: true
    min_height: 300
    max_height: 600
  accessibility:
    alt_text: true       # 차트별 대체 텍스트 자동 생성
    high_contrast: false  # 고대비 모드 옵션
    pattern_fills: false  # 색각 이상 대응 패턴 (향후 구현)
```

---

## 5. 인터랙션 규칙

```yaml
interactions:
  drill_down:
    - trigger: "섹터 차트 클릭 (도넛/트리맵/막대)"
      action: "해당 섹터의 종목 목록 필터링 표시"
      ui: "클릭한 섹터 이름을 상단에 표시 + '전체 보기' 리셋 버튼"
    - trigger: "종목 클릭 (어느 차트에서든)"
      action: "종목 상세 정보 확장"
      display: "종목명, 수익률, 평가금액, 매수가, 현재가, 비중"
    - trigger: "기간 차트에서 구간 드래그"
      action: "선택 구간 확대 (zoom)"
      reset: "더블클릭으로 원래 범위 복원"

  filters:
    - name: "마켓 필터"
      type: "multiselect"
      options: "동적 생성 (데이터에 존재하는 마켓만)"
      default: "전체"
      position: "사이드바 또는 차트 상단"
    - name: "수익/손실 필터"
      type: "radio"
      options: ["전체", "수익 종목만", "손실 종목만"]
      default: "전체"
    - name: "섹터 필터"
      type: "multiselect"
      options: "동적 생성 (데이터 기반)"
      default: "전체"
    - name: "정렬 기준"
      type: "selectbox"
      options: ["비중순", "수익률순", "평가금액순", "종목명순"]
      default: "비중순"

  filter_exception:
    no_results:
      description: "필터 적용 후 표시할 종목이 0개"
      action: "빈 차트 대신 안내 메시지 표시"
      message: "선택한 조건에 해당하는 종목이 없습니다."
      ui: "필터 리셋 버튼 제공"
    single_result:
      description: "필터 후 1개 종목만 남음"
      action: "차트 단순화 (도넛 → 카드, 분포 → 비활성화)"

  period_selector:
    condition: "시계열 데이터 존재 시"
    type: "radio"
    options: ["1주", "1개월", "3개월", "6개월", "1년", "전체"]
    default: "전체"
    exception:
      period_shorter_than_option:
        description: "데이터 기간이 선택 옵션보다 짧음"
        action: "해당 옵션 비활성화 (클릭 불가 + 툴팁: '데이터 부족')"
      no_data_in_period:
        action: "'해당 기간 데이터 없음' 메시지"
```

---

## 6. 숫자 표시 규칙

```yaml
number_display:
  currency:
    KRW:
      format: "₩{value:,}"
      precision: 0
      large_number:
        억: { threshold: 100000000, format: "₩{value}억" }
        만: { threshold: 10000, format: "₩{value}만" }
    USD:
      format: "${value:,.2f}"
      precision: 2
      large_number:
        M: { threshold: 1000000, format: "${value}M" }
        K: { threshold: 1000, format: "${value}K" }

  percentage:
    format: "{sign}{value}%"
    precision: 2
    sign_rule: "항상 부호 표시 (+ 또는 -)"
    zero_display: "0.00%"

  ratio:
    format: "{value}"
    precision: 2

  tooltip_vs_label:
    description: "차트 위 레이블은 간결하게, 툴팁은 상세하게"
    label: "₩1.2억"  or "+12.5%"
    tooltip: "₩123,456,789 (전체 비중 23.4%)"
```

---

## 7. 확장 규칙

```yaml
extension_points:
  new_chart_types:
    crypto:
      - type: "heatmap"
        title: "코인 간 상관관계"
        data: "correlation_matrix"
        min_data: { coins: 3, data_points: 30 }
      - type: "candlestick"
        title: "가격 캔들차트"
        data: "OHLCV"
        condition: "OHLCV 데이터 존재 시"
    bond:
      - type: "yield_curve"
        title: "수익률 곡선"
        data: "ytm by maturity"
      - type: "timeline"
        title: "만기 일정"
        data: "maturity dates"
    fund:
      - type: "rolling_return"
        title: "롤링 수익률"
        data: "N일 이동 수익률"
        window_options: [30, 60, 90, 180, 252]

  theme_customization:
    description: "향후 테마 커스터마이징 지원"
    options:
      - "다크 모드"
      - "색각 이상 대응 모드"
      - "프린트 모드 (흑백 최적화)"
    default: "라이트 모드"

  chart_export:
    formats: ["PNG", "SVG"]
    resolution: "2x (Retina)"
    method: "plotly.io.to_image() 또는 kaleido"
    exception:
      export_fail:
        action: "스크린샷 가이드 제공"
        message: "이미지 내보내기에 실패했습니다. 브라우저의 캡처 기능을 이용해주세요."
```
