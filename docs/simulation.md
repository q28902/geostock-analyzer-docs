# 시뮬레이터 (LAYER3/scripts/simulate_scenario.py)

## 역할

가상 시나리오 주입으로 LAYER1~4 통합 검증. 코드 변경·운영 데이터 수정 없이 LLM이 어떤 분석 내놓는지 측정.

## 주입 가능 차원

```python
# L2 단계
INJECT_HEADLINES: list[str]              # news_events에 추가
INJECT_ANOMALIES: list[CrossRuleHit]     # cross-rule 충돌 추가

# KR 단계 (LAYER4)
KR_SECTOR_OVERRIDES: dict[str, float]    # 19 ETF 평시 등락 → 가상 등락 덮어쓰기
KR_INJECT_NEWS_CLUSTERS                  # news_intel KR cluster 추가
KR_INJECT_NEWS_KR_DAILY_POLITICS / SOCECON

# L1 단계
L1_KEY_INDICES_OVERRIDES                 # 양자관계 점수 덮어쓰기
L1_GRADE_OVERRIDES
L1_REGIONAL_PROXIMITY_KR_OVERRIDES       # 5 hotspot
L1_SUPPLY_CHAIN_EXPOSURE_KR_OVERRIDES    # 7 산업
L1_MILITARY_POSTURE_OVERRIDES            # 5 지표
L1_SANCTIONS_INTENSITY_OVERRIDES         # 5 카테고리
```

## 사용 예 — 5/4 대만전쟁 발발

```python
INJECT_HEADLINES = [
    "긴급 — 중국 인민해방군 5/4 새벽 대만 본토 상륙작전 개시",
    "미군 항모전단 대만해협 진입, 미중 군사 직접충돌 초읽기",
    "TSMC 가오슝 팹 가동 전면 중단",
    ...
]
KR_SECTOR_OVERRIDES = {"반도체": -8.0, "방산": +12.0, "정유": +8.0, ...}
L1_KEY_INDICES_OVERRIDES = {"us_china": 1.0, "korea_china": 1.0, "global_tension": 5.0, ...}
```

## 5/3 시뮬 v1~v5 누적 결과

| 회차 | 주입 범위 | L4 결과 | 핵심 발견 |
|---|---|---|---|
| v1 | L2만 | 보합 (디커플링) | KR ETF 평시 데이터로 LAYER4가 한국 시장 반응 약하게 평가 |
| v2 | L2 (재실행) | 공포 | LLM 변동성 |
| v3 | L2 + KR ETF 충격 | 공포 (동조) | KR side 충격 주입 시 정상 반응 |
| v4 | L2 + KR + L1 4점수 max | 공포 (변화 없음) | L1 4 점수 단순 박기는 LLM 무시 |
| v5 | L2 + KR + L1 다차원 + 동적 가중 | 공포 (19/19 섹터 rationale L1 직접 인용) | **L1 다차원 + crisis_mode + L4 SYSTEM_PROMPT 동적 가중**으로 풀 가동 |

## 결론

- LAYER4의 한국 시장 분석은 **KR 데이터(ETF·뉴스)** 가 가장 강한 시그널
- LAYER1 양자관계 4 점수만으론 **LLM이 무시** — 다차원 + crisis_mode flag + SYSTEM_PROMPT 동적 가중 룰 필요
- 시뮬은 운영 데이터 변경 없이 코드 robustness 검증에 유용
