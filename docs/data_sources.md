# 외부 데이터 source 4 wire-up (2026-05-03)

LAYER1 add-on lens 4 차원의 정성 baseline → 실 데이터로 wire-up. 모두 무료 또는 키 옵션.

## 1. UN Comtrade (supply_chain_exposure_kr)

**source**: `https://comtradeapi.un.org/public/v1/preview/`
**key**: 불필요 (preview API)
**파일**: `LAYER1/geopolitical_affinity/data_fetchers/comtrade_supply_chain.py`

차원 매핑 (v2 정밀화):
- `semiconductor_taiwan`: HS 8541+8542 KOR←TWN 수입 (TSMC 후공정 의존)
- `semiconductor_china`: HS 8541+8542 KOR→CHN 수출 (메모리)
- `battery_china`: HS 8506+8507+283699+282590 KOR←CHN 수입 (양극재·전구체)
- `auto_china`: HS 8703+8708 KOR→CHN 수출
- `display_china`: HS 8528 KOR→CHN 수출
- `rare_earth_china`: HS 280530+284690+850511+850519+720299 KOR←CHN 수입
- `shipbuilding_global`: HS 8901+8902+8904+8905 KOR→world 절대값

비중 → 척도: 0~10%=1.5 / 10~20%=2.5 / 20~30%=3.5 / 30~40%=4.0 / 40~55%=4.5 / 55%+=5.0

## 2. Federal Register (sanctions_intensity)

**source**: `https://www.federalregister.gov/api/v1/documents.json`
**key**: 불필요
**파일**: `LAYER1/geopolitical_affinity/data_fetchers/federal_register_sanctions.py`

90일 발표 빈도 카운트:
- `us_china_tech_export`: "China export control Entity List"
- `us_china_tariff`: "China tariff Section 301"
- `us_kor_steel_auto`: "Korea steel section 232"
- `china_rare_earth_export`: "rare earth China export control"

빈도 → 척도: 0~3=1.5 / 3~8=2.5 / 8~15=3.5 / 15~25=4.0 / 25~40=4.5 / 40+=5.0

한계: 미 정부 발표만, 한중·중일 양국 발표 부재. 빈도 ≠ 강도 (누적 정책 발표 적어도 강도 큼)

## 3. CEPII GeoDist + Comtrade (regional_proximity_kr)

**파일**: `LAYER1/geopolitical_affinity/data_fetchers/regional_proximity.py`

5 hotspot의 한국 직접성 = `(거리 score + 교역 boost) × LAYER1 양자관계 multiplier`

거리 (km, 정적):
- TWN 1488 / CHN 952 / JPN 1156 / PHL 2625 / VNM 3458 / PRK 195

multiplier: 양자관계 1=적대→1.5, 5=우호→0.7

## 4. GDELT BigQuery (military_posture)

**source**: `gdelt-bq.gdeltv2.events`
**key**: GCP service account 또는 ADC
**비용**: 무료 (1TB/월 quota 안)
**파일**: `LAYER1/geopolitical_affinity/data_fetchers/gdelt_military_posture.py`

EventRootCode 17~20 (COERCE/ASSAULT/FIGHT/UNCONVENTIONAL_VIOLENCE) 30일 빈도:
- `korean_defcon`: KOR/PRK 양국 군사 이벤트
- `us_indopacom_alert`: USA + Indo-Pacific 8국
- `japan_sdf_deployment`: JPN actor (정밀화 후속 — RootCode 18·19·20만)
- `china_pla_taiwan`: CHN-TWN 양방향
- `carrier_groups_in_region`: USA + East Asia geo

## GCP setup (military_posture 활성화용)

```bash
brew install --cask google-cloud-sdk
echo 'source /opt/homebrew/share/google-cloud-sdk/path.zsh.inc' >> ~/.zshrc
source ~/.zshrc

gcloud auth application-default login   # 브라우저 OAuth
gcloud config set project YOUR_PROJECT_ID
```

확인: `ls ~/.config/gcloud/application_default_credentials.json`

## DART API (LAYER5 disclosures)

**source**: `https://opendart.fss.or.kr/`
**key**: 무료, 회원가입 후 발급
**라이브러리**: `dart-fss`
**환경변수**: `DART_API_KEY` (또는 `LAYER5/.env`)
