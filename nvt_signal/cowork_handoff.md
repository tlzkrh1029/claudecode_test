# Cowork 이관 작업: NVT 재구성 코드의 남은 검증

아래 작업을 순서대로 수행하고, 결과를 CSV 파일과 요약 문서(`nvt_tv_findings2.md`)로 정리해줘.

## 배경

- 차트: https://www.tradingview.com/chart/X3L1Uw1v/ (INDEX:BTCUSD)
- 2열 지표는 gliderfund의 보호 스크립트 "NVT - Network Value to Transactions" v1이다. 스크립트 ID는 `PUB;mGO6NQAZK4oHjEzHPwWWWTIYWDGJkBQF`이고, 설정은 Transaction Period 90, Force Daily View = false, Length Signal 13, Manual Levels = false이다.
- 이전 작업에서 2열의 NVT 계산식은 다음으로 확정했다(2014~2021 구간에서 중앙 오차 0.1% 이하).
  `nvt = close * TOTBC / request.security(ETRVU, timeframe.period, ta.sma(close, 90))`
- 이전 작업과 같은 방법(`TradingViewApi` → `model().dataSources()`에서 plot 값을 봉 단위로 추출)을 사용한다.
- 2열 plot ID: plot_0 Above-Value Line, plot_1 Top Capped Line, plot_2 Below-Value Line, plot_3 Zero Capped Line, plot_4 Above-Value Background 색상(colorer), plot_5 Below-Value Background 색상(colorer), plot_6 NVT, plot_7 NVT 선 색상(colorer), plot_8 Shapes(라벨), plot_11 Signal, plot_12 Bullish Cross, plot_13 Bearish Cross.
- colorer 값은 색상 인덱스다. 팔레트 기준으로 0 = #00FF00, 1 = #FF0000, 2 = #00FFFF, 3 = #474747, 4 = #FFFFFF, 5 = #008000, 6 = #000000으로 추정한다.

## 작업 1: 기준선과 색상 규칙 확인

1D, 2D, 5D, 1W, 2W, 1M, 3M, 4H 각각에서 2열의 모든 plot(plot_0 ~ plot_13) 값을 봉 단위로 추출한다. 3M처럼 값이 없는 시간 단위는 "값 없음"이라고 기록한다.
확인할 것:
1. plot_0과 plot_2가 시간 단위마다 상수인지, 그리고 그 값이 무엇인지(예상: 1D·2D·5D 150/45, 1W·2W 450/100, 1M 1000/333, 4H는 미확인).
2. plot_1과 plot_3의 값이 무엇인지. 상수인지, NVT를 따라가는지(예: max(NVT, 상단 기준선), min(NVT, 하단 기준선)), 아니면 0인지.
3. plot_7(NVT 선 색상 인덱스)이 언제 0, 1, 3이 되는지. NVT < 하단 기준선이면 0, NVT > 상단 기준선이면 1, 그 사이면 3이라는 규칙이 맞는지 봉 단위로 확인한다.
4. plot_4와 plot_5의 값이 언제 1/4, 4/5로 바뀌는지.

## 작업 2: 신호선과 교차 방향 확인 (차트 소유자로 로그인한 상태에서만)

1. 2열 설정에서 Show Crosses를 켠다.
2. 1D에서 plot_11(Signal)이 `ta.ema(NVT, 13)`과 같은지 확인한다.
3. plot_12(Bullish Cross)와 plot_13(Bearish Cross)에 값이 찍히는 봉에서, NVT가 Signal을 위로 돌파했는지 아래로 돌파했는지 기록한다.
4. Show Labels를 켜고 라벨이 어느 봉에 어떤 문구로 표시되는지 스크린샷으로 남긴다.
5. 끝나면 두 설정을 원래대로 끈다.

## 작업 3: 재구성 코드를 차트에 올려서 직접 비교 (차트 소유자로 로그인한 상태에서만)

1. Pine 편집기를 열고 아래 코드를 새 지표로 붙여 넣은 뒤 차트에 추가한다. 이 코드를 게시하거나 공유하지는 않는다.
2. 입력값을 2열과 똑같이 맞춘다(Transaction Period 90, Force Daily View = false, Length Signal 13).
3. 1D, 1W, 1M에서 재구성 지표의 "NVT"와 2열의 "NVT"를 봉 단위로 추출하고, 비율의 중앙값, 5~95% 구간, 최소~최대를 연도별로 정리한다. 특히 2021-11 이후 구간을 따로 정리한다.
4. 재구성 지표에서 컴파일 오류나 런타임 오류가 나면, 오류 메시지를 그대로 기록한다.
5. 작업 1에서 확인한 기준선·색상 규칙과 재구성 지표의 plot_0, plot_2, NVT 색상이 일치하는지 비교한다.
6. 끝나면 재구성 지표를 차트에서 제거하여 레이아웃을 원래대로 돌려놓는다.

## 결과물

- `tv_nvt2_<시간단위>.csv`: 날짜, close, 2열 plot_0~plot_13, 재구성 NVT
- `nvt_tv_findings2.md`: 작업 1~3의 결론, 각 추론 항목(기준선, 캡 라인, 색상 규칙, 교차 방향, 라벨)의 확정 여부, 재구성 코드에서 고쳐야 할 점

## 재구성 코드 (Pine v6)

```pine
//@version=6
//
// Reconstruction of gliderfund's protected "NVT (original) - Network Value to Transactions"
// (tradingview.com/script/50VFVta2, pine id PUB;mGO6NQAZK4oHjEzHPwWWWTIYWDGJkBQF, version 1).
//
// Taken from the script's public metaInfo (exact): input names/defaults/order, plot titles and order,
// the two fills, the colour palettes (lime/red/aqua/#474747 line, red/green backgrounds, label colours).
// Verified against values extracted from the chart (1D/1W/1M): the NVT formula and data feeds.
// Inferred (the source itself is encrypted): the automatic per-timeframe levels, the cap lines' values, the cross direction and the label text.
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
nvt = close * supply / volSma

//Levels: automatic per timeframe class, or manual
autoAbove = timeframe.ismonthly ? 1000.0 : timeframe.isweekly ? 450.0 : 150.0
autoBelow = timeframe.ismonthly ? 333.0  : timeframe.isweekly ? 100.0 : 45.0
useAuto   = not manualLevels and not forceDaily
aboveLevel = manualLevels ? float(manualAbove) : useAuto ? autoAbove : 150.0
belowLevel = manualLevels ? float(manualBelow) : useAuto ? autoBelow : 45.0

isAbove = nvt > aboveLevel
isBelow = nvt < belowLevel

//Background areas between invisible level lines and cap lines
pAbove  = plot(aboveLevel, "Above-Value Line", color = color.new(#808080, 100))
pTopCap = plot(math.max(nvt, aboveLevel), "Top Capped Line", color = color.new(#808080, 100))
pBelow  = plot(belowLevel, "Below-Value Line", color = color.new(#808080, 100))
pZero   = plot(math.min(nvt, belowLevel), "Zero Capped Line", color = color.new(#808080, 100))
fill(pAbove, pTopCap, color = hiAboveBg and isAbove ? color.new(#FF0000, 90) : color.new(#FFFFFF, 100), title = "Above-Value Background")
fill(pBelow, pZero,   color = hiBelowBg and isBelow ? color.new(#008000, 90) : color.new(#FFFFFF, 100), title = "Below-Value Background")

//NVT line: lime in the bottom area, red in the top area, dark grey (aqua in night mode) in between
neutral  = nightMode ? #00FFFF : #474747
nvtColor = hiBottom and isBelow ? #00FF00 : hiTop and isAbove ? #FF0000 : neutral
plot(showNvt ? nvt : na, "NVT", color = nvtColor, linewidth = 3)

//Label on the last bar
labelText = nightMode ? #000000 : #FFFFFF
plotshape(showLabels and barstate.islast ? nvt : na, "Shapes", style = shape.labelup, location = location.absolute,
     color = neutral, textcolor = labelText, text = "NVT")

//Signal line and crosses
signal = ta.ema(nvt, signalLen)
plot(showSignal ? signal : na, "Signal", color = #FF7F00)
bullCross = ta.crossover(nvt, signal)
bearCross = ta.crossunder(nvt, signal)
plot(showCrosses and bullCross ? nvt : na, "Bullish Cross", color = color.new(#474747, 10), linewidth = 4, style = plot.style_circles)
plot(showCrosses and bearCross ? nvt : na, "Bearish Cross", color = color.new(#474747, 50), linewidth = 4, style = plot.style_circles)

//Alerts
alertcondition(bullCross or bearCross, "Signal Line Cross", "NVT crossed its signal line")
alertcondition(bullCross, "Bullish Signal Line Cross", "NVT crossed above its signal line")
alertcondition(bearCross, "Bearish Signal Line Cross", "NVT crossed below its signal line")
alertcondition(ta.crossover(nvt, aboveLevel), "OverBought", "NVT entered the above-value area")
alertcondition(ta.crossunder(nvt, belowLevel), "OverSold", "NVT entered the below-value area")
```
