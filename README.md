//@version=6
indicator("TT Gold AI M30", overlay = true, max_labels_count = 500, max_lines_count = 500)

//========================
// INPUTS
//========================
groupTrend = "Trend"
ema21Len  = input.int(21, "EMA 21", group = groupTrend)
ema50Len  = input.int(50, "EMA 50", group = groupTrend)
ema100Len = input.int(100, "EMA 100", group = groupTrend)

groupRSI = "RSI"
rsiLen = input.int(14, "RSI Length", group = groupRSI)
rsiOB = input.int(70, "Overbought", group = groupRSI)
rsiOS = input.int(30, "Oversold", group = groupRSI)

groupMACD = "MACD"
fastLen = input.int(12, "Fast", group = groupMACD)
slowLen = input.int(26, "Slow", group = groupMACD)
signalLen = input.int(9, "Signal", group = groupMACD)

groupVolume = "Volume"
volLen = input.int(20, "Volume MA", group = groupVolume)

//========================
// EMA
//========================
ema21 = ta.ema(close, ema21Len)
ema50 = ta.ema(close, ema50Len)
ema100 = ta.ema(close, ema100Len)

plot(ema21, color=color.yellow, linewidth=2, title="EMA 21")
plot(ema50, color=color.orange, linewidth=2, title="EMA 50")
plot(ema100, color=color.red, linewidth=2, title="EMA 100")

//========================
// RSI
//========================
rsi = ta.rsi(close, rsiLen)

//========================
// MACD
//========================
macdLine = ta.ema(close, fastLen) - ta.ema(close, slowLen)
signalLine = ta.ema(macdLine, signalLen)
hist = macdLine - signalLine

//========================
// VOLUME
//========================
volMA = ta.sma(volume, volLen)
highVolume = volume > volMA

//========================
// TREND FILTER
//========================
bullTrend = ema21 > ema50 and ema50 > ema100
bearTrend = ema21 < ema50 and ema50 < ema100

bgcolor(bullTrend ? color.new(color.green, 92) : bearTrend ? color.new(color.red, 92) : na)//========================
// H1 TREND FILTER
//========================
ema21H1 = request.security(syminfo.tickerid, "60", ta.ema(close, ema21Len))
ema50H1 = request.security(syminfo.tickerid, "60", ta.ema(close, ema50Len))
ema100H1 = request.security(syminfo.tickerid, "60", ta.ema(close, ema100Len))

bullH1 = ema21H1 > ema50H1 and ema50H1 > ema100H1
bearH1 = ema21H1 < ema50H1 and ema50H1 < ema100H1

//========================
// ADX
//========================
adxLen = input.int(14, "ADX Length", group="ADX")
adxMin = input.float(20, "ADX Min", group="ADX")

[diplus, diminus, adx] = ta.dmi(adxLen, adxLen)

strongTrend = adx > adxMin

//========================
// ATR
//========================
atrLen = input.int(14, "ATR Length", group="ATR")
atr = ta.atr(atrLen)

//========================
// CANDLE FILTER
//========================
body = math.abs(close - open)
range = high - low

strongBull = close > open and body > range * 0.6
strongBear = close < open and body > range * 0.6

//========================
// MACD FILTER
//========================
macdBull = macdLine > signalLine and hist > 0
macdBear = macdLine < signalLine and hist < 0

//========================
// RSI FILTER
//========================
rsiBull = rsi > 55
rsiBear = rsi < 45

//========================
// BUY FILTER
//========================
buySignal =
     bullTrend and
     bullH1 and
     macdBull and
     rsiBull and
     highVolume and
     strongTrend and
     strongBull

//========================
// SELL FILTER
//========================
sellSignal =
     bearTrend and
     bearH1 and
     macdBear and
     rsiBear and
     highVolume and
     strongTrend and
     strongBear

//========================
// SIGNAL SCORE
//========================
score = 0

score += bullTrend or bearTrend ? 20 : 0
score += bullH1 or bearH1 ? 20 : 0
score += macdBull or macdBear ? 15 : 0
score += rsiBull or rsiBear ? 15 : 0
score += strongTrend ? 15 : 0
score += highVolume ? 10 : 0
score += strongBull or strongBear ? 5 : 0

//========================
// PLOT SIGNAL
//========================
plotshape(buySignal,
     title="BUY",
     style=shape.labelup,
     color=color.lime,
     text="BUY",
     textcolor=color.black,
     location=location.belowbar)

plotshape(sellSignal,
     title="SELL",
     style=shape.labeldown,
     color=color.red,
     text="SELL",
     textcolor=color.white,
     location=location.abovebar)//========================
// ENTRY - SL - TP
//========================

rrTP1 = input.float(1.0, "TP1 RR", step=0.1, group="Risk")
rrTP2 = input.float(2.0, "TP2 RR", step=0.1, group="Risk")
rrTP3 = input.float(3.0, "TP3 RR", step=0.1, group="Risk")
atrSL = input.float(1.5, "ATR SL", step=0.1, group="Risk")

var float entry = na
var float sl = na
var float tp1 = na
var float tp2 = na
var float tp3 = na

// BUY
if buySignal
    entry := close
    sl := low - atr * atrSL

    risk = entry - sl

    tp1 := entry + risk * rrTP1
    tp2 := entry + risk * rrTP2
    tp3 := entry + risk * rrTP3

// SELL
if sellSignal
    entry := close
    sl := high + atr * atrSL

    risk = sl - entry

    tp1 := entry - risk * rrTP1
    tp2 := entry - risk * rrTP2
    tp3 := entry - risk * rrTP3

//========================
// ENTRY LABEL
//========================

if buySignal
    label.new(
     bar_index,
     low,
     "BUY\nEntry: "+str.tostring(entry)+"\nSL: "+str.tostring(sl)+"\nTP1: "+str.tostring(tp1)+"\nTP2: "+str.tostring(tp2)+"\nTP3: "+str.tostring(tp3),
     style=label.style_label_up,
     color=color.lime,
     textcolor=color.black)

if sellSignal
    label.new(
     bar_index,
     high,
     "SELL\nEntry: "+str.tostring(entry)+"\nSL: "+str.tostring(sl)+"\nTP1: "+str.tostring(tp1)+"\nTP2: "+str.tostring(tp2)+"\nTP3: "+str.tostring(tp3),
     style=label.style_label_down,
     color=color.red,
     textcolor=color.white)

//========================
// ENTRY LINE
//========================

if buySignal or sellSignal

    line.new(bar_index, entry,
         bar_index+30, entry,
         color=color.aqua,
         width=2)

    line.new(bar_index, sl,
         bar_index+30, sl,
         color=color.red,
         style=line.style_dashed)

    line.new(bar_index, tp1,
         bar_index+30, tp1,
         color=color.green)

    line.new(bar_index, tp2,
         bar_index+30, tp2,
         color=color.green)

    line.new(bar_index, tp3,
         bar_index+30, tp3,
         color=color.green)

//========================
// ALERT
//========================

alertcondition(
 buySignal,
 title="BUY",
 message="TT Gold AI BUY")

alertcondition(
 sellSignal,
 title="SELL",
 message="TT Gold AI SELL")//========================
// DASHBOARD
//========================

var table dash = table.new(position.top_right, 2, 10)

if barstate.islast

    table.cell(dash,0,0,"TT Gold AI")
    table.cell(dash,1,0,"v1.0")

    trendTxt =
         bullTrend ? "UP" :
         bearTrend ? "DOWN" :
         "SIDE"

    table.cell(dash,0,1,"Trend")
    table.cell(dash,1,1,trendTxt)

    table.cell(dash,0,2,"RSI")
    table.cell(dash,1,2,str.tostring(rsi,"#.0"))

    table.cell(dash,0,3,"ADX")
    table.cell(dash,1,3,str.tostring(adx,"#.0"))

    table.cell(dash,0,4,"ATR")
    table.cell(dash,1,4,str.tostring(atr))

    table.cell(dash,0,5,"Score")
    table.cell(dash,1,5,str.tostring(score))

//========================
// KEEP LAST 2 SIGNALS
//========================

var label[] sigs = array.new<label>()

if buySignal

    lb =
      label.new(
      bar_index,
      low,
      "BUY")

    array.push(sigs,lb)

if sellSignal

    lb =
      label.new(
      bar_index,
      high,
      "SELL")

    array.push(sigs,lb)

while array.size(sigs)>2

    old=array.shift(sigs)

    label.delete(old)

//========================
// BAR COLOR
//========================

barcolor(
 buySignal ? color.lime :
 sellSignal ? color.red :
 na)

//========================
// BACKGROUND
//========================

bgcolor(
 buySignal ?
 color.new(color.green,92):

 sellSignal ?
 color.new(color.red,92):
 na)

//========================
// ALERTS
//========================

alertcondition(
 buySignal,
 title="BUY CONFIRMED",
 message="BUY")

alertcondition(
 sellSignal,
 title="SELL CONFIRMED",
 message="SELL")

//========================
// END
//========================
