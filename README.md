# Geostock Analyzer

> 지정학 → 매크로 → 미국 증시 → 한국 증시 → 한국 종목까지 5층 분석 파이프라인.

**Status**: PoC + 운영 자동화 (2026-05-03 기준)
**Code**: private (이 repo는 docs only)
**Docs**: 아키텍처·data source·운영 흐름 공개

## 5-Layer 구조

```
LAYER1 ─ 지정학 양자관계 + 한국 시각 lens (kr_macro_lens) + 글로벌 hotspot
   │
LAYER2 ─ 매크로 변수 44개 (yfinance + FRED + KOMIS + ECOS + FAO)
   │
LAYER3 ─ 미국 증시 GICS 11섹터 (Claude Opus zero-shot + EVENTCARD RAG)
   │
LAYER4 ─ 한국 증시 19섹터 (LLM, KR ETF + n8n news + crisis_mode 동적 가중)
   │
LAYER5 ─ 한국 종목 단위 trade signal (홀딩·관심·시총상위 universe + 컨센서스)
```

## 핵심 특징

- **add-only lens 패턴**: LAYER1 본체 양자관계(0~100 affinity) 산출은 그대로 두고, 한국 시각·글로벌 hotspot 차원을 별 모듈로 추가. fail시 fallback.
- **외부 데이터 4 wire-up**: UN Comtrade / Federal Register / CEPII GeoDist / GDELT BigQuery — 모두 무료·키 옵션
- **LLM 동적 가중**: LAYER1 `crisis_mode` 자동 판정 (global_tension≥4.5 또는 한국 직결 양자관계≤1.5) → LAYER4 SYSTEM_PROMPT가 위기 시 L1 비중 35~45% 1순위 격상
- **두 모드 분리**: LAYER5 `briefing` (매일 자동, 78% 토큰 절감) vs `deep` (사용자 ad-hoc) vs `analyze_single_stock` (단일 종목 5섹션 분석)
- **시뮬레이터**: `simulate_scenario.py` — 가상 시나리오(대만전쟁 등) 주입으로 LAYER1~4 통합 검증

## Documents

- [Architecture](docs/architecture.md) — 5층 데이터 흐름 + 입력·출력 스키마
- [LAYER1 — 지정학 affinity + add-on lens](docs/layer1.md)
- [LAYER2 — 매크로 변수](docs/layer2.md)
- [LAYER3 — 미국 증시 + EVENTCARD RAG](docs/layer3.md)
- [LAYER4 — 한국 증시 + crisis_mode](docs/layer4.md)
- [LAYER5 — 한국 종목 분석](docs/layer5.md)
- [Data Sources](docs/data_sources.md) — 외부 4 source 설정 가이드
- [Simulation](docs/simulation.md) — 시나리오 주입 사용법

## License

MIT (docs only). 코드는 별도 private repo.
