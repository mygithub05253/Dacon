# Analysis Skills — Navigation Index

> **Pipeline Position**: Stage 2
> **Input**: Standardized DataFrame from `parsing/*`
> **Output**: Metrics-enriched DataFrame + advanced metrics dict
> **Downstream**: `viz/*` and `insight/*` reference metric names from these files.
> **Recommended model**: `opus` — 수학적 공식, 리스크 지표, 크로스레퍼런스가 많아 고정밀 추론 필요

---

## Files

| File | Description |
|------|-------------|
| [basic-metrics.md](basic-metrics.md) | Per-stock indicators: return rate, valuation, profit/loss, portfolio weight. Includes formula, unit, display format, precision, color rules, and exception handling for each metric. |
| [portfolio-metrics.md](portfolio-metrics.md) | Portfolio-level aggregates: total return, total valuation/investment/P&L, stock counts, sector analysis, market analysis, and return distribution buckets with color-coded ranges. |
| [advanced-metrics.md](advanced-metrics.md) | Time-series risk indicators (requires trade history): daily returns generation, data sufficiency checks, Sharpe ratio, MDD, volatility, beta, Sortino, Calmar, win rate, avg win/loss ratio. |
| [benchmarks.md](benchmarks.md) | Benchmark index definitions (KR/US), comparison metrics (excess return, tracking error, information ratio), analysis mode auto-switching rules, and extension points for new asset classes. |
| [currency-rules.md](currency-rules.md) | Exchange rate source priority (API/daily/static), conversion timing, supported pairs, precision, staleness warnings. Cross-cutting supplement. |
| [tax-fee-impact.md](tax-fee-impact.md) | Tax categories (domestic/foreign/account type), fee categories, after-tax return estimation, display rules, regulatory notes. |
| [benchmark-selection.md](benchmark-selection.md) | Auto-benchmark selection based on portfolio composition: single-market, blended multi-market, available indices, display normalization. |
| [correlation-diversification.md](correlation-diversification.md) | Correlation matrix (Pearson), diversification ratio, cluster analysis. Requires 60+ trading days. |
| [income-analysis.md](income-analysis.md) | Dividend yield, return decomposition (price vs income), dividend calendar, tax impact on dividends. |
| [turnover-holding.md](turnover-holding.md) | Turnover ratio, FIFO holding period, churning detection, fee impact estimation. Requires trade history. |

---

> Original monolithic `analysis_rules.md` has been removed. All content is now in the files above.
