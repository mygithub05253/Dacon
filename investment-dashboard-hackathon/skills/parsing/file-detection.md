# File Detection Rules
### Supported Formats, Encoding, Delimiter & Header Detection

> **Pipeline**: Stage 1 | **Depends on**: none | **Used by**: analysis/*
>
> Covers how raw files are identified, decoded, and structurally parsed
> before any column mapping or normalization takes place.

---

## 1. Supported Formats

```yaml
supported_formats:
  description: "지원하는 파일 형식 목록"
  formats:
    - extension: ".csv"
      mime: "text/csv"
    - extension: ".tsv"
      mime: "text/tab-separated-values"
    - extension: ".xlsx"
      mime: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
      note: "첫 번째 시트만 읽음. 시트 선택 UI 제공."
    - extension: ".xls"
      mime: "application/vnd.ms-excel"
      note: "레거시 형식, openpyxl fallback"
    - extension: ".txt"
      mime: "text/plain"
      note: "구분자 자동 감지"

file_size_limit:
  description: "파일 크기 제한"
  max: "50MB"
  warning_at: "10MB"
  action_over_limit: "파일이 너무 큽니다 (최대 50MB). 데이터를 분할해주세요."

row_limit:
  description: "행 수 제한"
  max: 100000
  warning_at: 50000
  action_over_limit: "최근 {max}건만 분석합니다. 전체 분석이 필요하면 기간을 나눠서 업로드하세요."
```

---

## 2. Encoding Detection

```yaml
encoding_detection:
  description: "인코딩 감지 우선순위"
  strategy: "chardet auto-detect → fallback chain"
  priority: [UTF-8, UTF-8-SIG, EUC-KR, CP949, ASCII]
  fallback: UTF-8
  exception:
    decode_error:
      description: "디코딩 실패 시 처리"
      action: "다음 인코딩으로 재시도"
      max_retries: 4
      final_fallback: "errors='replace' 옵션으로 깨진 문자 대체 후 경고"
      message: "일부 문자가 깨졌을 수 있습니다. 원본 파일의 인코딩을 확인해주세요."
```

---

## 3. Delimiter Detection

```yaml
delimiter_detection:
  description: "구분자 자동 감지 전략"
  strategy: "csv.Sniffer + frequency analysis"
  candidates:
    - ","        # CSV default
    - "\t"       # TSV (키움증권 등)
    - "|"        # pipe-delimited (일부 금융 데이터)
    - ";"        # European CSV
  exception:
    mixed_delimiters:
      description: "한 파일에 여러 구분자 혼용"
      action: "가장 빈도 높은 구분자 선택 후 경고"
      message: "구분자가 일관되지 않습니다. ','로 처리했습니다."
    no_delimiter:
      description: "구분자 없이 공백만 존재"
      action: "\\s+ (연속 공백) 구분자로 시도"
```

---

## 4. Header Detection

```yaml
header_detection:
  description: "헤더 행 자동 탐지 전략"
  strategy: "row-by-row analysis to find header row"
  rules:
    - "첫 N행(최대 10행) 스캔하여 텍스트 비율이 80% 이상인 첫 행을 헤더로 인식"
    - "숫자만 있는 행은 데이터 행으로 판단"
  skip_patterns:
    - pattern: "계좌번호|조회일시|고객명|출력일"
      action: "해당 행 스킵 (증권사 메타 정보)"
    - pattern: "^\\s*$"
      action: "빈 행 스킵"
    - pattern: "^#|^//"
      action: "주석 행 스킵"
  exception:
    no_header_found:
      description: "헤더를 찾을 수 없는 경우"
      action: "자동 헤더 생성 (col_0, col_1, ...) 후 사용자에게 매핑 요청"
      message: "헤더를 자동 인식할 수 없습니다. 컬럼을 직접 지정해주세요."
    multi_level_header:
      description: "2행 이상의 병합 헤더 (증권사 리포트)"
      action: "행을 합쳐서 단일 헤더로 변환. 예: '수익률' + '(%)' → '수익률(%)'"
    duplicate_column_names:
      description: "같은 이름의 컬럼이 여러 개"
      action: "뒤에 _1, _2 접미사 추가 후 경고"
      message: "중복 컬럼명이 있어 자동으로 구분했습니다: {columns}"
```
