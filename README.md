# OpeningRangeBreakoutATR_NT8.cs
Opening Range Breakout indicator that marks the 9:30–9:45 ET high/low, then triggers a buy when price breaks above the range during 10:30–15:30 ET. Entry occurs at the next bar open. ATR-based stop loss and take profit are plotted dynamically. Designed to run on NinZaRenko charts (ID 12345).
OpeningRangeBreakoutATR_NT8.txt
#region Using declarations
using System;
using System.ComponentModel;
using System.ComponentModel.DataAnnotations;
using System.Xml.Serialization;
using System.Windows.Media;
using NinjaTrader.Data;
using NinjaTrader.Gui.Tools;
using NinjaTrader.NinjaScript;
using NinjaTrader.NinjaScript.DrawingTools;
using NinjaTrader.NinjaScript.Indicators;
#endregion

namespace NinjaTrader.NinjaScript.Indicators
{
    public class OpeningRangeBreakoutATR_NT8 : Indicator
    {
        private const int OpeningRangeSeriesBip = 1;

        private ATR atrPrimary;
        private Series<double> buySignalSeries;
        private Series<double> sellSignalSeries;
        private TimeZoneInfo easternTimeZone;

        private DateTime currentEtDate = DateTime.MinValue.Date;
        private double openingRangeHigh = double.NaN;
        private double openingRangeLow  = double.NaN;

        private bool inTrade;
        private double entryPrice;
        private double stopLossLevel;
        private double takeProfitLevel;

        protected override void OnStateChange()
        {
            if (State == State.SetDefaults)
            {
                Name        = "OpeningRangeBreakoutATR_NT8";
                Description = "Opening Range Breakout with ATR-based stop loss/take profit. Designed to plot on a 35/4 NinZaRenko chart while using a 1-minute series for the 9:30-9:45 ET opening range by default.";

                Calculate               = Calculate.OnEachTick;
                IsOverlay               = true;
                DrawOnPricePanel        = true;
                DisplayInDataBox        = true;
                PaintPriceMarkers       = true;
                IsSuspendedWhileInactive = true;

                StartHour   = 9;
                StartMinute = 30;
                EndHour     = 9;
                EndMinute   = 45;

                AtrLength     = 14;
                AtrMultiplier = 2.0;

                TradeStartHour   = 10;
                TradeStartMinute = 30;
                TradeEndHour     = 15;
                TradeEndMinute   = 30;

                Use1MinuteSeriesForOpeningRange = true;

                AddPlot(new Stroke(Brushes.Red, 2), PlotStyle.Line, "OpeningRangeHigh");
                AddPlot(new Stroke(Brushes.Blue, 2), PlotStyle.Line, "OpeningRangeLow");
                AddPlot(new Stroke(Brushes.Red, 1), PlotStyle.Line, "StopLoss");
                AddPlot(new Stroke(Brushes.LimeGreen, 1), PlotStyle.Line, "TakeProfit");
            }
            else if (State == State.Configure)
            {
                // Secondary 1-minute series used to build the 9:30-9:45 ET opening range
                // so the OR levels match the actual cash-session opening candle range while
                // still plotting on the primary NinZaRenko chart price scale.
                AddDataSeries(BarsPeriodType.Minute, 1);
            }
            else if (State == State.DataLoaded)
            {
                atrPrimary       = ATR(AtrLength);
                buySignalSeries  = new Series<double>(this, MaximumBarsLookBack.Infinite);
                sellSignalSeries = new Series<double>(this, MaximumBarsLookBack.Infinite);
                easternTimeZone  = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");
            }
        }

        protected override void OnBarUpdate()
        {
            if (BarsInProgress == OpeningRangeSeriesBip)
            {
                UpdateOpeningRangeFromMinuteSeries();
                return;
            }

            if (BarsInProgress != 0)
                return;

            if (CurrentBars[0] < Math.Max(AtrLength + 1, 2))
                return;

            if (Use1MinuteSeriesForOpeningRange && CurrentBars[OpeningRangeSeriesBip] < 0)
                return;

            DateTime primaryTimeEt = ToEastern(Time[0]);
            EnsureDateReset(primaryTimeEt);

            if (!Use1MinuteSeriesForOpeningRange)
                UpdateOpeningRangeFromPrimary(primaryTimeEt);

            // Only clear the current bar's signal values on the first tick of that bar so any
            // signal assigned to this bar persists for the rest of the bar.
            if (IsFirstTickOfBar)
            {
                buySignalSeries[0]  = double.NaN;
                sellSignalSeries[0] = double.NaN;
            }

            // Project OR levels and active SL/TP on the primary price panel.
            OpeningRangeHighSeries[0] = OpeningRangeReady ? openingRangeHigh : double.NaN;
            OpeningRangeLowSeries[0]  = OpeningRangeReady ? openingRangeLow  : double.NaN;
            StopLossSeries[0]         = inTrade ? stopLossLevel : double.NaN;
            TakeProfitSeries[0]       = inTrade ? takeProfitLevel : double.NaN;

            // We need the first tick of the new bar so we can reference the fully closed prior bar
            // and use the current bar open as the "open[i + 1]" equivalent from the source script.
            if (!IsFirstTickOfBar)
                return;

            if (CurrentBars[0] < 2)
                return;

            DateTime closedBarTimeEt = ToEastern(Time[1]);
            bool isInTradingWindow   = IsWithinWindowInclusive(closedBarTimeEt, TradeStartHour, TradeStartMinute, TradeEndHour, TradeEndMinute);
            bool enteredThisCall     = false;

            // Long entry only, preserving the source logic.
            if (!inTrade
                && OpeningRangeReady
                && isInTradingWindow
                && Close[2] < openingRangeHigh
                && Close[1] > openingRangeHigh)
            {
                inTrade       = true;
                enteredThisCall = true;
                entryPrice    = Open[0];

                double atrValue = atrPrimary[1];
                stopLossLevel   = entryPrice - (AtrMultiplier * atrValue);
                takeProfitLevel = entryPrice + (AtrMultiplier * atrValue);

                buySignalSeries[0] = 1.0;
                Draw.Text(this, $"Buy_{CurrentBar}", "Buy", 0, Low[0] - (2 * TickSize), Brushes.LimeGreen);

                // Source script starts the lines on i + 1 (the entry bar open).
                StopLossSeries[0]   = stopLossLevel;
                TakeProfitSeries[0] = takeProfitLevel;
            }

            // Preserve source behavior: once inTrade is true for bar i, the script also evaluates
            // exits against close[i] and paints SL/TP on bar i.
            if (inTrade)
            {
                StopLossSeries[1]   = stopLossLevel;
                TakeProfitSeries[1] = takeProfitLevel;

                if (Close[1] <= stopLossLevel)
                {
                    inTrade = false;
                    sellSignalSeries[1] = 1.0;
                    Draw.Text(this, $"SellSL_{CurrentBar}", "Sell (SL)", 1, High[1] + (2 * TickSize), Brushes.Red);
                }

                if (Close[1] >= takeProfitLevel)
                {
                    inTrade = false;
                    sellSignalSeries[1] = 1.0;
                    Draw.Text(this, $"SellTP_{CurrentBar}", "Sell (TP)", 1, High[1] + (2 * TickSize), Brushes.Red);
                }

                if (inTrade || enteredThisCall)
                {
                    StopLossSeries[0]   = stopLossLevel;
                    TakeProfitSeries[0] = takeProfitLevel;
                }
                else
                {
                    StopLossSeries[0]   = double.NaN;
                    TakeProfitSeries[0] = double.NaN;
                }
            }
        }

        private void UpdateOpeningRangeFromMinuteSeries()
        {
            if (!Use1MinuteSeriesForOpeningRange)
                return;

            DateTime minuteTimeEt = ToEastern(Time[0]);
            EnsureDateReset(minuteTimeEt);

            // NinjaTrader minute bars are end-stamped. For a 9:30-9:45 ET opening range,
            // the 1-minute bars to include are those ending after 9:30 and up to 9:45 inclusive.
            if (IsWithinWindowEndStamped(minuteTimeEt, StartHour, StartMinute, EndHour, EndMinute))
            {
                if (double.IsNaN(openingRangeHigh) || Highs[OpeningRangeSeriesBip][0] > openingRangeHigh)
                    openingRangeHigh = Highs[OpeningRangeSeriesBip][0];

                if (double.IsNaN(openingRangeLow) || Lows[OpeningRangeSeriesBip][0] < openingRangeLow)
                    openingRangeLow = Lows[OpeningRangeSeriesBip][0];
            }
        }

        private void UpdateOpeningRangeFromPrimary(DateTime primaryTimeEt)
        {
            if (IsWithinWindowInclusive(primaryTimeEt, StartHour, StartMinute, EndHour, EndMinute))
            {
                if (double.IsNaN(openingRangeHigh) || High[0] > openingRangeHigh)
                    openingRangeHigh = High[0];

                if (double.IsNaN(openingRangeLow) || Low[0] < openingRangeLow)
                    openingRangeLow = Low[0];
            }
        }

        private void EnsureDateReset(DateTime etTime)
        {
            if (currentEtDate != etTime.Date)
            {
                currentEtDate    = etTime.Date;
                openingRangeHigh = double.NaN;
                openingRangeLow  = double.NaN;
                // Source logic does not force a daily trade-state reset, so that is intentionally unchanged.
            }
        }

        private DateTime ToEastern(DateTime platformTime)
        {
            DateTime unspecified = DateTime.SpecifyKind(platformTime, DateTimeKind.Unspecified);
            return TimeZoneInfo.ConvertTime(unspecified, Core.Globals.GeneralOptions.TimeZoneInfo, easternTimeZone);
        }

        private bool IsWithinWindowInclusive(DateTime timeEt, int startHour, int startMinute, int endHour, int endMinute)
        {
            int current = ToTime(timeEt);
            int start   = startHour * 10000 + startMinute * 100;
            int end     = endHour * 10000 + endMinute * 100;
            return current >= start && current <= end;
        }

        private bool IsWithinWindowEndStamped(DateTime timeEt, int startHour, int startMinute, int endHour, int endMinute)
        {
            int current = ToTime(timeEt);
            int start   = startHour * 10000 + startMinute * 100;
            int end     = endHour * 10000 + endMinute * 100;
            return current > start && current <= end;
        }

        private bool OpeningRangeReady
        {
            get { return !double.IsNaN(openingRangeHigh) && !double.IsNaN(openingRangeLow); }
        }

        #region Properties
        [NinjaScriptProperty]
        [Range(0, 23)]
        [Display(Name = "Start Hour", GroupName = "Opening Range", Order = 1)]
        public int StartHour { get; set; }

        [NinjaScriptProperty]
        [Range(0, 59)]
        [Display(Name = "Start Minute", GroupName = "Opening Range", Order = 2)]
        public int StartMinute { get; set; }

        [NinjaScriptProperty]
        [Range(0, 23)]
        [Display(Name = "End Hour", GroupName = "Opening Range", Order = 3)]
        public int EndHour { get; set; }

        [NinjaScriptProperty]
        [Range(0, 59)]
        [Display(Name = "End Minute", GroupName = "Opening Range", Order = 4)]
        public int EndMinute { get; set; }

        [NinjaScriptProperty]
        [Range(1, 200)]
        [Display(Name = "ATR Length", GroupName = "ATR", Order = 1)]
        public int AtrLength { get; set; }

        [NinjaScriptProperty]
        [Range(0.1, 100.0)]
        [Display(Name = "ATR Multiplier", GroupName = "ATR", Order = 2)]
        public double AtrMultiplier { get; set; }

        [NinjaScriptProperty]
        [Range(0, 23)]
        [Display(Name = "Trade Start Hour", GroupName = "Trading Window", Order = 1)]
        public int TradeStartHour { get; set; }

        [NinjaScriptProperty]
        [Range(0, 59)]
        [Display(Name = "Trade Start Minute", GroupName = "Trading Window", Order = 2)]
        public int TradeStartMinute { get; set; }

        [NinjaScriptProperty]
        [Range(0, 23)]
        [Display(Name = "Trade End Hour", GroupName = "Trading Window", Order = 3)]
        public int TradeEndHour { get; set; }

        [NinjaScriptProperty]
        [Range(0, 59)]
        [Display(Name = "Trade End Minute", GroupName = "Trading Window", Order = 4)]
        public int TradeEndMinute { get; set; }

        [NinjaScriptProperty]
        [Display(Name = "Use 1-Minute Series For Opening Range", GroupName = "Opening Range", Order = 5)]
        public bool Use1MinuteSeriesForOpeningRange { get; set; }

        [Browsable(false)]
        [XmlIgnore]
        public Series<double> OpeningRangeHighSeries
        {
            get { return Values[0]; }
        }

        [Browsable(false)]
        [XmlIgnore]
        public Series<double> OpeningRangeLowSeries
        {
            get { return Values[1]; }
        }

        [Browsable(false)]
        [XmlIgnore]
        public Series<double> StopLossSeries
        {
            get { return Values[2]; }
        }

        [Browsable(false)]
        [XmlIgnore]
        public Series<double> TakeProfitSeries
        {
            get { return Values[3]; }
        }

        [Browsable(false)]
        [XmlIgnore]
        public Series<double> BuySignal
        {
            get { return buySignalSeries; }
        }

        [Browsable(false)]
        [XmlIgnore]
        public Series<double> SellSignal
        {
            get { return sellSignalSeries; }
        }
        #endregion
    }
}
