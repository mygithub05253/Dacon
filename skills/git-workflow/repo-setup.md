# Repository Setup & Maintenance

> Push rules, .gitignore template, contest folder management, and troubleshooting guide.

> **Trigger**: push, 푸시, .gitignore, force push, 대회 폴더, contest folder, troubleshoot, 트러블슈팅, sensitive data, 민감 정보

---

## 1. Push Rules

```yaml
push_rules:
  description: "push 전 확인 사항 및 force push 정책"

  before_push:
    checklist:
      - "git status로 의도하지 않은 파일 포함 여부 확인"
      - "git diff --staged로 변경 내용 최종 확인"
      - "커밋 메시지가 규칙을 따르는지 확인"
      - "민감 정보(API 키, 비밀번호) 포함 여부 확인"

  force_push:
    policy: "원칙적 금지"
    exception:
      - "rebase 후 자신의 feature 브랜치에만 허용"
      - "main에는 절대 force push 금지"
    command: "git push --force-with-lease  # force보다 안전"

  sensitive_files:
    never_push:
      - "*.env, .env.*"
      - "API 키, 토큰, 비밀번호가 포함된 파일"
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

## 2. .gitignore Template

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

    # Environment
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

    # Data (large files)
    *.csv
    !data/sample_*.csv
    *.xlsx
    !data/sample_*.xlsx

    # Temporary
    tmp/
    temp/
    *.log
```

---

## 3. Contest Folder Management

```yaml
contest_folder_rules:
  description: "새 대회 참여 시 폴더 구조 및 git 작업 규칙"

  new_contest_setup:
    steps:
      1: "main 브랜치에서 시작"
      2: "feature/<contest-name>/initial-setup 브랜치 생성"
      3: "대회 폴더 생성 (영어 케밥 케이스)"
      4: "폴더 내 README.md 작성 (대회 개요)"
      5: "필요한 하위 폴더 생성 (docs/, skills/, src/, data/)"
      6: "루트 README.md의 대회 목록 테이블에 추가"
      7: "커밋 후 main에 머지"

  folder_naming:
    format: "<contest-topic-keyword>"
    examples:
      - "investment-dashboard-hackathon"
      - "nlp-sentiment-analysis"
      - "image-classification-challenge"
    rules:
      - "영어 소문자 + 하이픈"
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

## 4. Troubleshooting Guide

### 4.1 Push Rejected

```yaml
push_rejected:
  description: "git push 시 rejected 에러"
  cause: "원격에 로컬에 없는 커밋이 있음"
  solution:
    - "git pull --rebase origin main"
    - "충돌 해소 후 git push"
  warning: "force push는 최후의 수단"
```

### 4.2 Merge Conflict

```yaml
merge_conflict:
  description: "merge 또는 rebase 시 CONFLICT 표시"
  solution:
    - "git status로 충돌 파일 확인"
    - "파일 열어서 <<<< ==== >>>> 마커 해소"
    - "git add <해소된 파일>"
    - "git rebase --continue 또는 git merge --continue"
```

### 4.3 Accidental Commit to Main

```yaml
accidental_commit_to_main:
  description: "main에 직접 커밋했는데 feature 브랜치에서 했어야 했음"
  solution:
    - "git branch feature/<name>  # 현재 커밋에서 브랜치 생성"
    - "git checkout main"
    - "git reset --hard HEAD~1  # main을 1커밋 되돌림"
    - "git checkout feature/<name>  # feature로 이동"
  warning: "이미 push했으면 이 방법 사용 금지"
```

### 4.4 Forgot to Pull

```yaml
forgot_to_pull:
  description: "작업 후 push 시 충돌 발생"
  solution:
    - "git stash  # 현재 변경 임시 저장"
    - "git pull origin main"
    - "git stash pop  # 변경 복원"
    - "충돌 있으면 해소 후 커밋"
```

### 4.5 Sensitive Data Pushed

```yaml
sensitive_data_pushed:
  description: "API 키나 비밀번호가 포함된 파일을 push함"
  solution:
    critical: "즉시 해당 키/토큰을 폐기하고 새로 발급"
    cleanup:
      - "git rm --cached <파일>"
      - ".gitignore에 추가"
      - "git commit -m 'chore: 민감 정보 파일 제거'"
      - "git push"
    note: "git 히스토리에 남아있으므로, 키 자체를 반드시 교체해야 함"
```

### 4.6 Large File Error

```yaml
large_file_error:
  description: "100MB 초과 파일 push 거부"
  solution:
    - ".gitignore에 해당 파일 패턴 추가"
    - "git rm --cached <대용량파일>"
    - "git commit -m 'chore: 대용량 파일 추적 제외'"
    - "필요하면 Git LFS 설정"
```
