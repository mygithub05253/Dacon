# Data Normalization Rules
### Number, Date, Market Detection & Currency Conversion

> **Pipeline**: Stage 1 | **Depends on**: none | **Used by**: analysis/*
>
> Transforms raw cell values into clean, standardized types
> after columns have been mapped to the standard schema.

---

## 1. Number Normalization

```yaml
number_normalization:
  description: "숫자 정규화 — 통화 기호, 콤마, 회계 표기 등 제거"
  remove_chars: [",", " ", "원", "$", "₩", "\\", "¥", "€", "주"]
  negative_formats:
    - "-1000"         # standard
    - "(1000)"        # accounting notation
    - "▼1000"         # 일부 증권사 (하락)
    - "△1000"         # 상승 기호 (양수로 처리)
    - "▲1000"         # 상승 기호
    - "-1,000.00"     # comma + decimal mixed

  percentage:
    description: "퍼센트 값 처리 규칙"
    detection: "값에 '%' 포함 여부"
    rules:
      - "5.2%"   → 5.2    # remove % sign, keep value
      - "0.052"  → 5.2    # decimal form → multiply by 100 (only when value < 1)
      - "-0.03"  → -3.0   # negative decimal same rule
      - "5.2"    → 5.2    # already percentage, keep as-is
    ambiguity:
      description: "0.052가 5.2%인지 0.052%인지 모호한 경우"
      rule: "같은 컬럼의 다른 값 범위 참조. 대부분 -100~100이면 그대로, 대부분 -1~1이면 ×100"

  exception:
    non_numeric:
      description: "숫자 컬럼에 텍스트 값 (예: 'N/A', '-', '해당없음')"
      action: "NaN으로 변환"
      log: true
    overflow:
      description: "비정상적으로 큰 숫자 (예: 999999999999)"
      threshold: "같은 컬럼 중앙값의 1000배 초과"
      action: "경고 표시 후 유지 (데이터 오류 가능성)"
    all_zero:
      description: "한 컬럼 전체가 0인 경우"
      action: "해당 컬럼 매핑 재검토 (잘못된 컬럼일 가능성)"
```

---

## 2. Date Normalization

```yaml
date_normalization:
  description: "다양한 날짜 형식을 표준 형식으로 변환"
  input_formats:
    - "YYYY-MM-DD"         # 2024-01-15 (ISO standard)
    - "YYYY/MM/DD"         # 2024/01/15
    - "YYYYMMDD"           # 20240115 (증권사 다수)
    - "YYYY.MM.DD"         # 2024.01.15
    - "MM/DD/YYYY"         # 01/15/2024 (US format)
    - "DD-MM-YYYY"         # 15-01-2024 (European format)
    - "DD/MM/YYYY"         # 15/01/2024
    - "YYYY년 MM월 DD일"   # Korean format
    - "YYYY-MM-DD HH:MM:SS"  # with timestamp
    - "MM-DD-YYYY"         # 01-15-2024
  output_format: "YYYY-MM-DD"

  ambiguity:
    description: "01/02/2024가 1월 2일인지 2월 1일인지 모호한 경우"
    strategy:
      - "파일 내 다른 날짜값에서 패턴 추론 (13 이상 값이 첫 자리면 DD-MM)"
      - "증권사 locale 기반 추론 (국내 → YYYY-MM-DD 계열 우선)"
      - "추론 불가 시 MM/DD (미국식) 기본 적용 후 경고"

  exception:
    invalid_date:
      description: "유효하지 않은 날짜"
      examples: ["2024-13-01", "2024-02-30", "0000-00-00"]
      action: "NaN 처리 후 경고. 해당 행 제외하지 않음."
    future_date:
      description: "오늘 이후 날짜"
      action: "경고 표시 (예약 거래이거나 데이터 오류)"
      message: "미래 날짜({date})가 포함되어 있습니다. 확인해주세요."
    very_old_date:
      description: "10년 이상 이전 날짜"
      threshold: "current_year - 10"
      action: "경고만 표시 (장기 보유일 수 있음)"
    mixed_formats:
      description: "한 컬럼에 여러 날짜 형식 혼용"
      action: "각 셀별로 개별 파싱 시도. 파싱 실패 셀은 NaN."
```

---

## 3. Market Auto-Detection

```yaml
market_detection:
  description: "ticker 패턴 기반 시장 자동 판별"

  KR:
    description: "한국 시장 종목 패턴"
    patterns:
      - regex: "^[0-9]{6}$"                 # 6자리 숫자 (KOSPI/KOSDAQ)
        examples: ["005930", "035420", "373220"]
      - regex: "^A[0-9]{6}$"                # A + 6자리 (일부 시스템)
        action: "A 제거 후 6자리로 처리"
      - regex: "^KR[0-9]{10}$"              # ISIN 코드 (한국)
        action: "5~10번째 자리 추출하여 종목코드로 변환"
    currency: KRW
    price_unit_check:
      description: "국내 주식인데 가격이 소수점 → 원 단위가 아닐 수 있음"
      rule: "가격 < 100 이면 경고 (단, ETF는 예외)"

  US:
    description: "미국 시장 종목 패턴"
    patterns:
      - regex: "^[A-Z]{1,5}$"               # standard ticker
        examples: ["AAPL", "TSLA", "VOO", "BRK.B"]
      - regex: "^[A-Z]{1,5}\\.[A-Z]$"       # class shares (BRK.B)
        action: "점(.) 포함하여 그대로 사용"
      - regex: "^US[0-9]{10}$"              # ISIN 코드 (미국)
        action: "별도 매핑 테이블로 ticker 변환"
    currency: USD
    price_unit_check:
      description: "미국 주식인데 가격이 10만 이상 → KRW으로 입력했을 수 있음"
      rule: "가격 > 100000 이면 통화 오류 경고"

  OTHER:
    description: "KR/US 패턴 모두 불일치"
    patterns:
      - regex: "^[A-Z0-9]{2,12}$"           # 기타 해외 주식/ETF
    action: "OTHER 마켓으로 분류, 통화는 사용자 입력 요청"
    message: "'{ticker}' 종목의 시장과 통화를 선택해주세요."

  exception:
    mixed_markets:
      description: "한 파일에 국내+해외 종목 혼합"
      action: "정상 처리. 각 종목별로 개별 마켓 판별."
      currency_note: "총 평가금액은 KRW 기준으로 환산하여 합산"
    ambiguous_ticker:
      description: "패턴이 KR과 US 모두 매칭 가능 (예: 5자리 숫자+문자)"
      action: "name 컬럼의 한글/영문 여부로 보조 판별"
      fallback: "사용자 확인 요청"
```

---

## 4. Currency Conversion

```yaml
currency_conversion:
  description: "혼합 통화 포트폴리오의 총합 계산 시 적용"
  base_currency: "KRW"  # 기본 표시 통화
  toggle: true            # 사용자가 USD/KRW 전환 가능

  exchange_rates:
    source: "고정 환율 (더미 데이터 사용 시) 또는 외부 API"
    fixed_rates:           # 외부 API 불가 시 fallback
      USD_KRW: 1350
      JPY_KRW: 9.0
      EUR_KRW: 1470
      CNY_KRW: 185
    api_source: "exchangerate-api.com (무료 티어)"
    cache_duration: "24시간"
    api_failure:
      action: "고정 환율 사용 후 경고"
      message: "실시간 환율을 가져올 수 없어 기본 환율(1 USD = ₩1,350)을 적용했습니다."

  calculation:
    description: "환산 계산 방식"
    per_stock: "원래 통화로 계산 후 표시"
    portfolio_total: "base_currency로 환산하여 합산"
    display: "원화 환산액과 원래 통화 병기"
```
