# Branch Convention

> Branch strategy and naming rules for solo development with Git Flow / GitHub Flow hybrid.

> **Trigger**: branch, 브랜치, branch naming, 브랜치 전략, feature branch, hotfix, develop

---

## 1. Branch Strategy (Git Flow + GitHub Flow Hybrid)

```yaml
branch_strategy:
  description: "솔로 개발에 적합한 간결한 하이브리드 전략"
```

### 1.1 Permanent Branches

```yaml
permanent_branches:
  main:
    description: "프로덕션 브랜치. 항상 안정적인 상태 유지."
    role: "production"
    protection:
      - "직접 push 가능 (솔로 개발이므로)"
      - "단, 대규모 변경은 PR 경유 권장"
    deploy: "main push → 자동 배포 (Streamlit Cloud 등)"

  develop:
    description: "개발 통합 브랜치. 다음 릴리즈 준비."
    role: "integration"
    usage: "optional — 프로젝트 규모가 커지면 도입"
    when_to_use:
      - "대회별 폴더에 기능이 많아질 때"
      - "여러 feature를 동시 개발할 때"
    when_to_skip:
      - "단순 문서 작업 위주일 때"
      - "feature → main 직접 머지로 충분할 때"
```

### 1.2 Temporary Branches

```yaml
temporary_branches:
  feature:
    description: "새로운 기능 개발용 브랜치"
    naming: "feature/<contest>/<description>"
    examples:
      - "feature/hackathon/dashboard-ui"
      - "feature/hackathon/parsing-preset"
      - "feature/new-contest/initial-setup"
    lifecycle: "생성 → 작업 → PR/머지 → 삭제"
    base: "main (또는 develop 사용 시 develop)"
    merge_to: "main (또는 develop)"

  fix:
    description: "버그 수정용 브랜치"
    naming: "fix/<contest>/<description>"
    examples:
      - "fix/hackathon/sharpe-division-zero"
      - "fix/hackathon/csv-encoding-error"
    lifecycle: "생성 → 수정 → PR/머지 → 삭제"
    base: "main"
    merge_to: "main"

  hotfix:
    description: "배포된 서비스의 긴급 수정"
    naming: "hotfix/<description>"
    examples:
      - "hotfix/deploy-config"
      - "hotfix/critical-data-loss"
    base: "main"
    merge_to: "main"
    urgency: "즉시 머지, PR 생략 가능"

  docs:
    description: "문서 전용 변경"
    naming: "docs/<description>"
    examples:
      - "docs/skills-refinement"
      - "docs/readme-update"
    base: "main"
    merge_to: "main"
    note: "소규모 문서 수정은 main 직접 커밋도 허용"

  release:
    description: "대회 제출 또는 릴리즈 준비"
    naming: "release/<version>"
    examples:
      - "release/v1.0"
      - "release/hackathon-submit"
    base: "develop (또는 main)"
    merge_to: "main + develop (양쪽)"
    when_to_use: "대회 제출 직전 최종 점검 시"
```

---

## 2. Branch Naming Rules

```yaml
branch_naming:
  description: "브랜치 이름 작성 규칙"
  format: "<type>/<contest>/<description>"
  rules:
    - "영어 소문자 + 하이픈(-) 사용"
    - "슬래시(/)로 계층 구분"
    - "설명은 2~4단어, 간결하게"
    - "한국어 금지 (git 호환성 이슈)"
    - "이슈 번호가 있으면 포함 가능: feature/hackathon/42-add-chart"
  forbidden:
    - "대문자 사용"
    - "공백, 언더스코어(_)"
    - "의미 없는 이름 (test, temp, asdf)"
```
