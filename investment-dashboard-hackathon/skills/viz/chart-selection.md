# Chart Selection Rules — 차트 자동 선택 규칙

> Auto-select the best chart type based on data characteristics.
> Evaluate conditions top-to-bottom; first match wins.
> Each chart has a `min_data` gate — if unmet, fall through to fallback.
>
> **Reference**: report files use `{id}.{type}` to reference charts.
> e.g. `portfolio_composition.donut`, `return_analysis.horizontal_bar`, `time_series.line`, `distribution.histogram`

---

## 1. Chart Auto-Selection Rules

### 1.1 Portfolio Composition

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

### 1.2 Return Analysis

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

### 1.3 Time Series (requires trade history)

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

### 1.4 Risk / Return

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

### 1.5 Distribution

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

### 1.6 Market / Currency Composition

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

### 1.7 Stock Detail Table

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

## 2. Fallback Chain

> Sequential fallback when a chart cannot render.

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
