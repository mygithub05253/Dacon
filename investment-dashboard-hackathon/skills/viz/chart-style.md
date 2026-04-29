# Chart Style Rules — 차트 스타일 규칙

> Color palette, common chart style, and number display rules.

---

## 1. Color Palette

### 1.1 Profit / Loss Colors

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
```

### 1.2 Sequential Palette

```yaml
  sequential:
    primary: ["#E6F1FB", "#85B7EB", "#378ADD", "#185FA5", "#042C53"]
```

### 1.3 Categorical Palette

```yaml
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

---

## 2. Chart Common Style

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

## 3. Number Display Rules

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
