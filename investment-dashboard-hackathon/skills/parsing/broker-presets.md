# Broker Presets & Error Handling
### Per-Broker CSV Profiles, Error Recovery & Extension Points

> **Pipeline**: Stage 1 | **Depends on**: none | **Used by**: analysis/*
>
> Pre-defined parsing profiles for major Korean brokers,
> comprehensive error handling rules, and extension guides
> for adding new asset types or broker presets.

---

## 1. Broker Presets

```yaml
broker_presets:
  description: "주요 증권사별 CSV 특성을 사전 정의하여 자동 감지 정확도 향상"

  kiwoom:
    broker_name: "키움증권"
    encoding: EUC-KR
    delimiter: "\t"
    skip_rows: 0
    header_pattern: "종목번호.*종목명.*보유수량"
    column_map:
      ticker: "종목번호"
      name: "종목명"
      quantity: "보유수량"
      avg_price: "매입단가"
      current_price: "현재가"
    special_rules:
      - "종목번호 앞에 'A' 접두사 제거"
      - "금액에 콤마 포함"

  mirae_asset:
    broker_name: "미래에셋증권"
    encoding: UTF-8
    delimiter: ","
    skip_rows: 2               # 상단 2행 계좌 정보
    header_pattern: "종목코드.*종목명.*잔고수량"
    column_map:
      ticker: "종목코드"
      name: "종목명"
      quantity: "잔고수량"
      avg_price: "평균매입가"
      current_price: "현재가"

  samsung:
    broker_name: "삼성증권"
    encoding: EUC-KR
    delimiter: ","
    skip_rows: 1
    header_pattern: "종목코드.*종목.*보유주수"
    column_map:
      ticker: "종목코드"
      name: "종목"
      quantity: "보유주수"
      avg_price: "매수평균가"
      current_price: "종가"

  toss:
    broker_name: "토스증권"
    encoding: UTF-8
    delimiter: ","
    skip_rows: 0
    header_pattern: "종목코드.*종목명.*보유수량"
    column_map:
      ticker: "종목코드"
      name: "종목명"
      quantity: "보유수량"
      avg_price: "평균단가"
      current_price: "현재가"
    special_rules:
      - "해외 종목은 별도 시트/파일로 분리됨"

  korea_investment:
    broker_name: "한국투자증권"
    encoding: UTF-8-SIG
    delimiter: ","
    skip_rows: 3
    header_pattern: "종목번호.*종목명.*잔고"
    column_map:
      ticker: "종목번호"
      name: "종목명"
      quantity: "잔고"
      avg_price: "매입평균가"
      current_price: "현재가"

  auto_detection:
    description: "증권사 자동 감지 전략"
    strategy: "파일 첫 10행에서 broker_presets의 header_pattern과 매칭"
    fallback: "프리셋 매칭 실패 시 범용 매핑 전략(column-mapping.md § 3) 사용"
    confidence_display: "'{broker}' 형식으로 자동 인식했습니다."
```

---

## 2. File-Level Errors

```yaml
file_errors:
  description: "파일 수준 에러 처리"

  empty_file:
    condition: "파일 크기 0 또는 헤더만 존재"
    action: "샘플 데이터 사용 제안"
    message: "파일이 비어있습니다. 샘플 데이터로 시작해보시겠습니까?"
    recovery: "샘플 데이터 3종 제공 (국내주식, 미국ETF, 혼합 포트폴리오)"

  corrupted_file:
    condition: "파일 읽기 자체 실패"
    action: "에러 메시지 표시 + 파일 형식 가이드 제공"
    message: "파일을 읽을 수 없습니다. CSV, XLSX 형식인지 확인해주세요."

  wrong_extension:
    condition: "확장자와 실제 내용 불일치 (예: .csv인데 바이너리)"
    action: "내용 기반 형식 재감지 시도"
    fallback: "파일 형식을 확인해주세요."

  password_protected:
    condition: "암호화된 Excel 파일"
    action: "비밀번호 해제 후 다시 업로드 요청"
```

---

## 3. Data-Level Errors

```yaml
data_errors:
  description: "데이터 수준 에러 처리"

  missing_required_column:
    condition: "ticker 또는 name 모두 매핑 실패"
    action: "사용자에게 컬럼 매핑 UI 표시"
    severity: "critical"
    message: "종목을 식별할 수 있는 컬럼을 찾지 못했습니다. 종목코드 또는 종목명 컬럼을 선택해주세요."

  missing_price_columns:
    condition: "avg_price와 current_price 모두 없음"
    action: "평가금액(valuation) 또는 손익금액 컬럼에서 역산 시도"
    fallback: "가격 정보 없이 종목 목록과 수량만 표시"
    message: "가격 정보를 찾을 수 없습니다. 수익률 분석은 제한됩니다."

  all_nan_column:
    condition: "매핑된 컬럼의 모든 값이 NaN"
    action: "해당 필드를 '데이터 없음' 처리 후 경고"
    dependent_features: "해당 필드에 의존하는 차트/인사이트 비활성화"

  negative_quantity:
    condition: "수량이 음수"
    action: "공매도로 해석하거나, abs() 적용 후 경고"
    strategy: "거래 유형 컬럼이 있으면 매도로 처리, 없으면 abs() + 경고"

  zero_avg_price:
    condition: "매수가가 0"
    action: "수익률 계산 시 division by zero 방지. 해당 종목 수익률 = N/A 처리"
    message: "'{name}'의 매수가가 0입니다. 수익률을 계산할 수 없습니다."

  duplicate_ticker:
    condition: "같은 종목코드가 여러 행"
    strategy:
      holdings_data: "수량 합산, 매수가는 가중평균"
      trade_data: "거래 내역이면 각 행 유지 (시계열)"
    detection: "ticker + date 조합으로 판별"
    message: "동일 종목({ticker})이 {count}건 있어 합산했습니다."

  single_stock:
    condition: "종목이 1개만 존재"
    action: "포트폴리오 분석 대신 단일 종목 분석 모드로 전환"
    disabled_features: ["섹터 분석", "분산도 평가", "비중 차트"]
    message: "종목이 1개입니다. 단일 종목 분석 모드로 전환합니다."

  too_few_rows:
    condition: "데이터 행이 0개"
    action: "파싱 결과 확인 요청 + 샘플 데이터 제안"

  mixed_data_types:
    condition: "보유 현황과 거래 내역이 혼합된 파일"
    detection: "trade_type 컬럼 존재 여부 + 같은 ticker 다른 날짜 다수"
    action: "거래 내역으로 인식하여 보유 현황을 재계산"
    message: "거래 내역 데이터로 인식했습니다. 최종 보유 현황을 자동으로 계산합니다."
```

---

## 4. Data Quality Warnings

```yaml
data_quality_warnings:
  description: "데이터 품질 경고 규칙"

  stale_data:
    condition: "기준일이 30일 이상 이전"
    message: "데이터 기준일이 {date}입니다. 현재가가 실제와 다를 수 있습니다."
    severity: "low"

  extreme_return:
    condition: "수익률 절대값 > 500%"
    message: "'{name}'의 수익률이 {return}%입니다. 액면분할/병합 또는 데이터 오류일 수 있습니다."
    severity: "medium"

  price_currency_mismatch:
    condition: "KR 종목인데 가격 < 100 또는 US 종목인데 가격 > 100000"
    message: "'{name}'의 가격({price})이 해당 시장 기준으로 비정상적입니다. 통화 단위를 확인해주세요."
    severity: "high"

  missing_rate_high:
    condition: "전체 행의 30% 이상이 NaN 포함"
    message: "데이터의 {rate}%에 빈 값이 있습니다. 분석 결과가 불완전할 수 있습니다."
    severity: "medium"
```

---

## 5. Extension Points

```yaml
extension_points:
  description: "새로운 자산군 추가 시 수정할 항목 가이드"

  new_asset_types:
    crypto:
      description: "암호화폐 자산군"
      ticker_patterns:
        - regex: "^[A-Z]{2,10}$"           # BTC, ETH, SOL
        - regex: "^[A-Z]+-[A-Z]+$"         # BTC-USDT (거래쌍)
      additional_schema_fields:
        wallet_address: { type: string, required: false }
        network: { type: string, enum: [ETH, SOL, BNB, BTC] }
        staking_status: { type: boolean, required: false }
      column_synonyms_extension:
        ticker: [코인명, 코인, Token, Coin, Asset]
        quantity: [보유수량, Amount, Balance, Holdings]
      broker_presets:
        upbit: { broker_name: "업비트", encoding: UTF-8, delimiter: "," }
        bithumb: { broker_name: "빗썸", encoding: EUC-KR, delimiter: "," }
        binance: { broker_name: "바이낸스", encoding: UTF-8, delimiter: "," }

    bond:
      description: "채권 자산군"
      ticker_patterns:
        - regex: "^[A-Z]{2}[0-9]{10}$"     # ISIN
        - regex: "^KR[0-9]{10}$"            # 한국 채권 ISIN
      additional_schema_fields:
        coupon_rate: { type: float, unit: "%" }
        maturity_date: { type: datetime }
        face_value: { type: float }
        credit_rating: { type: string }
      column_synonyms_extension:
        coupon_rate: [이자율, 표면이율, Coupon, Rate]
        maturity_date: [만기일, 상환일, Maturity]

    real_estate:
      description: "부동산/리츠 자산군"
      ticker_patterns:
        - regex: "^REIT-[0-9]+$"           # REITs
      additional_schema_fields:
        property_type: { type: string, enum: [오피스, 리테일, 물류, 주거] }
        dividend_yield: { type: float }
        nav: { type: float, description: "순자산가치" }

    fund:
      description: "펀드 자산군"
      ticker_patterns:
        - regex: "^[A-Z0-9]{8,15}$"        # 펀드 코드
      additional_schema_fields:
        fund_type: { type: string, enum: [주식형, 채권형, 혼합형, MMF] }
        expense_ratio: { type: float, description: "총보수" }
        benchmark: { type: string }

  new_broker_preset:
    description: "새로운 증권사 프리셋 추가 절차"
    steps:
      1: "해당 증권사 CSV 샘플 수집"
      2: "인코딩, 구분자, skip_rows 파악"
      3: "column_map 정의"
      4: "broker_presets에 추가"
      5: "auto_detection 패턴 등록"
```
