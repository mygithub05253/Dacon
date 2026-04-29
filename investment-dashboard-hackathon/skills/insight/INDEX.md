# Insight Skill — INDEX

> **Pipeline**: Stage 3-B (parallel with viz/*)
> **Input**: Metrics from `analysis/*` (return_rate, weight, mdd, sharpe_ratio, etc.)
> **Output**: Insight card list (type, severity, template message)
> **Downstream**: `report/*` places insight cards into the report layout.

---

## Sub-files

| File | Description |
|------|-------------|
| [pattern-rules.md](pattern-rules.md) | 인사이트 유형 정의, 패턴 감지 규칙 (집중도, 수익률, 분산, 고급 지표, 특수 상황), 충돌/중복 해소 |
| [display-rules.md](display-rules.md) | 우선순위/정렬, 최대 표시 수, 카드 포맷/스타일, 요약 생성, 확장 포인트 (AI 보강 포함) |

## Processing Order

1. **Detect** patterns from analysis metrics (pattern-rules.md)
2. **Resolve** conflicts and deduplicate (pattern-rules.md section 3)
3. **Sort & Display** with card format and summary (display-rules.md)
