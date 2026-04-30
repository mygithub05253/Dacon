# Currency Rules — Exchange Rate Handling

> Defines unified exchange rate handling rules to prevent calculation errors
> from inconsistent currency conversion across all analysis stages.
> Source: `analysis_rules.md` Stage 2 supplement

---

## Pipeline Position

```yaml
stage: 2_supplement
referenced_by:
  - "basic-metrics.md (mixed_currency exceptions)"
  - "portfolio-metrics.md (cross-currency aggregation)"
  - "advanced-metrics.md (multi-currency risk)"
  - "report/section-specs (환율 표기 규칙)"
```

---

## 1. Exchange Rate Sources

```yaml
# 환율 데이터 소스 우선순위

source_priority:
  primary:
    name: "real_time_api"
    providers: ["exchangerate-api.com", "open.er-api.com"]
    cache_duration:
      market_hours: "4h"       # 장중: 4시간 캐시
      after_hours: "24h"       # 장외: 24시간 캐시
    fallback_trigger:
      - "timeout > 3s"
      - "HTTP status >= 400"
      - "response body invalid"

  secondary:
    name: "daily_close_rate"
    providers: ["Bank of Korea (한국은행)", "FRED"]
    update_schedule: "once daily at market close"
    # 설명: 장 마감 기준 고시환율. API 실패 시 사용.

  tertiary:
    name: "static_fallback"
    description: "하드코딩된 기본 환율 (최후 수단)"
    rates:
      USD_KRW: 1350.00
      EUR_KRW: 1470.00
      JPY_KRW: 9.00
      GBP_KRW: 1710.00
      CNY_KRW: 186.00
    last_updated: "2025-01-01"
    staleness_warning:
      threshold_days: 90
      message: "환율 데이터가 {days}일 전 기준입니다. 실제 환율과 차이가 클 수 있습니다."
```

---

## 2. Conversion Timing Rules

```yaml
# 환율 적용 시점 규칙

timing_rules:
  portfolio_valuation:
    rule: "use latest available rate"
    # 설명: 포트폴리오 현재 가치 평가 시 최신 환율 사용

  historical_return:
    rule: "use rate at each calculation date"
    # 설명: 과거 수익률 계산 시 해당 날짜의 환율 적용

  transaction_level:
    rule: "use rate at trade date (if available)"
    fallback: "use nearest available rate within 3 business days"
    # 설명: 개별 거래는 거래일 환율 적용, 없으면 근접일 사용

  display_rule:
    footer_required: true
    format: "환율: {rate} ({source}, {date} 기준)"
    # 설명: 환율 출처와 기준일을 항상 하단에 표시
```

---

## 3. Supported Currency Pairs

```yaml
# 지원 통화 및 교차환율 규칙

supported_currencies:
  - { code: "KRW", symbol: "₩", name: "대한민국 원" }
  - { code: "USD", symbol: "$", name: "미국 달러" }
  - { code: "EUR", symbol: "€", name: "유로" }
  - { code: "JPY", symbol: "¥", name: "일본 엔" }
  - { code: "GBP", symbol: "£", name: "영국 파운드" }
  - { code: "CNY", symbol: "¥", name: "중국 위안" }

cross_rate_rule:
  method: "USD intermediary"
  formula: "A→B = (A→USD) * (USD→B)"
  # 설명: 직접 환율이 없는 통화쌍은 USD를 매개로 계산

unknown_currency:
  action: "flag_warning, do not convert"
  message: "'{currency}' 통화를 인식할 수 없습니다. 환산 없이 원본 금액으로 표시합니다."
```

---

## 4. Precision & Rounding

```yaml
# 정밀도 및 반올림 규칙

exchange_rate_precision: 4   # 소수점 4자리

converted_amount_precision:
  KRW: 0    # 정수 (원 단위)
  USD: 2    # 소수점 2자리 (센트)
  EUR: 2
  JPY: 0    # 정수 (엔 단위)
  GBP: 2
  CNY: 2

display_formatting:
  locale_aware: true
  thousands_separator: true
  examples:
    KRW: "₩1,350,000"
    USD: "$1,000.00"
    JPY: "¥150,000"
```

---

## 5. Edge Cases

```yaml
# 예외 상황 처리

weekend_holiday:
  condition: "환율 데이터가 없는 주말/공휴일"
  action: "use last business day rate"
  display: "직전 영업일 기준"

extreme_volatility:
  condition: "daily change > 5%"
  action: "show_warning"
  message: "환율이 전일 대비 {change}% 변동했습니다. 환산 금액에 큰 영향이 있을 수 있습니다."
  include_in_calculation: true

stale_rate:
  condition: "rate age > 7 days"
  action: "prominent_banner"
  message: "환율 데이터가 {days}일 전 기준입니다. 최신 환율을 확인해주세요."
  banner_style: "warning"
```

---

## 6. Extension Points

```yaml
# 향후 확장 고려사항

extensions:
  crypto_rates:
    description: "암호화폐 환율 (BTC, ETH 등)"
    note: "별도 소스 필요 (CoinGecko, Binance API)"
    status: "planned"

  realtime_streaming:
    description: "장중 실시간 스트리밍 환율"
    note: "WebSocket 기반, 인트라데이 포트폴리오에 필요"
    status: "future"

  user_custom_rate:
    description: "사용자 직접 환율 입력"
    note: "특정 거래에 실제 적용된 환율을 수동 입력"
    status: "planned"
```
