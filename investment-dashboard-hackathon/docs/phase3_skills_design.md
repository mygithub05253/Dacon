# Phase 3. Skills 설계 전략

> **현재 상태**: 5개 모놀리식 파일 → **30개 세분화 스킬 파일** (5개 폴더 + INDEX.md)
> **파이프라인**: parsing → analysis → viz + insight → report (4-Stage)

---

## 설계 3원칙

### 1. 선언적 (Declarative)
Skills 파일은 **"무엇을"** 정의합니다. **"어떻게"**는 코드가 처리합니다.
- 예: `basic-metrics.md`에서 "수익률 = (현재가 - 매수가) / 매수가 x 100" 이라고 선언하면, 코드가 이를 읽고 계산 함수를 생성

### 2. 교체 가능 (Swappable)
개별 스킬 파일 단위로 교체하면 동일한 파이프라인으로 다른 자산군에 대응합니다.
- 예: `broker-presets.md`만 추가하면 새로운 증권사 포맷 즉시 지원
- 각 스킬 파일에 `extension_points` 섹션을 포함하여 확장 방법을 명시

### 3. 바이브코딩 친화 (Vibe-coding Friendly)
AI가 읽고 코드를 생성할 수 있는 구조화된 YAML 포맷을 사용합니다.
- **영어 구조 + 한국어 설명**: 필드명/키는 영어, 설명은 한국어 → AI 파싱 효율 극대화
- 규칙은 YAML 코드블록으로 정의 → AI가 파싱하여 Python 코드로 변환 가능
- 자연어 설명 + 구조화된 규칙이 공존하는 하이브리드 포맷

---

## 폴더 구조 (30 Files + 5 INDEX)

```
investment-dashboard-hackathon/skills/
├── parsing/          ← Stage 1: 데이터 인식·정규화
│   ├── INDEX.md
│   ├── file-detection.md        파일 형식 감지 (CSV/Excel/JSON, 인코딩, 구분자)
│   ├── column-mapping.md        컬럼명 → 표준 스키마 매핑 (22개 매핑 규칙)
│   ├── normalization.md         데이터 타입 정규화 (날짜, 통화, 수량)
│   ├── broker-presets.md        증권사별 프리셋 (키움, 미래에셋, NH, 삼성, 한투)
│   └── data-integrity-check.md  파싱 후 교차검증 게이트 (cross-validation)
│
├── analysis/         ← Stage 2: 지표 계산
│   ├── INDEX.md
│   ├── basic-metrics.md         기본 지표 (수익률, 손익, 평가금액, 비중)
│   ├── portfolio-metrics.md     포트폴리오 지표 (가중수익률, 샤프비율, MDD, 베타)
│   ├── advanced-metrics.md      고급 지표 (정보비율, 트레이너비율, 젠센알파 등)
│   ├── benchmarks.md            벤치마크 데이터 소스 및 비교 로직
│   ├── benchmark-selection.md   자동 벤치마크 선택 알고리즘 (국내/해외/혼합)
│   ├── currency-rules.md        환율 처리 규칙 (KRW/USD 혼합 포트폴리오)
│   ├── tax-fee-impact.md        세금·수수료 영향 분석 (검증된 한국 세율, 자본시장법/금소법)
│   ├── correlation-diversification.md  상관행렬, 분산비율, 분산효과 측정
│   ├── income-analysis.md       배당/이자 소득 vs 자본이득 분리 분석
│   └── turnover-holding.md      회전율, 보유기간 분석, 과잉매매(churning) 감지
│
├── viz/              ← Stage 3-A: 시각화
│   ├── INDEX.md
│   ├── chart-selection.md       조건→차트 자동 매핑 (9종 차트 유형)
│   ├── chart-style.md           색상 체계, 폰트, 반응형 스타일
│   └── layout.md                대시보드 레이아웃, 그리드 시스템, 인터랙션
│
├── insight/          ← Stage 3-B: 인사이트 생성
│   ├── INDEX.md
│   ├── pattern-rules.md         패턴 감지 규칙 (18개), 우선순위, 충돌 해소
│   ├── display-rules.md         인사이트 카드 렌더링, 심각도 표시, 요약 생성
│   └── safety-disclaimer.md     면책조항, 금지 표현, AI 출력 검증 필터
│
└── report/           ← Stage 4: 리포트 구성
    ├── INDEX.md
    ├── report-flow.md           리포트 생성 파이프라인, 섹션 순서
    ├── section-specs.md         섹션별 명세 (9개 섹션), 필수/선택 구분
    ├── export-rules.md          내보내기 포맷 (PDF, HTML, PNG)
    └── error-handling.md        에러 5단계 분류, 폴백 전략, 사용자 메시지
```

---

## 파이프라인 의존 관계

```
[Stage 1] parsing/ (5 files)
  file-detection → column-mapping → normalization → broker-presets
                                                          ↓
                                              data-integrity-check ← 교차검증 게이트
                                                          ↓
                                                   standard_schema 출력
                                                          ↓
[Stage 2] analysis/ (10 files)
  basic-metrics → portfolio-metrics → advanced-metrics
  benchmarks + benchmark-selection → 벤치마크 비교
  currency-rules → 환율 적용
  tax-fee-impact → 세후 수익률
  correlation-diversification → 분산 분석
  income-analysis → 소득 유형 분리
  turnover-holding → 매매 패턴 분석
                          ↓
           ┌──────────────┴──────────────┐
           ↓                             ↓
[Stage 3-A] viz/ (3 files)    [Stage 3-B] insight/ (3 files)
  chart-selection                pattern-rules
  chart-style                    display-rules
  layout                         safety-disclaimer
           ↓                             ↓
           └──────────────┬──────────────┘
                          ↓
[Stage 4] report/ (4 files)
  report-flow → section-specs → export-rules
                                error-handling (전 단계 적용)
```

**핵심**: 각 스킬 파일은 이전 단계의 출력 스키마만 참조합니다. `data-integrity-check.md`가 Stage 1의 교차검증 게이트 역할을 하여, 불완전한 데이터가 Stage 2로 넘어가는 것을 차단합니다.

---

## 핵심 특징

### 금융 안전성 (Financial Safety)
- **`safety-disclaimer.md`**: 면책조항 필수 삽입, 금지 표현 목록 ("보장", "확실한 수익" 등), AI 출력 검증 필터
- **`tax-fee-impact.md`**: 검증된 한국 세율 (양도소득세, 배당소득세, 증권거래세), 자본시장법/금소법 참조
- 모든 인사이트에 "투자 권유가 아님" 면책조항 자동 부착

### 전문가 수준 분석 (Professional-grade Analysis)
- **상관행렬 및 분산비율**: `correlation-diversification.md`에서 포트폴리오 분산 효과 정량화
- **소득 분석**: `income-analysis.md`에서 배당/이자 소득 vs 자본이득을 분리하여 세금 효율 분석
- **과잉매매 감지**: `turnover-holding.md`에서 회전율 기반 churning 여부 판단

### 데이터 무결성 (Data Integrity)
- **교차검증 게이트**: `data-integrity-check.md`가 파싱 완료 후 다음을 검증:
  - 필수 컬럼 존재 여부
  - 데이터 타입 일관성
  - 합계 금액 cross-check
  - 날짜 범위 유효성
- 검증 실패 시 Stage 2 진입을 차단하고 사용자에게 피드백 제공

### 규제 준수 (Regulatory Compliance)
- 한국 자본시장법 및 금융소비자보호법(금소법) 요구사항 반영
- 세율은 실제 한국 세법 기준으로 검증 (2024-2025 기준)
- 투자 조언/추천으로 오인될 수 있는 표현 자동 필터링

---

## 하이브리드 모델 전략

각 INDEX.md에 `recommended_model` 필드가 포함되어 있어 AI 모델을 작업 복잡도에 따라 배정합니다.

| 폴더 | recommended_model | 근거 |
|------|-------------------|------|
| `parsing/` | **sonnet** | 규칙 기반 매핑, 단일 파일 처리 |
| `analysis/` | **opus** | 멀티스텝 계산, 크로스레퍼런스 검증 필요 |
| `viz/` | **sonnet** | 템플릿 기반 차트 생성, 스타일 적용 |
| `insight/` | **opus** | 패턴 간 충돌 해소, 맥락 기반 판단 필요 |
| `report/` | **sonnet** | 정해진 섹션 구조 따라 조립 |

### 적용 규칙
- 메인 분석 파이프라인 = Opus, 보조 탐색/포맷팅 = Sonnet/Haiku
- 판단이 어려우면 Sonnet으로 시작, 복잡도가 높아지면 Opus로 전환
- 병렬 에이전트 실행 시 모델 혼합 사용 가능

---

## 모놀리식 → 세분화 마이그레이션 요약

| 기존 (삭제됨) | 현재 (30 files) | 변경 사항 |
|---------------|-----------------|-----------|
| `parsing_rules.md` (1 file) | `parsing/` (5 files + INDEX) | 파일 감지, 컬럼 매핑, 정규화, 프리셋, 무결성 검증으로 분리 |
| `analysis_rules.md` (1 file) | `analysis/` (10 files + INDEX) | 기본→고급→벤치마크→세금→상관→소득→회전율로 세분화 |
| `visualization_rules.md` (1 file) | `viz/` (3 files + INDEX) | 차트 선택, 스타일, 레이아웃 분리 |
| `insight_rules.md` (1 file) | `insight/` (3 files + INDEX) | 패턴 규칙, 표시 규칙, 안전 면책 분리 |
| `report_rules.md` (1 file) | `report/` (4 files + INDEX) | 플로우, 섹션 명세, 내보내기, 에러 처리 분리 |

**세분화 이유**: 개별 스킬 파일이 작고 독립적이므로 AI 컨텍스트 윈도우를 효율적으로 사용하고, 필요한 규칙만 로드하여 정확도를 높입니다.

---

## 평가 기준 대응

| 평가 항목 | Skills 대응 전략 |
|-----------|-----------------|
| **분석 규칙 명확성** | 30개 파일에 YAML 포맷으로 모든 규칙을 구조화, formula/condition 명시 |
| **시각화 기준** | `chart-selection.md`에 조건→차트 매핑을 우선순위와 함께 정의 |
| **인사이트 생성 규칙** | `pattern-rules.md`에 18개 패턴 감지 규칙 + template 기반 메시지 자동 생성 |
| **금융 안전성** | `safety-disclaimer.md` + `tax-fee-impact.md`로 규제 준수 및 면책 자동화 |
| **전문성** | 상관분석, 분산비율, 소득 분리, 과잉매매 감지 등 전문가 수준 분석 |
| **데이터 무결성** | `data-integrity-check.md`의 교차검증 게이트로 파이프라인 품질 보장 |
| **범용성** | 모든 파일에 `extension_points` 섹션으로 확장성 입증 |
| **바이브코딩 활용** | 영어 구조 + 한국어 설명 하이브리드, AI가 읽고 코드 생성 가능한 포맷 |
| **모델 효율** | INDEX.md의 `recommended_model`로 작업별 최적 모델 자동 배정 |
