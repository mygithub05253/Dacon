# Pull Request Convention

> Rules for creating, reviewing, and merging pull requests.

> **Trigger**: PR, pull request, 풀 리퀘스트, merge PR, squash, PR 생성, PR 머지

---

## 1. PR Creation Rules

```yaml
pr_creation:
  description: "PR 생성 시 제목, 본문, 라벨 규칙"

  title_format: "[<type>] <Korean description>"
  title_examples:
    - "[feat] 투자 대시보드 섹터별 차트 추가"
    - "[fix] CSV 파싱 시 EUC-KR 인코딩 오류 수정"
    - "[docs] Skills.md 문서 간 교차 참조 정합성 개선"

  rules:
    - "하나의 PR = 하나의 목적 (기능, 버그 수정, 문서 등)"
    - "PR 크기는 가능한 작게 유지 (300줄 이내 권장)"
    - "WIP PR은 제목에 [WIP] 접두사 추가 — Draft PR 활용"
    - "스크린샷이 도움되면 첨부 (UI 변경 시 필수)"

  labels:
    - "feat / fix / docs / refactor / chore"
    - "대회명 (hackathon, etc.)"
    - "priority: high / medium / low"
```

---

## 2. PR Body Template

```yaml
body_template:
  description: "PR 본문 작성 시 사용할 템플릿"
  content: |
    ## 변경 사항
    <!-- 무엇을 변경했는지 -->

    ## 변경 이유
    <!-- 왜 변경했는지 -->

    ## 테스트
    <!-- 어떻게 확인했는지 -->
    - [ ] 로컬 실행 확인
    - [ ] 기존 기능 영향 없음 확인

    ## 관련 이슈
    <!-- Closes #이슈번호 -->
```

---

## 3. PR Merge Strategy

```yaml
pr_merge:
  description: "PR 머지 방식 및 체크리스트"

  strategy:
    default: "Squash and Merge"
    reason: "feature 브랜치의 잡다한 커밋을 하나로 정리"
    exceptions:
      - condition: "커밋 히스토리를 보존해야 할 때 (릴리즈 등)"
        use: "Merge Commit"
      - condition: "커밋이 이미 깔끔하게 정리되어 있을 때"
        use: "Rebase and Merge"
```

---

## 4. Merge Checklist

```yaml
merge_checklist:
  description: "머지 전후 확인 사항"

  before_merge:
    - "충돌 없음 확인"
    - "변경 사항 최종 리뷰"
    - "테스트 통과 확인 (CI 있으면)"

  after_merge:
    - "원격 feature 브랜치 삭제"
    - "로컬 feature 브랜치 삭제"
    - "main 브랜치로 전환"
    - "main pull 받기"
```
