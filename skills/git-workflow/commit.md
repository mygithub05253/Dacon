# Commit Convention

> Rules for writing commit messages: format, types, scope, subject, body, footer, and granularity.

> **Trigger**: commit, 커밋, commit message, 커밋 메시지, type, scope, subject, body, footer

---

## 1. Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

```yaml
format_rules:
  description: "커밋 메시지 포맷 규칙"
  type_and_scope: "English lowercase"
  subject: "Korean or English, max 50 chars"
  body: "Korean, wrap at 72 chars"
  footer: "optional, for issue/PR references"
```

---

## 2. Types

```yaml
types:
  description: "커밋 타입 목록 — 변경 성격을 나타내는 접두사"
  feat:     "새로운 기능 추가"
  fix:      "버그 수정"
  docs:     "문서 변경 (README, SKILL.md, 기획서 등)"
  style:    "코드 포맷팅, 세미콜론 누락 등 (동작 변경 없음)"
  refactor: "리팩토링 (기능 변경 없이 코드 구조 개선)"
  test:     "테스트 추가 또는 수정"
  chore:    "빌드, 설정, 패키지 등 기타 변경"
  perf:     "성능 개선"
  ci:       "CI/CD 파이프라인 변경"
  design:   "UI/UX, 디자인 관련 변경"
  data:     "데이터 파일, 샘플 데이터 추가/변경"
  init:     "프로젝트 초기 설정"
```

---

## 3. Scope

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
    - analysis     # 분석/지표 관련
    - viz          # 시각화 관련
    - insight      # 인사이트 관련
    - report       # 리포트 관련
    - dashboard    # 대시보드 UI 관련
    - skills       # Skills.md 문서 관련
    - deploy       # 배포 관련
    - data         # 데이터 파일 관련
    - git          # git 워크플로우 관련
```

---

## 4. Subject Rules

```yaml
subject_rules:
  description: "커밋 제목 작성 규칙"
  language: "Korean preferred, English allowed"
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
    - "feat: 수정함."                    # 모호 + 마침표
    - "업데이트"                          # type 누락, 내용 불명확
    - "feat(parsing): Add new feature"    # 제목이 모호
    - "fix: 여러 가지 버그 수정"          # 한 커밋에 여러 변경 금지
```

---

## 5. Body Rules

```yaml
body_rules:
  description: "커밋 본문 작성 규칙 — '왜' 변경했는지에 집중"
  language: "Korean"
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

---

## 6. Footer Rules

```yaml
footer_rules:
  description: "커밋 푸터 규칙 — 이슈 참조, breaking change, 공동 작업자"
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

---

## 7. Commit Granularity

```yaml
commit_granularity:
  description: "하나의 커밋 = 하나의 논리적 변경"
  rules:
    - "기능 추가와 버그 수정을 같은 커밋에 섞지 않는다"
    - "코드 변경과 포맷팅 변경을 분리한다"
    - "관련 파일은 하나의 커밋에 묶는다 (예: 규칙 변경 + 해당 테스트)"
    - "WIP(작업 중) 커밋은 push 전에 squash한다"
  size_guideline:
    small: "1~5개 파일, 50줄 이내 변경 — 이상적"
    medium: "5~15개 파일, 200줄 이내 — 괜찮음"
    large: "15개+ 파일 또는 200줄+ — 분할 검토"
    exception: "초기 설정, 대규모 리팩토링은 큰 커밋 허용"
```
