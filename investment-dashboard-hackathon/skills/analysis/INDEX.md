# Analysis Skills — Navigation Index

> **Pipeline Position**: Stage 2
> **Input**: Standardized DataFrame from `parsing/*`
> **Output**: Metrics-enriched DataFrame + advanced metrics dict
> **Downstream**: `visualization_rules.md` and `insight_rules.md` reference metric names from these files.
> **Recommended model**: `opus` — 수학적 공식, 리스크 지표, 크로스레퍼런스가 많아 고정밀 추론 필요

---

## Files

| File | Description |
|------|-------------|
| [basic-metrics.md](basic-metrics.md) | Per-stock indicators: return rate, valuation, profit/loss, portfolio weight. Includes formula, unit, display format, precision, color rules, and exception handling for each metric. |
| [portfolio-metrics.md](portfolio-metrics.md) | Portfolio-level aggregates: total return, total valuation/investment/P&L, stock counts, sector analysis, market analysis, and return distribution buckets with color-coded ranges. |
| [advanced-metrics.md](advanced-metrics.md) | Time-series risk indicators (requires trade history): daily returns generation, data sufficiency checks, Sharpe ratio, MDD, volatility, beta, Sortino, Calmar, win rate, avg win/loss ratio. |
| [benchmarks.md](benchmarks.md) | Benchmark index definitions (KR/US), comparison metrics (excess return, tracking error, information ratio), analysis mode auto-switching rules, and extension points for new asset classes. |

---

## Original Source

All content is derived from `../analysis_rules.md` (preserved, not deleted).

- Sections 1 --> `basic-metrics.md`
- Sections 2 + 3 --> `portfolio-metrics.md`
- Sections 4.0 - 4.6 --> `advanced-metrics.md`
- Sections 5 + 6 + 7 --> `benchmarks.md`
