# Architecture (v2 + 2026-05-03 확장)

## 데이터 흐름

```
[외부 데이터]
  FBIC / GDELT / Comtrade / Federal Register / CEPII / DART / FDR / Naver / yfinance / FRED / KOMIS / ECOS / FAO
       │
       ▼
LAYER1 ── 양자관계 affinity 1,122 페어 + key_indices(10) + add-on lens
       │
       │  ┌─ kr_macro_lens (4 차원, 한국 시각)
       │  └─ global_hotspots (6 hotspot, 모든 시장)
       ▼
LAYER2 ── 매크로 변수 44개 + anomaly + news_events + LLM correction queue
       ▼
LAYER3 ── Claude Opus 4.5 zero-shot
       │  ├─ EVENTCARD RAG (303장 + bge-m3 임베딩 hybrid)
       │  └─ 출력: market_phase + GICS 11섹터 (impact·confidence·rationale)
       ▼
LAYER4 ── KR data + Claude Opus 4.5
       │  ├─ KR sector ETF 19종 + ECOS 6 + n8n news_kr_daily + news_intel KR cluster
       │  ├─ crisis_mode 자동 판정 (L1 변수 임계)
       │  └─ 출력: KR 19섹터 + market_phase
       ▼
LAYER5 ── 종목 단위 trade signal
          ├─ universe = sector_top5 (95) + watchlist (사용자)
          ├─ 데이터: OHLCV(FDR) + Flow(Naver) + Fundamentals(Naver+yfinance) + 컨센서스(Naver, watchlist만) + Disclosures(DART)
          └─ 두 모드: briefing (매일 자동) / deep (사용자 ad-hoc) / single_stock_deep (단·중·장)
```

## LAYER1 add-on lens 메커니즘

```python
# engine.py 끝에 try/except wrap
def get_snapshot(year=None):
    out = {  # 기존 키 (변동 X)
        'year': year, 'timestamp': ..., 'n_pairs': ...,
        'pairs': pairs,  # 1,122 양자관계 affinity 0~100
        'key_indices': {us_china, korea_china, ...},
    }
    try:
        out.update(kr_macro_lens.compute_all_kr_dimensions(out))
    except Exception as e:
        out["_kr_lens_error"] = str(e)
    try:
        out.update(global_hotspots.compute_all_global_hotspots(out))
    except Exception as e:
        out["_global_hotspots_error"] = str(e)
    return out
```

기존 affinity·grade·pairs 산출은 add-on이 fail해도 영향 없음.

## crisis_mode 자동 판정 (LAYER4)

```python
crisis = (
    global_tension >= 4.5                              # 긴장 척도 max
    or min(korea_china, korea_japan) <= 1.5            # 한국 직결 양자 적대
    or us_china <= 1.3                                 # 미중 극단 적대
    or DEFCON <= 3                                     # 군사 격상
    or china_pla_taiwan >= 4.5                         # 대만 군사 활동
)
```

평시 → 가중 L3 65~75% / L2 15~25% / L1 5~10%
위기 → **L1 35~45% 1순위 격상** + L3 25~35% + KR ETF 5~15% (위기 시 ETF 평시 데이터일 가능성으로 신뢰도 하락)

## 운영 자동화

- 매 영업일 18:00 KST: L2 → L3 daily snapshot (launchd `com.layer3.daily`)
- 금 17:00 KST: 다음 영업일 forecast 박제 (`com.layer3.weekly-forecast`)
- 화 09:00 KST: 직전 월요일 close 자동 평가 (`com.layer3.eval-forecast`)
- LAYER5 launchd 등록 = 잔여
- 알림: Geostock_bot 통합 채팅창
