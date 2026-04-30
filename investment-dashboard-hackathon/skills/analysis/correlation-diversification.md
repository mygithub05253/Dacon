# Correlation & Diversification Analysis

> Stage 2 supplement (requires trade_history mode).
> Input: Daily price series from advanced-metrics (min 60 trading days).
> Output: Correlation matrix, diversification ratio, cluster groups.
> Referenced by: `viz/chart-selection` (heatmap), `insight/pattern-rules` (concentration alerts).
> Recommended model: opus

---

## 1. Correlation Matrix

```yaml
correlation_matrix:
  description: "종목 간 상관관계 행렬 생성"
  method: "pearson"
  input: "daily_returns per stock (from advanced-metrics daily_returns_generation)"

  parameters:
    min_overlap_days: 60
    data_source: "daily_returns (adjusted close)"

  rules:
    pair_calculation:
      description: "각 종목 쌍에 대해 Pearson 상관계수 계산"
      formula: "corr(stock_A_daily_returns, stock_B_daily_returns)"
    insufficient_overlap:
      condition: "겹치는 거래일 < 60"
      action: "해당 쌍을 'N/A'로 표시, 보간(interpolation) 금지"
      message: "거래일 겹침이 부족하여 상관계수를 산출할 수 없습니다."
    significance_test:
      method: "t-test for correlation coefficient"
      threshold: 0.05
      action: "p-value > 0.05인 상관계수에 '통계적으로 유의하지 않음' 표시"

  output:
    format: "N×N matrix (N = number of stocks)"
    diagonal: 1.0
    symmetry: true
```

---

## 2. Interpretation Rules

```yaml
interpretation:
  description: "상관계수 해석 기준"

  thresholds:
    high:
      condition: "|r| > 0.7"
      label: "높은 상관관계"
      color: "#E24B4A"
    moderate:
      condition: "0.4 < |r| <= 0.7"
      label: "중간 상관관계"
      color: "#BA7517"
    low:
      condition: "|r| <= 0.4"
      label: "낮은 상관관계"
      color: "#639922"
    inverse:
      condition: "r < -0.3"
      label: "역상관"
      note: "잠재적 헤지 페어 — 한 종목 하락 시 다른 종목이 상승할 수 있음"

  display:
    chart_type: "heatmap"
    color_scale: "diverging"
    color_map:
      negative_1: "#E24B4A"   # red
      zero: "#FFFFFF"          # white
      positive_1: "#2563EB"    # blue
    annotation: "각 셀에 상관계수 값 표시 (소수점 2자리)"
    ordering: "hierarchical clustering 순서로 정렬 (유사 종목 인접 배치)"
```

---

## 3. Diversification Ratio

```yaml
diversification_ratio:
  description: "분산투자 효과 정량화"
  formula: "DR = sum(w_i * sigma_i) / sigma_portfolio"
  explanation: "개별 종목 가중 변동성의 합 / 포트폴리오 전체 변동성"

  parameters:
    w_i: "종목별 비중 (시가 기준)"
    sigma_i: "종목별 연환산 변동성"
    sigma_portfolio: "포트폴리오 연환산 변동성 (상관관계 반영)"

  interpretation:
    excellent:
      condition: "DR >= 1.5"
      label: "우수한 분산투자 효과"
      color: "#27680A"
    adequate:
      condition: "1.2 <= DR < 1.5"
      label: "적정한 분산투자"
      color: "#639922"
    limited:
      condition: "DR < 1.2"
      label: "분산투자 효과가 제한적입니다"
      color: "#D85A30"

  rules:
    dr_equals_1:
      description: "DR = 1이면 모든 종목이 완전 상관 (분산투자 효과 없음)"
    dr_greater_than_1:
      description: "DR > 1이면 분산투자가 포트폴리오 변동성을 낮추고 있음"

  display:
    format: "단일 수치 + 해석 텍스트"
    precision: 2
```

---

## 4. Cluster Analysis

```yaml
cluster_analysis:
  description: "유사하게 움직이는 종목 그룹 식별"
  activation: "종목 수 >= 8일 때만 활성화"

  method:
    algorithm: "hierarchical_clustering"
    linkage: "ward"
    distance_metric: "1 - correlation"
    cut_threshold:
      correlation: 0.6
      description: "상관계수 > 0.6인 종목들을 같은 클러스터로 분류"

  output:
    cluster_labels: "종목별 클러스터 번호"
    dendrogram: "optional — 클러스터 계층 시각화"

  insight:
    same_cluster: "같은 클러스터 내 종목들은 유사하게 움직입니다."
    recommendation: "분산투자 강화를 위해 다른 클러스터의 종목을 추가하는 것을 고려할 수 있습니다."

  exceptions:
    fewer_than_8_stocks:
      action: "클러스터 분석 비활성화"
      message: "클러스터 분석에는 8종목 이상이 필요합니다."
    all_one_cluster:
      action: "경고 표시"
      message: "모든 종목이 하나의 클러스터에 속합니다 — 분산투자 효과가 제한적입니다."
```

---

## 5. Edge Cases

```yaml
edge_cases:
  single_stock:
    condition: "포트폴리오 내 종목이 1개"
    action: "correlation-diversification 모듈 전체 비활성화"
    message: "분산투자 분석에는 2종목 이상 필요합니다."

  same_sector_concentration:
    condition: "모든 종목이 동일 섹터"
    action: "경고 배너 표시"
    message: "동일 섹터 집중 — 섹터 리스크에 노출되어 있습니다."
    severity: "warning"

  insufficient_data:
    condition: "특정 종목의 거래일 < 60일"
    action: "해당 종목을 행렬에서 제외"
    message: "{ticker}은(는) 데이터 부족으로 상관관계 분석에서 제외되었습니다."
    display: "제외 종목 목록을 별도 표시"

  mixed_asset_class:
    condition: "암호화폐 + 주식 혼합 포트폴리오"
    action: "별도 계산 (거래시간 상이)"
    crypto_trading_days: 365
    stock_trading_days: 252
    message: "암호화폐와 주식은 거래시간이 달라 별도로 분석합니다."

  two_stocks_only:
    condition: "종목이 정확히 2개"
    action: "상관계수 1개만 표시 (행렬 대신 단일 값)"
    display: "단일 상관계수 카드 + 분산비율"
```

---

## 6. Safety Disclaimer

```yaml
safety_disclaimer:
  messages:
    - "상관관계는 과거 데이터 기반이며, 미래 상관관계를 보장하지 않습니다."
    - "시장 위기 시 상관관계가 급등하는 경향이 있습니다 (correlation breakdown)."
    - "분산투자는 손실을 완전히 방지하지 않습니다."
  display: "분석 결과 하단에 항상 표시"
  style: "muted text, smaller font size"
```
