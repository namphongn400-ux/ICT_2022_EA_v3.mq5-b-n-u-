//+------------------------------------------------------------------+
//|              ICT 2022 Mentorship EA  v3.0                        |
//|   Fix: Calibrate filters, diagnostic logging, session fix        |
//+------------------------------------------------------------------+
#property copyright "ICT 2022 EA v3"
#property version   "3.00"

#include <Trade\Trade.mqh>

//=== INPUT PARAMETERS ===

input group "=== BIAS ==="
input ENUM_TIMEFRAMES  InpBiasTF1        = PERIOD_H4;
input ENUM_TIMEFRAMES  InpBiasTF2        = PERIOD_H1;
input bool             InpRequireBothTF  = false;        // true=H4+H1 phải đồng thuận, false=chỉ cần H4
input int              InpSwingLookback  = 20;

input group "=== ENTRY ==="
input ENUM_TIMEFRAMES  InpEntryTF        = PERIOD_M5;
input int              InpSweepLookback  = 40;
input int              InpMSSLookback    = 30;
input int              InpFVGLookback    = 20;
input int              InpStateTimeout   = 40;

input group "=== SIGNAL QUALITY FILTER ==="
input double           InpMinSweepWickPips = 3.0;        // Wick tối thiểu (pips) — thực tế hơn ATR
input double           InpMinBodyRatio     = 0.40;       // Body/Range nến MSS tối thiểu
input double           InpMinFVGPips       = 2.0;        // FVG tối thiểu (pips)
input bool             InpFVGZoneFilter    = true;        // Lọc Discount/Premium zone

input group "=== RISK MANAGEMENT ==="
input double           InpRiskPercent    = 0.75;
input double           InpDefaultRR      = 2.5;
input int              InpSLBufferPoints = 80;
input int              InpMaxTrades      = 1;
input int              InpCooldownBars   = 15;

input group "=== SESSION (Broker Server Time offset vs UTC) ==="
input bool             InpUseLondon      = true;
input bool             InpUseNewYork     = true;
input int              InpBrokerGMTOffset = 3;           // Broker UTC+? (thường là 2 hoặc 3, kiểm tra broker)
// London 07:00-10:00 UTC | NY 13:30-17:00 UTC

input group "=== MISC ==="
input int              InpMagicNumber    = 202200;
input bool             InpEnableAlerts   = false;
input bool             InpEnableDiag     = true;         // Log diagnostic chi tiết

//=== ENUMS & STRUCTS ===
enum ESTATE { STATE_WAIT_SWEEP, STATE_WAIT_MSS, STATE_WAIT_FVG, STATE_ORDER_PLACED };
struct SwingPoint { double price; datetime time; bool valid; };
struct FVGZone    { double top; double bottom; datetime time; bool isBullish; bool valid; };

//=== GLOBALS ===
CTrade   g_trade;
ESTATE   g_state        = STATE_WAIT_SWEEP;
bool     g_biasIsBullish = false;
double   g_sweepLevel   = 0;
datetime g_sweepTime    = 0;
datetime g_mssTime      = 0;
datetime g_lastBarTime  = 0;
int      g_barsInState  = 0;
int      g_cooldown     = 0;
int      g_atrHandle    = INVALID_HANDLE;
int      g_diagCount    = 0; // Đếm lần check để tránh spam log

//+------------------------------------------------------------------+
int OnInit()
{
    g_trade.SetExpertMagicNumber(InpMagicNumber);
    g_trade.SetDeviationInPoints(20);
    g_atrHandle = iATR(_Symbol, InpEntryTF, 14);
    if(g_atrHandle == INVALID_HANDLE) { Print("ATR init failed"); return INIT_FAILED; }
    ResetState();

    // In ATR hiện tại để debug threshold
    double atr[];
    ArraySetAsSeries(atr, true);
    CopyBuffer(g_atrHandle, 0, 1, 1, atr);
    double atrPips = atr[0] / (_Point * 10);
    Print("=== ICT 2022 EA v3 | ", _Symbol, " | ATR(14) M5 = ", DoubleToString(atrPips, 1),
          " pips | SweepWick min = ", DoubleToString(InpMinSweepWickPips, 1), " pips ===");
    Print("Broker GMT Offset = UTC+", InpBrokerGMTOffset,
          " | London session = ", InpLondonStartH(), ":00 - ", InpLondonEndH(), ":00 server time");
    Print("NY session = ", InpNYStartH(), ":30 - ", InpNYEndH(), ":00 server time");
    return INIT_SUCCEEDED;
}

// Helper: Convert UTC hours to broker server time
int InpLondonStartH() { return 7  + InpBrokerGMTOffset; }
int InpLondonEndH()   { return 10 + InpBrokerGMTOffset; }
int InpNYStartH()     { return 13 + InpBrokerGMTOffset; }
int InpNYEndH()       { return 17 + InpBrokerGMTOffset; }

void OnDeinit(const int reason) {
    if(g_atrHandle != INVALID_HANDLE) IndicatorRelease(g_atrHandle);
}

//+------------------------------------------------------------------+
//| MAIN LOOP                                                         |
//+------------------------------------------------------------------+
void OnTick()
{
    datetime currentBar = iTime(_Symbol, InpEntryTF, 0);
    if(currentBar == g_lastBarTime) return;
    g_lastBarTime = currentBar;
    g_diagCount++;

    // Cooldown
    if(g_cooldown > 0) { g_cooldown--; return; }

    // === SESSION CHECK ===
    if(!IsInSession())
    {
        CancelICTPendingOrders();
        if(g_diagCount % 12 == 0) // Log mỗi 1 giờ (~12 nến M5)
            Diag("Ngoài session. Server time: " + TimeToString(TimeCurrent(), TIME_MINUTES));
        return;
    }

    if(CountActiveTrades() >= InpMaxTrades) return;

    // === BƯỚC 1: BIAS ===
    bool bias1Valid, bias2Valid;
    bool bias1 = false, bias2 = false;
    bias1Valid = GetBias(InpBiasTF1, bias1);
    bias2Valid = GetBias(InpBiasTF2, bias2);

    if(!bias1Valid)
    {
        Diag("Bias H4 không xác định được → skip");
        if(g_state != STATE_WAIT_SWEEP) ResetState();
        return;
    }

    if(InpRequireBothTF && bias2Valid && bias1 != bias2)
    {
        Diag("Bias mâu thuẫn H4(" + (bias1?"Bull":"Bear") + ") ≠ H1(" + (bias2?"Bull":"Bear") + ") → skip");
        if(g_state != STATE_WAIT_SWEEP) ResetState();
        return;
    }

    g_biasIsBullish = bias1;
    string biasStr  = g_biasIsBullish ? "BULL ▲" : "BEAR ▼";

    // === BƯỚC 2: SWEEP ===
    if(g_state == STATE_WAIT_SWEEP)
    {
        SwingPoint sweep;
        if(CheckLiquiditySweep(g_biasIsBullish, sweep))
        {
            g_sweepLevel  = sweep.price;
            g_sweepTime   = sweep.time;
            g_barsInState = 0;
            g_state       = STATE_WAIT_MSS;
            Print("[ICT v3] ✓ SWEEP | Bias:", biasStr,
                  " | Level:", DoubleToString(g_sweepLevel, _Digits),
                  " | ", TimeToString(g_sweepTime, TIME_MINUTES));
            if(InpEnableAlerts) Alert(_Symbol, " ICT SWEEP @ ", DoubleToString(g_sweepLevel, _Digits));
        }
        else
            Diag("WAIT_SWEEP | Bias:" + biasStr + " | Không tìm thấy sweep đủ điều kiện");
        return;
    }

    // === BƯỚC 3: MSS ===
    if(g_state == STATE_WAIT_MSS)
    {
        g_barsInState++;
        if(g_barsInState > InpStateTimeout)
        {
            Diag("Timeout chờ MSS (" + IntegerToString(InpStateTimeout) + " bars). Reset.");
            ResetState(); return;
        }

        // Nếu bias đổi chiều thì reset
        if(bias1Valid && bias1 != g_biasIsBullish)
        {
            Diag("Bias đổi chiều trong khi chờ MSS. Reset.");
            ResetState(); return;
        }

        SwingPoint mss;
        string reason = "";
        if(CheckMSS_WithDisplacement(g_biasIsBullish, g_sweepTime, mss, reason))
        {
            g_mssTime     = mss.time;
            g_barsInState = 0;
            g_state       = STATE_WAIT_FVG;
            Print("[ICT v3] ✓ MSS | ", (g_biasIsBullish ? "↑ Phá đỉnh" : "↓ Phá đáy"),
                  " @ ", DoubleToString(mss.price, _Digits),
                  " | Bar:", IntegerToString(g_barsInState));
            if(InpEnableAlerts) Alert(_Symbol, " ICT MSS @ ", DoubleToString(mss.price, _Digits));
        }
        else
            Diag("WAIT_MSS [" + IntegerToString(g_barsInState) + "/" +
                 IntegerToString(InpStateTimeout) + "] | " + reason);
        return;
    }

    // === BƯỚC 4: FVG ===
    if(g_state == STATE_WAIT_FVG)
    {
        g_barsInState++;
        if(g_barsInState > InpStateTimeout)
        {
            Diag("Timeout chờ FVG (" + IntegerToString(InpStateTimeout) + " bars). Reset.");
            ResetState(); return;
        }

        FVGZone fvg;
        string reason = "";
        if(FindFVG_WithQuality(g_biasIsBullish, g_mssTime, fvg, reason))
        {
            if(PlaceICTOrder(fvg))
                g_state = STATE_ORDER_PLACED;
            else
                ResetState();
        }
        else
            Diag("WAIT_FVG [" + IntegerToString(g_barsInState) + "/" +
                 IntegerToString(InpStateTimeout) + "] | " + reason);
        return;
    }

    // === ORDER PLACED ===
    if(g_state == STATE_ORDER_PLACED)
    {
        if(CountActiveTrades() == 0)
        {
            Diag("Lệnh đóng. Cooldown " + IntegerToString(InpCooldownBars) + " bars.");
            g_cooldown = InpCooldownBars;
            ResetState();
        }
    }
}

//+------------------------------------------------------------------+
//| BƯỚC 1: Bias                                                     |
//+------------------------------------------------------------------+
bool GetBias(ENUM_TIMEFRAMES tf, bool &isBullish)
{
    double h[], l[], c[];
    ArraySetAsSeries(h, true); ArraySetAsSeries(l, true); ArraySetAsSeries(c, true);
    int bars = InpSwingLookback * 4;
    if(CopyHigh (_Symbol, tf, 1, bars, h) <= 0) return false;
    if(CopyLow  (_Symbol, tf, 1, bars, l) <= 0) return false;
    if(CopyClose(_Symbol, tf, 1, bars, c) <= 0) return false;

    double sh[3]; int shCnt = 0;
    double sl[3]; int slCnt = 0;

    for(int i = 2; i < bars - 2 && (shCnt < 3 || slCnt < 3); i++)
    {
        if(shCnt < 3 && h[i] > h[i-1] && h[i] > h[i-2] && h[i] > h[i+1] && h[i] > h[i+2])
            sh[shCnt++] = h[i];
        if(slCnt < 3 && l[i] < l[i-1] && l[i] < l[i-2] && l[i] < l[i+1] && l[i] < l[i+2])
            sl[slCnt++] = l[i];
    }

    if(shCnt >= 2 && slCnt >= 2)
    {
        bool hh = sh[0] > sh[1], hl = sl[0] > sl[1];
        bool lh = sh[0] < sh[1], ll = sl[0] < sl[1];
        if(hh && hl) { isBullish = true;  return true; }
        if(lh && ll) { isBullish = false; return true; }
    }

    // Fallback EMA50
    int ema = iMA(_Symbol, tf, 50, 0, MODE_EMA, PRICE_CLOSE);
    if(ema == INVALID_HANDLE) return false;
    double ev[];
    ArraySetAsSeries(ev, true);
    CopyBuffer(ema, 0, 1, 3, ev);
    IndicatorRelease(ema);
    if(ev[0] > ev[2] && c[0] > ev[0]) { isBullish = true;  return true; }
    if(ev[0] < ev[2] && c[0] < ev[0]) { isBullish = false; return true; }
    return false;
}

//+------------------------------------------------------------------+
//| BƯỚC 2: Liquidity Sweep — dùng pips thay vì ATR multiplier      |
//+------------------------------------------------------------------+
bool CheckLiquiditySweep(bool bullish, SwingPoint &result)
{
    result.valid = false;
    double h[], l[], c[], o[];
    datetime t[];
    ArraySetAsSeries(h, true); ArraySetAsSeries(l, true);
    ArraySetAsSeries(c, true); ArraySetAsSeries(o, true);
    ArraySetAsSeries(t, true);

    int bars = InpSweepLookback + 5;
    if(CopyHigh (_Symbol, InpEntryTF, 1, bars, h) <= 0) return false;
    if(CopyLow  (_Symbol, InpEntryTF, 1, bars, l) <= 0) return false;
    if(CopyClose(_Symbol, InpEntryTF, 1, bars, c) <= 0) return false;
    if(CopyOpen (_Symbol, InpEntryTF, 1, bars, o) <= 0) return false;
    if(CopyTime (_Symbol, InpEntryTF, 1, bars, t) <= 0) return false;

    // Wick tối thiểu tính bằng pips (dễ calibrate hơn ATR multiplier)
    double minWick = InpMinSweepWickPips * 10 * _Point; // 3 pips = 30 points 5-digit

    if(bullish)
    {
        for(int i = 3; i < InpSweepLookback - 2; i++)
        {
            // Fractal swing low
            if(!(l[i] < l[i-1] && l[i] < l[i-2] && l[i] < l[i+1] && l[i] < l[i+2]))
                continue;

            double swingLow  = l[i];
            double lowerWick = MathMin(o[0], c[0]) - l[0];

            if(l[0] < swingLow && c[0] > swingLow && lowerWick >= minWick)
            {
                result.price = swingLow;
                result.time  = t[0];
                result.valid = true;
                return true;
            }
            break; // Chỉ lấy swing low gần nhất
        }
    }
    else
    {
        for(int i = 3; i < InpSweepLookback - 2; i++)
        {
            if(!(h[i] > h[i-1] && h[i] > h[i-2] && h[i] > h[i+1] && h[i] > h[i+2]))
                continue;

            double swingHigh = h[i];
            double upperWick = h[0] - MathMax(o[0], c[0]);

            if(h[0] > swingHigh && c[0] < swingHigh && upperWick >= minWick)
            {
                result.price = swingHigh;
                result.time  = t[0];
                result.valid = true;
                return true;
            }
            break;
        }
    }
    return false;
}

//+------------------------------------------------------------------+
//| BƯỚC 3: MSS + Displacement                                       |
//+------------------------------------------------------------------+
bool CheckMSS_WithDisplacement(bool bullish, datetime sweepTime,
                                SwingPoint &result, string &reason)
{
    result.valid = false;
    reason = "";
    double h[], l[], o[], c[];
    datetime t[];
    ArraySetAsSeries(h, true); ArraySetAsSeries(l, true);
    ArraySetAsSeries(o, true); ArraySetAsSeries(c, true);
    ArraySetAsSeries(t, true);

    int bars = InpMSSLookback + 5;
    if(CopyHigh (_Symbol, InpEntryTF, 1, bars, h) <= 0) return false;
    if(CopyLow  (_Symbol, InpEntryTF, 1, bars, l) <= 0) return false;
    if(CopyOpen (_Symbol, InpEntryTF, 1, bars, o) <= 0) return false;
    if(CopyClose(_Symbol, InpEntryTF, 1, bars, c) <= 0) return false;
    if(CopyTime (_Symbol, InpEntryTF, 1, bars, t) <= 0) return false;

    // Displacement: body/range check
    double body  = MathAbs(c[0] - o[0]);
    double range = h[0] - l[0];
    if(range <= 0) { reason = "Doji nến"; return false; }

    double bodyRatio = body / range;
    if(bodyRatio < InpMinBodyRatio)
    {
        reason = "Body ratio=" + DoubleToString(bodyRatio, 2) +
                 " < min=" + DoubleToString(InpMinBodyRatio, 2);
        return false;
    }

    if(bullish && c[0] < o[0]) { reason = "Cần nến tăng (bullish)"; return false; }
    if(!bullish && c[0] > o[0]) { reason = "Cần nến giảm (bearish)"; return false; }

    // Tìm index sweep
    int sweepIdx = -1;
    for(int i = 1; i < bars; i++)
        if(t[i] <= sweepTime) { sweepIdx = i; break; }
    if(sweepIdx < 0) { reason = "Không tìm thấy sweep bar"; return false; }

    if(bullish)
    {
        double targetHigh = 0;
        for(int i = 1; i < MathMin(sweepIdx + 1, bars); i++)
            if(h[i] > targetHigh) targetHigh = h[i];

        if(targetHigh <= 0) { reason = "Không có swing high sau sweep"; return false; }
        if(c[0] <= targetHigh)
        {
            reason = "Close=" + DoubleToString(c[0], _Digits) +
                     " chưa phá đỉnh=" + DoubleToString(targetHigh, _Digits);
            return false;
        }
        result.price = targetHigh; result.time = t[0]; result.valid = true;
        return true;
    }
    else
    {
        double targetLow = DBL_MAX;
        for(int i = 1; i < MathMin(sweepIdx + 1, bars); i++)
            if(l[i] < targetLow) targetLow = l[i];

        if(targetLow == DBL_MAX) { reason = "Không có swing low sau sweep"; return false; }
        if(c[0] >= targetLow)
        {
            reason = "Close=" + DoubleToString(c[0], _Digits) +
                     " chưa phá đáy=" + DoubleToString(targetLow, _Digits);
            return false;
        }
        result.price = targetLow; result.time = t[0]; result.valid = true;
        return true;
    }
}

//+------------------------------------------------------------------+
//| BƯỚC 4: FVG                                                      |
//+------------------------------------------------------------------+
bool FindFVG_WithQuality(bool bullish, datetime afterTime,
                          FVGZone &result, string &reason)
{
    result.valid = false;
    reason = "";
    double h[], l[];
    datetime t[];
    ArraySetAsSeries(h, true); ArraySetAsSeries(l, true); ArraySetAsSeries(t, true);

    int bars = InpFVGLookback + 5;
    if(CopyHigh(_Symbol, InpEntryTF, 1, bars, h) <= 0) return false;
    if(CopyLow (_Symbol, InpEntryTF, 1, bars, l) <= 0) return false;
    if(CopyTime(_Symbol, InpEntryTF, 1, bars, t) <= 0) return false;

    double minFVGSize = InpMinFVGPips * 10 * _Point;

    // Discount/Premium midpoint từ H4
    double midpoint = GetRangeMidpoint();

    int fvgCount = 0; // Đếm bao nhiêu FVG tìm thấy nhưng bị filter

    for(int i = 1; i < bars - 2; i++)
    {
        if(t[i] < afterTime) break;

        if(bullish)
        {
            double fvgBottom = h[i+1];
            double fvgTop    = l[i-1];
            if(fvgTop <= fvgBottom) continue;

            double fvgSize = fvgTop - fvgBottom;
            if(fvgSize < minFVGSize) { fvgCount++; continue; }

            if(InpFVGZoneFilter && midpoint > 0 && fvgBottom > midpoint)
            {
                fvgCount++;
                reason = "FVG ▲ trong Premium zone (cần Discount). Midpoint=" +
                         DoubleToString(midpoint, _Digits);
                continue;
            }

            // Chưa bị fill
            if(l[0] < fvgTop) { fvgCount++; reason = "FVG đã bị fill"; continue; }

            result.bottom = fvgBottom; result.top = fvgTop;
            result.time = t[i]; result.isBullish = true; result.valid = true;
            Print("[ICT v3] FVG ▲ [", DoubleToString(fvgBottom,_Digits),
                  " - ", DoubleToString(fvgTop,_Digits), "] ",
                  DoubleToString(fvgSize/_Point,0), "pts");
            return true;
        }
        else
        {
            double fvgTop    = l[i+1];
            double fvgBottom = h[i-1];
            if(fvgTop <= fvgBottom) continue;

            double fvgSize = fvgTop - fvgBottom;
            if(fvgSize < minFVGSize) { fvgCount++; continue; }

            if(InpFVGZoneFilter && midpoint > 0 && fvgTop < midpoint)
            {
                fvgCount++;
                reason = "FVG ▼ trong Discount zone (cần Premium). Midpoint=" +
                         DoubleToString(midpoint, _Digits);
                continue;
            }

            if(h[0] > fvgBottom) { fvgCount++; reason = "FVG đã bị fill"; continue; }

            result.top = fvgTop; result.bottom = fvgBottom;
            result.time = t[i]; result.isBullish = false; result.valid = true;
            Print("[ICT v3] FVG ▼ [", DoubleToString(fvgBottom,_Digits),
                  " - ", DoubleToString(fvgTop,_Digits), "] ",
                  DoubleToString(fvgSize/_Point,0), "pts");
            return true;
        }
    }

    if(reason == "")
        reason = "Không tìm thấy FVG hợp lệ sau MSS (" +
                 IntegerToString(fvgCount) + " FVG bị lọc)";
    return false;
}

//+------------------------------------------------------------------+
double GetRangeMidpoint()
{
    double biasH[], biasL[];
    ArraySetAsSeries(biasH, true); ArraySetAsSeries(biasL, true);
    double rHigh = 0, rLow = DBL_MAX;
    int bars = InpSwingLookback * 2;
    if(CopyHigh(_Symbol, InpBiasTF1, 1, bars, biasH) > 0 &&
       CopyLow (_Symbol, InpBiasTF1, 1, bars, biasL) > 0)
    {
        for(int i = 0; i < bars; i++) {
            if(biasH[i] > rHigh) rHigh = biasH[i];
            if(biasL[i] < rLow)  rLow  = biasL[i];
        }
        if(rHigh > rLow) return (rHigh + rLow) / 2.0;
    }
    return 0;
}

//+------------------------------------------------------------------+
bool PlaceICTOrder(FVGZone &fvg)
{
    double slBuf = InpSLBufferPoints * _Point;

    if(fvg.isBullish)
    {
        double entry = fvg.bottom;
        double sl    = g_sweepLevel - slBuf;
        if(sl >= entry) { Diag("SL >= Entry, skip"); return false; }

        double tp = FindBiasSwingExtreme(true);
        if(tp <= entry) tp = entry + (entry - sl) * InpDefaultRR;

        if(SymbolInfoDouble(_Symbol, SYMBOL_ASK) <= entry)
        { Diag("Giá đã fill qua FVG Buy. Skip."); return false; }

        double lot = CalculateLotSize((entry - sl) / _Point);
        if(lot <= 0) return false;

        if(!g_trade.BuyLimit(lot, entry, _Symbol, sl, tp, ORDER_TIME_DAY, 0, "ICT_BUY_v3"))
        { Print("[ICT v3] ✗ BuyLimit: ", g_trade.ResultRetcodeDescription()); return false; }

        Print("[ICT v3] ✓ BUY LIMIT | E:", DoubleToString(entry,_Digits),
              " SL:", DoubleToString(sl,_Digits), " TP:", DoubleToString(tp,_Digits),
              " Lot:", DoubleToString(lot,2),
              " RR:", DoubleToString((tp-entry)/(entry-sl),1));
        if(InpEnableAlerts) Alert(_Symbol, " ICT BUY @ ", DoubleToString(entry,_Digits));
        return true;
    }
    else
    {
        double entry = fvg.top;
        double sl    = g_sweepLevel + slBuf;
        if(sl <= entry) { Diag("SL <= Entry, skip"); return false; }

        double tp = FindBiasSwingExtreme(false);
        if(tp <= 0 || tp >= entry) tp = entry - (sl - entry) * InpDefaultRR;

        if(SymbolInfoDouble(_Symbol, SYMBOL_BID) >= entry)
        { Diag("Giá đã fill qua FVG Sell. Skip."); return false; }

        double lot = CalculateLotSize((sl - entry) / _Point);
        if(lot <= 0) return false;

        if(!g_trade.SellLimit(lot, entry, _Symbol, sl, tp, ORDER_TIME_DAY, 0, "ICT_SELL_v3"))
        { Print("[ICT v3] ✗ SellLimit: ", g_trade.ResultRetcodeDescription()); return false; }

        Print("[ICT v3] ✓ SELL LIMIT | E:", DoubleToString(entry,_Digits),
              " SL:", DoubleToString(sl,_Digits), " TP:", DoubleToString(tp,_Digits),
              " Lot:", DoubleToString(lot,2),
              " RR:", DoubleToString((entry-tp)/(sl-entry),1));
        if(InpEnableAlerts) Alert(_Symbol, " ICT SELL @ ", DoubleToString(entry,_Digits));
        return true;
    }
}

//+------------------------------------------------------------------+
double FindBiasSwingExtreme(bool findHigh)
{
    double arr[];
    ArraySetAsSeries(arr, true);
    int bars = InpSwingLookback * 3;
    double result = findHigh ? 0 : DBL_MAX;

    if(findHigh) {
        if(CopyHigh(_Symbol, InpBiasTF1, 1, bars, arr) <= 0) return 0;
        for(int i = 2; i < bars-2; i++)
            if(arr[i] > arr[i-1] && arr[i] > arr[i+1] && arr[i] > result) result = arr[i];
        return result;
    } else {
        if(CopyLow(_Symbol, InpBiasTF1, 1, bars, arr) <= 0) return 0;
        for(int i = 2; i < bars-2; i++)
            if(arr[i] < arr[i-1] && arr[i] < arr[i+1] && arr[i] < result) result = arr[i];
        return (result == DBL_MAX) ? 0 : result;
    }
}

//+------------------------------------------------------------------+
double CalculateLotSize(double slPoints)
{
    if(slPoints <= 0) return 0;
    double balance  = AccountInfoDouble(ACCOUNT_BALANCE);
    double riskAmt  = balance * InpRiskPercent / 100.0;
    double tickVal  = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
    double tickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
    if(tickSize <= 0 || tickVal <= 0) return 0;

    double pointVal = (tickVal / tickSize) * _Point;
    if(pointVal <= 0) return 0;

    double lot  = riskAmt / (slPoints * pointVal);
    double minL = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
    double maxL = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
    double step = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);

    double freeMargin  = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
    double oneLotMgn   = SymbolInfoDouble(_Symbol, SYMBOL_MARGIN_INITIAL);
    if(oneLotMgn > 0)
        lot = MathMin(lot, (freeMargin * 0.05) / oneLotMgn);

    lot = MathFloor(lot / step) * step;
    return MathMax(minL, MathMin(maxL, lot));
}

//+------------------------------------------------------------------+
//| Session Filter dùng server time + broker GMT offset              |
//+------------------------------------------------------------------+
bool IsInSession()
{
    MqlDateTime dt;
    // Dùng TimeCurrent() (server time) + điều chỉnh ngược về UTC
    datetime utcTime = TimeCurrent() - InpBrokerGMTOffset * 3600;
    TimeToStruct(utcTime, dt);
    if(dt.day_of_week == 0 || dt.day_of_week == 6) return false;
    int h = dt.hour, m = dt.min;
    bool london = InpUseLondon  && h >= 7  && h < 10;
    bool ny     = InpUseNewYork && ((h == 13 && m >= 30) || (h > 13 && h < 17));
    return (london || ny);
}

//+------------------------------------------------------------------+
int CountActiveTrades()
{
    int cnt = 0;
    for(int i = PositionsTotal()-1; i >= 0; i--)
        if(PositionGetSymbol(i) == _Symbol &&
           PositionGetInteger(POSITION_MAGIC) == InpMagicNumber) cnt++;
    for(int i = OrdersTotal()-1; i >= 0; i--)
        if(OrderGetTicket(i) > 0 &&
           OrderGetString(ORDER_SYMBOL) == _Symbol &&
           OrderGetInteger(ORDER_MAGIC) == InpMagicNumber) cnt++;
    return cnt;
}

void CancelICTPendingOrders()
{
    for(int i = OrdersTotal()-1; i >= 0; i--) {
        ulong ticket = OrderGetTicket(i);
        if(ticket > 0 && OrderGetString(ORDER_SYMBOL) == _Symbol &&
           OrderGetInteger(ORDER_MAGIC) == InpMagicNumber)
            g_trade.OrderDelete(ticket);
    }
}

void ResetState()
{
    g_state = STATE_WAIT_SWEEP;
    g_sweepLevel = 0; g_sweepTime = 0;
    g_mssTime = 0; g_barsInState = 0;
}

void Diag(string msg) {
    if(InpEnableDiag)
        Print("[ICT v3 DIAG] ", TimeToString(TimeCurrent(), TIME_DATE|TIME_MINUTES), " | ", msg);
}
