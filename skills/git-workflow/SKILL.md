# Git Workflow Skill — Git 워크플로우 규칙

> 이 스킬은 Claude Code에서 git 작업(commit, push, pull, merge, branch, PR)을 수행할 때 참조하는 규칙입니다.
> 사용자가 git 관련 요청을 하면 이 문서의 규칙을 따��� 실행합니다.
>
> **트리거**: commit, push, pull, merge, branch, PR, 커밋, 푸시, 브랜치, 머지 등 git 관련 키워드

---

## 1. 커밋 규칙 (Commit Convention)

### 1.1 커밋 메시지 포맷

```
<type>(<scope>): <subject>

<body>

<footer>
```

**규칙:**
- `type`과 `scope`는 영어 소문자
- `subject`(제목)는 한국어 또는 영어 (50자 이내)
- `body`(본문)는 한국어 자유 작성 (72자에서 줄바꿈)
- `footer`는 이슈/PR 참조용 (선택)

### 1.2 타입 (Type)

```yaml
types:
  feat:     "새로운 기능 추가"
  fix:      "버그 수정"
  docs:     "문서 변경 (README, SKILL.md, 기획서 등)"
  style:    "코드 포맷팅, 세미콜론 누락 등 (동작 변경 없음)"
  refactor: "리팩토링 (기능 변경 없이 코드 구조 개선)"
  test:     "테스트 추가 또는 수정"
  chore:    "빌드, 설정, 패키지 등 기타 변경"
  perf:     "성능 개선"
  ci:       "CI/CD 파이프��인 변경"
  design:   "UI/UX, 디자인 관련 변경"
  data:     "데이터 파일, 샘플 데이터 추가/변경"
  init:     "프로젝트 초기 설정"
```

### 1.3 스코프 (Scope)

```yaml
scopes:
  description: "변경 영역을 괄호 안에 표기. 선택 사항이지만 권장."
  examples:
    - "feat(parsing): 키움증권 CSV 프리셋 추가"
    - "fix(analysis): 샤프비율 0 나누기 에러 처리"
    - "docs(skills): visualization_rules.md 차트 규칙 보강"
    - "chore(deploy): Streamlit Cloud 배포 설정"
  project_scopes:
    - parsing      # 데이터 파싱 관련
    - analysis     # 분석/지�� 관련
    - viz          # 시각화 관련
    - insight      # 인사이트 관��
    - report       # 리포트 관련
    - dashboard    # 대시보드 UI 관련
    - skills       # Skills.md 문서 관련
    - deploy       # 배포 관련
    - data         # 데이터 파일 관련
    - git          # git 워크플로우 관련
```

### 1.4 제목 작성 규칙

```yaml
subject_rules:
  language: "한국어 우선, 영어도 허용"
  max_length: 50
  style:
    - "명령형/서술형으로 작성 ('추가', '수정', '개선', '제거')"
    - "마침표 붙이지 않음"
    - "첫 글자 대문자 불필요 (한국어이므로)"
  good_examples:
    - "feat(parsing): 증권사 자동 감지 로직 추가"
    - "fix(analysis): MDD 계산 시 빈 데이터 예외 처리"
    - "docs: 루트 README.md 대회 목록 업데이트"
    - "refactor(viz): 차트 색상 팔레트 상수 분리"
    - "chore: .gitignore에 __pycache__ 추가"
  bad_examples:
    - "feat: 수정함."                    # 모호 + 마침���
    - "업데이트"                          # type 누락, 내용 불명확
    - "feat(parsing): Add new feature"    # 제목이 모���
    - "fix: 여러 가지 버그 수정"          # 한 커밋에 여러 변경 금지
```

### 1.5 본문 작성 규칙

```yaml
body_rules:
  language: "한국어"
  when_required:
    - "변경 이유가 제목만으로 불명확할 때"
    - "breaking change가 있을 때"
    - "여러 파일을 변경했을 때"
    - "복잡한 로직 변경일 때"
  when_skip:
    - "단순 오타 수정"
    - "한 줄 변경"
    - "docs 타입의 단순 문서 수정"
  format:
    - "제목과 빈 줄로 구분"
    - "'무엇을'보다 '왜' 변경했는지에 집중"
    - "72자에서 줄바꿈"
  example: |
    feat(insight): 환율 리스크 인사이트 규칙 추가

    해외 자산 비중 20~80% 구간에서 환율 변동 영향을
    사용자에게 알려주는 인사이트가 없었음.

    - currency_risk 규칙 추가 (US weight 20~80%)
    - insight_rules.md 분산 투자 섹션에 반영
```

### 1.6 푸터 규칙

```yaml
footer_rules:
  issue_reference:
    format: "Closes #123, Refs #456"
    when: "GitHub Issue와 연결된 작업일 때"
  breaking_change:
    format: "BREAKING CHANGE: 설명"
    when: "하위 호환이 깨지는 변경일 때"
  co_author:
    format: "Co-authored-by: Name <email>"
    when: "협업 커밋일 때"
```

### 1.7 커밋 단위 규���

```yaml
commit_granularity:
  principle: "하나의 커밋 = 하나의 논리적 변경"
  rules:
    - "기능 추가와 버그 수정을 같은 커밋에 섞지 않는다"
    - "코드 변경과 포맷팅 변경을 분리��다"
    - "관련 파일은 하나의 커밋에 묶는다 (예: 규칙 변경 + 해당 테스트)"
    - "WIP(작업 중) 커밋은 push 전에 squash한다"
  size_guideline:
    small: "1~5개 파일, 50줄 이내 변경 — 이상적"
    medium: "5~15개 파일, 200줄 이내 — 괜찮음"
    large: "15개+ 파일 또는 200줄+ — 분할 검토"
    exception: "초기 설정, 대규모 리팩토링은 큰 커밋 허용"
```

---

## 2. 브랜치 ��칙 (Branch Convention)

### 2.1 브랜치 전략 (Git Flow + GitHub Flow 하이브리드)

```yaml
branch_strategy:
  description: "솔로 개발에 적합한 간결한 하이브리드 전략"

  permanent_branches:
    main:
      role: "프로덕션 브랜치. 항상 안정적인 상태 유지."
      protection:
        - "직접 push 가능 (솔로 개발이므로)"
        - "단, 대규모 변경은 PR 경유 권장"
      deploy: "main push → 자동 배포 (Streamlit Cloud 등)"

    develop:
      role: "개발 통합 브랜치. 다음 릴리즈 준비."
      usage: "선택적 — 프로젝트 규모가 커지면 도입"
      when_to_use:
        - "대회별 폴더에 기능이 많아질 때"
        - "여러 feature를 동시 개발할 때"
      when_to_skip:
        - "단순 문서 작업 위주일 때"
        - "feature → main 직접 머지로 충분할 때"

  temporary_branches:
    feature:
      naming: "feature/<대회>/<기능>"
      examples:
        - "feature/hackathon/dashboard-ui"
        - "feature/hackathon/parsing-preset"
        - "feature/new-contest/initial-setup"
      lifecycle: "생성 ��� 작업 → PR/머지 → 삭제"
      base: "main (또는 develop 사용 시 develop)"
      merge_to: "main (또는 develop)"

    fix:
      naming: "fix/<대회>/<설명>"
      examples:
        - "fix/hackathon/sharpe-division-zero"
        - "fix/hackathon/csv-encoding-error"
      lifecycle: "생성 → 수정 → PR/머지 → 삭제"
      base: "main"
      merge_to: "main"

    hotfix:
      naming: "hotfix/<설명>"
      examples:
        - "hotfix/deploy-config"
        - "hotfix/critical-data-loss"
      description: "배포된 서비스의 긴급 수정"
      base: "main"
      merge_to: "main"
      urgency: "즉시 머지, PR 생��� 가능"

    docs:
      naming: "docs/<설명>"
      examples:
        - "docs/skills-refinement"
        - "docs/readme-update"
      description: "문서 전용 변경"
      base: "main"
      merge_to: "main"
      note: "소규모 문서 수정은 main 직접 커밋도 허용"

    release:
      naming: "release/<버전>"
      examples:
        - "release/v1.0"
        - "release/hackathon-submit"
      description: "대회 제출 또는 ���리즈 준비"
      base: "develop (또는 main)"
      merge_to: "main + develop (양쪽)"
      when_to_use: "대회 제출 직전 최종 점검 시"
```

### 2.2 브랜치 네이밍 규칙

```yaml
branch_naming:
  format: "<type>/<contest>/<description>"
  rules:
    - "영어 소문자 + 하이픈(-) 사용"
    - "슬래시(/)로 ��층 구분"
    - "설명은 2~4단어, 간결하게"
    - "한국어 금지 (git 호환성 이슈)"
    - "이슈 번호가 있으면 포함 가능: feature/hackathon/42-add-chart"
  forbidden:
    - "대문자 사용"
    - "공백, 언더스코어(_)"
    - "의미 ���는 이름 (test, temp, asdf)"
```

---

## 3. PR ���칙 (Pull Request Convention)

### 3.1 PR 생성 규칙

```yaml
pr_creation:
  title_format: "[<type>] <한국어 설명>"
  title_examples:
    - "[feat] 투자 대시보드 섹터별 차트 추가"
    - "[fix] CSV 파싱 시 EUC-KR 인코딩 오류 수정"
    - "[docs] Skills.md 문서 ��� 교��� 참조 정합성 개선"

  body_template: |
    ## 변경 사���
    <!-- 무엇을 변경했는지 -->

    ## 변경 이유
    <!-- 왜 변경했는지 -->

    ## 테스트
    <!-- 어떻게 확인했는지 -->
    - [ ] 로컬 실행 확인
    - [ ] 기존 기능 영향 없음 확인

    ## 관련 이슈
    <!-- Closes #이슈번호 -->

  rules:
    - "하나의 PR = 하나의 목적 (기능, 버그 수정, 문서 등)"
    - "PR 크기는 가능한 작게 유지 (300줄 이내 권장)"
    - "WIP PR은 제목에 [WIP] 접두사 추가 — Draft PR 활용"
    - "스크린샷이 도움되면 첨부 (UI 변경 시 ��수)"

  labels:
    - "feat / fix / docs / refactor / chore"
    - "대회명 (hackathon, etc.)"
    - "priority: high / medium / low"
```

### 3.2 PR 머지 규칙

```yaml
pr_merge:
  strategy:
    default: "Squash and Merge"
    reason: "feature 브랜치의 ���다한 커밋을 하나로 정리"
    exceptions:
      - condition: "커밋 히스토리를 보존해야 할 때 (릴리즈 등)"
        use: "Merge Commit"
      - condition: "커밋이 이미 깔끔하게 정리���어 있을 때"
        use: "Rebase and Merge"

  checklist:
    before_merge:
      - "충돌 없�� 확인"
      - "변경 사항 최종 리뷰"
      - "테스트 통과 확인 (CI 있으면)"
    after_merge:
      - "원격 feature 브랜치 삭제"
      - "로컬 feature 브랜치 삭제"
      - "main 브랜치로 전환"
      - "main pull 받기"
```

---

## 4. Pull 규칙

```yaml
pull_rules:
  before_work:
    description: "작업 시작 전 항상 ��신 상태 동기화"
    commands:
      - "git checkout main"
      - "git pull origin main"
      - "git checkout <작업브랜치>"
      - "git rebase main  # 또는 git merge main"

  rebase_vs_merge:
    rebase:
      when: "feature 브랜치가 뒤처져 있고, 아직 push 안 했을 때"
      benefit: "깔끔한 히스토리"
      command: "git rebase main"
    merge:
      when: "이미 push한 브랜치이거나, 충돌이 복잡할 때"
      benefit: "안전함, 히스토리 보존"
      command: "git merge main"

  conflict_resolution:
    steps:
      1: "충돌 파일 확인: git status"
      2: "각 파일의 충돌 마커(<<<< ==== >>>>) 해소"
      3: "해소 후 git add <파일>"
      4: "git rebase --continue 또는 git commit"
    principles:
      - "충돌 해소 시 양쪽 변경 ���도를 모두 반영"
      - "잘 모르��으면 충돌을 일으킨 커밋의 의도를 ���저 파악"
      - "해소 후 반드시 동작 확인"
```

---

## 5. Merge 규칙

```yaml
merge_rules:
  feature_to_main:
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

  hotfix_to_main:
    method: "직접 머지 (긴급성)"
    steps:
      1: "git checkout main"
      2: "git merge hotfix/<name>"
      3: "git push origin main"
      4: "git branch -d hotfix/<name>"

  no_merge_rules:
    - "main에서 feature로 머지하지 않는다 (rebase 사용)"
    - "feature 브랜치끼리 직접 머지하지 않는���"
    - "���돌이 있는 상태로 머지하지 않는다"
```

---

## 6. Push 규칙

```yaml
push_rules:
  before_push:
    checklist:
      - "git status로 의도하지 않은 파�� 포함 여부 확인"
      - "git diff --staged로 변경 내용 최종 확인"
      - "커밋 메시지가 규칙을 따르는지 확인"
      - "민감 정보(API 키, 비밀번호) ���함 여부 확인"

  force_push:
    policy: "원칙적 금지"
    exception:
      - "rebase 후 자신의 feature 브랜치에만 허용"
      - "main에는 절대 force push 금지"
    command: "git push --force-with-lease  # force보다 안전"

  sensitive_files:
    never_push:
      - "*.env, .env.*"
      - "API ��, 토큰, 비밀번호가 포함된 파일"
      - "credentials.json, secrets.yaml"
      - "개인정보가 포함된 데이터 파일"
    gitignore_required:
      - "__pycache__/"
      - "*.pyc"
      - ".env"
      - ".vscode/"
      - ".idea/"
      - "node_modules/"
      - "*.egg-info/"
      - ".streamlit/secrets.toml"
```

---

## 7. .gitignore 기본 템플릿

```yaml
gitignore_template:
  description: "새 대회 폴더 생성 시 적용할 .gitignore 항목"
  content: |
    # Python
    __pycache__/
    *.py[cod]
    *$py.class
    *.egg-info/
    dist/
    build/
    .eggs/

    # 환경
    .env
    .env.*
    .venv/
    venv/

    # IDE
    .vscode/
    .idea/
    *.swp
    *.swo
    *~

    # OS
    .DS_Store
    Thumbs.db

    # Streamlit
    .streamlit/secrets.toml

    # 데이터 (대용량)
    *.csv
    !data/sample_*.csv
    *.xlsx
    !data/sample_*.xlsx

    # 임시
    tmp/
    temp/
    *.log
```

---

## 8. 실행 명령어 레퍼런스

> Claude Code에서 git 작업 요청 시 아래 ���령어를 조합하�� 실행합니다.

### 8.1 새 기능 개발 시작

```bash
git checkout main
git pull origin main
git checkout -b feature/<contest>/<description>
# 작업 수행
git add -A
git commit -m "<type>(<scope>): <subject>"
git push -u origin feature/<contest>/<description>
# GitHub에서 PR 생성
```

### 8.2 빠른 문서 수정 (main 직접)

```bash
git checkout main
git pull origin main
# 문서 수정
git add -A
git commit -m "docs(<scope>): <subject>"
git push origin main
```

### 8.3 머지 후 정리

```bash
git checkout main
git pull origin main
git branch -d feature/<contest>/<description>           # 로컬 삭제
git push origin --delete feature/<contest>/<description> # 원격 삭제
```

### 8.4 현재 상��� 점검

```bash
git status                    # 변경 파일 확인
git log --oneline -10         # 최근 커밋 10개
git branch -a                 # 모든 브랜치
git remote -v                 # 원격 저장소 확인
git diff --stat               # 변경 통계
```

---

## 9. 대회별 폴더 관리 규칙

```yaml
contest_folder_rules:
  description: "새 대회 참여 시 폴더 구조 및 git 작업 규칙"

  new_contest_setup:
    steps:
      1: "main 브랜치에서 시작"
      2: "feature/<대회명>/initial-setup 브랜치 생성"
      3: "대회 폴더 생성 (영어 케밥 케이스)"
      4: "폴더 내 README.md 작성 (대회 개요)"
      5: "필요한 하위 폴더 생성 (docs/, skills/, src/, data/)"
      6: "루트 README.md의 대회 목록 테이블에 추가"
      7: "커밋 후 main�� 머지"

  folder_naming:
    format: "<대회-주제-키워드>"
    examples:
      - "investment-dashboard-hackathon"
      - "nlp-sentiment-analysis"
      - "image-classification-challenge"
    rules:
      - "영어 소문자 + ���이픈"
      - "대회 내용을 유추할 수 있는 이름"
      - "너무 길지 않게 (3~5 단어)"

  folder_structure_template: |
    <contest-name>/
    ├── README.md          # 대회 개요, 평가 기준, 제출물 설명
    ├── docs/              # 기획서, 설계 문서
    ├── skills/            # Skills.md 규칙 문서 (해당 시)
    ├── src/               # 소스 코드
    ├── data/              # 샘플 데이터 (대용량은 .gitignore)
    └── assets/            # 이미지, 리소스

  root_readme_update:
    description: "새 대회 추가 시 루트 README.md의 테이블에 행 추가"
    format: "| 대회명 | 기간 | 주제 | 폴더 링크 |"
```

---

## 10. 트러블슈팅 가이드

```yaml
troubleshooting:
  common_issues:
    push_rejected:
      symptom: "git push 시 rejected 에러"
      cause: "원격에 로컬에 없는 커밋이 있음"
      solution:
        - "git pull --rebase origin main"
        - "충돌 해소 후 git push"
      warning: "force push는 최후의 수단"

    merge_conflict:
      symptom: "merge 또는 rebase 시 CONFLICT 표시"
      solution:
        - "git status로 충돌 파일 확인"
        - "파일 열어서 <<<< ==== >>>> 마커 해소"
        - "git add <해소된 파일>"
        - "git rebase --continue 또는 git merge --continue"

    accidental_commit_to_main:
      symptom: "main에 직접 커밋했는데 feature 브랜치에서 했어야 했음"
      solution:
        - "git branch feature/<name>  # 현재 커밋에서 브랜치 생성"
        - "git checkout main"
        - "git reset --hard HEAD~1  # main을 1커밋 되돌림"
        - "git checkout feature/<name>  # feature로 이동"
      warning: "이미 push했으면 이 방법 사용 금지"

    forgot_to_pull:
      symptom: "작업 후 push 시 충돌 발생"
      solution:
        - "git stash  # 현재 변경 임시 저장"
        - "git pull origin main"
        - "git stash pop  # 변경 복원"
        - "충돌 있으면 해소 후 커밋"

    sensitive_data_pushed:
      symptom: "API 키나 비밀번호가 포함된 파일을 push함"
      solution:
        critical: "즉시 해당 키/토큰을 폐기하고 새로 발급"
        cleanup:
          - "git rm --cached <파일>"
          - ".gitignore에 추가"
          - "git commit -m 'chore: 민감 정보 파일 제거'"
          - "git push"
        note: "git 히스토리에 남아있으므로, 키 자체를 반드시 교체해야 함"

    large_file_error:
      symptom: "100MB 초과 파일 push 거부"
      solution:
        - ".gitignore에 해당 파일 패턴 ���가"
        - "git rm --cached <대용량파일>"
        - "git commit -m 'chore: 대용량 파일 추적 제외'"
        - "필요하면 Git LFS 설정"
```
