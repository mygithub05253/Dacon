# Benchmarks — Comparison, Mode Switching & Extensions

> Defines benchmark indices, comparison metrics, analysis mode auto-switching,
> and extension points for new asset classes.
> Source: `analysis_rules.md` Sections 5 + 6 + 7

---

## 1. Benchmark Definitions

```yaml
benchmarks:
  KR:
    - name: "KOSPI"
      description: "국내 대표 지수 (유가증권시장)"
      default: true
      ticker: "^KS11"
    - name: "KOSDAQ"
      description: "코스닥 지수"
      ticker: "^KQ11"
    - name: "KOSPI200"
      description: "대형주 200종 지수"
      ticker: "^KS200"

  US:
    - name: "S&P 500"
      description: "미국 대형주 500종 지수"
      default: true
      ticker: "^GSPC"
    - name: "NASDAQ"
      description: "나스닥 종합 지수"
      ticker: "^IXIC"
    - name: "DOW"
      description: "다우존스 산업평균"
      ticker: "^DJI"

  selection_rule:
    single_market: "해당 시장의 default 벤치마크"
    mixed_market: "비중 기준 주력 시장의 벤치마크 + 보조 시장 벤치마크 병기"
```

---

## 2. Comparison Metrics

```yaml
comparison_metrics:
  - excess_return:
      formula: "portfolio_return - benchmark_return"
      label: "초과수익률"
  - tracking_error:
      formula: "std(daily_excess_return) * sqrt(252)"
      label: "추적 오차"
  - information_ratio:
      formula: "annualized_excess_return / tracking_error"
      label: "정보 비율"
      exception:
        tracking_error_zero: "information_ratio = N/A"

exceptions:
  benchmark_data_missing:
    action: "벤치마크 비교 섹션 전체 비활성화"
    message: "벤치마크 데이터를 가져올 수 없어 비교 분석을 제공할 수 없습니다."
    fallback: "벤치마크 없이 절대 수익률만 표시"
  period_mismatch:
    description: "포트폴리오 기간과 벤치마크 기간 불일치"
    action: "겹치는 기간만 비교. 제외 기간 명시."
```

---

## 3. Analysis Mode Auto-Switching

```yaml
analysis_modes:
  description: "데이터 특성에 따라 분석 깊이를 자동 조절"

  full_mode:
    condition: "보유 현황 + 거래 내역 모두 존재, 종목 5개 이상, 30일+ 데이터"
    features: "모든 기본 지표 + 고급 지표 + 벤치마크 비교 + 전체 시각화"

  standard_mode:
    condition: "보유 현황만 존재, 종목 3개 이상"
    features: "기본 지표 + 집계 분석 + 기본 시각화"
    disabled: "시계열 차트, 샤프/MDD/베타, 벤치마크 비교"

  minimal_mode:
    condition: "보유 현황만 존재, 종목 1~2개"
    features: "종목별 수익률, 평가금액, 손익만 표시"
    disabled: "분산도 분석, 섹터 분석, 대부분의 차트"

  trade_history_mode:
    condition: "거래 내역만 존재 (보유 현황 없음)"
    action: "거래 내역에서 현재 보유 현황을 재계산"
    calculation: "매수 수량 합산 - 매도 수량 합산 = 현재 보유량"
    avg_price_calc: "가중평균 매수가 (FIFO 또는 이동평균)"

  mode_display:
    show_current_mode: true
    upgrade_suggestion: "더 상세한 분석을 위해 거래 내역도 업로드해보세요."
```

---

## 4. Extension Points

```yaml
extension_points:
  new_asset_metrics:
    crypto:
      additional_metrics:
        - market_cap_weight: "시가총액 기준 비중"
        - staking_yield: "스테이킹 수익률 (APY)"
        - impermanent_loss: "LP 비영구적 손실"
        - dominance: "전체 크립토 시장 대비 비중"
      modified_parameters:
        annualize_factor: 365   # 24/7 trading
        risk_free_rate: 0       # no crypto base rate

    bond:
      additional_metrics:
        - ytm: "만기수익률 (Yield to Maturity)"
        - duration: "듀레이션 (금리 민감도)"
        - modified_duration: "수정 듀레이션"
        - coupon_income: "이자 수익 (연간)"
        - accrued_interest: "경과 이자"
      modified_parameters:
        return_includes_coupon: true
        benchmark: "국고채 금리 곡선"

    real_estate:
      additional_metrics:
        - cap_rate: "자본환원율 (NOI / 매입가)"
        - nav_premium: "NAV 대비 프리미엄/할인율"
        - dividend_yield: "배당수익률"
        - occupancy_rate: "임대율"
      modified_parameters:
        annualize_factor: 12    # monthly data

    fund:
      additional_metrics:
        - alpha: "젠센의 알파 (초과수익)"
        - r_squared: "R² (벤치마크 설명력)"
        - expense_impact: "보수 차감 영향"
        - tracking_difference: "벤치마크 추적 차이"

  dividend_handling:
    description: "배당금 처리 규칙"
    modes:
      total_return: "배당 재투자 가정 (총수익률)"
      price_return: "배당 제외 (가격 수익률만)"
    default: "total_return"
    detection: "trade_type에 DIVIDEND가 있으면 배당 내역 포함"

  stock_split_handling:
    description: "액면분할/병합 처리"
    detection: "같은 종목의 가격이 전일 대비 40% 이상 급변"
    action: "경고 표시. 자동 보정은 하지 않음 (데이터 신뢰성 이슈)."
    message: "'{name}' 가격이 급변했습니다. 액면분할/병합이 있었다면 매수가를 조정해주세요."

  custom_benchmark:
    description: "사용자 정의 벤치마크 추가 가능"
    input_format: "CSV (date, return 컬럼)"
    validation: "날짜 형식 검증, 수익률 범위 검증 (-100~+500%)"
```
