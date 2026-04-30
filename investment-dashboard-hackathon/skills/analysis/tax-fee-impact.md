# Tax & Fee Impact — 세후 수익률 및 비용 시뮬레이션 규칙

> 투자 수익에 대한 세금·수수료의 실질 영향을 시뮬레이션하기 위한 규칙 정의.
> 모든 수치는 검증된 한국 세법·금융 규정 기준.
> Source: `analysis_rules.md` Stage 2 supplement

```yaml
effective_date: "2025-01-01"
last_verified: "2026-04-30"
data_basis: "2025-2026년 한국 세법 및 금융규정"
base_rate_bok: 2.50            # 한국은행 기준금리 (2026.04 현재)
```

---

## Pipeline Position

```yaml
stage: 2_supplement
referenced_by:
  - "basic-metrics.md (after-tax return calculation)"
  - "report/section-specs (footer labels, tax disclaimer)"
```

---

## 1. 국내 주식 세금

```yaml
domestic_stocks_kr:
  # 1.1 양도소득세
  capital_gains_tax:
    major_shareholder:
      # 대주주 기준 (소득세법 시행령 제157조)
      criteria:
        kospi:
          ownership_pct: 0.01        # 지분율 1%
          market_cap: 5000000000     # 시가총액 50억원
          logic: "OR"                # 둘 중 하나 충족 시
        kosdaq:
          ownership_pct: 0.02        # 지분율 2%
          market_cap: 5000000000     # 시가총액 50억원
          logic: "OR"
      tax_rate:
        bracket_1:
          threshold: 300000000       # 과세표준 3억원 이하
          rate: 0.20                 # 20%
        bracket_2:
          threshold_above: 300000000 # 과세표준 3억원 초과
          rate: 0.25                 # 25%
      local_tax: "+10% of 국세"      # 지방소득세 별도
    general_investor:
      rate: 0.00                     # 비과세
      note: "금융투자소득세(금투세) 2025년 폐지 확정"
      status: "abolished"
      # 2024.12 국회 본회의 금투세 폐지안 의결

  # 1.2 배당소득세
  dividend_tax:
    rate: 0.154                      # 15.4%
    breakdown:
      income_tax: 0.14              # 소득세 14%
      local_tax: 0.014              # 지방소득세 1.4%
    withholding: true
    note: "배당 수령 시 원천징수"

  # 1.3 금융소득종합과세
  financial_income_comprehensive:
    threshold: 20000000              # 이자+배당 합산 2,000만원
    condition: "초과 시 종합소득에 합산 과세"
    max_rate: 0.495                  # 최고 49.5% (지방소득세 포함)
    note: "이자소득 + 배당소득 연간 합계 기준"
```

---

## 2. 해외 주식 세금

```yaml
foreign_stocks:
  # 2.1 양도소득세
  capital_gains_tax:
    rate: 0.22                       # 22%
    breakdown:
      national_tax: 0.20            # 국세 20%
      local_tax: 0.02               # 지방소득세 2%
    exemption: 2500000               # 연간 기본공제 250만원 (KRW)
    formula: "(annual_gains - 2,500,000) * 0.22"
    filing_period: "매년 5월 (전년도분 확정신고)"
    note: "해외주식 양도소득은 국내와 달리 일반 투자자도 과세"

  # 2.2 배당소득세 (미국 주식)
  dividend_tax:
    us_stocks:
      withholding_rate: 0.15         # 미국 원천징수 15%
      treaty: "한미 조세조약"
      additional_kr_tax: 0.00        # 국내 추가 납부 없음
      note: "조약 세율 15%가 국내 세율 14%를 초과 → 추가 과세 없음"
    other_countries:
      rule: "현지 원천징수 + 한국 15.4%와의 차이분 추가 납부"
      note: "조세조약 국가별 상이, 외국납부세액공제 적용 가능"
```

---

## 3. 계좌 유형별 세금 영향

```yaml
account_type_impact:
  general_account:
    description: "일반 위탁계좌"
    tax_treatment: "full_tax"

  isa_account:
    description: "ISA (개인종합자산관리계좌)"
    tax_treatment: "partial_exemption"
    mandatory_period: "3년 (의무가입기간)"
    exemption_limit:
      general: 2000000               # 일반형 비과세 200만원
      low_income: 4000000            # 서민형/농어민형 비과세 400만원
    excess_rate: 0.099               # 초과분 9.9% 분리과세
    note: "만기 후 연금계좌 전환 시 추가 세액공제 혜택"

  pension_savings:
    description: "연금저축"
    tax_treatment: "deferred"
    annual_deduction_limit: 6000000  # 세액공제 한도 600만원/년
    annual_deposit_limit: 18000000   # 납입 한도 1,800만원/년 (연금계좌 합산)
    withdrawal_tax_by_age:
      under_70: 0.055               # 70세 미만 5.5%
      age_70_to_80: 0.044           # 70~80세 4.4%
      age_80_plus: 0.033            # 80세 이상 3.3%
    early_withdrawal: 0.165          # 중도 인출 시 기타소득세 16.5%

  irp_account:
    description: "IRP (개인형퇴직연금)"
    tax_treatment: "deferred"
    combined_deduction_limit: 9000000 # 연금저축+IRP 합산 세액공제 900만원/년
    annual_deposit_limit: 18000000    # 납입 한도 1,800만원/년 (연금계좌 합산)
    note: "퇴직급여 수령 시 퇴직소득세 적용 (별도 계산)"
```

---

## 4. 거래 수수료 (2025년 기준)

```yaml
fees:
  # 4.1 증권거래세 (매도 시)
  securities_transaction_tax:
    note: "2025년 기준 세율. 2023년 이후 단계적 인하 적용."
    kospi:
      transaction_tax: 0.0003       # 증권거래세 0.03%
      agriculture_tax: 0.0015       # 농어촌특별세 0.15%
      total: 0.0018                 # 합계 0.18%
    kosdaq:
      transaction_tax: 0.0018       # 증권거래세 0.18%
      agriculture_tax: 0.0002       # 농어촌특별세 0.02%
      total: 0.0020                 # 합계 0.20%

  # 4.2 증권사 매매수수료
  trading_commission:
    description: "온라인(MTS/HTS) 기준"
    typical_range: "0.01% ~ 0.05%"
    default_estimate: 0.00015       # 0.015%
    event_free: true                # 신규 계좌 이벤트 시 무료 빈번
    note: "증권사·이벤트에 따라 0% 가능"

  # 4.3 거래소 수수료 (유관기관 제비용)
  exchange_fee:
    krx_rate: 0.00004               # 약 0.004%
    note: "KRX 거래소 자체 수수료, 증권사 수수료와 별도"

  # 4.4 환전 수수료 (해외주식)
  fx_conversion_fee:
    typical_range: "0.1% ~ 1.0%"
    default_estimate: 0.0025        # 0.25%
    note: "원화→달러 환전 스프레드. 증권사·환전 방식에 따라 상이"

  # 4.5 계좌 유지 수수료
  account_maintenance:
    typical: 0                       # 대부분 무료 (온라인 증권사)
```

---

## 5. 계산 규칙

```yaml
calculation:
  after_tax_return:
    formula: "pre_tax_return - estimated_tax_impact"
    display_as: "range"              # 정확한 값이 아닌 범위로 표시
    note: "세후 수익률은 추정치 — 정확한 값은 개인 상황에 따라 다름"

  fee_adjusted_return:
    formula: "gross_return - total_fees"
    total_fees: "trading_commission + exchange_fee + securities_tax + fx_fee"

  display_priority:
    - "pre_tax_return (세전 수익률)"
    - "fee_adjusted_return (수수료 차감 수익률)"
    - "after_tax_return (추정 세후 수익률)"

  rounding:
    return_rate: 2                   # 소수점 2자리
    tax_amount: 0                    # 원 단위 (KRW)
```

---

## 6. 표시 규칙

```yaml
display:
  default_basis: "pre_tax"
  toggle_option: true
  toggle_label: "추정 세후 보기"

  label_requirement:
    rule: "항상 어떤 기준인지 명시"
    pre_tax_label: "세전"
    after_tax_label: "추정 세후"
    fee_adjusted_label: "수수료 차감"

  disclaimer:
    required: true
    text: "세금 계산은 추정치이며, 실제 세금은 개인의 소득·계좌 유형·거래 규모에 따라 다릅니다. 투자 판단의 근거로 사용하지 마십시오."
    position: "해당 수치 하단 또는 리포트 푸터"
    style: "muted text, smaller font"

  comparison_view:
    format: "세전 +12.5% → 추정 세후 +9.8%"
    note: "사용자가 세금 영향을 직관적으로 파악하도록 병렬 표시"
```

---

## 7. 예외 상황 및 한계

```yaml
limitations:
  tax_bracket_unknown:
    description: "사용자의 정확한 과세 구간을 알 수 없음"
    action: "일반적인 세율로 추정, 면책 문구 표시"
    message: "개인 소득 수준에 따라 실제 세율이 다를 수 있습니다."

  tax_law_changes:
    description: "세법 변경 빈번"
    action: "effective_date 필드로 관리, 매년 업데이트"
    current_basis: "2025-2026년 기준"

  multi_year_carryover:
    description: "해외주식 손익통산/이월공제"
    action: "단일연도 기준으로만 계산. 이월공제 미반영."
    message: "전년도 손실 이월공제는 반영되지 않습니다."

  comprehensive_taxation_limit:
    description: "금융소득종합과세 시뮬레이션 한계"
    action: "2,000만원 초과 여부만 경고, 정확한 종합과세 세율은 미산출"
    message: "금융소득이 2,000만원을 초과할 경우 종합과세 대상이 될 수 있습니다."

  extension_api:
    description: "실제 세금 계산기 API 연동"
    status: "future"
    note: "국세청 홈택스 API 또는 세무 SaaS 연동 고려"
```

---

## 8. 법적 참조 및 규제 근거

```yaml
regulatory_reference:
  tax_law:
    - name: "소득세법 제94조"
      scope: "양도소득 과세 대상 자산 정의"
    - name: "소득세법 시행령 제157조"
      scope: "대주주 요건 (지분율·시가총액 기준)"
    - name: "증권거래세법 제8조"
      scope: "증권거래세 세율"
    - name: "조세특례제한법 제91조의18"
      scope: "ISA 비과세·분리과세 특례"
    - name: "소득세법 제20조의3"
      scope: "연금소득 과세"

  financial_regulation:
    - name: "자본시장법 제9조 제4항"
      scope: "투자권유 정의"
    - name: "자본시장법 제47조"
      scope: "투자광고 규제"
    - name: "금융소비자보호법 제17조"
      scope: "적합성 원칙"
    - name: "금융소비자보호법 제18조"
      scope: "적정성 원칙"
    - name: "금융소비자보호법 제19조"
      scope: "설명의무"

  treaty:
    - name: "한미 조세조약 (제12조)"
      scope: "미국 배당소득 원천징수 15% 제한세율"

  dashboard_compliance:
    rule: "대시보드는 투자 조언이 아닌 정보 제공 목적"
    required_disclaimer: >
      본 대시보드의 세금·수수료 시뮬레이션은 참고용 추정치이며,
      정확한 세금 계산은 세무 전문가에게 문의하시기 바랍니다.
      자본시장법 및 금융소비자보호법에 따라 투자권유에 해당하지 않습니다.
```
