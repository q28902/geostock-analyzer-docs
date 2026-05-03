# LIMITATIONS — 현재 PoC 한계 및 정직 보고

> 외부 평가자·기여자가 시스템을 읽기 전에 이 문서를 먼저 읽기 권장.
> 모든 보고된 수치·시뮬·rationale의 신뢰도 경계가 명시됨.

## 1. 검증 메타데이터 부재 (가장 중요)

`docs/layer1.md` 안 LAYER1 적중률 표:

| 검증 | 적중률 |
|------|--------|
| 등급 적중 | 73% |
| 방향 적중 | 87% |
| 순위 적중 | 100% |
| **종합** | **87%** |

**누락**:
- 검증 데이터셋 (몇 년치, 몇 페어, 어떤 ground truth)
- baseline 비교 (random은 몇 %?)
- 측정일·측정자
- 표본 크기 — 특히 "순위 적중 100%"는 n이 작으면 자동 100%

**현재 상태**: 이전 세션 측정값 인용, 메타데이터 미명시. **위 수치는 절대 마케팅 자료로 사용 금지**. 신규 측정 필요.

**개선 후보**: 21-case harness(법률봇 패턴) + 3-repetition averaging + baseline(random affinity 분포) 동시 측정.

## 2. Mock 데이터 잔존

PoC 단계에서 일부 변수는 실데이터 wire-up 안 됨:

| 위치 | Mock 변수 | 영향 |
|------|----------|------|
| L2 slow_variables (16/16) | mock 2 | 14/16 실데이터 (87.5%) |
| L4 외인 매매 | pykrx 차단 → Naver 크롤링 우회 (5/3) | watchlist 종목만, universe sector_top은 부재 |
| L4 한국 정치·사경 뉴스 | n8n news_kr_daily (5/2 수정 후 운영) | 일부 누락 케이스 가능 |
| L1 add-on lens baseline 22 값 | 정성 추정 (외부 source 4종 wire-up으로 대체 진행 중) | 외부 source 활성 시 derived |

**시뮬레이션 v5 "L4 풀 가동" 표현의 한계**:
LAYER1 다차원 + crisis_mode + L4 SYSTEM_PROMPT 동적 가중까지 활성화된 상태이지만, **외인 매매 데이터가 watchlist 한정**이라 universe 종목 분석에선 그 데이터 없이 LLM이 추정. "풀 가동"은 LAYER1·LAYER4 분석 메커니즘 풀 가동일 뿐, 입력 데이터 풀 가용은 아님.

## 3. branch 머지 평가 — 본질은 추적·교정 사이클 (임계는 trigger)

**사용자 지적 정정**: 임계값 박는 것 자체가 목적이 아니다. 본질은:
- **티커별 raw % 차이 누적 추적** (예측 midpoint vs 실제 등락률)
- **교정 전후 cohort 비교**로 차이 감소량 정량화
- 임계는 "교정 여부 결정 trigger" + "회귀 알림 기준"일 뿐

**구현** (5/3 적용):
- `_eval_market()`에 `abs_diff` per ticker 박음 — `pred_midpoint`(라벨→%) vs `actual_pct` 절대 차이
- 시장 평균 `avg_abs_diff` + `max_abs_diff` 메트릭 — 교정 baseline
- `compare_cohorts(log_path, cohort_field, market)` — 누적 `_log.jsonl`에서 cohort tag별 평균 abs_diff 비교 + delta 계산 + regression 알림 (after > before면 회귀)
- `LABEL_MIDPOINT` dict — 강한호재 +5%, 호재 +1.75%, 중립 0%, 악재 -1.75%, 강한악재 -5% (typical 중간값)

**측정 사이클**:
1. forecast 생성 시 cohort tag 자동 부착 (`prompt_version`/`schema_version`/`model`/`git_sha` 등 — 후속)
2. 매 영업일 actual 주입 후 평가 → `_log.jsonl`에 sector별 `abs_diff` 누적
3. 교정 (예: SYSTEM_PROMPT 변경, RAG 임베딩 도입) 시 cohort 분기
4. `compare_cohorts()` 로 교정 전후 평균 abs_diff 차이 측정
5. 회귀 발생 시 (after > before) 즉시 알림 → 교정 롤백 결정

**MERGE_THRESHOLDS는 trigger**: 직관적 합격선 (방향 70%/강도 60%/avg_abs_diff <2%p 등). 단 본질은 추적·축적·정량 교정 효과.

**5/3 추가 적용**:
- ✅ `generate_forecast.py`에 `build_cohort_tag()` — forecast JSON에 `cohort` 자동 부착:
  - `prompt_sha` (LAYER3·4 SYSTEM_PROMPT sha, 8자리) — prompt 변경 시 자동 cohort 분기
  - `git_sha` (current commit short SHA) — 코드 변경 추적
  - `schema_l3`/`schema_l4` — 호환 깨질 변경 추적
  - `tagged_at` (UTC ISO)
- 다음 forecast부터 cohort tag 박힘 → `compare_cohorts(cohort_field="cohort.prompt_sha")` 같이 호출 가능
- 매 영업일 누적 → 교정 전후 평균 abs_diff 차이 정량화 가능

**21-case harness 폐기 (5/3)**:
- 사용자 명시 정정: 21 시나리오 정의가 본질 X
- 본질 = 매일 자연 발생하는 forecast vs actual 누적 + cohort 비교
- 인위적 시나리오 generation 불필요 — 자연 운영 데이터로 충분

## 4. LLM 비결정성 처리 부재

LAYER3·LAYER4·LAYER5 모두 Claude Opus 4.5 zero-shot 호출:
- temperature·seed 명시 X
- simulation v1 vs v2 결과 차이 = LLM 변동성 (외부 변동 X)

**영향**:
- 매일 18:00 daily snapshot 신뢰도 경계 모호
- 같은 시점 결과 차이가 데이터 변화인지 LLM 무작위인지 구분 불가
- "예측이 맞았다" 결론 신뢰성 약화

**개선 후보**:
- ⚠️ **Claude CLI는 `--temperature` 옵션 미지원** (검증 끝남: `claude -p --help` 결과 `--model`만). SDK 직접 호출이 아니라 CLI 사용 한 LAYER3·4·5는 temperature 강제 불가
- **2026-05-03 적용**: `DETERMINISTIC_NOTE` 상수 — SYSTEM_PROMPT에 "결정론적 출력 강제" 자연어 박음. 효과 약하지만 비용 0
- **추가 후보 (보수적)**: 각 layer 호출에 3회 평균 (비용 3배, agreement_pct 메타 박기). Layer3/4/5 dataclass에 `n_calls`/`determinism_note` 필드 이미 추가 완료
- **근본 해결**: Anthropic SDK 직접 호출로 전환 + `temperature=0` (CLI subscription → API 비용 모델 변경 큰 작업)

## 5. Layer 간 인터페이스 schema 버전 관리 부재

현재 dataclass:
```python
@dataclass
class Layer2Output:
    asof: str
    layer1_context: dict
    slow_variables: list[VariableState]
    # ... schema_version 필드 없음
```

**위험**:
- L2 스키마 변경 시 L3가 silent break
- 캐시·로그 파일이 옛 schema이면 분석 불가
- 사용자가 4/20 채팅에서 "두 채팅창 schema 불일치" 직접 우려한 패턴이 코드에서 잠복

**개선 후보** (5분 작업):
- 각 Layer Output dataclass에 `schema_version: str = "1.0"` 추가
- L3는 받은 L2의 버전을 검증 (`assert l2.schema_version == "1.0"`)
- 호환 깨질 변경 시 minor → major bump

## 6. 재무·공시 데이터 시계 미스매치 (5/3 적용)

LAYER5 입력은 시계가 다른 데이터의 혼합:
- OHLCV: 실시간 (전일 close)
- 외인·기관 매매: 5d/30d 누적
- PER/PBR/EPS/BPS: **분기 또는 연간 결산** (3개월+ lag 가능)
- 공시: 즉시

**위험**: prompt에 시점 라벨 없으면 LLM이 분기 데이터를 실시간으로 오인. 1Q 결과를 4Q 시장 분석에 쓰는 위험.

**5/3 적용**:
- `StockFundamentals`에 `as_of_period` 필드 (예: "2025.12") + `fetched_at` (UTC ISO)
- Naver 메인 페이지 th `(2025.12)` 패턴 추출
- `_stock_to_dict`/`_stock_to_dict_brief` 둘 다 prompt payload에 박음
- SYSTEM_PROMPT 분석 원칙 11번: "as_of XXXX.XX 데이터 기준 — 이미 시장에 반영된 정보" rationale 명시 의무

## 7. portfolio_weight 결정론적 정규화 (5/3 적용)

LAYER5 출력의 `portfolio_weight: 0~1`을 LLM이 통째로 결정하던 위험:
- Kelly·equal-weight·confidence-weighted 어느 것? 명세 없음
- 합계 1.0 보장 X
- 기존 holdings 비중 reconcile X

**5/3 적용**:
- `normalize_portfolio_weights(stocks, holdings_map, cash_buffer=0.10)` 후처리 함수
- 공식: `raw = max(0, SIGNAL_SCORE) × CONFIDENCE_WEIGHT` (long-only)
  - 강한매수=2.0 / 매수=1.0 / 중립·매도·강한매도=0
  - 높음=1.0 / 중간=0.7 / 낮음=0.4
- `weight = raw / sum(raw) × (1 - cash_buffer)` (현금 buffer 10%)
- 합계 ≤ 0.9 보장
- LLM이 weight 출력해도 무시·덮어씀 — signal·confidence만 사용
- SYSTEM_PROMPT 분석 원칙 12번에 명시

**검증**: 4 stock (강매수+높음/매수+중간/중립/매도) → 0.667/0.233/0/0 합계 0.900 정상.

## 8. 법적 안전장치 — 개인 연구용 명시 (5/3 적용)

LAYER5가 매수·매도 신호 직접 생성 → 자본시장법상 유사투자자문업 해석 여지. 자기 사용 한정이라도 명시 안전.

**5/3 적용**:
- LAYER5 README 최상단 ⚠️ Disclaimer 섹션:
  - "research signals for personal study only. NOT investment advice."
  - 자본시장법 투자권유·자문 X
  - 자기 사용 한정. 외부 배포·상업화 금지
  - LLM 비결정성·데이터 lag·mock 잔존 → reference only
- 모든 Telegram 알림 (poc_run·analyze_stock) 끝에 면책 문구 박음:
  - "⚠️ 개인 연구용 신호. 투자 자문 X. 손익 책임 사용자."
- watchlist.json `.gitignore` 강제 (개인 보유 정보 절대 공개 X) — 양 repo 모두

---

## 운영 의사결정에 미치는 영향

| 보완 | 즉시 위험 | 중기 위험 |
|------|----------|----------|
| 1. 검증 메타데이터 | 외부 신뢰도 X | 채용·OSS 평가 시 부정 평가 |
| 2. Mock 잔존 | 시뮬 결과 oversell 위험 | 실 시장 적용 시 정확도 갭 |
| 3. 평가 임계 | 5/8 머지 결정 자의적 | 시스템 수명 단축 |
| 4. LLM 비결정성 | 매일 결과 신뢰도 모호 | 백테스트 무의미 |
| 5. Schema 버전 | 다음 schema 변경 시 silent break | 디버깅 비용 누적 |

## 우선순위 (보수적)

1. ✅ **#5 Schema version** (5/3 적용) — Layer2~5 dataclass `schema_version` + run_layer3/4/5 진입 시 호환 검증
2. ✅ **#4 DETERMINISTIC_NOTE prompt** (5/3 적용) — CLI temperature 미지원 → 자연어 강제. n_calls·determinism_note 메타 필드 박음
3. ✅ **#3 추적·교정 사이클** (5/3 적용) — abs_diff per ticker + compare_cohorts + cohort tag 자동 부착 (prompt_sha/git_sha/schema)
4. ❌ **#1 21-case harness — 폐기** (5/3) — 본질 = 자연 운영 누적 + cohort 비교. 인위 시나리오 불필요
5. **#2 Mock 갭** (지속) — 외부 source wire-up 진행 중, 잔여 후속
6. ✅ **#6 시계 라벨** (5/3 적용) — `as_of_period` "2025.12" 추출 + prompt 박음
7. ✅ **#7 portfolio_weight 결정론** (5/3 적용) — `normalize_portfolio_weights` confidence-weighted long-only + cash buffer 10%
8. ✅ **#8 법적 안전장치** (5/3 적용) — README Disclaimer + 알림 면책 문구 + watchlist.json gitignore

---

이 문서는 시스템의 한계를 가리지 않기 위해 존재. 누락·새로 발견된 한계는 PR로 추가.
