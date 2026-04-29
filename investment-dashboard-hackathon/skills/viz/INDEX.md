# Visualization Skill — INDEX

> **Pipeline**: Stage 3-A
> **Input**: Metrics from `analysis/*` (return_rate, weight, volatility, etc.)
> **Output**: Plotly chart objects + layout metadata
> **Downstream**: `report/*` references chart IDs (`portfolio_composition`, `return_analysis`, etc.)

---

## Sub-files

| File | Description |
|------|-------------|
| [chart-selection.md](chart-selection.md) | 차트 자동 선택 규칙 — 데이터 특성별 차트 타입 매칭 + fallback chain |
| [chart-style.md](chart-style.md) | 색상 팔레트, 차트 공통 스타일, 숫자 표시 규칙 |
| [layout.md](layout.md) | 대시보드 레이아웃, KPI 카드, 인터랙션, 확장 포인트 |

## Processing Order

1. **Select** chart type per data slot (chart-selection.md)
2. **Apply** color palette + common style (chart-style.md)
3. **Place** charts in dashboard grid (layout.md)
