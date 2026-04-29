# 월간 해커톤 — 투자 데이터를 시각화하라

> Skills.md 기반 투자 대시보드 설계 대회

---

## 대회 개요

| 항목 | 내용 |
|------|------|
| 대회명 | 월간 해커톤 : 투자 데이터를 시각화하라 |
| 플랫폼 | [Dacon](https://dacon.io/) |
| 기간 | 2025년 4월 |
| 핵심 과제 | Skills.md 기반으로 투자 데이터 분석·시각화 규칙을 설계하고, 이를 기반으로 대시보드를 자동 생성하는 시스템 구축 |

---

## 평가 기준

| 항목 | 배점 | 핵심 포인트 |
|------|------|-------------|
| **범용성** | 25점 | 다양한 투자 데이터 구조에 대응, 재사용 가능성 |
| **Skills.md 설계** | 25점 | 분석 규칙 명확성, 시각화 기준, 인사이트 생성 규칙의 체계성 |
| **대시보드 자동 생성** | 25점 | 자동 분석, 시각화 적절성, UI 완성도 |
| **바이브코딩 활용** | 15점 | Skills.md → 코드 자동 생성 구조, AI 친화적 문서 설계 |
| **실용성 및 창의성** | 10점 | 실제 활용 가능성, 확장 기능, UX |

---

## 제출물

1. **기획서 (PDF)** — 서비스 개요, 분석 흐름, 대시보드 구성, Skills.md 설계 방향, 확장 기능
2. **Skills.md (5종, PDF 변환 제출)** — 분석/시각화/인사이트/리포트 규칙 문서
3. **웹 페이지 (배포 URL)** — Skills.md 기반 바이브코딩으로 구현한 Streamlit 대시보드

---

## 서비스 컨셉 — AlphaFolio

**"투자 데이터를 올리면, AI가 분석하고 대시보드를 자동으로 만들어주는 서비스"**

핵심 차별화: 분석 로직을 코드가 아닌 **Skills.md 문서**에 선언적으로 정의하여, 문서만 교체하면 다른 자산군(채권, 암호화폐 등)에도 대응 가능한 구조.

---

## 프로젝트 구조

```
investment-dashboard-hackathon/
├── README.md                        ← 이 파일
├── AlphaFolio_기획서_이동원.pdf     ← 제출용 기획서
├── docs/                            ← Phase별 설계 문서
│   ├── phase1_service_concept.md    ← 서비스 컨셉 정의
│   ├── phase2_analysis_flow.md      ← 분석 파이프라인 흐름
│   ├── phase3_skills_design.md      ← Skills.md 설계 전략
│   ├── phase4_dashboard_design.md   ← 대시보드 UI 설계
│   └── phase5_vibecoding_extension.md ← 바이브코딩 & 확장 기능
├── skills/                          ← Skills.md 규칙 문서 (핵심)
│   ├── parsing_rules.md             ← 데이터 파싱 규칙
│   ├── analysis_rules.md            ← 투자 지표 분석 규칙
│   ├── visualization_rules.md       ← 시각화 규칙
│   ├── insight_rules.md             ← AI 인사이트 생성 규칙
│   └── report_rules.md              ← 리포트 구성 규칙
├── src/                             ← Streamlit 대시보드 소스코드
├── data/                            ← 샘플/더미 데이터
└── assets/                          ← 이미지, 아이콘 등 리소스
```

---

## Skills.md 파이프라인

Skills.md 5종은 아래와 같은 의존 관계로 동작합니다:

```
CSV 업로드
    ↓
parsing_rules.md   → 파일 인식, 컬럼 매핑, 데이터 정규화 (표준 스키마 생성)
    ↓
analysis_rules.md  → 수익률·비중·고급 지표(샤프·MDD·베타) 계산
    ↓
    ├── visualization_rules.md  → 조건 기반 차트 자동 선택 & 렌더링
    └── insight_rules.md        → 13개 패턴 감지 → 인사이트 카드 생성
            ↓
    report_rules.md  → 섹션 구성, 레이아웃, 내보내기(대시보드/PDF/CSV)
```

---

## 기술 스택

| 영역 | 기술 | 선택 이유 |
|------|------|-----------|
| 프론트/대시보드 | Streamlit | 빠른 프로토타이핑, Python 단일 스택 |
| 데이터 분석 | pandas, numpy | 투자 데이터 가공·지표 계산 표준 도구 |
| 시각화 | Plotly, Altair | 인터랙티브 차트, Streamlit 네이티브 지원 |
| 배포 | Streamlit Cloud | 무료 배포, URL 즉시 공유 가능 |

---

## 설계 3원칙

1. **선언적 (Declarative)** — Skills.md는 "무엇을" 정의하고, 코드가 "어떻게"를 처리
2. **교체 가능 (Swappable)** — 모든 문서에 `extension_points` 섹션 포함, 규칙 교체만으로 다른 자산군 대응
3. **바이브코딩 친화** — YAML 구조화 포맷으로 AI가 읽고 코드 생성 가능
