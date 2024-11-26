# trader
grid trading
// This Pine Script™ code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © ponlawatsanjan

//@version=5
strategy("Gold Follow Trend TF 15m Trailing stop", overlay=true, margin_long=100, margin_short=100)
KAMAlength = input(48, title="KAMAlength")
KAMAfastKAMAlength = input(2, title="KAMAfast KAMAlength")
KAMAslowKAMAlength = input(30, title="KAMAslow KAMAlength")


KAMA_direction = math.abs(close - close[KAMAlength])
KAMA_volatility = math.sum(math.abs(close - close[1]), KAMAlength)
KAMA_ER = KAMA_direction / KAMA_volatility


KAMAfastSC = 2 / (KAMAfastKAMAlength + 1)
KAMAslowSC = 2 / (KAMAslowKAMAlength + 1)
SC = math.pow(KAMA_ER * (KAMAfastSC - KAMAslowSC) + KAMAslowSC, 2)


var float KAMA = na
if (na(KAMA))
    KAMA := close
else
    KAMA := KAMA + SC * (close - KAMA)

plot(KAMA, title="KAMA", color=color.blue)


barColor = color.new(color.white, 100)
if (low > KAMA)
    barColor := color.green
else if (high < KAMA)
    barColor := color.red
else
    barColor := color.yellow

barcolor(barColor)

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
Mom_lengthbase = input(48, title="Mom Length base")



calculateMom(length) =>
    highestHigh = ta.highest(high, length)
    lowestLow = ta.lowest(low, length)
    smaClose = ta.wma(close, length)
    avgBase = math.avg(math.avg(highestHigh, lowestLow), smaClose)
    mom = ta.linreg(close - avgBase, length, 0)
    mom


Mom_base = calculateMom(Mom_lengthbase)
momlookback = input(48)

MomSlobeUp = Mom_base > nz(Mom_base[momlookback]) 
MomSlobeDown = Mom_base < nz(Mom_base[momlookback]) 
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
ADXlen = input(48)
ADXdilen = input(48)
ADXstatlen = input(480)


dirmov(len) =>
    up = ta.change(high)
    down = -ta.change(low)
    plusDM = na(up) ? na : (up > down and up > 0 ? up : 0)
    minusDM = na(down) ? na : (down > up and down > 0 ? down : 0)
    truerange = ta.wma(ta.tr, len)
    plus = fixnan(100 * ta.wma(plusDM, len) / truerange)
    minus = fixnan(100 * ta.wma(minusDM, len) / truerange)
    [plus, minus]


adx(dilen, adxlen) =>
    [plus, minus] = dirmov(dilen)
    sum = plus + minus
    adx = 100 * ta.wma(math.abs(plus - minus) / (sum == 0 ? 1 : sum), adxlen)
    adx


ADX_sig = adx(ADXdilen, ADXlen)

ADXMean = ta.wma(ADX_sig, ADXstatlen)
ADXLowest = ta.lowest(ADX_sig, ADXstatlen)
ADXHighest = ta.highest(ADX_sig, ADXstatlen)
ADXLow = ADXMean - (ADXMean - ADXLowest) / 2
ADXHigh = ADXMean + (ADXHighest - ADXMean) / 2
adxMid_M_H = ADXMean + (ADXHigh - ADXMean) / 2
adxMid_L_M = ADXMean - (ADXMean - ADXLow) / 2

ADXslobelookback = input(10)
ADXslobeUP = ADX_sig > nz(ADX_sig[ADXslobelookback])
ADXslobeDOWN = ADX_sig < nz(ADX_sig[ADXslobelookback])

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

ATR_length = input(48)
ATR_statlen = input(480)

ATR = ta.rma(ta.tr(true), ATR_length)

ATRMean = ta.wma(ATR, ATR_statlen)
ATRLowest = ta.lowest(ATR, ATR_statlen)
ATRHighest = ta.highest(ATR, ATR_statlen)
ATRlow = ATRMean - (ATRMean - ATRLowest) / 2
ATRhigh = ATRMean + (ATRHighest - ATRMean) / 2
ATRMid_M_H = ATRMean + (ATRhigh - ATRMean) / 2
ATRMid_L_M = ATRMean - (ATRMean - ATRlow) / 2

ATRslobelookback = input(10)
ATRslobeUP = ATR > nz(ATR[ATRslobelookback])
ATRslobeDOWN = ATR < nz(ATR[ATRslobelookback])

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

int     BAR_LOOKBACK    = input.int(10, "Bar Lookback")
int     ATR_LENGTH      = input.int(48, "ATR Length")
float   ATR_MULTIPLIER  = input.float(1.0, "ATR Multiplier")


float atrValue = ta.atr(ATR_LENGTH)


var float trailingStopLoss = na 
float longStop  = ta.lowest(low, BAR_LOOKBACK) - (atrValue * ATR_MULTIPLIER)
float shortStop = ta.highest(high, BAR_LOOKBACK) + (atrValue * ATR_MULTIPLIER)


bool canTakeTrades = not na(atrValue)
bgcolor(canTakeTrades ? na : color.red)


highest = ta.highest(2)
lowest = ta.lowest(2)
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

MAIN_CONDITION_BOTH = ATRslobeUP and ADXslobeUP and ATR > ATRMean and ta.crossover(ADX_sig,ADXMean) or ta.crossover(ADX_sig,adxMid_M_H) or ta.crossover(ADX_sig,ADXHigh) 

var float Entryprice = na


Long_TP_Mul = input(3)
Long_tp = Entryprice + ATR * Long_TP_Mul


LongCondition_ture = close > KAMA

if (LongCondition_ture and MomSlobeUp and MAIN_CONDITION_BOTH and canTakeTrades and strategy.position_size == 0)
    Entryprice := close
    strategy.entry("Long", strategy.long,qty = 2)

if strategy.position_size == 2 and high >= Long_tp 
    strategy.close("Long", qty = 1)



Short_TP_Mul = input(3)
Short_tp = Entryprice - ATR * Long_TP_Mul

ShortCondition_ture = close < KAMA

if (ShortCondition_ture and MomSlobeDown and MAIN_CONDITION_BOTH and canTakeTrades and strategy.position_size == 0)
    Entryprice := close
    strategy.entry("Short", strategy.short,qty = 2)
    


if strategy.position_size == -2 and low <= Short_tp 
    strategy.close("Short", qty = 1)



if (strategy.position_size > 0)
    if (na(trailingStopLoss) or longStop > trailingStopLoss)
        trailingStopLoss := longStop
else if (strategy.position_size < 0)
    if (na(trailingStopLoss) or shortStop < trailingStopLoss)
        trailingStopLoss := shortStop
else
    trailingStopLoss := na


strategy.exit("Long Exit",  "Long",  stop=trailingStopLoss)
strategy.exit("Short Exit", "Short", stop=trailingStopLoss)


plot(trailingStopLoss, "Stop Loss", color.red, 1, plot.style_linebr)


