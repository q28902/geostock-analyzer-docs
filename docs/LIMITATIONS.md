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

## 3. branch 머지 평가 임계값 부재 (정량화 필요)

LAYER3·LAYER4 모두 `feature/*` branch 미머지 상태. 평가 항목 명시:
- 매일 1~2회 호출 → 시장 결과와 대조
- rationale 품질 (LAYER2 변수 활용도)
- 모순 분석 일관성
- 비용 (subscription 한도 영향)

**부재**:
- "rationale 품질" 점수화 방법 (3점 척도? Claude critique?)
- "일관성" 측정 (같은 입력 3회 호출 시 결과 일치율)
- "시장 결과와 대조" 합격 임계값 (방향 70%? 등급 50%?)

**개선 후보**:
- 21-case harness (법률봇에서 검증된 패턴): 미리 정의된 21 시나리오 × 3회 평균
- 사전 합격 임계값 박제 — 후행 정성 결정 회피
- 4/3 채팅에서 사용자가 직접 발견했던 "1주 후 정성 결정" 함정 회피

## 4. LLM 비결정성 처리 부재

LAYER3·LAYER4·LAYER5 모두 Claude Opus 4.5 zero-shot 호출:
- temperature·seed 명시 X
- simulation v1 vs v2 결과 차이 = LLM 변동성 (외부 변동 X)

**영향**:
- 매일 18:00 daily snapshot 신뢰도 경계 모호
- 같은 시점 결과 차이가 데이터 변화인지 LLM 무작위인지 구분 불가
- "예측이 맞았다" 결론 신뢰성 약화

**개선 후보**:
- 각 layer 호출에 3회 평균 (비용 3배, 신뢰도 ↑)
- 또는 temperature 명시 (Claude CLI는 default 사용 — 명시 시 0 가능)
- daily snapshot에 `n_calls`·`agreement_pct` 메타 박기

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

1. **#5 Schema version** (5분, 즉시) → 다음 변경 시 silent break 방지
2. **#4 temperature=0** (10분) → 즉시 일관성 향상
3. **#1 검증 메타** (1~2시간) → 21-case harness 또는 즉시 메타 표시
4. **#3 평가 임계** (1~2시간) → 5/8 머지 결정 전에 정량화 필수
5. **#2 Mock 갭** (지속) → 외부 source wire-up은 진행 중, 잔여는 후속

---

이 문서는 시스템의 한계를 가리지 않기 위해 존재. 누락·새로 발견된 한계는 PR로 추가.
