# LAYER2 — 매크로 변수 + anomaly + LLM correction queue

## 역할

LAYER1 양자관계 + 실시간 시장 데이터 → 정량 변수 44개 + cross-rule anomaly + LLM correction 큐

## 변수 분포 (40/44 실 데이터, 90.9%)

- **slow_variables (16)**: 정책·금리·환율 등 (FRED 9 + ECOS derived 1 + KOMIS 3 + FAO 1 + mock 2)
- **fast_variables (24)**: yfinance 15 + KOMIS 6 + Apify 4 + 기타 (브랜치 도킹 시 27/29)

## 분류 로직

`thresholds.json` 임계값 — `normal | caution | alert` 3단계 분류 + momentum_flag (5d 변동률) + direction_flip (금리 방향 전환) + inverted (낮을수록 악화 지표)

## Cross-rule Anomaly

변수 간 충돌·동시 발생 패턴 검출. 예시:
- `oil_tariff_stagflation`: WTI alert + tariff escalating + CPI caution → 스태그플레이션 신호

## 출력

```python
@dataclass
class Layer2Output:
    asof: str
    layer1_context: dict           # L1 raw 전달 (해석 X)
    slow_variables: list[VariableState]
    fast_variables: list[VariableState]
    anomalies: list[CrossRuleHit]
    llm_correction_queue: list[dict]
    news_events: list[dict]        # 뉴스 raw 전달
```

## 2026-05-03 변경

- **`get_layer1_snapshot()` 헬퍼**: 실 LAYER1 엔진 우선 호출 + mock fallback
- env `LAYER1_USE_REAL_ENGINE=0` 시 mock 강제 (디버깅·롤백)
- sys.path 자동 추가 (LAYER1 패키지 venv install 안 돼도 작동)
