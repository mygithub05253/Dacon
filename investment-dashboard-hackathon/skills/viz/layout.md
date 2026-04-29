# Layout & Interaction Rules — 레이아웃 및 인터랙션 규칙

> Dashboard layout (KPI cards, chart placement, responsive), interactions (drill-down, filters, period selector), and extension points.

---

## 1. Dashboard Layout

### 1.1 KPI Card Area (top)

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

### 1.2 Chart Placement Rules

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

## 2. Interactions

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

## 3. Extension Points

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
