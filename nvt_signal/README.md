# NVT Signal 역추적 결과

## 결론

차트 https://www.tradingview.com/chart/X3L1Uw1v/ 의 두 번째 열 지표는 gliderfund가 게시한 보호 스크립트 **"NVT (original) - Network Value to Transactions"**(tradingview.com/script/50VFVta2)의 **버전 1**입니다. 스크립트 ID는 `PUB;mGO6NQAZK4oHjEzHPwWWWTIYWDGJkBQF`이며, 네 번째 열은 같은 스크립트의 버전 2(2024-11-05)입니다.

`nvt_signal_reconstructed.pine`은 이 지표를 Pine v6로 재구성한 코드입니다. 차트에서 원본과 재구성 코드의 plot 값을 봉 단위로 추출해 비교한 결과, 13개 경우(59,461행)에서 모든 plot 값과 na 위치, 색상이 10자리 유효숫자 수준까지 일치했습니다.

```
NVT = BITSTAMP:BTCUSD 종가 × QUANDL:BCHAIN/TOTBC / SMA(QUANDL:BCHAIN/ETRVU, 90)
```

- 이동평균은 데이터 심볼 자체의 봉(2009-01부터, 0 값 포함)으로 계산합니다.
- `Force Daily View`를 끄면 차트 시간 단위의 봉으로 계산합니다. 따라서 주봉에서는 90주, 월봉에서는 90개월을 평균합니다.
- 분봉 차트와 `Force Daily View`를 켠 경우에는 모든 값을 일봉에서 가져옵니다.
- 원본이 2011-08-18부터 시작하는 이유는 BITSTAMP 가격 데이터가 그날 시작하기 때문입니다.

## 확정된 규칙

| 항목 | 규칙 |
|---|---|
| 자동 기준선 (상단/하단) | 분봉 200/60, 일봉 계열 150/45, 주봉 계열 450/100, 월봉 1000/333. `Force Daily View`를 켜면 모든 시간 단위에서 150/45입니다. |
| 기준선과 캡 라인 표시 | NVT가 기준선을 넘었고, 해당 Highlight Background 옵션이 켜진 봉에서만 값이 있습니다. |
| 캡 라인 | 차트 봉 365개 구간의 NVT 최고값·최저값(na 제외)입니다. `bar_index < 364`이면 `상단 × 1.1`, `하단 / 1.1`을 씁니다. |
| NVT 선 색상 | 하단 미만 #00FF00, 상단 초과 #FF0000, 그 사이 #474747(Night Mode에서는 #00FFFF) |
| 채우기 색상 | 빨강·녹색, 투명도 90. Night Mode에서는 NVT가 상단을 넘은 봉에서 두 채우기 모두 흰색입니다. |
| 신호선과 교차 | `ta.ema(nvt, 13)`. crossover는 Bullish, crossunder는 Bearish이며, 점은 신호선 값 위치에 찍힙니다. |
| 라벨 | 마지막 봉의 NVT 위치에 "NVT"(labelup, tiny, 투명도 30) |

## 근거

1. **공개 메타데이터**: 입력값의 이름·기본값·순서·최솟값, plot 제목과 순서, 채우기, 색상 팔레트, 숨김 스타일, 라벨 투명도, 알림 6개의 제목과 메시지를 그대로 옮겼습니다.
2. **차트 값 대조**: 4H, 1D, 2D, 5D, 1W, 2W, 1M과 설정 조합 6가지(교차·라벨 표시, Night Mode, Highlight Background 끔, Highlight Areas 끔, Force Daily, Manual Levels)에서 원본과 재구성 코드를 비교했습니다. 불일치는 0건입니다.
3. **독립 데이터 재계산**: 차트와 별개로 확보한 Quandl BCHAIN 원본(QuantConnect 제공, 2009-01 ~ 2021-11)과 BITSTAMP 종가로 NVT를 다시 계산했습니다. 1D 3,737봉 중 3,733봉, 1W 535봉 중 533봉이 원본과 5e-10 이내로 같았습니다. 나머지는 데이터 파일 마지막 며칠의 수정치입니다. 차트 종가를 쓰면 중앙 오차가 0.15%, 최대 오차가 50%로 커지므로, 가격 기준은 BITSTAMP로 확정됩니다.

## 한계와 주의 사항

- **데이터 피드 중단**: `QUANDL:BCHAIN/ETRVU`와 `TOTBC`는 2026-06-22 이후 새 봉이 없습니다. 그 이후의 NVT는 원본과 재구성 코드 모두 "BITSTAMP 종가 × 상수"이므로, 최근 값은 가격만 반영합니다. TradingView 공식 레퍼런스에는 QUANDL 요청이 더 이상 유효하지 않다고 적혀 있으므로, 앞으로는 두 지표 모두 오류로 멈출 수 있습니다.
- **최종 수정분의 차트 실행 여부**: 차트에서 완전 일치를 확인한 것은 Cowork가 만든 v4입니다. 최종 코드는 v4에 메타데이터 항목(알림 6개, 입력 최솟값, 숨김 스타일, 라벨 투명도, precision 제거)만 더했습니다. 이 추가분은 plot 값에 영향을 주지 않지만, 차트에서 다시 실행해 보지는 않았습니다.
- **알림 조건**: 알림의 제목과 메시지는 메타데이터에서 가져왔지만, 조건식은 제목에서 추론했습니다. plot 값으로는 검증할 수 없습니다.
- **검증하지 않은 경우**: INDEX:BTCUSD 이외의 차트(특히 USD가 아닌 차트), 틱 차트, 3M 이상의 월봉(이동평균이 계산되지 않아 값이 없음), Highlight 옵션을 하나만 끈 경우, Night Mode와 Highlight 옵션을 함께 바꾼 경우입니다.

## 파일

- `nvt_signal_reconstructed.pine`: 두 번째 열 지표를 재구성한 Pine v6 코드입니다.
