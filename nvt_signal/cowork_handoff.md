# Cowork 이관 작업 (4차): 데이터 피드 확인과 4열(v2) 계산식 찾기

아래 작업을 순서대로 수행하고, 결과를 CSV 파일과 요약 문서(`nvt_tv_findings4.md`)로 정리해줘. 작업 A와 B는 필수이고, 작업 C는 시간이 남으면 수행한다.

## 배경

- 차트: https://www.tradingview.com/chart/X3L1Uw1v/ (INDEX:BTCUSD). 차트 소유 계정(`tlzkrh1029`)으로 로그인한 상태에서 수행한다.
- 2열: gliderfund "NVT - Network Value to Transactions" **v1**. 계산식은 `BITSTAMP:BTCUSD 종가 × QUANDL:BCHAIN/TOTBC / SMA(QUANDL:BCHAIN/ETRVU, 90)`으로 확정되었다. 그런데 `QUANDL:BCHAIN/*` 피드는 2026-06-22 이후 새 봉이 없어서, 그 뒤로는 NVT가 "가격 × 상수"가 되었다.
- 4열: 같은 스크립트의 **v2**(2024-11-05, 설정 Transaction Period 90, Force Daily View = true, Length Signal 13). 작성자는 "온체인 거래량(On-Chain Bitcoin Volume)으로 바꿨다"고만 밝혔다. 이미 알려진 사실은 다음과 같다.
  - 06-22 이후에도 값이 계속 바뀐다.
  - 2009-10-05(차트 첫 봉)부터 값이 있다.
  - 마지막 며칠은 가격이 움직여도 값이 같다(예: 2026-09-23~25 모두 28.112952). 즉 실시간 가격이 아니라 늦게 갱신되는 온체인 데이터만 쓰는 것으로 보인다.
  - 값의 크기는 2열의 약 1/10이다.
- TradingView 심볼 검색에서 찾은 후보 심볼:
  - Quandl 대체 피드: `BCHAIN:ETRVU`, `BCHAIN:TOTBC`, `BCHAIN:MKTCP`
  - 온체인 거래량: `INTOTHEBLOCK:BTC_TXVOLUMEUSD`, `COINMETRICS:BTC_TXVOLUMEUSD`, `INTOTHEBLOCK:BTC_TXVOLUME`, `COINMETRICS:BTC_TXVOLUME`
  - 시가총액·공급량: `GLASSNODE:BTC_MARKETCAP`, `COINMETRICS:BTC_MARKETCAPFF`, `GLASSNODE:BTC_SUPPLY`
  - NVT 자체: `INTOTHEBLOCK:BTC_NETWORKVALUETOTRANSACTION`
- 추출 방법은 이전과 같다(`TradingViewApi` → `model().dataSources()`에서 plot 값을 봉 단위로 읽기, 과거 데이터를 끝까지 불러온 뒤 추출). plot은 제목으로 짝을 맞춘다.
- 원본 지표(2열, 4열)의 설정은 작업 C를 제외하고 바꾸지 않는다.

## 작업 A: 원시 데이터 추출 (필수)

1. Pine 편집기에서 아래 "진단 코드"를 새 지표로 추가한다. 저장하거나 게시하지 않는다.
2. 차트를 **1D**로 두고, 진단 지표의 모든 plot과 2열 `NVT`, 4열 `NVT`, 차트 `close`를 봉 단위로 추출한다. 과거 데이터는 끝까지 불러온다.
3. 심볼마다 다음을 기록한다.
   - 심볼이 열리는지 여부. 열리지 않는 심볼은 값이 전부 na로 나온다.
   - 첫 값 날짜와 마지막으로 값이 **바뀐** 날짜
   - 오늘 기준으로 며칠 늦게 갱신되는지
4. 1W로 바꿔서 같은 추출을 한 번 더 한다.
5. 끝나면 진단 지표를 제거한다.

## 작업 B: 분석 (필수)

작업 A의 CSV로 다음을 계산한다.

1. **Quandl과 새 BCHAIN 피드 비교**: `QUANDL:BCHAIN/ETRVU`와 `BCHAIN:ETRVU`, `TOTBC`끼리, `MKTCP`끼리 겹치는 구간의 비율(중앙값, 최소~최대)을 구한다. 새 BCHAIN 피드가 2026-06-22 이후에도 갱신되는지 확인한다.
2. **v1 부활 가능성**: 진단 지표의 `v1 with BCHAIN` plot이 2026-06-22 이전 구간에서 2열 NVT와 얼마나 같은지(비율의 중앙값, 5~95% 구간, 최대 오차), 그리고 06-22 이후에도 `NVT ÷ BITSTAMP 종가`가 매일 바뀌는지 확인한다.
3. **4열(v2) 계산식 찾기**: 아래 후보를 4열 NVT와 비교해서, 연도별 중앙 상대 오차가 가장 작은 조합을 찾는다. 모든 이동평균은 90일, 데이터 심볼 자체의 일봉 기준이다. 필요하면 1~2일 시차도 시험한다.
   - 분자: `GLASSNODE:BTC_MARKETCAP`, `COINMETRICS:BTC_MARKETCAPFF`, `BCHAIN:MKTCP`, `GLASSNODE:BTC_SUPPLY × 가격(BITSTAMP, 차트 종가)`
   - 분모: `SMA90(INTOTHEBLOCK:BTC_TXVOLUMEUSD)`, `SMA90(COINMETRICS:BTC_TXVOLUMEUSD)`, `SMA90(INTOTHEBLOCK:BTC_TXVOLUME × 가격)`, `SMA90(COINMETRICS:BTC_TXVOLUME × 가격)`, `SMA90(BCHAIN:ETRVU)`
   - 비교 대상: `INTOTHEBLOCK:BTC_NETWORKVALUETOTRANSACTION`의 90일 SMA
   - 진단 코드에는 대표 조합 몇 개를 미리 계산해 두었다(`v2 cand` plot). 나머지 조합은 원시 값으로 직접 계산한다.
4. 오차가 1e-6 이하인 조합을 찾으면 그 조합을 "확정"으로, 찾지 못하면 가장 가까운 조합과 남은 오차의 패턴(특정 기간, 일정한 배율 등)을 기록한다.

## 작업 C: v2의 이동평균 길이 확인 (선택)

1. 4열(v2)의 Transaction Period를 90에서 30으로 바꾸고 1D에서 4열 NVT를 추출한다.
2. 작업 B에서 찾은 조합의 SMA 길이를 30으로 바꿔서 같은지 확인한다.
3. 끝나면 Transaction Period를 **90으로 되돌린다.**

## 마무리

- 진단 지표를 차트에서 제거한다.
- 2열과 4열의 설정이 처음 상태와 같은지 확인한다(4열 Transaction Period 90).
- 시간 단위를 원래대로 1M으로 되돌린다.

## 결과물

- `tv_nvt4_raw_1D.csv`, `tv_nvt4_raw_1W.csv`: 날짜, close, 2열 NVT, 4열 NVT, 진단 지표의 모든 plot(열 이름에 plot 제목 사용)
- `tv_nvt4_v2_period30_1D.csv`: 작업 C 결과(수행한 경우)
- `nvt_tv_findings4.md`: 심볼별 상태 표, Quandl과 BCHAIN 비교, v1 부활 가능성, v2 계산식 후보별 오차 표와 결론

## 진단 코드 (Pine v6)

```pine
//@version=6
indicator("NVT feed diagnostic", shorttitle = "NVT-diag4", overlay = false)

// Every series is taken from daily bars of its own symbol. Unavailable symbols return na instead of an error.
d(sym) => request.security(sym, "D", close, ignore_invalid_symbol = true)
dsma(sym) => request.security(sym, "D", ta.sma(close, 90), ignore_invalid_symbol = true)

bs      = d("BITSTAMP:BTCUSD")
qVol    = d("QUANDL:BCHAIN/ETRVU")
qSup    = d("QUANDL:BCHAIN/TOTBC")
qCap    = d("QUANDL:BCHAIN/MKTCP")
bVol    = d("BCHAIN:ETRVU")
bSup    = d("BCHAIN:TOTBC")
bCap    = d("BCHAIN:MKTCP")
itbVol  = d("INTOTHEBLOCK:BTC_TXVOLUMEUSD")
cmVol   = d("COINMETRICS:BTC_TXVOLUMEUSD")
itbVolN = d("INTOTHEBLOCK:BTC_TXVOLUME")
cmVolN  = d("COINMETRICS:BTC_TXVOLUME")
gnCap   = d("GLASSNODE:BTC_MARKETCAP")
cmCapFF = d("COINMETRICS:BTC_MARKETCAPFF")
gnSup   = d("GLASSNODE:BTC_SUPPLY")
itbNvt  = d("INTOTHEBLOCK:BTC_NETWORKVALUETOTRANSACTION")

bVolSma   = dsma("BCHAIN:ETRVU")
itbVolSma = dsma("INTOTHEBLOCK:BTC_TXVOLUMEUSD")
cmVolSma  = dsma("COINMETRICS:BTC_TXVOLUMEUSD")
itbNvtSma = dsma("INTOTHEBLOCK:BTC_NETWORKVALUETOTRANSACTION")

plot(bs,      "raw BITSTAMP close")
plot(qVol,    "raw QUANDL ETRVU")
plot(qSup,    "raw QUANDL TOTBC")
plot(qCap,    "raw QUANDL MKTCP")
plot(bVol,    "raw BCHAIN ETRVU")
plot(bSup,    "raw BCHAIN TOTBC")
plot(bCap,    "raw BCHAIN MKTCP")
plot(itbVol,  "raw ITB TXVOLUMEUSD")
plot(cmVol,   "raw CM TXVOLUMEUSD")
plot(itbVolN, "raw ITB TXVOLUME")
plot(cmVolN,  "raw CM TXVOLUME")
plot(gnCap,   "raw GN MARKETCAP")
plot(cmCapFF, "raw CM MARKETCAPFF")
plot(gnSup,   "raw GN SUPPLY")
plot(itbNvt,  "raw ITB NVT")

plot(bs * bSup / bVolSma, "v1 with BCHAIN")
plot(gnCap / itbVolSma,   "v2 cand GN cap / ITB vol")
plot(gnCap / cmVolSma,    "v2 cand GN cap / CM vol")
plot(cmCapFF / cmVolSma,  "v2 cand CM capFF / CM vol")
plot(bCap / itbVolSma,    "v2 cand BCHAIN cap / ITB vol")
plot(itbNvtSma,           "v2 cand SMA90 of ITB NVT")
```
