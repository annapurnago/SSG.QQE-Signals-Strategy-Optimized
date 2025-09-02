qqe-nifty.pine
//@version=6
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © colinmck

strategy("QQE Signals Strategy Optimized", overlay=true, default_qty_type=strategy.percent_of_equity, default_qty_value=50, commission_type=strategy.commission.percent, commission_value=0.1, pyramiding=0)

// Inputs
RSI_Period = input.int(14, title='RSI Length')
SF = input.int(5, title='RSI Smoothing')
QQE = input.float(4.238, title='Fast QQE Factor')
ThreshHold = input.int(10, title="Thresh-hold")

// Risk Management Inputs
useTrailingSL = input.bool(true, title="Use Trailing Stop Loss")
trailOffset = input.float(2.0, title="Trailing SL Offset (%)", minval=0.1, maxval=10.0)
useTakeProfit = input.bool(true, title="Use Take Profit")
tpMultiplier = input.float(2.0, title="TP Multiplier (RR Ratio)", minval=1.0, maxval=5.0)
useVolatilityFilter = input.bool(true, title="Use Volatility Filter")
atrPeriod = input.int(14, title="ATR Period")
minAtrRatio = input.float(0.5, title="Min ATR Ratio", minval=0.1, maxval=2.0)
useTrendFilter = input.bool(true, title="Use EMA Trend Filter")
emaPeriod = input.int(50, title="EMA Period")
maxDrawdownPct = input.int(20, title="Max Drawdown % to Stop Trading", minval=5, maxval=50)

// Calculate equity for drawdown protection
var float peakEquity = na
var bool tradingEnabled = true

currentEquity = strategy.equity
if na(peakEquity) or currentEquity > peakEquity
    peakEquity := currentEquity

drawdownPct = ((peakEquity - currentEquity) / peakEquity) * 100
if drawdownPct >= maxDrawdownPct
    tradingEnabled := false
    strategy.cancel_all()
    strategy.close_all()

// QQE Calculation
src = close
Wilders_Period = RSI_Period * 2 - 1

Rsi = ta.rsi(src, RSI_Period)
RsiMa = ta.ema(Rsi, SF)
AtrRsi = math.abs(RsiMa[1] - RsiMa)
MaAtrRsi = ta.ema(AtrRsi, Wilders_Period)
dar = ta.ema(MaAtrRsi, Wilders_Period) * QQE

var float longband = 0.0
var float shortband = 0.0
var int trend = 0

DeltaFastAtrRsi = dar
RSIndex = RsiMa
newshortband = RSIndex + DeltaFastAtrRsi
newlongband = RSIndex - DeltaFastAtrRsi
longband := RSIndex[1] > longband[1] and RSIndex > longband[1] ? math.max(longband[1], newlongband) : newlongband
shortband := RSIndex[1] < shortband[1] and RSIndex < shortband[1] ? math.min(shortband[1], newshortband) : newshortband
cross_1 = ta.cross(longband[1], RSIndex)
trend := ta.cross(RSIndex, shortband[1]) ? 1 : cross_1 ? -1 : nz(trend[1], 1)
FastAtrRsiTL = trend == 1 ? longband : shortband

// Find all the QQE Crosses
var int QQExlong = 0
QQExlong := nz(QQExlong[1])
var int QQExshort = 0
QQExshort := nz(QQExshort[1])
QQExlong := FastAtrRsiTL < RSIndex ? QQExlong + 1 : 0
QQExshort := FastAtrRsiTL > RSIndex ? QQExshort + 1 : 0

// Strategy Conditions
longCondition = QQExlong == 1
shortCondition = QQExshort == 1

// Additional Filters
atrValue = ta.atr(atrPeriod)
avgAtr = ta.sma(atrValue, 20)
volatilityOk = not useVolatilityFilter or (atrValue > avgAtr * minAtrRatio)

emaValue = ta.ema(close, emaPeriod)
trendFilterLong = not useTrendFilter or close > emaValue
trendFilterShort = not useTrendFilter or close < emaValue

// Final filtered conditions
filteredLongCondition = longCondition and volatilityOk and trendFilterLong and tradingEnabled
filteredShortCondition = shortCondition and volatilityOk and trendFilterShort and tradingEnabled

// Store signal candle high and low for stop loss
var float longStopLoss = na
var float shortStopLoss = na
var float longTakeProfit = na
var float shortTakeProfit = na
var float longRisk = na
var float shortRisk = na

if filteredLongCondition
    longStopLoss := low * 0.995  // 0.5% below low for buffer
    longRisk := close - longStopLoss
    longTakeProfit := useTakeProfit ? close + (longRisk * tpMultiplier) : na
    shortStopLoss := na
    shortTakeProfit := na
    shortRisk := na
    
if filteredShortCondition
    shortStopLoss := high * 1.005  // 0.5% above high for buffer
    shortRisk := shortStopLoss - close
    shortTakeProfit := useTakeProfit ? close - (shortRisk * tpMultiplier) : na
    longStopLoss := na
    longTakeProfit := na
    longRisk := na

// Color the signal candles
barcolor(filteredLongCondition ? color.green : filteredShortCondition ? color.red : na)

// Execute trades with proper risk management
if filteredLongCondition
    strategy.entry("Long", strategy.long)
    strategy.exit("Long SL", "Long", stop=longStopLoss, limit=longTakeProfit)
    if useTrailingSL and not na(longRisk)
        trailPoints = longRisk * trailOffset / 100
        strategy.exit("Long Trail", "Long", trail_points=trailPoints, trail_offset=trailPoints)
    
if filteredShortCondition
    strategy.entry("Short", strategy.short)
    strategy.exit("Short SL", "Short", stop=shortStopLoss, limit=shortTakeProfit)
    if useTrailingSL and not na(shortRisk)
        trailPoints = shortRisk * trailOffset / 100
        strategy.exit("Short Trail", "Short", trail_points=trailPoints, trail_offset=trailPoints)

// Close opposite positions when new signals occur
if filteredLongCondition
    strategy.close("Short", comment="Close Short")
    
if filteredShortCondition
    strategy.close("Long", comment="Close Long")

// Plotting
plot(not na(longStopLoss) ? longStopLoss : na, color=color.red, style=plot.style_linebr, linewidth=1, title="Long Stop Loss")
plot(not na(shortStopLoss) ? shortStopLoss : na, color=color.red, style=plot.style_linebr, linewidth=1, title="Short Stop Loss")
plot(not na(longTakeProfit) ? longTakeProfit : na, color=color.green, style=plot.style_linebr, linewidth=1, title="Long Take Profit")
plot(not na(shortTakeProfit) ? shortTakeProfit : na, color=color.green, style=plot.style_linebr, linewidth=1, title="Short Take Profit")

// Drawdown protection alert - FIXED bgcolor syntax
bgcolor(drawdownPct >= maxDrawdownPct ? color.new(color.red, 85) : na)
plotshape(drawdownPct >= maxDrawdownPct, title="Trading Halted", text="HALTED", style=shape.labeldown, location=location.abovebar, color=color.red, size=size.large)

// Plotting for visual reference
plotshape(filteredLongCondition, title="QQE long", text="L", textcolor=color.white, style=shape.labelup, location=location.belowbar, color=color.green, size=size.small)
plotshape(filteredShortCondition, title="QQE short", text="S", textcolor=color.white, style=shape.labeldown, location=location.abovebar, color=color.red, size=size.small)
