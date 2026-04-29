# Phase 5. 바이브코딩 전략 + 확장 기능 설계

---

## 1. 바이브코딩 정의와 AlphaFolio에서의 의미

바이브코딩(Vibe-coding)은 자연어 또는 구조화된 문서를 AI에게 전달하면 AI가 동작하는 코드를 생성하는 개발 방식입니다.

AlphaFolio에서 바이브코딩은 단순히 "AI에게 코딩을 맡기는 것"이 아닙니다. **Skills.md 문서 자체가 프로그래밍 언어의 역할**을 합니다. YAML로 정의된 규칙이 곧 실행 명세이고, AI는 이를 Python으로 번역하는 컴파일러 역할을 합니다.

```
전통적 개발:   요구사항(자연어) → 개발자(해석) → 코드(Python)
바이브코딩:    Skills.md(YAML) → AI(변환) → 코드(Python)
AlphaFolio:   Skills.md(YAML) → AI 변환 + 런타임 파싱 → 대시보드
```

---

## 2. 3계층 아키텍처

### 계층 1: 설계 시점 코드 생성 (Design-time Generation)

프로젝트를 처음 만들 때 Skills.md를 AI에게 넘겨 파이프라인 코드를 생성합니다.

```yaml
layer_1:
  when: "프로젝트 초기 개발, 대규모 규칙 변경 시"
  scope: "pipeline/*.py, components/*.py 전체"
  frequency: "비정기 (구조적 변경 시)"
  coverage: "전체 규칙의 약 30% (복잡한 알고리즘, 새 차트 유형 등)"
```

### 계층 2: 런타임 규칙 해석 (Runtime Interpretation)

생성된 코드가 실행 시점에 Skills.md의 YAML을 직접 읽고 조건을 평가합니다. 규칙을 바꾸면 코드 수정 없이 즉시 반영됩니다.

```yaml
layer_2:
  when: "사용자가 데이터를 업로드할 때마다"
  scope: "인사이트 규칙, 컬럼 매핑 사전, 차트 선택 조건, 에러 메시지 등"
  frequency: "매 실행"
  coverage: "전체 규칙의 약 60%"
```

### 계층 3: 사용자 자연어 확장 (Future — User-time Generation)

사용자가 대시보드에서 자연어로 분석을 요청하면 AI가 즉석으로 분석 코드를 생성하여 실행합니다. MVP 이후 확장 기능입니다.

```yaml
layer_3:
  when: "사용자가 대시보드 내에서 자연어 질문 시"
  scope: "ad-hoc 분석 코드"
  frequency: "사용자 요청 시"
  coverage: "미래 확장 (약 10%)"
  status: "MVP 이후"
```

---

## 3. 프롬프트 엔지니어링 전략

### 3.1 프롬프트 설계 원칙

```yaml
prompt_principles:
  # ─── 원칙 1: 역할 고정 ───
  role_anchoring:
    description: "매 프롬프트의 첫 문장에서 AI의 역할을 명확히 고정"
    reason: "역할이 모호하면 AI가 설명문을 쓰거나 의견을 내기 시작함"
    pattern: |
      당신은 AlphaFolio 프로젝트의 {stage_name} 모듈을 구현하는
      시니어 Python 백엔드 개발자입니다.
      코드만 생성하세요. 설명은 주석으로 넣으세요.

  # ─── 원칙 2: 입출력 스키마 명시 ───
  io_schema:
    description: "AI에게 입력 타입과 출력 타입을 코드로 보여줌"
    reason: "자연어 설명보다 Python dataclass/TypedDict가 정확"
    pattern: |
      ## 입력
      ```python
      @dataclass
      class ParsedStock:
          ticker: str
          name: str
          market: str       # "KR" | "US"
          quantity: int
          avg_price: float
          current_price: Optional[float]
          currency: str     # "KRW" | "USD"
      ```
      ## 출력
      ```python
      @dataclass
      class AnalysisResult:
          return_rate: Optional[float]  # None이면 계산 불가
          valuation: float
          profit_loss: Optional[float]
          weight: float               # 0~100
          flags: List[str]            # ["estimated", "extreme_return"] 등
      ```

  # ─── 원칙 3: 규칙을 통째로 넘기기 ───
  full_context:
    description: "Skills.md 전문을 프롬프트에 포함 (요약하지 않음)"
    reason: |
      요약하면 예외 처리, fallback chain, 에지 케이스가 빠짐.
      Skills.md 파일 하나가 보통 300~500줄 — 현대 LLM의 컨텍스트 안에 충분히 들어감.
    exception: "파일이 1000줄 이상이면 관련 섹션만 발췌"

  # ─── 원칙 4: 안티패턴 명시 ───
  negative_examples:
    description: "AI가 자주 하는 실수를 '하지 마세요' 목록으로 방지"
    reason: "AI는 과도하게 추상화하거나, 불필요한 클래스 계층을 만드는 경향이 있음"
    pattern: |
      ## 하지 마세요
      - ABC 추상 클래스, 팩토리 패턴 등 과도한 추상화
      - 외부 라이브러리 추가 (pandas, plotly, streamlit만 사용)
      - 비동기(async) 코드
      - 하나의 함수가 50줄을 넘기는 것
      - 타입 힌트 없는 함수

  # ─── 원칙 5: 점진적 생성 ───
  incremental:
    description: "한 번에 전체 모듈을 생성하지 않고, 기능 단위로 나눠서 생성"
    reason: |
      한 번에 500줄 코드를 생성하면 품질이 낮음.
      50~100줄 단위로 나눠 생성하고, 각 단위를 검증 후 다음으로 진행.
    pattern:
      step_1: "데이터 로드 + 인코딩 감지 함수"
      step_2: "컬럼 매핑 함수 (동의어 사전 포함)"
      step_3: "데이터 정규화 함수 (숫자, 날짜, 마켓)"
      step_4: "에러 핸들링 + 경고 수집 함수"
      step_5: "통합 파이프라인 함수 (위 함수들 호출)"
```

### 3.2 단계별 프롬프트 실전 예시

#### 예시 A: parser.py 생성 — 인코딩 감지 함수

```yaml
prompt_example_parser_encoding:
  purpose: "parsing_rules.md의 인코딩 규칙 → Python 함수"
  
  prompt: |
    당신은 AlphaFolio의 CSV 파서를 구현하는 Python 개발자입니다.
    아래 Skills.md 규칙에 따라 파일 인코딩을 감지하는 함수를 작성하세요.

    ## Skills.md 규칙 (parsing_rules.md 발췌)
    ```yaml
    encoding_detection:
      strategy: "chardet 라이브러리로 자동 감지 → 실패 시 순차 시도"
      priority:
        - UTF-8
        - UTF-8-SIG
        - EUC-KR
        - CP949
        - ASCII
      fallback: UTF-8
      exception:
        decode_error:
          action: "다음 인코딩으로 재시도"
          max_retries: 4
          final_fallback: "errors='replace' 옵션으로 깨진 문자 대체 후 경고"
          message: "일부 문자가 깨졌을 수 있습니다. 원본 파일의 인코딩을 확인해주세요."
    ```

    ## 입력
    - file_content: bytes (업로드된 파일의 raw bytes)

    ## 출력
    ```python
    @dataclass
    class EncodingResult:
        encoding: str           # 감지된 인코딩명
        confidence: float       # 0.0~1.0
        text: str               # 디코딩된 텍스트
        warnings: List[str]     # 경고 메시지 목록
    ```

    ## 하지 마세요
    - chardet 외 다른 인코딩 감지 라이브러리 사용
    - 파일을 디스크에 쓰는 행위
    - 50줄을 넘기는 함수

  expected_output_structure: |
    def detect_encoding(file_content: bytes) -> EncodingResult:
        # 1. chardet로 감지 시도
        # 2. confidence 낮으면 priority 순서대로 재시도
        # 3. 모두 실패 시 UTF-8 + errors='replace'
        # 4. 경고 메시지 수집
        ...
```

#### 예시 B: analyzer.py 생성 — 수익률 계산 함수

```yaml
prompt_example_analyzer_return:
  purpose: "analysis_rules.md의 수익률 규칙 → Python 함수"
  
  prompt: |
    당신은 AlphaFolio의 투자 지표 계산 모듈을 구현하는 Python 개발자입니다.
    아래 Skills.md 규칙에 따라 종목별 수익률을 계산하는 함수를 작성하세요.

    ## Skills.md 규칙 (analysis_rules.md 발췌)
    ```yaml
    metric: return_rate
    formula: "(current_price - avg_price) / avg_price * 100"
    unit: "%"
    precision: 2
    exception:
      avg_price_zero:
        condition: "avg_price == 0"
        action: "return_rate = N/A"
        message: "'{name}' 매수가가 0이어서 수익률을 계산할 수 없습니다."
      current_price_missing:
        condition: "current_price가 NaN"
        action: "return_rate = N/A"
        ui_display: "현재가 없음"
      extreme_return:
        condition: "abs(return_rate) > 500"
        action: "경고 표시. 계산은 포함하되 시각적 경고."
        message: "수익률이 {return}%입니다. 데이터를 확인해주세요."
        include_in_calculation: true
    ```

    ## 입력
    ```python
    @dataclass
    class ParsedStock:
        ticker: str
        name: str
        avg_price: float
        current_price: Optional[float]
    ```

    ## 출력
    ```python
    @dataclass
    class ReturnResult:
        return_rate: Optional[float]  # None = 계산 불가
        flags: List[str]              # ["extreme_return"] 등
        warnings: List[str]           # 사용자에게 보여줄 경고
    ```

    ## 하지 마세요
    - numpy 사용 (순수 Python으로 충분)
    - 반올림을 빼먹는 것 (precision: 2 지켜야 함)
    - exception 블록의 모든 케이스를 빠뜨리는 것

  expected_output: |
    def calc_return_rate(stock: ParsedStock) -> ReturnResult:
        warnings = []
        flags = []

        if stock.avg_price == 0:
            warnings.append(f"'{stock.name}' 매수가가 0이어서 수익률을 계산할 수 없습니다.")
            return ReturnResult(return_rate=None, flags=["avg_price_zero"], warnings=warnings)

        if stock.current_price is None or math.isnan(stock.current_price):
            return ReturnResult(return_rate=None, flags=["current_price_missing"], warnings=warnings)

        rate = round((stock.current_price - stock.avg_price) / stock.avg_price * 100, 2)

        if abs(rate) > 500:
            flags.append("extreme_return")
            warnings.append(f"수익률이 {rate}%입니다. 데이터를 확인해주세요.")

        return ReturnResult(return_rate=rate, flags=flags, warnings=warnings)
```

#### 예시 C: 차트 선택 엔진 — visualization_rules.md → Python

```yaml
prompt_example_chart_selector:
  purpose: "visualization_rules.md의 차트 선택 규칙 → 동적 차트 선택기"
  
  prompt: |
    당신은 AlphaFolio의 차트 선택 엔진을 구현하는 Python 개발자입니다.
    아래 Skills.md 규칙에 따라 데이터 조건에 맞는 차트를 자동 선택하는 함수를 작성하세요.

    중요: 이 함수는 visualization_rules.md의 YAML을 런타임에 파싱하여 동작해야 합니다.
    하드코딩하지 마세요. 새 규칙이 YAML에 추가되면 코드 수정 없이 반영되어야 합니다.

    ## Skills.md 규칙 구조 (visualization_rules.md)
    ```yaml
    chart_rules:
      - id: "portfolio_composition"
        condition: "weight 데이터 존재"
        priority: 1
        charts:
          - type: "donut"
            min_data: { stocks: 2 }
            exception:
              single_stock:
                action: "도넛 대신 단일 KPI 카드"
            fallback:
              condition: "렌더링 실패"
              replace_with: "horizontal_bar"
    ```

    ## 핵심 요구사항
    1. chart_rules 배열을 priority 순서로 순회
    2. condition을 데이터에 대해 평가
    3. min_data 충족 여부 확인
    4. 불충족 시 fallback chain 실행
    5. exception 조건도 확인하여 차트 유형 교체

    ## 출력
    ```python
    @dataclass
    class ChartSelection:
        chart_type: str           # "donut", "horizontal_bar", "treemap" 등
        title: str
        config: dict
        is_fallback: bool         # fallback으로 선택되었는지
        fallback_reason: Optional[str]
    ```
```

### 3.3 프롬프트 체이닝 전략

```yaml
prompt_chaining:
  description: "하나의 모듈을 여러 번의 프롬프트로 점진적으로 구축하는 전략"

  chain_for_parser:
    prompt_1:
      focus: "인코딩 감지 + 파일 읽기"
      output: "detect_encoding(), read_file()"
      lines: "~60줄"
    prompt_2:
      focus: "컬럼 매핑 (동의어 사전 로드 + 매핑 알고리즘)"
      input: "prompt_1의 출력을 '기존 코드'로 전달"
      output: "load_synonyms(), map_columns()"
      lines: "~80줄"
    prompt_3:
      focus: "데이터 정규화 (숫자, 날짜, 마켓 식별)"
      output: "normalize_numbers(), detect_market(), normalize_dates()"
      lines: "~70줄"
    prompt_4:
      focus: "증권사 프리셋 자동 감지"
      output: "detect_broker(), apply_preset()"
      lines: "~50줄"
    prompt_5:
      focus: "통합 파이프라인 + 에러 핸들링"
      input: "prompt 1~4의 모든 함수를 '기존 코드'로 전달"
      output: "parse_file() — 모든 함수를 호출하는 메인 함수"
      lines: "~40줄"
    total: "~300줄, 5회 프롬프트"

  chain_for_analyzer:
    prompt_1: "종목별 기본 지표 (return_rate, valuation, profit_loss, weight)"
    prompt_2: "포트폴리오 전체 지표 (총 수익률, 총 평가금액, 종목 통계)"
    prompt_3: "집계 분석 (섹터별, 마켓별, 수익률 분포)"
    prompt_4: "고급 지표 (샤프, MDD, 변동성, 베타) + 분석 모드 판별"
    prompt_5: "통합 함수 + 모드별 분기 + 에러 핸들링"
    total: "~400줄, 5회 프롬프트"

  chain_for_visualizer:
    prompt_1: "YAML 파서 + 차트 선택 엔진"
    prompt_2: "Plotly 차트 빌더 (도넛, 바, 트리맵)"
    prompt_3: "Plotly 차트 빌더 (라인, 산점도, 히스토그램)"
    prompt_4: "fallback chain 실행기 + 텍스트 폴백"
    prompt_5: "KPI 카드 렌더러 + 통합 함수"
    total: "~450줄, 5회 프롬프트"

  chain_for_insight:
    prompt_1: "YAML 규칙 로더 + 조건 평가 엔진"
    prompt_2: "패턴 감지 루프 (모든 규칙 순회 + suppress/dedup)"
    prompt_3: "템플릿 렌더링 + 인사이트 카드 생성"
    prompt_4: "종합 요약 생성 + 에스컬레이션 판별"
    total: "~250줄, 4회 프롬프트"

  chain_for_reporter:
    prompt_1: "섹션 구성 엔진 + 모드별 활성화"
    prompt_2: "PDF 생성 (차트 이미지 변환 + 레이아웃)"
    prompt_3: "CSV 내보내기 + ZIP 패키징"
    total: "~200줄, 3회 프롬프트"
```

---

## 4. AI 생성 코드 검증 전략

### 4.1 검증 파이프라인

```yaml
verification_pipeline:
  description: "AI가 생성한 코드를 프로덕션에 넣기 전에 반드시 거치는 단계"

  stage_1_static_check:
    name: "정적 검사"
    tools: ["mypy (타입 체크)", "ruff (린팅)", "black (포맷팅)"]
    automated: true
    pass_criteria:
      - "mypy 에러 0건"
      - "ruff 에러 0건 (경고는 허용)"
      - "black 포맷 일치"
    common_ai_mistakes:
      - "Optional을 쓰고 None 체크를 빼먹음 → mypy가 잡음"
      - "미사용 import → ruff가 잡음"
      - "일관성 없는 따옴표 → black이 교정"

  stage_2_unit_test:
    name: "단위 테스트"
    method: "AI에게 테스트 코드도 같이 생성 요청"
    prompt_addition: |
      위 함수에 대한 pytest 테스트도 작성하세요.
      반드시 다음 케이스를 포함하세요:
      1. 정상 케이스 (happy path)
      2. Skills.md에 정의된 모든 exception 케이스
      3. 빈 데이터 / None 입력
      4. 경계값 (threshold 근처)
    
    example_test_cases:
      calc_return_rate:
        - case: "정상 — 매수가 100, 현재가 112"
          expected: "return_rate = 12.0, flags = []"
        - case: "매수가 0"
          expected: "return_rate = None, flags = ['avg_price_zero']"
        - case: "현재가 None"
          expected: "return_rate = None, flags = ['current_price_missing']"
        - case: "극단 수익률 — 매수가 10, 현재가 1000"
          expected: "return_rate = 9900.0, flags = ['extreme_return']"
        - case: "수익률 정확히 500%"
          expected: "return_rate = 500.0, flags = [] (500은 포함하지 않음, >500이 조건)"
        - case: "음수 매수가"
          expected: "예외 처리 (데이터 오류)"

  stage_3_integration_test:
    name: "통합 테스트 (샘플 데이터 End-to-End)"
    test_data:
      - name: "sample_kr.csv"
        description: "키움증권 형식 국내 10종목"
        expected: "full_mode, 10종목 분석, 인사이트 3개 이상"
      - name: "sample_us.csv"
        description: "영문 미국 ETF 5종목"
        expected: "standard_mode, 5종목, USD 통화"
      - name: "sample_mixed.csv"
        description: "KR+US 혼합 15종목"
        expected: "standard_mode, 혼합 통화 처리, 마켓 비교 활성"
      - name: "sample_minimal.csv"
        description: "종목 1개, 최소 컬럼"
        expected: "minimal_mode, single_stock 예외 처리"
      - name: "sample_broken.csv"
        description: "깨진 인코딩 + 누락 컬럼"
        expected: "경고 메시지 + 부분 파싱 성공 또는 에러 화면"
    pass_criteria:
      - "5개 샘플 모두 크래시 없이 결과 생성"
      - "에러 케이스에서 적절한 에러 메시지 표시"
      - "KPI 값이 수동 계산 결과와 일치"

  stage_4_visual_review:
    name: "시각적 검증"
    method: "Streamlit 로컬 실행 + 화면 확인"
    checklist:
      - "차트가 빈 영역 없이 렌더링되는가"
      - "숫자 포맷이 Skills.md 규칙과 일치하는가 (소수점, 통화 기호)"
      - "색상이 규칙과 일치하는가 (green=positive, red=negative)"
      - "모바일 뷰에서 레이아웃이 깨지지 않는가"
      - "필터 적용 시 모든 차트가 갱신되는가"
      - "에러 상태에서 안내 메시지가 뜨는가"

  stage_5_edge_case_stress:
    name: "엣지 케이스 스트레스 테스트"
    cases:
      - "종목 0개 (빈 CSV)"
      - "종목 1개"
      - "종목 200개 (대량)"
      - "모든 종목 수익률 0%"
      - "모든 종목 수익률 -100%"
      - "현재가 전부 NaN"
      - "매수가 전부 0"
      - "통화 혼합 (KRW + USD + EUR)"
      - "날짜 형식 혼합 (2024-01-01 + 01/01/2024 + 20240101)"
      - "파일 크기 49MB (한계치)"
```

### 4.2 AI 코드 리뷰 체크리스트

```yaml
ai_code_review_checklist:
  description: "AI 생성 코드를 받았을 때 개발자가 확인하는 항목"

  must_check:
    - item: "Skills.md 예외 처리 누락 여부"
      how: "Skills.md의 exception 블록 수와 코드의 try/except 수를 대조"
      common_miss: "3개 예외 중 1~2개만 구현하고 나머지를 빠뜨림"

    - item: "하드코딩된 매직 넘버"
      how: "500, 30, 50 같은 숫자가 코드에 직접 있으면 Skills.md의 threshold와 대조"
      fix: "상수를 Skills.md에서 읽어오도록 변경"

    - item: "None/NaN 처리"
      how: "Optional 타입 필드에 대한 None 체크가 모든 경로에 있는지"
      common_miss: "happy path만 구현하고 None 경로를 빠뜨림"

    - item: "fallback chain 완전성"
      how: "Skills.md의 fallback 단계 수와 코드의 try/except 중첩 수 비교"
      common_miss: "2~3단계 중 마지막 단계(텍스트 폴백)를 빠뜨림"

    - item: "반환 타입 일관성"
      how: "함수 시그니처의 반환 타입과 실제 return 문이 일치하는지"
      common_miss: "어떤 경로에서는 dict를, 다른 경로에서는 dataclass를 반환"

    - item: "경고 메시지 수집"
      how: "Skills.md의 message 필드가 코드에서 warnings 리스트에 추가되는지"
      common_miss: "경고를 print()로 출력하거나 아예 무시"

  nice_to_check:
    - "함수 길이 50줄 이하"
    - "중복 코드 없음"
    - "docstring 포함"
    - "타입 힌트 완전성"
```

---

## 5. 런타임 규칙 해석 엔진 설계

### 5.1 YAML 규칙 로더

```yaml
rule_loader:
  description: "Skills.md에서 YAML 코드블록을 추출하여 Python dict로 변환"
  
  design:
    loading_strategy: "앱 시작 시 1회 로드 → 메모리 캐싱"
    cache: "@st.cache_resource (앱 재시작 전까지 유지)"
    
    parsing_method: |
      1. Skills.md 파일을 텍스트로 읽기
      2. ```yaml ... ``` 코드블록 정규식으로 추출
      3. 각 블록을 yaml.safe_load()로 파싱
      4. 섹션 ID 기준으로 dict에 저장
    
    error_handling:
      yaml_syntax_error: "해당 블록 스킵 + 경고 로그"
      file_not_found: "기본값 사용 (하드코딩된 최소 규칙)"
      encoding_error: "UTF-8로 강제 로드"

  auto_reload:
    description: "개발 중 Skills.md 수정 시 자동 반영"
    method: "파일 수정 시간 체크 → 변경 감지 시 캐시 무효화"
    production: "비활성화 (배포 후에는 재시작으로 반영)"
```

### 5.2 조건 평가 엔진

```yaml
condition_evaluator:
  description: "Skills.md의 condition 문자열을 데이터에 대해 평가"
  
  supported_conditions:
    simple_comparison: "any stock weight > 50%"
    field_existence: "weight 데이터 존재"
    count_check: "종목 수 >= 5"
    mode_check: "analysis_mode == 'full_mode'"
    combined: "return_rate < -30% AND weight > 20%"
    null_check: "current_price가 NaN"
    
  evaluation_strategy: |
    condition 문자열을 정규식으로 분해하여 Python 표현식으로 매핑.
    eval()은 사용하지 않음 (보안 리스크). 
    대신 사전 정의된 평가 함수 맵으로 처리.
    
  function_map:
    "any stock {field} > {value}": "lambda df, f, v: (df[f] > v).any()"
    "all stock {field} > {value}": "lambda df, f, v: (df[f] > v).all()"
    "{field} 데이터 존재": "lambda df, f: f in df.columns and df[f].notna().any()"
    "종목 수 >= {n}": "lambda df, n: len(df) >= n"
    "analysis_mode == '{mode}'": "lambda state, m: state.analysis_mode == m"
    
  fallback: "매칭 안 되는 condition → 항상 False (안전 기본값)"
```

### 5.3 어떤 규칙이 런타임이고, 어떤 규칙이 설계 시점인가

```yaml
runtime_vs_designtime:
  description: "각 규칙의 해석 시점을 명확히 구분"

  runtime_interpreted:
    description: "코드 수정 없이 Skills.md만 수정하면 동작 변경"
    rules:
      parsing_rules:
        - "컬럼 동의어 사전 (synonym_map) — 매핑명 추가/삭제"
        - "증권사 프리셋 (broker_presets) — 새 증권사 패턴 추가"
        - "인코딩 우선순위 — 순서 변경"
        - "에러/경고 메시지 텍스트"
        - "파일 크기 제한 값"
      analysis_rules:
        - "수익률 구간 분류 기준 (buckets)"
        - "분석 모드 전환 임계값 (min_stocks, min_days)"
        - "지표 표시 format, precision"
      visualization_rules:
        - "차트 선택 조건 + 우선순위"
        - "색상 팔레트 값"
        - "min_data 임계값"
        - "top_n, others_threshold 값"
        - "차트 config (hole, sort, show_values 등)"
      insight_rules:
        - "패턴 감지 조건 (condition)의 임계값"
        - "메시지 템플릿 텍스트"
        - "인사이트 유형 색상/아이콘"
        - "suppress/dedup 규칙"
        - "에스컬레이션 조건"
      report_rules:
        - "섹션 순서, required 여부"
        - "모드별 활성화 매핑"
        - "description_template 텍스트"
        - "푸터 면책 사항 텍스트"

  design_time_generated:
    description: "AI에게 코드 생성을 의뢰하고, 검증 후 코드에 반영"
    rules:
      - "수익률 계산 공식 (formula 필드)"
      - "MDD / 샤프비율 / 변동성 등 고급 지표 알고리즘"
      - "Plotly 차트 렌더링 함수 (새 차트 유형)"
      - "PDF 생성 로직"
      - "데이터 정규화 알고리즘 (날짜 파싱, 숫자 정리)"
      - "fallback chain의 실제 렌더링 코드"
```

---

## 6. 버전 관리 전략

```yaml
version_control:
  # ─── Skills.md 버전 관리 ───
  skills_versioning:
    method: "Git으로 Skills.md도 소스코드와 동일하게 관리"
    branch_strategy:
      main: "안정 버전 Skills.md"
      feature: "새 규칙 추가 시 feature/add-crypto-rules 등"
    
    commit_convention:
      format: "[skills:{doc_name}] {변경 내용}"
      examples:
        - "[skills:insight] 환율 리스크 인사이트 규칙 추가"
        - "[skills:parsing] 토스증권 프리셋 컬럼명 수정"
        - "[skills:viz] 캔들스틱 차트 규칙 추가"
        - "[skills:analysis] 소르티노 비율 지표 추가"
    
    review_process:
      런타임_규칙_수정:
        review: "변경 diff 확인 → 샘플 데이터 재실행으로 검증"
        approver: "개발자 본인 (1인 프로젝트)"
        risk: "낮음 (코드 변경 없으므로)"
      설계시점_규칙_추가:
        review: "변경 diff 확인 → AI 코드 재생성 → 테스트 → 반영"
        approver: "개발자 본인"
        risk: "중간 (새 코드 추가되므로)"

  # ─── AI 생성 코드 추적 ───
  generated_code_tracking:
    method: "각 생성 파일 상단에 메타데이터 주석"
    template: |
      # ─── AUTO-GENERATED ───
      # source: skills/{doc_name}.md
      # section: {section_id}
      # generated_by: Claude (claude-sonnet-4-6)
      # generated_at: 2026-04-28
      # prompt_version: v1.2
      # manual_edits: {있으면 변경 이력 기술}
      # ──────────────────────
    
    purpose:
      - "어떤 Skills.md 규칙에서 유래했는지 추적"
      - "재생성 시 어떤 프롬프트 버전을 사용했는지 확인"
      - "수동 수정이 있었는지 표시 (재생성 시 충돌 방지)"

  # ─── 변경 영향 분석 ───
  change_impact:
    description: "Skills.md 한 곳을 수정하면 어디까지 영향 받는지 매핑"
    
    impact_map:
      parsing_rules_change:
        직접_영향: ["pipeline/parser.py"]
        간접_영향: ["pipeline/analyzer.py (입력 스키마가 바뀔 수 있음)"]
        검증: "파싱 결과 df의 컬럼이 표준 스키마와 일치하는지"
      
      analysis_rules_change:
        직접_영향: ["pipeline/analyzer.py"]
        간접_영향: ["pipeline/visualizer.py (지표명 참조)", "pipeline/insight_engine.py (지표값 참조)"]
        검증: "계산 결과가 수동 계산과 일치하는지"
      
      visualization_rules_change:
        직접_영향: ["pipeline/visualizer.py"]
        간접_영향: ["components/chart_renderer.py"]
        검증: "차트 렌더링 성공 여부, fallback 동작 확인"
      
      insight_rules_change:
        직접_영향: ["pipeline/insight_engine.py"]
        간접_영향: ["components/insight_cards.py"]
        검증: "인사이트 생성 수, 충돌 해소 정상 동작"
      
      report_rules_change:
        직접_영향: ["pipeline/reporter.py", "pages/*.py"]
        간접_영향: ["utils/export.py"]
        검증: "모든 페이지 렌더링 + PDF 생성"
```

---

## 7. 개발 타임라인

```yaml
development_timeline:
  total_estimate: "5일 (1인 개발 기준)"
  
  day_1:
    focus: "파이프라인 핵심 — 파서 + 분석기"
    tasks:
      - task: "parser.py 생성 (프롬프트 5회)"
        hours: 3
        detail: "인코딩→컬럼매핑→정규화→프리셋→통합"
      - task: "analyzer.py 생성 (프롬프트 5회)"
        hours: 3
        detail: "기본지표→전체지표→집계→고급→통합"
      - task: "단위 테스트 작성 + 검증"
        hours: 2
    deliverable: "파싱 + 분석 파이프라인 동작 확인"

  day_2:
    focus: "시각화 + 인사이트 엔진"
    tasks:
      - task: "visualizer.py 생성 (프롬프트 5회)"
        hours: 3
        detail: "YAML파서→도넛/바/트리맵→라인/산점도→fallback→통합"
      - task: "insight_engine.py 생성 (프롬프트 4회)"
        hours: 2
        detail: "규칙로더→패턴감지→템플릿→요약"
      - task: "통합 테스트 (5개 샘플 데이터)"
        hours: 3
    deliverable: "전체 파이프라인 End-to-End 동작"

  day_3:
    focus: "Streamlit UI — 홈 + 포트폴리오 + 성과분석"
    tasks:
      - task: "main.py (홈 페이지) 생성"
        hours: 2
        detail: "랜딩→업로드→로딩→KPI→인사이트→미니차트"
      - task: "포트폴리오 페이지 생성"
        hours: 2
        detail: "도넛+트리맵+마켓비교"
      - task: "성과분석 페이지 생성"
        hours: 2
        detail: "수익률바+섹터비교+분포+기여도"
      - task: "공통 사이드바 + 필터 연동"
        hours: 2
    deliverable: "3페이지 동작 확인"

  day_4:
    focus: "나머지 페이지 + 내보내기"
    tasks:
      - task: "리스크 페이지 생성"
        hours: 2
      - task: "종목상세 페이지 생성"
        hours: 1.5
      - task: "리포트 페이지 + PDF 생성"
        hours: 3
      - task: "시각적 검증 + 엣지 케이스 테스트"
        hours: 1.5
    deliverable: "6페이지 전체 동작"

  day_5:
    focus: "배포 + 최종 검증 + 문서"
    tasks:
      - task: "Streamlit Cloud 배포"
        hours: 1
      - task: "배포 환경 테스트 (5개 샘플)"
        hours: 2
      - task: "에러 케이스 최종 검증"
        hours: 2
      - task: "기획서 최종 정리"
        hours: 3
    deliverable: "배포 URL + 기획서 PDF + Skills.md PDF"
```

---

## 8. 확장 기능 로드맵

### 8.1 자산군 확장

모든 Skills.md에 이미 `extension_points`가 정의되어 있습니다. 활성화만 하면 새 자산군을 지원합니다.

```yaml
asset_extensions:
  crypto:
    priority: "MVP 이후 첫 확장"
    effort: "3~5일"
    skills_changes:
      parsing: "티커 패턴 (BTC, ETH), 거래소 프리셋 (업비트, 바이낸스), 24/7 마켓"
      analysis: "스테이블코인 비율, DeFi 노출도, 24h 변동성"
      visualization: "히트맵 (코인 상관관계), 스테이블 게이지"
      insight: "스테이블코인 비율 < 10%, 단일 토큰 집중, DeFi 분산"
      report: "DeFi 포지션 섹션, 스테이블코인 비율 섹션"
    new_code: "거래소 API 연동, 24h 변동성 계산"

  bond:
    priority: "Phase 3"
    effort: "5~7일"
    skills_changes:
      parsing: "채권 ID (ISIN), 만기일/쿠폰율/신용등급 컬럼"
      analysis: "듀레이션, 수정듀레이션, YTM, 이자수익 분리"
      visualization: "수익률 곡선, 만기 래더, 신용등급 분포"
      insight: "만기 집중, 금리 민감도, 쿠폰 일정"
      report: "만기 일정, 이자 수령 캘린더, 듀레이션 분석 섹션"
    new_code: "채권 가격 계산, YTM 반복 계산 (Newton-Raphson)"

  real_estate:
    priority: "Phase 4"
    effort: "3~5일"
    skills_changes:
      parsing: "REITs 티커, 임대 수익 컬럼"
      analysis: "배당수익률, NAV 프리미엄/디스카운트, 입주율"
      visualization: "배당 이력, NAV 추이"
      insight: "NAV 할인율 과다, 배당 컷 리스크"
      report: "입주율 현황, NAV 분석 섹션"

  multi_asset:
    priority: "Phase 5"
    effort: "7~10일"
    description: "주식+채권+암호화폐 통합 분석, 자산군 간 상관관계, 리밸런싱 제안"
```

### 8.2 기능 확장

```yaml
feature_extensions:
  benchmark:
    priority: "MVP 포함"
    features: ["KOSPI/S&P 500 비교", "초과 수익률(알파)", "추적 오차"]
    data: "yfinance API"

  simulation:
    priority: "Phase 2"
    features: ["종목 교체 시뮬레이션", "비중 조정", "리밸런싱 제안", "백테스트"]
    ui: "새 페이지 '🔮 시뮬레이션' + 슬라이더 인터랙션"

  comparison:
    priority: "Phase 3"
    features: ["포트폴리오 A vs B 비교", "시점별 변화 추적"]
    ui: "새 페이지 '🆚 비교' + 2개 파일 업로드"

  ai_chat:
    priority: "Phase 4"
    features: ["자연어 분석 질의", "맞춤 설명", "투자 용어 해설"]
    implementation: "st.chat_input + Claude API"

  external_data:
    priority: "Phase 5"
    features: ["실시간 주가", "환율 업데이트", "섹터 자동 매핑", "뉴스 연동"]
```

---

## 9. 바이브코딩 효과 정량화

```yaml
quantified_impact:
  rule_total: "~80개 규칙 (5개 Skills.md 합산)"
  exception_total: "~120개 예외 처리"
  
  automation_breakdown:
    fully_auto: "60% — Skills.md 수정만으로 코드 변경 없이 동작 변경"
    ai_assisted: "30% — Skills.md + AI 프롬프트 → 코드 생성 → 리뷰"
    manual: "10% — 알고리즘/외부 연동 등 직접 구현"

  speed_comparison:
    rule_addition:
      traditional: "30분~1시간 (코드 수정 + 테스트)"
      vibecoding: "5~15분 (YAML 추가 또는 프롬프트 1회)"
      speedup: "3~4×"
    new_asset_class:
      traditional: "1~2주 (전체 파이프라인 재구현)"
      vibecoding: "3~5일 (extension_points 활성화 + AI 코드 생성)"
      speedup: "2~3×"
    bug_fix:
      traditional: "원인 파악 → 코드 수정 → 테스트"
      vibecoding: "Skills.md 규칙 확인 → 규칙 수정 또는 AI 코드 재생성 → 테스트"
      benefit: "규칙이 명시적이라 원인 파악이 빠름"

  code_quality:
    consistency: "모든 모듈이 동일한 패턴 (condition → action → exception)"
    coverage: "Skills.md에 정의된 예외 = 반드시 구현된 예외"
    documentation: "코드 상단 메타데이터로 원본 규칙 추적 가능"
```

---

## 10. 평가 기준 대응

| 평가 항목 (배점) | 대응 전략 |
|-----------------|----------|
| 바이브코딩 활용 (15점) | 3계층 아키텍처 + 구체적 프롬프트 템플릿 + 코드 변환 예시 + 5단계 검증 파이프라인. 규칙의 60%가 코드 수정 없이 동작 변경 가능. |
| 범용성 (25점) | 자산군 확장 = Skills.md extension_points 활성화. 런타임 규칙 해석으로 동적 대응. 변경 영향 분석 매핑으로 안전한 확장. |
| Skills.md 설계 (25점) | AI 친화 YAML 설계 원칙 5가지 (역할 고정, IO 스키마, 전문 전달, 안티패턴, 점진적 생성). 프롬프트 체이닝으로 모듈별 22회 프롬프트 계획. |
| 대시보드 자동 생성 (25점) | Skills.md → 프롬프트 체이닝 → 파이프라인 코드 → 대시보드 자동 구성의 일관된 워크플로우. |
| 실용성/창의성 (10점) | 5일 개발 타임라인, 버전 관리 전략, AI 코드 리뷰 체크리스트로 실무 적용 가능성 입증. |
