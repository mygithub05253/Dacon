# Dacon 프로젝트 설정

## Git 워크플로우

이 레포지토리의 git 작업은 아래 세분화된 스킬 문서의 규칙을 따릅니다:

→ **[skills/git-workflow/INDEX.md](skills/git-workflow/INDEX.md)** (목차)

| 작업 | 참조 파일 |
|------|-----------|
| 커밋 | `skills/git-workflow/commit.md` |
| 브랜치 | `skills/git-workflow/branch.md` |
| PR | `skills/git-workflow/pr.md` |
| 머지/풀 | `skills/git-workflow/merge-pull.md` |
| 레포 설정/트러블슈팅 | `skills/git-workflow/repo-setup.md` |

git 관련 요청 시 해당 작업에 맞는 파일만 참조하세요. 전체가 필요하면 INDEX.md를 참조합니다.

## 프로젝트 구조

- 각 대회는 독립 폴더로 관리 (예: `investment-dashboard-hackathon/`)
- 새 대회 추가 시 `skills/git-workflow/repo-setup.md`의 Contest Folder 규칙을 따름
- 루트 `README.md`의 대회 목록 테이블을 항상 최신 상태로 유지

## 해커톤 Skills.md 구조

대회별 스킬은 세분화되어 관리됩니다:

```
investment-dashboard-hackathon/skills/
├── parsing/     → file-detection, column-mapping, normalization, broker-presets, data-integrity-check
├── analysis/    → basic-metrics, portfolio-metrics, advanced-metrics, benchmarks, currency-rules, tax-fee-impact, benchmark-selection, correlation-diversification, income-analysis, turnover-holding
├── viz/         → chart-selection, chart-style, layout
├── insight/     → pattern-rules, display-rules, safety-disclaimer
└── report/      → report-flow, section-specs, export-rules, error-handling
```

```
smart-factory-hackathon/skills/
├── data-pipeline/ → dataset-specs, sensor-profiling, preprocessing, feature-engineering, class-balance, data-leakage, streaming-sim
├── modeling/      → train-pipeline, evaluation, threshold-engine, model-ops
├── xai/           → shap-analysis, action-guide, alert-rules
├── ui/            → page-structure, dashboard-components, chart-style, interaction, ux-manufacturing
└── quality/       → testing, demo-prep, deployment, scoring-strategy
```

각 폴더의 `INDEX.md`에서 파이프라인 위치와 파일 목록을 확인할 수 있습니다.

## 하이브리드 모델 전략

작업 복잡도에 따라 모델을 다르게 배정합니다:

| 복잡도 | 모델 | 기준 | 예시 |
|--------|------|------|------|
| **높음** | Opus 4.6 | 멀티스텝 설계, 깊은 분석, 크로스레퍼런스 검증, 대량 코드 생성 | 대시보드 전체 구현, 스킬 설계/리팩터링, 포트폴리오 분석 로직 |
| **중간** | Sonnet | 단일 파일 수정, 정해진 규칙 적용, 템플릿 기반 작업 | 커밋 메시지 작성, 차트 스타일 적용, 단일 스킬 파일 수정 |
| **낮음** | Haiku | 단순 검색, 파일 탐색, 포맷 확인 | 파일 찾기, 변수명 grep, 구조 확인 |

### 적용 규칙

- Agent spawn 시 `model` 파라미터로 지정: `opus`, `sonnet`, `haiku`
- 각 스킬 INDEX.md의 `recommended_model` 필드를 참고
- 판단이 어려우면 **Sonnet으로 시작**, 복잡도가 높아지면 Opus로 전환
- 병렬 에이전트 실행 시: 메인 분석 = Opus, 보조 탐색 = Sonnet/Haiku

## 레포 정보

- **원격**: https://github.com/mygithub05253/Dacon.git
- **기본 브랜치**: main
- **소유자**: 이동원 (kik328288@gmail.com)
