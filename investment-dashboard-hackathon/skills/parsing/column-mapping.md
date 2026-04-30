# Column Mapping Rules
### Standard Schema, Synonym Dictionary & Mapping Pipeline

> **Pipeline**: Stage 1 | **Depends on**: none | **Used by**: analysis/*
>
> Defines the canonical schema all parsed data must conform to,
> the synonym dictionary for matching heterogeneous column names,
> and the multi-step mapping strategy.

---

## 1. Standard Schema

```yaml
standard_schema:
  description: "모든 파싱 결과가 준수해야 하는 표준 스키마"

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
    description: "보유 수량"
    validation:
      min: 0             # 0 허용 (매도 완료 종목)
      negative_handling:
        if_trade_type_exists: preserve_sign  # 공매도(short) 포지션 유지
        if_trade_type_missing: flag_warning  # 사용자 확인 요청, abs() 자동 적용 금지
        warning_message: "음수 수량이 감지되었습니다. 공매도 포지션인지 확인해주세요."
        # description: 공매도 손익 부호 반전 방지. 자동 abs() 금지
      float_handling:
        KR_market: round_to_integer  # 한국 주식은 1주 단위
        US_market: allow_float       # 미국 주식 소수점 매매 허용 (Robinhood, IBKR 등)
        OTHER: allow_float           # 기타 시장은 float 허용
        # description: 시장에 따라 소수점 주식 허용 여부 결정. 한국만 정수 강제

  avg_price:
    type: float
    required: true
    description: "평균 매입가"
    validation:
      min: 0             # 0 이하 불허
      zero_handling: "0이면 해당 행 제외 후 경고"
      extreme_check: "같은 시장 종목 대비 100배 이상 차이 시 통화 오류 의심 경고"

  current_price:
    type: float
    required: false
    description: "현재가"
    fallback:
      strategy_1: null  # avg_price 복사 삭제 — 수익률이 0%로 왜곡됨
      strategy_2: mark_as_unavailable
      display: "현재가 미제공"
      return_rate: null  # 수익률 산출 불가로 표시
      # description: 현재가 없으면 N/A 처리. avg_price 복사는 수익률 0% 왜곡 유발
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
    description: "섹터/산업 분류"
    derivation_sources:
      - "CSV에 섹터/산업 컬럼이 있으면 그대로 사용"
      - "없으면 ticker 기반 섹터 매핑 테이블 참조"
      - "매핑 실패 시 '기타' 섹터로 분류"

  date:
    type: datetime
    required: false
    description: "거래/기준 일자"
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

  # --- Phase 3 추가 필드 (optional) ---
  account_type:
    type: enum
    values: ["일반", "ISA", "ISA_서민형", "연금저축", "IRP", "OTHER"]
    required: false
    default: "일반"
    synonyms: ["계좌유형", "계좌종류", "account", "계좌구분"]
    # description: 계좌 유형별 세제 혜택 차등 적용 (참조: tax-fee-impact.md)

  purchase_date:
    type: date
    required: false
    synonyms: ["매수일", "매수일자", "buy_date", "거래일", "trade_date"]
    format: auto_detect  # 참조: normalization.md date_locale_rule
    # description: 보유기간 계산 및 장기보유 특별공제 판단용

  dividend_amount:
    type: number
    required: false
    unit: currency
    synonyms: ["배당금", "배당액", "dividend", "분배금", "배당수익"]
    # description: 배당금 수령 내역 (참조: income-analysis.md)

  dividend_date:
    type: date
    required: false
    synonyms: ["배당일", "배당지급일", "ex_dividend_date", "배당기준일"]
    # description: 배당 지급일/기준일 (income-analysis.md 배당캘린더용)
```

---

## 2. Column Synonym Dictionary

```yaml
column_synonyms:
  description: "컬럼명 동의어 사전 — 증권사별 다양한 이름을 표준 필드에 매핑"

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

---

## 3. Mapping Strategy & Pipeline

```yaml
mapping_strategy:
  description: "6단계 매핑 파이프라인"
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
    description: "매핑되지 않은 컬럼 처리"
    action: "무시하되, 매핑되지 않은 컬럼 목록을 로그에 표시"
    message: "다음 컬럼은 분석에 사용되지 않습니다: {columns}"
```
