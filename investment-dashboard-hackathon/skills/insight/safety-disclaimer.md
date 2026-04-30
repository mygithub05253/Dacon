# Safety & Disclaimer Rules
### 인사이트·리포트·AI 생성 콘텐츠에 적용되는 면책 및 안전 규칙

> **Pipeline**: Cross-cutting concern — applies to insight/*, report/*, and any user-facing text
>
> Referenced by: pattern-rules.md, display-rules.md, report/section-specs.md
> Enforcement: pre-render validation on all generated text

---

## 1. Mandatory Disclaimers

```yaml
mandatory_disclaimers:
  description: "모든 리포트·대시보드에 반드시 표시해야 하는 면책 문구"

  global_footer:
    description: "리포트 하단에 항상 표시"
    texts:
      - "본 리포트는 정보 제공 목적으로만 작성되었으며, 투자 권유가 아닙니다."
      - "과거 수익률이 미래 수익을 보장하지 않습니다."
      - "투자 판단의 책임은 투자자 본인에게 있습니다."
      - "세전 기준 수익률이며, 실제 수익은 세금 및 수수료에 따라 달라질 수 있습니다."
    style:
      font_size: "12px"
      color: "var(--text-secondary)"
      border_top: "1px solid var(--border-light)"
      padding_top: "16px"

  per_insight:
    description: "각 인사이트 카드 하단에 부착"
    text: "본 정보는 데이터 기반 참고 자료이며, 개인 투자 상황에 따라 다를 수 있습니다."
    style:
      font_size: "11px"
      color: "var(--text-muted)"

  ai_generated:
    description: "AI 보강 기능 활성화 시 표시"
    text: "AI가 생성한 분석이며, 전문 투자 자문을 대체하지 않습니다."
    condition: "ai_enhancement is active"
    icon: "🤖"
    style:
      background: "var(--bg-ai-badge)"
      border_radius: "4px"
      padding: "4px 8px"
```

---

## 2. Forbidden Expressions (금지 표현)

```yaml
forbidden_expressions:
  description: "대시보드 내 모든 텍스트에서 차단해야 하는 표현"

  direct_action_commands:
    description: "직접적 매매 지시 표현"
    patterns:
      - "매수하세요"
      - "매도하세요"
      - "매수 추천"
      - "매도 추천"
      - "반드시 ~해야 합니다"
      - "즉시 ~하세요"
      - "~할 것을 권합니다"
      - "~을 추천합니다"

  specific_product_recommendations:
    description: "특정 상품·증권사 지목 행위"
    patterns:
      - regex: "KODEX|TIGER|ARIRANG|KBSTAR.*매수|매도"
        note: "특정 ETF 티커 + 매매 동사 조합"
      - regex: "키움증권|삼성증권|미래에셋.*에서.*매수|매도|가입"
        note: "특정 증권사 + 행동 동사 조합"

  certainty_claims:
    description: "미래에 대한 확정적 표현"
    patterns:
      - "반드시 오를 것입니다"
      - "확실히 하락합니다"
      - "보장됩니다"
      - "확정적"
      - regex: "반드시.*될 것"
      - regex: "100%.*수익"
```

---

## 3. Allowed Expressions (허용 표현)

```yaml
allowed_expressions:
  description: "안전한 표현 가이드라인"

  neutral_observation:
    description: "중립적 관찰 표현"
    examples:
      - "~인 것으로 나타납니다"
      - "데이터에 따르면 ~"
      - "~한 경향이 있습니다"
      - "~할 수 있습니다"
      - "~로 분석됩니다"

  general_guidance:
    description: "일반적 안내 (특정 행동 지시 아님)"
    examples:
      - "포트폴리오 구성을 점검해보시기 바랍니다"
      - "분산투자 관점에서 검토가 필요할 수 있습니다"
      - "전문가와 상담을 권장합니다"
      - "추가적인 리서치를 고려해보실 수 있습니다"
```

---

## 4. AI Output Validation Pipeline

```yaml
ai_validation_pipeline:
  description: "AI 생성 텍스트가 사용자에게 표시되기 전 검증 단계"

  pre_check:
    action: "생성 텍스트에서 Section 2의 금지 표현 스캔"
    method: "regex matching + keyword lookup"
    scan_targets:
      - "insight card body"
      - "summary text"
      - "ai_enhancement output"

  on_violation:
    action: "전체 AI 출력을 fallback 템플릿으로 교체"
    fallback_template: "[{insight_type}]에 대한 상세 분석은 전문가와 상담하시기 바랍니다."
    log:
      enabled: true
      fields: ["timestamp", "original_text", "violated_pattern"]
      visibility: "debug only — 사용자에게 비노출"

  post_check:
    action: "disclaimer 부착 확인"
    ensure:
      - "per_insight disclaimer 존재"
      - "ai_generated badge 존재 (ai_enhancement 활성 시)"
```

---

## 5. Data Freshness Warnings

```yaml
data_freshness:
  description: "데이터 기준일에 따른 경고 표시"
  rules:
    - condition: "data_age > 1 day"
      severity: "info"
      message: "데이터 기준일: {date} (실시간이 아닙니다)"
      display: "리포트 상단 배지"

    - condition: "data_age > 7 days"
      severity: "warning"
      message: "⚠️ 주의: 데이터가 {days}일 전 기준입니다. 현재 시세와 다를 수 있습니다."
      display: "리포트 상단 경고 배너"
      style:
        background: "var(--bg-warning)"
        border: "1px solid var(--border-warning)"

    - condition: "exchange_rate_age > 24h"
      severity: "warning"
      message: "환율 기준: {rate_date} (최신 환율과 다를 수 있습니다)"
      display: "환율 관련 수치 옆에 표시"
```

---

## 6. Regulatory Reference

```yaml
regulatory_reference:
  description: "관련 법규 참조 및 서비스 성격 명시"

  applicable_laws:
    - name: "자본시장법 제47조"
      topic: "투자광고 제한"
      relevance: "리포트가 투자광고로 해석되지 않도록 면책 문구 필수"

    - name: "금융소비자보호법"
      topic: "적합성·적정성 원칙"
      relevance: "개인별 투자 성향을 고려한 맞춤 자문 불가 — 일반 정보만 제공"

  service_disclaimer:
    text: "이 대시보드는 투자자문업 등록 서비스가 아닙니다."
    display: "global_footer 내 또는 About 페이지"

  note: |
    위 법규는 참고용이며, 법적 자문을 구성하지 않습니다.
    실제 서비스 출시 시 법률 검토가 필요합니다.
```
