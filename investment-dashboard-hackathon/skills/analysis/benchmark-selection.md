# Benchmark Selection — Auto-Selection Rules

> Defines rules for automatically selecting appropriate benchmarks
> based on portfolio composition and market exposure.
> Source: `analysis_rules.md` Stage 2 supplement

---

## Pipeline Position

```yaml
stage: 2_supplement
referenced_by:
  - "benchmarks.md (index definitions, data sources)"
  - "report/section-specs (comparison charts, legend labels)"
```

---

## 1. Single-Market Portfolio

```yaml
# 단일 시장 포트폴리오 벤치마크

single_market_rules:
  kr_stocks:
    condition: "KR market weight == 100%"
    benchmark: "KOSPI"
    # 설명: 국내 주식만 보유 시 KOSPI 지수 비교

  us_stocks:
    condition: "US market weight == 100%"
    benchmark: "S&P 500"
    # 설명: 미국 주식만 보유 시 S&P 500 비교

  sector_concentrated:
    condition: "single sector weight > 70%"
    action: "use sector ETF index instead of broad market"
    examples:
      - { sector: "IT/반도체", benchmark: "KODEX 반도체 or XLK" }
      - { sector: "바이오/헬스케어", benchmark: "KODEX 바이오 or XLV" }
      - { sector: "금융", benchmark: "KODEX 은행 or XLF" }
    fallback: "broad market index if no sector match"
    # 설명: 특정 섹터 집중 시 섹터 ETF 지수가 더 적절한 비교 대상
```

---

## 2. Multi-Market Portfolio (Blended Benchmark)

```yaml
# 멀티마켓 포트폴리오 혼합 벤치마크

blended_benchmark:
  calculation:
    step_1: "calculate market weights: KR_weight, US_weight, OTHER_weight"
    step_2: "assign benchmark per market"
    step_3: "blended_return = sum(market_weight_i * market_benchmark_return_i)"
    # 설명: 시장별 비중에 따라 벤치마크 수익률을 가중 합산

  display:
    format: "혼합 벤치마크 (KOSPI {kr_pct}% + S&P 500 {us_pct}%)"
    example: "혼합 벤치마크 (KOSPI 60% + S&P 500 40%)"

  rebalance:
    frequency: "monthly"
    method: "recalculate weights to match current portfolio market exposure"
    # 설명: 포트폴리오 비중 변화에 맞춰 매월 벤치마크 비중 재조정

  threshold:
    min_weight_for_inclusion: 5    # 5% 미만 시장은 벤치마크에서 제외
    # 설명: 비중이 너무 작은 시장은 벤치마크 복잡도만 높이므로 제외
```

---

## 3. Available Benchmarks

```yaml
# 사용 가능한 벤치마크 목록

benchmarks:
  kr_equity:
    - { code: "KOSPI", name: "코스피", type: "broad_market" }
    - { code: "KOSDAQ", name: "코스닥", type: "growth_market" }
    - { code: "KRX300", name: "KRX 300", type: "broad_large_mid" }

  us_equity:
    - { code: "SPX", name: "S&P 500", type: "broad_market" }
    - { code: "IXIC", name: "NASDAQ Composite", type: "tech_heavy" }
    - { code: "DJI", name: "Dow Jones Industrial", type: "blue_chip" }

  global:
    - { code: "MXWO", name: "MSCI World", type: "developed_markets" }
    - { code: "MXEF", name: "MSCI Emerging Markets", type: "emerging_markets" }

  bond:
    - { code: "KR_GOV_3Y", name: "국고채 3년", type: "kr_government" }
    - { code: "US_10Y", name: "US Treasury 10Y", type: "us_government" }

  selection_logic:
    rule: "match portfolio's market cap profile"
    large_cap_dominant: "KOSPI or S&P 500"
    small_cap_dominant: "KOSDAQ or Russell 2000"
    # 설명: 포트폴리오의 시가총액 분포에 맞는 지수 선택
```

---

## 4. Data Requirements

```yaml
# 벤치마크 데이터 요구사항

data:
  required: "benchmark daily close prices for comparison period"

  source_priority:
    primary: "API (Yahoo Finance, KRX Open API)"
    secondary: "local cache (SQLite / CSV)"
    tertiary: "hardcoded monthly data (최후 수단)"

  minimum_overlap:
    trading_days: 30
    description: "포트폴리오와 최소 30거래일 이상 겹쳐야 비교 가능"
    # 설명: 데이터 부족 시 벤치마크 비교 비활성화

  update_frequency: "daily at market close"
  cache_expiry: "24h"
```

---

## 5. Display Rules

```yaml
# 벤치마크 표시 규칙

display:
  selection_transparency:
    rule: "항상 어떤 벤치마크가 선택되었고 왜 선택되었는지 표시"
    format: "벤치마크: {name} (포트폴리오의 {reason}에 기반하여 자동 선택)"
    example: "벤치마크: KOSPI (국내 주식 100% 포트폴리오)"

  chart_legend:
    portfolio_label: "내 포트폴리오"
    benchmark_label: "{benchmark_name}"
    period_display: true       # 비교 기간 표시

  normalization:
    method: "base_100"
    rule: "포트폴리오 시작일 기준 양쪽 모두 100으로 정규화"
    # 설명: 금액 차이 무관하게 상대 성과 비교 가능

  user_override:
    enabled: true
    description: "사용자가 자동 선택된 벤치마크를 수동 변경 가능"
    ui: "드롭다운 메뉴로 Available Benchmarks 목록 제공"
```

---

## 6. Edge Cases

```yaml
# 예외 상황 처리

edge_cases:
  short_history:
    condition: "portfolio history < 30 trading days"
    action: "skip benchmark comparison"
    message: "포트폴리오 이력이 30거래일 미만이어서 벤치마크 비교를 제공할 수 없습니다."
    alternative: "단순 수익률만 표시"

  unknown_market:
    condition: "종목의 시장 구분 불가"
    action: "exclude from market weight calculation"
    message: "'{name}' 종목의 시장을 식별할 수 없어 벤치마크 비중 계산에서 제외합니다."

  crypto_holdings:
    condition: "포트폴리오에 암호화폐 포함"
    action: "no standard benchmark available"
    display: "암호화폐 부분은 벤치마크 비교 대상에서 제외됩니다."
    # 설명: 암호화폐는 표준 벤치마크가 없어 포트폴리오 단독 표시

  all_cash:
    condition: "portfolio is 100% cash or money market"
    action: "use risk-free rate as benchmark"
    benchmark: "KR_GOV_3Y or equivalent deposit rate"

  benchmark_data_missing:
    condition: "선택된 벤치마크의 데이터를 가져올 수 없음"
    action: "try next benchmark in same category, then skip"
    message: "벤치마크 데이터를 불러올 수 없습니다. 포트폴리오 수익률만 표시합니다."
```
