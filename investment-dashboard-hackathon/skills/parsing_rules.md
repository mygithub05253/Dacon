# Parsing Rules — 데이터 파싱 규칙

> AlphaFolio 데이터 입력 계층의 규칙을 정의합니다.
> 이 문서를 참조하여 다양한 형식의 투자 CSV를 표준 구조로 변환합니다.
>
> **파이프라인 위치**: Stage 1 (최초 진입점)
> **출력**: 표준 스키마(standard_schema)로 정규화된 DataFrame
> **하위 의존**: analysis_rules.md가 이 문서의 표준 스키마 필드명을 참조합니다.

---

## 1. 파일 인식 규칙

### 1.1 지원 파일 형식
```yaml
supported_formats:
  - extension: ".csv"
    mime: "text/csv"
  - extension: ".tsv"
    mime: "text/tab-separated-values"
  - extension: ".xlsx"
    mime: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    note: "첫 번째 시트만 읽음. 시트 선택 UI 제공."
  - extension: ".xls"
    mime: "application/vnd.ms-excel"
    note: "레거시 형식, openpyxl fallback"
  - extension: ".txt"
    mime: "text/plain"
    note: "구분자 자동 감지"

file_size_limit:
  max: "50MB"
  warning_at: "10MB"
  action_over_limit: "파일이 너무 큽니다 (최대 50MB). 데이터를 분할해주세요."

row_limit:
  max: 100000
  warning_at: 50000
  action_over_limit: "최근 {max}건만 분석합니다. 전체 분석이 필요하면 기간을 나눠서 업로드하세요."
```

### 1.2 인코딩 감지 우선순위
```yaml
encoding_detection:
  strategy: "chardet 라이브러리로 자동 감지 → 실패 시 순차 시도"
  priority:
    - UTF-8
    - UTF-8-SIG       # BOM 포함 (Excel에서 저장 시)
    - EUC-KR           # 국내 증권사 다수
    - CP949            # EUC-KR 확장
    - ASCII
  fallback: UTF-8
  exception:
    decode_error:
      action: "다음 인코딩으로 재시도"
      max_retries: 4
      final_fallback: "errors='replace' 옵션으로 깨진 문자 대체 후 경고"
      message: "일부 문자가 깨졌을 수 있습니다. 원본 파일의 인코딩을 확인해주세요."
```

### 1.3 구분자 감지
```yaml
delimiter_detection:
  strategy: "csv.Sniffer + 빈도 분석"
  candidates:
    - ","        # CSV 기본
    - "\t"       # TSV (키움증권 등)
    - "|"        # 파이프 구분 (일부 금융 데이터)
    - ";"        # 유럽식 CSV
  exception:
    mixed_delimiters:
      description: "한 파일에 여러 구분자 혼용"
      action: "가장 빈도 높은 구분자 선택 후 경고"
      message: "구분자가 일관되지 않습니다. ','로 처리했습니다."
    no_delimiter:
      description: "구분자 없이 공백만 존재"
      action: "\\s+ (연속 공백) 구분자로 시도"
```

### 1.4 헤더 감지
```yaml
header_detection:
  strategy: "행별 특성 분석으로 헤더 행 자동 탐지"
  rules:
    - "첫 N행(최대 10행) 스캔하여 텍스트 비율이 80% 이상인 첫 행을 헤더로 인식"
    - "숫자만 있는 행은 데이터 행으로 판단"
  skip_patterns:
    - pattern: "계좌번호|조회일시|고객명|출력일"
      action: "해당 행 스킵 (증권사 메타 정보)"
    - pattern: "^\\s*$"
      action: "빈 행 스킵"
    - pattern: "^#|^//"
      action: "주석 행 스킵"
  exception:
    no_header_found:
      action: "자동 헤더 생성 (col_0, col_1, ...) 후 사용자에게 매핑 요청"
      message: "헤더를 자동 인식할 수 없습니다. 컬럼을 직접 지정해주세요."
    multi_level_header:
      description: "2행 이상의 병합 헤더 (증권사 리포트)"
      action: "행을 합쳐서 단일 헤더로 변환. 예: '수익률' + '(%)' → '수익률(%)'"
    duplicate_column_names:
      description: "같은 이름의 컬럼이 여러 개"
      action: "뒤에 _1, _2 접미사 추가 후 경고"
      message: "중복 컬럼명이 있어 자동으로 구분했습니다: {columns}"
```

---

## 2. 컬럼 매핑 규칙

### 2.1 표준 스키마
```yaml
standard_schema:
  ticker:
    type: string
    required: true
    description: "종목 식별 코드"
    validation: "빈 문자열, null, NaN 불허"
  name:
    type: string
    required: true
    description: "종목명"
    validation: "빈 문자열 불허. 공백만 있는 경우 trim 후 검증"
  market:
    type: string
    enum: [KR, US, OTHER]
    derived: true
    derivation_rule: "ticker 패턴 기반. 어디에도 매칭 안 되면 OTHER"
  quantity:
    type: integer
    required: true
    validation:
      min: 0             # 0 허용 (매도 완료 종목)
      negative_handling: "음수면 abs() 적용 후 경고"
      float_handling: "소수점 있으면 반올림 후 경고 (소수점 주식 미지원)"
  avg_price:
    type: float
    required: true
    validation:
      min: 0             # 0 이하 불허
      zero_handling: "0이면 해당 행 제외 후 경고"
      extreme_check: "같은 시장 종목 대비 100배 이상 차이 시 통화 오류 의심 경고"
  current_price:
    type: float
    required: false
    fallback:
      strategy_1: "avg_price 값 복사 (수익률 0% 처리)"
      strategy_2: "외부 API로 현재가 조회 시도 (yfinance)"
      strategy_3: "사용자에게 수동 입력 요청"
    validation:
      negative_handling: "음수면 abs() 적용 후 경고"
      zero_handling: "0이면 상장폐지 가능성 경고"
  currency:
    type: string
    enum: [KRW, USD, JPY, EUR, CNY, OTHER]
    derived: true
    derivation_rule: "market 기반 추론. KR→KRW, US→USD. 명시적 통화 컬럼 있으면 우선"
  sector:
    type: string
    derived: true
    derivation_sources:
      - "CSV에 섹터/산업 컬럼이 있으면 그대로 사용"
      - "없으면 ticker 기반 섹터 매핑 테이블 참조"
      - "매핑 실패 시 '기타' 섹터로 분류"
  date:
    type: datetime
    required: false
    fallback: "오늘 날짜 (datetime.now())"
  trade_type:
    type: string
    required: false
    enum: [BUY, SELL, DIVIDEND]
    description: "거래 유형 (거래 내역 CSV인 경우)"
  invested_amount:
    type: float
    derived: true
    derivation_rule: "avg_price * quantity"
    description: "종목별 투자 원금 (포트폴리오 전체 투자금액 합산에 사용)"
    note: "CSV에 투자금액 컬럼이 명시적으로 있으면 해당 값 우선 사용"
```

### 2.2 컬럼명 동의어 사전
```yaml
column_synonyms:
  ticker:
    ko: [종목코드, 종목번호, 코드, 종목코드(6자리), 상품코드, 단축코드]
    en: [Symbol, Ticker, Code, Stock Code, Asset Code, ISIN]
    pattern: ["*코드*", "*번호*", "*symbol*", "*ticker*"]
  name:
    ko: [종목명, 종목이름, 상품명, 자산명, 펀드명, ETF명]
    en: [Name, Stock Name, Asset Name, Description, Security Name]
    pattern: ["*종목*명*", "*name*"]
  quantity:
    ko: [보유수량, 수량, 잔고수량, 보유주수, 잔고, 주수, 보유량]
    en: [Quantity, Shares, Qty, Holdings, Position, Volume]
    pattern: ["*수량*", "*주수*", "*shares*", "*qty*"]
  avg_price:
    ko: [평균매입가, 매수단가, 평균단가, 매입평균가, 평균매수가, 취득단가, 매입가]
    en: [Avg Cost, Average Price, Cost Basis, Purchase Price, Avg Price]
    pattern: ["*평균*가*", "*매입*가*", "*매수*가*", "*avg*", "*cost*"]
  current_price:
    ko: [현재가, 종가, 시장가, 기준가, 평가단가, 전일종가]
    en: [Current Price, Last Price, Close, Market Price, Price, Last]
    pattern: ["*현재*", "*종가*", "*close*", "*price*"]
  date:
    ko: [일자, 날짜, 거래일, 기준일, 조회일자, 체결일, 결제일, 거래일시]
    en: [Date, Trade Date, Settlement Date, As Of Date, Timestamp]
    pattern: ["*일자*", "*날짜*", "*date*"]
  trade_type:
    ko: [거래구분, 매매구분, 거래유형, 구분, 매수매도, 거래종류]
    en: [Type, Side, Trade Type, Action, Transaction Type, Buy/Sell]
    trade_value_mapping:
      BUY: [매수, 매입, 입고, Buy, B, 1, 신규매수]
      SELL: [매도, 매출, 출고, Sell, S, 2, 전량매도, 일부매도]
      DIVIDEND: [배당, 이자, 분배금, Dividend, Div, D, 배당금입금]
  valuation:
    ko: [평가금액, 평가액, 보유금액, 자산가치]
    en: [Market Value, Valuation, Value, Total Value]
    note: "평가금액 컬럼이 있으면 current_price 대신 역산 가능"
  profit_loss:
    ko: [손익금액, 평가손익, 실현손익, 수익금, 손익]
    en: [P&L, Profit Loss, Gain Loss, Unrealized P&L]
```

### 2.3 매핑 전략
```yaml
mapping_strategy:
  pipeline:
    step_1_exact_match:
      description: "컬럼명이 동의어 사전에 정확히 일치 (대소문자 무시)"
      confidence: 1.0
    step_2_pattern_match:
      description: "와일드카드 패턴 매칭 (*코드*, *price* 등)"
      confidence: 0.9
    step_3_fuzzy_match:
      description: "유사도 80% 이상 (Levenshtein distance, 자소 분리 기반)"
      confidence: 0.7
      min_similarity: 0.8
    step_4_type_inference:
      description: "컬럼 값 패턴으로 타입 추론"
      confidence: 0.5
      rules:
        - pattern: "6자리 숫자가 80% 이상"
          infer: "ticker (KR)"
        - pattern: "1~5자리 영문 대문자가 80% 이상"
          infer: "ticker (US)"
        - pattern: "정수 + 값 범위 1~999999"
          infer: "quantity"
        - pattern: "소수점 포함 + 값 > 100"
          infer: "price (avg or current)"
        - pattern: "날짜 형식 매칭률 80% 이상"
          infer: "date"
        - pattern: "매수|매도|Buy|Sell 포함 80% 이상"
          infer: "trade_type"
    step_5_reverse_engineer:
      description: "이미 매핑된 컬럼 관계로 나머지 추론"
      rules:
        - "평가금액 있고 수량 있으면: current_price = 평가금액 / 수량"
        - "손익금액 있고 평가금액, 투자금액 있으면: profit_loss 검증"
        - "수익률 있고 매수가 있으면: current_price = avg_price * (1 + 수익률/100)"
    step_6_user_confirm:
      description: "자동 매핑 실패 또는 confidence < 0.7인 컬럼은 사용자 확인"
      ui: "드롭다운 선택 (표준 스키마 필드 목록 + '사용 안 함' 옵션)"
      message: "자동 매핑된 결과를 확인해주세요. 잘못된 항목이 있으면 수정하세요."

  conflict_resolution:
    description: "같은 표준 필드에 여러 컬럼이 매핑될 때"
    strategy: "confidence가 높은 컬럼 우선, 동점이면 왼쪽(첫 번째) 컬럼 우선"
    user_override: true  # 사용자가 직접 선택 가능

  unmapped_columns:
    action: "무시하되, 매핑되지 않은 컬럼 목록을 로그에 표시"
    message: "다음 컬럼은 분석에 사용되지 않습니다: {columns}"
```

---

## 3. 데이터 정규화 규칙

### 3.1 숫자 정규화
```yaml
number_normalization:
  remove_chars: [",", " ", "원", "$", "₩", "\\", "¥", "€", "주"]
  negative_formats:
    - "-1000"         # 표준
    - "(1000)"        # 회계 표기
    - "▼1000"         # 일부 증권사
    - "△1000"         # 상승 기호 (양수로 처리)
    - "▲1000"         # 상승 기호
    - "-1,000.00"     # 콤마 + 소수점 혼용
  percentage:
    detection: "값에 '%' 포함 여부"
    rules:
      - "5.2%"   → 5.2    # % 기호 제거, 값 유지
      - "0.052"  → 5.2    # 소수형이면 ×100 (단, 값 < 1일 때만)
      - "-0.03"  → -3.0   # 음수 소수도 동일
      - "5.2"    → 5.2    # 이미 백분율이면 유지
    ambiguity:
      description: "0.052가 5.2%인지 0.052% 인지 모호한 경우"
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

### 3.2 날짜 정규화
```yaml
date_normalization:
  input_formats:
    - "YYYY-MM-DD"         # 2024-01-15 (국제 표준)
    - "YYYY/MM/DD"         # 2024/01/15
    - "YYYYMMDD"           # 20240115 (증권사 다수)
    - "YYYY.MM.DD"         # 2024.01.15
    - "MM/DD/YYYY"         # 01/15/2024 (미국식)
    - "DD-MM-YYYY"         # 15-01-2024 (유럽식)
    - "DD/MM/YYYY"         # 15/01/2024
    - "YYYY년 MM월 DD일"   # 한국어 표기
    - "YYYY-MM-DD HH:MM:SS"  # 타임스탬프 포함
    - "MM-DD-YYYY"         # 01-15-2024
  output_format: "YYYY-MM-DD"
  
  ambiguity:
    description: "01/02/2024가 1월 2일인지 2월 1일인지 모호"
    strategy:
      - "파일 내 다른 날짜값에서 패턴 추론 (13 이상 값이 첫 자리면 DD-MM)"
      - "증권사 locale 기반 추론 (국내 → YYYY-MM-DD 계열 우선)"
      - "추론 불가 시 MM/DD (미국식) 기본 적용 후 경고"
  
  exception:
    invalid_date:
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

### 3.3 마켓 자동 식별
```yaml
market_detection:
  KR:
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
    patterns:
      - regex: "^[A-Z]{1,5}$"               # 표준 ticker
        examples: ["AAPL", "TSLA", "VOO", "BRK.B"]
      - regex: "^[A-Z]{1,5}\\.[A-Z]$"       # 클래스 구분 (BRK.B)
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

### 3.4 통화 환산 규칙
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
    per_stock: "원래 통화로 계산 후 표시"
    portfolio_total: "base_currency로 환산하여 합산"
    display: "원화 환산액과 원래 통화 병기"
```

---

## 4. 에러 처리 규칙

### 4.1 파일 수준 에러
```yaml
file_errors:
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

### 4.2 데이터 수준 에러
```yaml
data_errors:
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

### 4.3 데이터 품질 경고
```yaml
data_quality_warnings:
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

## 5. 증권사별 파싱 프리셋

```yaml
broker_presets:
  description: "주요 증권사별 CSV 특성을 사전 정의하여 자동 감지 정확도 향상"
  
  키움증권:
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

  미래에셋증권:
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

  삼성증권:
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

  토스증권:
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

  한국투자증권:
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
    strategy: "파일 첫 10행에서 broker_presets의 header_pattern과 매칭"
    fallback: "프리셋 매칭 실패 시 범용 매핑 전략(2.3) 사용"
    confidence_display: "'{broker}' 형식으로 자동 인식했습니다."
```

---

## 6. 확장 규칙

```yaml
extension_points:
  new_asset_type:
    description: "새로운 자산군 추가 시 수정할 항목 가이드"
    
    crypto:
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
        업비트: { encoding: UTF-8, delimiter: "," }
        빗썸: { encoding: EUC-KR, delimiter: "," }
        바이낸스: { encoding: UTF-8, delimiter: "," }

    bond:
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
      ticker_patterns:
        - regex: "^REIT-[0-9]+$"           # REITs
      additional_schema_fields:
        property_type: { type: string, enum: [오피스, 리테일, 물류, 주거] }
        dividend_yield: { type: float }
        nav: { type: float, description: "순자산가치" }

    fund:
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
