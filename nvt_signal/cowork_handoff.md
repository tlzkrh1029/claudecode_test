TradingView 차트 https://www.tradingview.com/chart/X3L1Uw1v/ 를 브라우저에서 열고, 아래 값을 읽어서 표로 정리해줘.

배경:
- 두 번째 열은 gliderfund의 보호 스크립트 "NVT (original) - Network Value to Transactions" v1이다. 설정은 Transaction Period 90, Force Daily View = false이다.
- 세 번째 열은 aamonkey의 "NVT Dynamic Colored with Signals"(DNVT_aam)이며, nvt = (MKTCP / sma(ETRVU, 90)) / 4 로 계산한다.
- 확인하고 싶은 가설: 두 번째 열의 NVT 값이 모든 봉에서 세 번째 열 "NVT" 값의 정확히 4배이다.

할 일:
1. 차트를 1D로 두고 데이터 창(Data Window)을 연다.
2. 2017-12-17, 2018-12-15, 2020-03-13, 2021-04-14, 2022-11-21 근처의 봉과 가장 최근 봉 하나에 마우스를 올린다. 각 봉에서 두 번째 열의 "NVT" 값과 세 번째 열의 "NVT" 값을 읽는다. 해당 날짜에 값이 없으면 가까운 날짜를 쓴다.
3. 차트를 1W와 1M으로 바꿔서, 1M 기준 2018-12, 2021-03 봉과 가장 최근 봉에서 같은 작업을 반복한다.
4. 날짜, 시간 단위, 2열 값, 3열 값, 2열/3열 비율을 표로 정리한다.
5. 추가로 확인할 것: 2열과 3열 선이 2023~2024년 이후에도 이어지는지, 아니면 끊기거나 평평해지는지 알려줘.
