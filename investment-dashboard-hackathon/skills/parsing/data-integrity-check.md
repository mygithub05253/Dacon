# Data Integrity Check — Post-Parsing Validation
### 파싱 후 교차 검증으로 분석 파이프라인 진입 전 데이터 오류 차단

> **Pipeline**: Stage 1 final step | **Depends on**: normalization | **Used by**: analysis/*
>
> Input: Standardized DataFrame from normalization
> Output: Validated DataFrame + integrity_report
> Blocking: if critical check fails, halt pipeline and show error

---

## 1. Arithmetic Cross-Validation

```yaml
arithmetic_checks:
  description: "수치 간 산술적 정합성 검증"

  invested_amount_check:
    rule: "invested_amount ≈ avg_price * quantity"
    tolerance: 0.02    # 2% (수수료·반올림 허용)
    actions:
      - deviation: "> 2%"
        severity: "warning"
        action: "해당 행 flag_warning 표시"
        message: "투자금액이 평균단가 × 수량과 {diff}% 차이납니다."
      - deviation: "> 10%"
        severity: "error"
        action: "해당 행 분석 제외"
        message: "투자금액 불일치가 심각합니다 ({diff}%). 이 종목은 분석에서 제외됩니다."

  valuation_check:
    rule: "valuation ≈ current_price * quantity"
    tolerance: 0.02
    skip_if: "current_price is null"
    actions:
      - deviation: "> 2%"
        severity: "warning"
      - deviation: "> 10%"
        severity: "error"

  profit_loss_check:
    rule: "profit_loss ≈ valuation - invested_amount"
    tolerance: 0.01    # 1%
    sign_check: "profit_loss의 부호가 (current_price - avg_price) 부호와 일치해야 함"
    sign_mismatch_severity: "error"
    message: "손익 부호가 가격 변동 방향과 불일치합니다."
```

---

## 2. Logical Consistency Checks

```yaml
logical_checks:
  description: "데이터의 논리적 일관성 검증"
  rules:
    - id: "no_duplicate_tickers"
      check: "ticker 중복 없음 (merge 이후)"
      severity: "error"
      message: "중복 종목이 발견되었습니다: {tickers}. 합산 처리됩니다."

    - id: "positive_quantity"
      check: "quantity > 0 (공매도 플래그 없는 경우)"
      severity: "error"
      message: "수량이 0 이하인 종목이 있습니다: {tickers}"

    - id: "positive_prices"
      check: "avg_price > 0 AND current_price > 0"
      severity: "error"
      message: "가격이 0 이하인 데이터가 있습니다."

    - id: "positive_invested"
      check: "invested_amount > 0"
      severity: "error"
      message: "투자금액이 0 이하인 종목: {tickers}"

    - id: "weight_sum"
      check: "sum(weight) ≈ 100%"
      tolerance: 0.005   # 0.5%
      severity: "warning"
      message: "비중 합계가 {sum}%입니다. 100%와 차이가 있습니다."

    - id: "no_future_dates"
      check: "trade_date <= today"
      severity: "warning"
      message: "미래 날짜의 거래가 포함되어 있습니다: {dates}"
```

---

## 3. Statistical Outlier Detection

```yaml
outlier_detection:
  description: "통계적 이상치 탐지"
  rules:
    - id: "return_rate_zscore"
      check: "return_rate의 Z-score"
      threshold: 4
      severity: "warning"
      message: "⚠️ {ticker}의 수익률({return}%)이 통계적 이상치입니다 (Z={z})."

    - id: "concentration_warning"
      check: "single stock weight > 50%"
      severity: "warning"
      message: "⚠️ {ticker}의 비중이 {weight}%입니다. 집중 투자 리스크에 유의하세요."

    - id: "price_ratio_suspicious"
      check: "avg_price / current_price > 10 OR < 0.1"
      severity: "warning"
      message: "⚠️ {ticker}의 평균단가 대비 현재가 비율이 비정상적입니다 ({ratio}x)."
```

---

## 4. Completeness Check

```yaml
completeness_check:
  description: "필수/권장 필드 존재 여부 및 완성도 점수"
  required_fields:
    - "ticker"
    - "quantity"
    - "avg_price"
  recommended_fields:
    - "current_price"
    - "invested_amount"
    - "sector"
    - "market"

  scoring:
    formula: "filled_fields / total_fields * 100"
    thresholds:
      - score: ">= 80%"
        mode: "full_analysis"
        label: "완전"
      - score: ">= 60%"
        mode: "standard_analysis"
        label: "양호"
      - score: "< 60%"
        mode: "degraded_analysis"
        label: "부족"
        message: "데이터 완성도가 낮아 일부 분석이 제한됩니다 ({score}%)."
```

---

## 5. Integrity Report Output

```yaml
integrity_report:
  description: "검증 결과 리포트 구조"
  summary:
    fields:
      - total_rows: "전체 행 수"
      - passed: "검증 통과 행 수"
      - warnings: "경고 건수"
      - errors: "오류 건수"

  per_row_detail:
    fields:
      - row_index: "행 번호"
      - check_name: "검증 항목 ID"
      - severity: "error | warning | info"
      - message: "사용자 표시 메시지"

  recommendation:
    all_passed: "✅ 전체 검증 통과 — 분석을 시작합니다."
    has_issues: "⚠️ {n}개 항목 확인 필요 — 아래 상세 내역을 참고하세요."

  display:
    position: "분석 결과 상단"
    style: "collapsible section"
    default_state: "collapsed (오류 없을 시) | expanded (오류 있을 시)"
```

---

## 6. Error Handling

```yaml
error_handling:
  description: "심각도별 처리 전략"
  levels:
    critical:
      description: "파이프라인 중단"
      conditions:
        - "유효 행 0건"
        - "모든 가격이 음수"
        - "스키마 불일치 (필수 컬럼 누락)"
      action: "분석 중단, 사용자에게 오류 메시지 표시"
      message: "데이터를 분석할 수 없습니다. 파일을 확인 후 다시 업로드해주세요."

    warning:
      description: "경고 표시 후 계속 진행"
      conditions:
        - "개별 행 이상치"
        - "선택 필드 누락"
        - "비중 합계 불일치"
      action: "해당 행/필드에 플래그 표시, 분석 계속"

    info:
      description: "참고 사항 기록"
      conditions:
        - "미미한 반올림 차이"
        - "자동 보정된 값"
      action: "리포트에 기록, 사용자 알림 없음"
```
