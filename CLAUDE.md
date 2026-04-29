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
├── parsing/     → file-detection, column-mapping, normalization, broker-presets
├── analysis/    → basic-metrics, portfolio-metrics, advanced-metrics, benchmarks
├── viz/         → chart-selection, chart-style, layout
├── insight/     → pattern-rules, display-rules
└── report/      → report-flow, section-specs, export-rules, error-handling
```

각 폴더의 `INDEX.md`에서 파이프라인 위치와 파일 목록을 확인할 수 있습니다.

## 레포 정보

- **원격**: https://github.com/mygithub05253/Dacon.git
- **기본 브랜치**: main
- **소유자**: 이동원 (kik328288@gmail.com)
