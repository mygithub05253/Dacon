# Merge & Pull Rules

> Rules for pulling latest changes, merging branches, rebase vs merge decisions, and conflict resolution.

> **Trigger**: pull, merge, 머지, rebase, 리베이스, conflict, 충돌, 충돌 해소, git pull, git merge

---

## 1. Pull Rules

```yaml
pull_rules:
  description: "작업 시작 전 항상 최신 상태 동기화"

  before_work:
    commands:
      - "git checkout main"
      - "git pull origin main"
      - "git checkout <working-branch>"
      - "git rebase main  # 또는 git merge main"
```

---

## 2. Rebase vs Merge

```yaml
rebase_vs_merge:
  description: "상황에 따른 rebase / merge 선택 기준"

  rebase:
    when: "feature 브랜치가 뒤처져 있고, 아직 push 안 했을 때"
    benefit: "깔끔한 히스토리"
    command: "git rebase main"

  merge:
    when: "이미 push한 브랜치이거나, 충돌이 복잡할 때"
    benefit: "안전함, 히스토리 보존"
    command: "git merge main"
```

---

## 3. Conflict Resolution

```yaml
conflict_resolution:
  description: "충돌 발생 시 해소 절차 및 원칙"

  steps:
    1: "충돌 파일 확인: git status"
    2: "각 파일의 충돌 마커(<<<< ==== >>>>) 해소"
    3: "해소 후 git add <파일>"
    4: "git rebase --continue 또는 git commit"

  principles:
    - "충돌 해소 시 양쪽 변경 의도를 모두 반영"
    - "잘 모르겠으면 충돌을 일으킨 커밋의 의도를 먼저 파악"
    - "해소 후 반드시 동작 확인"
```

---

## 4. Merge Rules

### 4.1 Feature to Main

```yaml
feature_to_main:
  description: "feature 브랜치를 main에 머지하는 규칙"
  method: "PR 경유 권장, 소규모는 직접 머지 허용"

  threshold:
    pr_required: "파일 5개 이상 변경 또는 로직 변경"
    direct_ok: "문서 수정, 설정 변경, 단일 파일 소규모 수정"

  steps_pr:
    1: "feature 브랜치에서 최종 커밋"
    2: "GitHub에서 PR 생성"
    3: "변경 사항 리뷰"
    4: "Squash and Merge"
    5: "feature 브랜치 삭제"

  steps_direct:
    1: "git checkout main"
    2: "git merge feature/<name>"
    3: "git push origin main"
    4: "git branch -d feature/<name>"
```

### 4.2 Hotfix to Main

```yaml
hotfix_to_main:
  description: "긴급 수정을 main에 직접 머지"
  method: "direct merge"
  steps:
    1: "git checkout main"
    2: "git merge hotfix/<name>"
    3: "git push origin main"
    4: "git branch -d hotfix/<name>"
```

### 4.3 Prohibited Merges

```yaml
no_merge_rules:
  description: "금지된 머지 패턴"
  rules:
    - "main에서 feature로 머지하지 않는다 (rebase 사용)"
    - "feature 브랜치끼리 직접 머지하지 않는다"
    - "충돌이 있는 상태로 머지하지 않는다"
```
