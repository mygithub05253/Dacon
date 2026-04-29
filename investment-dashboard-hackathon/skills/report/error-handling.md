# Error Handling — 에러 상태 전체 매핑

> Complete error state map: pipeline errors by stage and graceful degradation levels 1-5.

---

## 1. Pipeline Errors

```yaml
error_state_map:
  description: "리포트 생성 과정에서 발생 가능한 모든 에러와 대응"

  pipeline_errors:
    - stage: "parsing"
      error: "파싱 완전 실패 (유효 데이터 0건)"
      report_action: "리포트 생성 불가 페이지 표시"
      message: "업로드된 파일에서 투자 데이터를 인식할 수 없습니다. 지원하는 형식을 확인해 주세요."
      show: ["파일 형식 가이드 링크", "샘플 파일 다운로드 버튼"]

    - stage: "parsing"
      error: "부분 파싱 성공 (일부 행 오류)"
      report_action: "성공 데이터로 리포트 생성 + 경고"
      message: "전체 {total}건 중 {failed}건의 데이터를 처리하지 못했습니다."
      footer_note: true

    - stage: "analysis"
      error: "지표 계산 일부 실패"
      report_action: "계산 성공 지표로 리포트 생성 + 해당 섹션 부분 표시"
      missing_metric_display: "N/A"

    - stage: "analysis"
      error: "지표 계산 전체 실패"
      report_action: "원시 데이터 테이블만 표시"
      sections_available: ["header", "stock_detail_table (원본 데이터)", "footer"]
      message: "분석에 필요한 데이터가 부족합니다. 원본 데이터를 확인해 주세요."

    - stage: "visualization"
      error: "특정 차트 렌더링 실패"
      report_action: "해당 차트의 fallback_chain 실행"
      final_fallback: "텍스트 요약 또는 숨김"

    - stage: "visualization"
      error: "전체 차트 렌더링 실패"
      report_action: "텍스트 전용 리포트 모드"
      message: "차트 생성에 실패했습니다. 텍스트 기반 분석 결과를 표시합니다."

    - stage: "insight"
      error: "인사이트 생성 실패"
      report_action: "인사이트 섹션 스킵, 나머지 정상 표시"
      notice: "인사이트 분석을 수행할 수 없었습니다."

    - stage: "report"
      error: "PDF 생성 실패"
      report_action: "대시보드 정상 + PDF 버튼에 에러 표시"
      alternative: "CSV 다운로드 또는 브라우저 인쇄(Ctrl+P) 안내"

    - stage: "report"
      error: "메모리 부족 (대용량 데이터)"
      report_action: "데이터 샘플링 후 재시도"
      threshold: "종목 500개 초과"
      sampling: "평가금액 상위 100개로 축소"
      message: "데이터가 너무 많아 상위 100개 종목으로 분석합니다."
```

---

## 2. Graceful Degradation Levels

```yaml
  graceful_degradation_levels:
    level_1:
      name: "완전 리포트"
      condition: "모든 파이프라인 정상"
      sections: "전체"

    level_2:
      name: "부분 리포트"
      condition: "일부 지표/차트 누락"
      sections: "사용 가능한 섹션만 + 누락 안내"

    level_3:
      name: "최소 리포트"
      condition: "분석 대부분 실패"
      sections: ["header", "kpi_summary (가능한 것만)", "stock_detail_table", "footer"]
      message: "제한된 데이터로 기본 리포트를 생성했습니다."

    level_4:
      name: "원시 데이터"
      condition: "분석 전체 실패"
      sections: ["header (에러 표시)", "stock_detail_table (원본)", "footer"]
      message: "분석에 실패하여 원본 데이터만 표시합니다."

    level_5:
      name: "에러 페이지"
      condition: "파싱부터 실패"
      sections: ["에러 메시지", "파일 형식 안내", "샘플 다운로드"]
```
