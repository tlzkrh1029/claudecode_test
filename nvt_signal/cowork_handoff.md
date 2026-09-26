# Cowork 이관 작업 (3차): 수정한 재구성 코드의 최종 검증

아래 작업을 순서대로 수행하고, 결과를 CSV 파일과 요약 문서(`nvt_tv_findings3.md`)로 정리해줘. 작업 A와 작업 B는 필수이고, 작업 C는 시간이 남으면 수행한다.

## 배경

- 차트: https://www.tradingview.com/chart/X3L1Uw1v/ (INDEX:BTCUSD). 차트 소유 계정(`tlzkrh1029`)으로 로그인한 상태에서 수행한다.
- 원본(2열): gliderfund "NVT - Network Value to Transactions" v1, `PUB;mGO6NQAZK4oHjEzHPwWWWTIYWDGJkBQF`. 현재 설정은 Transaction Period 90, Force Daily View = false, Length Signal 13, Manual Levels = false, Show Crosses = false, Show Labels = false이다.
- 2차 작업에서 확정한 원본 규칙을 반영해서 재구성 코드를 수정했다(아래 "재구성 코드" 참고). 수정 내용은 다음과 같다.
  - 분봉 기준선 200/60 추가
  - 기준선과 캡 라인은 NVT가 기준선을 넘은 봉에서만 표시
  - 캡 라인은 `ta.highest/lowest(nvt, 365)`, `bar_index < 364`이면 `상단 × 1.1`, `하단 / 1.1`
  - 교차 점은 신호선 값 위치
  - 라벨 크기 tiny
  - alertcondition 삭제
  - 2011-08-18 이전에 끝나는 봉은 na
- 추출 방법은 이전과 같다(`TradingViewApi` → `model().dataSources()`에서 plot 값을 봉 단위로 읽기, 과거 데이터를 끝까지 불러온 뒤 추출).
- **plot 비교는 plot ID가 아니라 plot 제목으로 짝을 맞춘다.** 재구성 지표는 컴파일러가 plot ID와 색상 번호를 원본과 다르게 매길 수 있다. 제목은 원본과 같게 맞춰 두었다(Above-Value Line, Top Capped Line, Below-Value Line, Zero Capped Line, NVT, Shapes, Signal, Bullish Cross, Bearish Cross).

## 작업 A: 수정한 재구성 코드와 원본의 봉 단위 비교 (필수)

1. Pine 편집기에서 아래 "재구성 코드"를 새 지표로 붙여 넣고 차트에 추가한다. 저장하거나 게시하지 않는다.
2. 재구성 지표의 입력값에서 **Force Daily View across all Time Frames만 false로 바꾼다.** 나머지는 기본값(원본과 같음)으로 둔다.
3. 4H, 1D, 2D, 5D, 1W, 2W, 1M 각각에서 원본과 재구성 지표의 모든 plot 값을 봉 단위로 추출한다.
4. 제목별로 다음을 확인한다.
   - Above-Value Line, Below-Value Line: 값과 na 위치가 같은지(불일치 봉 수)
   - Top Capped Line, Zero Capped Line: 값의 상대 오차(중앙값, 최대)와 na 위치 불일치 봉 수. 특히 `ta.highest`/`ta.lowest`가 창 안의 na를 건너뛰는지 확인한다.
   - NVT: 비율의 중앙값, 5~95% 구간, 최소~최대(전체, 2014-01 이전, 2014-01 이후)
   - NVT 선 색상: 색 자체가 같은지(#00FF00/#FF0000/#474747). 불일치 봉은 NVT가 기준선 근처라서 생긴 것인지 확인한다.
   - Signal: 비율의 중앙값과 최대 오차
   - 첫 값이 나오는 봉의 날짜가 원본과 같은지
5. Show Crosses와 Show Labels를 **원본과 재구성 지표 모두** 켜고 1D에서 한 번 더 추출한다. Bullish Cross, Bearish Cross가 찍히는 봉과 값이 같은지, 라벨이 같은 봉에 같은 크기로 보이는지 확인하고 스크린샷을 남긴다. 확인이 끝나면 원본의 두 설정을 다시 끈다.
6. 컴파일 오류, 런타임 오류, 경고 메시지가 나오면 그대로 기록한다.

## 작업 B: 원본의 미확인 설정 조합 확인 (필수)

원본의 설정을 하나씩 바꾸고, 1D, 1W, 1M에서 plot 값을 추출한다. 각 항목이 끝나면 **반드시 원래 설정으로 되돌린다.**

1. Force Daily View = true: 1W, 1M에서 Above-Value Line, Below-Value Line의 값(150/45인지, 시간 단위별 값인지), NVT가 1D 값과 같은지 확인한다. 4H에서도 한 번 확인한다.
2. Night Mode = true: NVT 선 색상, 채우기 색상, 라벨 배경과 글자색 인덱스를 기록한다.
3. Highlight Top Areas = false, Highlight Bottom Areas = false: NVT 선 색상 인덱스가 어떻게 바뀌는지 기록한다.
4. Highlight Above-Value Background = false, Highlight Below-Value Background = false: 채우기 색상 인덱스(plot_4, plot_5)와 기준선·캡 라인 값이 어떻게 바뀌는지 기록한다.
5. Manual Above/Below Value Levels = true (150/45 그대로): 1W, 1M에서 기준선이 150/45로 바뀌는지 확인한다.
6. 같은 조합을 재구성 지표에도 적용해서 결과가 같은지 비교한다.

## 작업 C: 2011-08-18 시작 조건의 원인 확인 (선택)

원본이 2011-08-18 이전 봉에 값을 내지 않는 이유를 확인한다. 가설은 "원본이 비트스탬프 가격을 함께 쓴다"이다.

1. 아래 "진단 코드"를 새 지표로 추가한다(저장하거나 게시하지 않는다).
2. 1D, 1W에서 진단 지표의 plot과 원본 NVT를 추출한다.
3. 원본 NVT와 가장 잘 맞는 후보(A: 차트 종가, B: BITSTAMP 종가, C: BITSTAMP 시작 조건만 적용)를 연도별 중앙 오차로 비교한다. 특히 2011~2013년 오차가 줄어드는지 본다.
4. 끝나면 진단 지표를 제거한다.

## 마무리

- 추가한 재구성 지표와 진단 지표를 모두 차트에서 제거한다.
- 원본의 설정이 처음 상태(위 "배경"의 설정)와 같은지 확인한다.
- 시간 단위를 원래대로 1M으로 되돌린다.

## 결과물

- `tv_nvt3_<시간단위>.csv`: 날짜, close, 원본 plot 전체, 재구성 plot 전체(열 이름에 plot 제목 사용)
- `tv_nvt3_settings_<항목>.csv`: 작업 B의 설정별 추출값
- `tv_nvt3_diag_<시간단위>.csv`: 작업 C의 진단값
- `nvt_tv_findings3.md`: 작업 A~C의 결론, 불일치 항목과 원인, 재구성 코드에서 더 고쳐야 할 점(코드 조각 포함)

## 재구성 코드 (Pine v6)

```pine
//@version=6
//
// Reconstruction of gliderfund's protected "NVT (original) - Network Value to Transactions"
// (tradingview.com/script/50VFVta2, pine id PUB;mGO6NQAZK4oHjEzHPwWWWTIYWDGJkBQF, version 1).
//
// Taken from the script's public metaInfo (exact): input names/defaults/order, plot titles and order,
// the two fills, the colour palettes (lime/red/aqua/#474747 line, red/green backgrounds, label colours).
// Verified bar by bar against values extracted from the chart (4H/1D/2D/5D/1W/2W/1M): the NVT formula
// and data feeds, the per-timeframe levels, the cap lines, the line colours, the signal EMA, the cross
// direction and position, and the label.
// Not verified: Force Daily = true levels, Night Mode and the Highlight switches turned off, and why the
// original starts on 2011-08-18 (reproduced here with a date condition).
//
// NVT = close x total supply / SMA(estimated on-chain tx volume in USD, Transaction Period)
//
indicator("NVT - Network Value to Transactions", shorttitle = "NVT", overlay = false, precision = 2)

//Inputs (same names, defaults and order as the original)
nightMode    = input.bool(false, "Night Mode")
showNvt      = input.bool(true,  "Show NVT Line")
txPeriod     = input.int(90,     "Transaction Period", minval = 1)
forceDaily   = input.bool(true,  "Force Daily View across all Time Frames")
showSignal   = input.bool(true,  "Show Signal Line")
signalLen    = input.int(13,     "Length Signal", minval = 1)
showCrosses  = input.bool(false, "Show Crosses")
hiTop        = input.bool(true,  "Highlight Top Areas")
hiBottom     = input.bool(true,  "Highlight Bottom Areas")
hiAboveBg    = input.bool(true,  "Highlight Above-Value Background")
hiBelowBg    = input.bool(true,  "Highlight Below-Value Background")
manualLevels = input.bool(false, "Manual Above/Below Value Levels")
manualAbove  = input.int(150,    "Manual Above-Value Level")
manualBelow  = input.int(45,     "Manual Below-Value Level")
showLabels   = input.bool(false, "Show Labels")

//Data (Quandl feeds; the original's values match these to ~0.1% on 2014-2021 bars)
volSymbol    = "QUANDL:BCHAIN/ETRVU"   // estimated transaction volume, USD
supplySymbol = "QUANDL:BCHAIN/TOTBC"   // total bitcoins in circulation

// The volume SMA runs on the data symbol's own bars (history from 2009-01), not on the chart's bars.
// Force Daily: daily bars on every timeframe. Off: bars of the chart's timeframe (90 weeks on 1W, 90 months on 1M).
volTf  = forceDaily ? "D" : timeframe.period
volSma = request.security(volSymbol, volTf, ta.sma(close, txPeriod))
supply = request.security(supplySymbol, "D", close)

// Network value from the chart's own price x supply, so the line moves with the live price.
// The original shows nothing on bars that close on or before 2011-08-18.
started = time_close > timestamp("UTC", 2011, 8, 18, 0, 0)
nvt = started ? close * supply / volSma : na

//Levels: automatic per timeframe class, or manual
autoAbove = timeframe.ismonthly ? 1000.0 : timeframe.isweekly ? 450.0 : timeframe.isintraday ? 200.0 : 150.0
autoBelow = timeframe.ismonthly ? 333.0  : timeframe.isweekly ? 100.0 : timeframe.isintraday ? 60.0  : 45.0
useAuto   = not manualLevels and not forceDaily
aboveLevel = manualLevels ? float(manualAbove) : useAuto ? autoAbove : 150.0
belowLevel = manualLevels ? float(manualBelow) : useAuto ? autoBelow : 45.0

isAbove = nvt > aboveLevel
isBelow = nvt < belowLevel

//Background areas: level line to the 365-bar extreme, drawn only while NVT is beyond the level
topCap = bar_index >= 364 ? ta.highest(nvt, 365) : aboveLevel * 1.1
botCap = bar_index >= 364 ? ta.lowest(nvt, 365)  : belowLevel / 1.1
pAbove  = plot(isAbove ? aboveLevel : na, "Above-Value Line", color = color.new(#808080, 100))
pTopCap = plot(isAbove ? topCap     : na, "Top Capped Line",  color = color.new(#808080, 100))
pBelow  = plot(isBelow ? belowLevel : na, "Below-Value Line", color = color.new(#808080, 100))
pZero   = plot(isBelow ? botCap     : na, "Zero Capped Line", color = color.new(#808080, 100))
fill(pAbove, pTopCap, color = hiAboveBg ? color.new(#FF0000, 90) : color.new(#FFFFFF, 100), title = "Above-Value Background")
fill(pBelow, pZero,   color = hiBelowBg ? color.new(#008000, 90) : color.new(#FFFFFF, 100), title = "Below-Value Background")

//NVT line: lime in the bottom area, red in the top area, dark grey (aqua in night mode) in between
neutral  = nightMode ? #00FFFF : #474747
nvtColor = hiBottom and isBelow ? #00FF00 : hiTop and isAbove ? #FF0000 : neutral
plot(showNvt ? nvt : na, "NVT", color = nvtColor, linewidth = 3)

//Label on the last bar
labelText = nightMode ? #000000 : #FFFFFF
plotshape(showLabels and barstate.islast ? nvt : na, "Shapes", style = shape.labelup, location = location.absolute,
     color = neutral, textcolor = labelText, text = "NVT", size = size.tiny)

//Signal line and crosses
signal = ta.ema(nvt, signalLen)
plot(showSignal ? signal : na, "Signal", color = #FF7F00)
bullCross = ta.crossover(nvt, signal)
bearCross = ta.crossunder(nvt, signal)
plot(showCrosses and bullCross ? signal : na, "Bullish Cross", color = color.new(#474747, 10), linewidth = 4, style = plot.style_circles)
plot(showCrosses and bearCross ? signal : na, "Bearish Cross", color = color.new(#474747, 50), linewidth = 4, style = plot.style_circles)
```

## 진단 코드 (작업 C용, Pine v6)

```pine
//@version=6
indicator("NVT start diagnostic", shorttitle = "NVT-diag3", overlay = false, precision = 4)
volSma  = request.security("QUANDL:BCHAIN/ETRVU", timeframe.period, ta.sma(close, 90))
supply  = request.security("QUANDL:BCHAIN/TOTBC", "D", close)
bsClose = request.security("BITSTAMP:BTCUSD", timeframe.period, close, ignore_invalid_symbol = true)
plot(close * supply / volSma,   "A chart close")
plot(bsClose * supply / volSma, "B bitstamp close")
plot(na(bsClose) ? na : close * supply / volSma, "C chart close, bitstamp start")
plot(bsClose, "Bitstamp close")
```
