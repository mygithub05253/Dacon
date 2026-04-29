# Parsing Rules — Index
### Navigation & Pipeline Overview

> **Pipeline position**: Stage 1 (entry point)
> **Input**: Raw CSV / XLSX file uploaded by user
> **Output**: Standardized DataFrame conforming to `standard_schema`
> **Depends on**: none
> **Used by**: `analysis/*` skills
> **Recommended model**: `sonnet` — 규칙 기반 매핑/정규화, 판단보다 절차 중심

---

## Files

| # | File | Sections | Description |
|---|------|----------|-------------|
| 1 | [file-detection.md](file-detection.md) | 1.1 ~ 1.4 | Supported formats, encoding detection, delimiter detection, header detection |
| 2 | [column-mapping.md](column-mapping.md) | 2.1 ~ 2.3 | Standard schema definition, column synonym dictionary, 6-step mapping pipeline |
| 3 | [normalization.md](normalization.md) | 3.1 ~ 3.4 | Number normalization, date normalization, market auto-detection, currency conversion |
| 4 | [broker-presets.md](broker-presets.md) | 5 + 4 + 6 | Broker presets (5 brokers), error handling (file/data/quality), extension points |

---

## Processing Order

```
Raw File
  │
  ├─ 1. file-detection    → detect format, encoding, delimiter, header
  ├─ 2. column-mapping     → match columns to standard_schema
  ├─ 3. normalization      → clean values, detect market, convert currency
  └─ 4. broker-presets     → apply broker preset (if detected), handle errors
  │
  ▼
Standardized DataFrame  →  analysis/* skills
```

---

## Original Source

The monolithic source file is preserved at:
[`../parsing_rules.md`](../parsing_rules.md)
