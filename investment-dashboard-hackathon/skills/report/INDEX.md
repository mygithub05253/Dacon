# Report Skill — INDEX

> **Pipeline**: Stage 4 (final output layer)
> **Input**: Charts from `viz/*` + Insights from `insight/*` + Metrics from `analysis/*`
> **Output**: Dashboard (Streamlit) / PDF report / CSV data
> **Upstream**: `parsing/*` -> `analysis/*` -> `viz/*` + `insight/*` -> this skill

---

## Sub-files

| File | Description |
|------|-------------|
| [report-flow.md](report-flow.md) | 리포트 흐름 — 스토리 구조, 섹션 순서, 모드별 섹션 맵, 렌더링 실패 폴백 |
| [section-specs.md](section-specs.md) | 섹션별 상세 규칙 — header, KPI, 인사이트, 구성, 수익률, 리스크, 마켓 비교, 종목 테이블, 푸터 |
| [export-rules.md](export-rules.md) | 내보내기 — 대시보드 뷰, PDF, CSV, 반응형 레이아웃, 접근성 |
| [error-handling.md](error-handling.md) | 에러 상태 — 파이프라인 단계별 에러 매핑, 단계적 품질 저하 (level 1-5) |

## Processing Order

1. **Assemble** sections based on analysis mode (report-flow.md)
2. **Render** each section with specs and fallbacks (section-specs.md)
3. **Export** to dashboard / PDF / CSV (export-rules.md)
4. **Handle** errors gracefully at every stage (error-handling.md)
