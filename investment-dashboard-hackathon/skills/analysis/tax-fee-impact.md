# Tax & Fee Impact — After-Tax Return Rules

> Defines tax and fee simulation rules for showing users
> the real impact on their investment returns.
> Source: `analysis_rules.md` Stage 2 supplement

---

## Pipeline Position

```yaml
stage: 2_supplement
referenced_by:
  - "basic-metrics.md (after-tax return calculation)"
  - "report/section-specs (footer labels, tax disclaimer)"
```

---

## 1. Tax Categories

```yaml
# 세금 카테고리별 규칙

domestic_stocks_kr:
  # 1.1 국내 주식
  capital_gains_tax:
    major_shareholder:
      rate: 0.22          # 22% (지방세 포함)
      condition: "대주주 요건 충족 시"
      # 설명: 대주주 양도소득세 — 양도차익에 대해 22% 과세
    general_investor:
      rate: 0.00          # 비과세 (2025년 기준)
      note: "금융투자소득세 유예 상태"
      effective_date: "2025-01-01"
      # 설명: 일반 투자자 주식 양도소득 비과세 (금투세 유예)
  dividend_tax:
    rate: 0.154           # 15.4% (소득세 14% + 지방소득세 1.4%)
    withholding: true
    # 설명: 배당소득세 — 배당금 수령 시 원천징수

foreign_stocks:
  # 1.2 해외 주식
  capital_gains_tax:
    rate: 0.22            # 22% (지방세 포함)
    exemption: 2500000    # 연간 기본공제 250만원
    unit: "KRW"
    formula: "(annual_gains - 2,500,000) * 0.22"
    # 설명: 해외주식 양도소득세 — 연간 수익 250만원 초과분에 22%
  dividend_tax:
    us_stocks:
      withholding_rate: 0.15    # 미국 원천징수 15%
      additional_kr_tax: 0.00   # 조세조약에 의해 추가 과세 없음
    other_countries:
      rule: "원천징수세율 + 한국 추가과세 (15.4%와의 차이분)"
      # 설명: 조세조약에 따라 국가별 상이

account_type_impact:
  # 1.3 계좌 유형별 세금 영향
  general_account:
    description: "일반 위탁계좌"
    tax_treatment: "full_tax"
    # 설명: 모든 세금 정상 적용

  isa_account:
    description: "ISA (개인종합자산관리계좌)"
    tax_treatment: "partial_exemption"
    exemption_limit:
      general: 2000000      # 일반형 200만원
      income_type: 4000000  # 서민형/청년형 400만원
    excess_rate: 0.099      # 초과분 9.9% 분리과세
    # 설명: ISA 비과세 한도 내 면제, 초과분 저율 과세

  pension_account:
    description: "연금저축/IRP"
    tax_treatment: "deferred"
    withdrawal_tax:
      pension_income: "3.3% ~ 5.5%"    # 연금소득세 (연령별 차등)
      lump_sum: "16.5%"                 # 일시 인출 시 기타소득세
    # 설명: 납입 시 세액공제, 인출 시 과세 (과세이연)
```

---

## 2. Fee Categories

```yaml
# 수수료 카테고리

fees:
  trading_commission:
    description: "매매 수수료 (증권사별 상이)"
    typical_range: "0.01% ~ 0.50%"
    default_estimate: 0.015    # 0.015% (온라인 기준)
    # 설명: MTS/HTS 기준 온라인 수수료. 증권사 이벤트로 무료인 경우 많음.

  exchange_fee:
    description: "거래소 수수료 (유관기관 제비용)"
    krx_rate: 0.00004          # 약 0.004%
    # 설명: KRX 거래소 자체 수수료. 증권사 수수료와 별도.

  fx_conversion_fee:
    description: "환전 수수료 (해외주식)"
    typical_range: "0.1% ~ 1.0%"
    default_estimate: 0.25     # 0.25%
    # 설명: 원화→달러 환전 시 스프레드. 증권사/환전 방식에 따라 상이.

  account_maintenance:
    description: "계좌 유지 수수료"
    typical: 0                 # 대부분 무료 (온라인 증권사)
```

---

## 3. Calculation Rules

```yaml
# 세후/수수료 차감 수익률 계산

calculation:
  after_tax_return:
    formula: "pre_tax_return - estimated_tax_impact"
    display_as: "range"        # 정확한 값이 아닌 범위로 표시
    # 설명: 세후 수익률은 추정치 — 정확한 값은 개인 상황에 따라 다름

  fee_adjusted_return:
    formula: "gross_return - total_fees"
    total_fees: "trading_commission + exchange_fee + fx_fee"

  display_priority:
    - "pre_tax_return (세전 수익률)"
    - "fee_adjusted_return (수수료 차감 수익률)"
    - "after_tax_return (추정 세후 수익률)"

  rounding:
    return_rate: 2             # 소수점 2자리
    tax_amount: 0              # 원 단위 (KRW)
```

---

## 4. Display Rules

```yaml
# 화면 표시 규칙

display:
  default_basis: "pre_tax"     # 기본: 세전 표시
  toggle_option: true          # 세후 토글 제공
  toggle_label: "추정 세후 보기"

  label_requirement:
    rule: "항상 어떤 기준인지 명시"
    pre_tax_label: "세전"
    after_tax_label: "추정 세후"
    fee_adjusted_label: "수수료 차감"

  disclaimer:
    required: true
    text: "세금 계산은 추정치이며, 실제 세금은 개인 상황에 따라 다릅니다."
    position: "해당 수치 하단 또는 리포트 푸터"
    style: "muted text, smaller font"

  comparison_view:
    description: "세전 vs 세후 비교 표시"
    format: "세전 +12.5% → 추정 세후 +9.8%"
    # 설명: 사용자가 세금 영향을 직관적으로 파악하도록 병렬 표시
```

---

## 5. Edge Cases / Limitations

```yaml
# 예외 상황 및 한계

limitations:
  tax_bracket_unknown:
    description: "사용자의 정확한 과세 구간을 알 수 없음"
    action: "일반적인 세율로 추정, 면책 문구 표시"
    message: "개인 소득 수준에 따라 실제 세율이 다를 수 있습니다."

  tax_law_changes:
    description: "세법 변경 빈번"
    action: "effective_date 필드로 관리, 매년 업데이트"
    current_basis: "2025년 기준"
    # 설명: 금투세 시행 시 별도 업데이트 필요

  financial_investment_income_tax:
    description: "금융투자소득세 (금투세) 시행 대비"
    status: "유예 중 (2025년 기준)"
    planned_rate: 0.22         # 시행 시 5,000만원 초과분 22%
    action: "시행 확정 시 domestic_stocks_kr.capital_gains_tax 업데이트"

  multi_year_carryover:
    description: "해외주식 손익통산/이월공제"
    action: "단일연도 기준으로만 계산. 이월공제 미반영."
    message: "전년도 손실 이월공제는 반영되지 않습니다."

  extension_api:
    description: "실제 세금 계산기 API 연동"
    status: "future"
    note: "국세청 홈택스 API 또는 세무 SaaS 연동 고려"
```
