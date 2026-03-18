# Building-financial-data-API-endpoint
from flask import Flask, request, jsonify
import yfinance as yf
import pandas as pd
import numpy as np
from datetime import datetime

app = Flask(__name__)


def get_ticker(symbol):
    """Helper to create a yfinance Ticker object."""
    return yf.Ticker(symbol.upper())


# ══════════════════════════════════════════════════════════════
# ENDPOINT 1: Company Information
# GET /company/<symbol>
# ══════════════════════════════════════════════════════════════
@app.route("/company/<symbol>", methods=["GET"])
def company_info(symbol):
    """
    Retrieve detailed company information for a given stock symbol.
    Returns: full name, business summary, industry, sector, key officers.
    """
    try:
        ticker = get_ticker(symbol)
        info = ticker.info

        # Validate that the symbol actually returned meaningful data
        if not info or info.get("quoteType") is None:
            return jsonify({"error": f"Symbol '{symbol}' not found or invalid."}), 404

        # Extract key officers with their names and titles
        officers = [
            {
                "name": officer.get("name", "N/A"),
                "title": officer.get("title", "N/A"),
                "age": officer.get("age", "N/A"),
                "total_pay": officer.get("totalPay", "N/A"),
            }
            for officer in info.get("companyOfficers", [])
        ]

        data = {
            "symbol": symbol.upper(),
            "full_name": info.get("longName", "N/A"),
            "display_name": info.get("shortName", "N/A"),
            "business_summary": info.get("longBusinessSummary", "N/A"),
            "industry": info.get("industry", "N/A"),
            "industry_key": info.get("industryKey", "N/A"),
            "sector": info.get("sector", "N/A"),
            "sector_key": info.get("sectorKey", "N/A"),
            "website": info.get("website", "N/A"),
            "headquarters": {
                "city": info.get("city", "N/A"),
                "state": info.get("state", "N/A"),
                "country": info.get("country", "N/A"),
            },
            "full_time_employees": info.get("fullTimeEmployees", "N/A"),
            "phone": info.get("phone", "N/A"),
            "exchange": info.get("exchange", "N/A"),
            "quote_type": info.get("quoteType", "N/A"),
            "key_officers": officers,
        }

        return jsonify(data), 200

    except Exception as e:
        return jsonify({"error": f"An unexpected error occurred: {str(e)}"}), 500


# ══════════════════════════════════════════════════════════════
# ENDPOINT 2: Real-Time Stock Market Data
# GET /stock/<symbol>
# ══════════════════════════════════════════════════════════════
@app.route("/stock/<symbol>", methods=["GET"])
def stock_market_data(symbol):
    """
    Retrieve real-time stock market data for a given symbol.
    Returns: market state, price, change, % change, volume, and more.
    """
    try:
        ticker = get_ticker(symbol)
        info = ticker.info

        if not info or info.get("quoteType") is None:
            return jsonify({"error": f"Symbol '{symbol}' not found or invalid."}), 404

        # Current price falls back across multiple possible fields
        current_price = (
            info.get("currentPrice")
            or info.get("regularMarketPrice")
            or info.get("ask")
            or info.get("bid")
        )
        previous_close = (
            info.get("previousClose")
            or info.get("regularMarketPreviousClose")
        )

        # Calculate price change and percentage change
        price_change = None
        pct_change = None
        if current_price is not None and previous_close is not None:
            price_change = round(current_price - previous_close, 4)
            pct_change = round((price_change / previous_close) * 100, 4)

        data = {
            "symbol": symbol.upper(),
            "company_name": info.get("shortName", "N/A"),
            "market_state": info.get("marketState", "N/A"),  # REGULAR, PRE, POST, CLOSED
            "currency": info.get("currency", "N/A"),
            "exchange": info.get("exchange", "N/A"),
            "pricing": {
                "current_price": current_price,
                "previous_close": previous_close,
                "price_change": price_change,
                "percentage_change": pct_change,
                "open": info.get("open") or info.get("regularMarketOpen"),
                "day_low": info.get("dayLow") or info.get("regularMarketDayLow"),
                "day_high": info.get("dayHigh") or info.get("regularMarketDayHigh"),
                "bid": info.get("bid"),
                "ask": info.get("ask"),
                "bid_size": info.get("bidSize"),
                "ask_size": info.get("askSize"),
            },
            "52_week": {
                "low": info.get("fiftyTwoWeekLow"),
                "high": info.get("fiftyTwoWeekHigh"),
                "change_percent": info.get("52WeekChange"),
            },
            "moving_averages": {
                "50_day": info.get("fiftyDayAverage"),
                "200_day": info.get("twoHundredDayAverage"),
            },
            "volume": {
                "current": info.get("volume") or info.get("regularMarketVolume"),
                "average_10day": info.get("averageVolume10days"),
                "average_3month": info.get("averageVolume"),
            },
            "fundamentals": {
                "market_cap": info.get("marketCap"),
                "enterprise_value": info.get("enterpriseValue"),
                "trailing_pe": info.get("trailingPE"),
                "forward_pe": info.get("forwardPE"),
                "price_to_book": info.get("priceToBook"),
                "trailing_eps": info.get("trailingEps"),
                "forward_eps": info.get("forwardEps"),
                "dividend_rate": info.get("dividendRate"),
                "dividend_yield": info.get("dividendYield"),
                "beta": info.get("beta"),
            },
            "timestamp": datetime.utcnow().isoformat() + "Z",
        }

        return jsonify(data), 200

    except Exception as e:
        return jsonify({"error": f"An unexpected error occurred: {str(e)}"}), 500


# ══════════════════════════════════════════════════════════════
# ENDPOINT 3: Historical Market Data
# POST /historical/<symbol>
# Body: { "start_date": "YYYY-MM-DD", "end_date": "YYYY-MM-DD", "interval": "1d" }
# ══════════════════════════════════════════════════════════════
@app.route("/historical/<symbol>", methods=["POST"])
def historical_data(symbol):
    """
    Retrieve historical OHLCV data for a symbol over a specified date range.
    Accepts a JSON body with start_date, end_date, and optional interval.
    Valid intervals: 1m, 2m, 5m, 15m, 30m, 60m, 90m, 1h, 1d, 5d, 1wk, 1mo, 3mo
    """
    try:
        body = request.get_json(force=True, silent=True) or {}
        start_date = body.get("start_date")
        end_date = body.get("end_date")
        interval = body.get("interval", "1d")  # Default to daily

        # Validate required fields
        if not start_date or not end_date:
            return jsonify({
                "error": "Request body must include 'start_date' and 'end_date' in YYYY-MM-DD format."
            }), 400

        # Validate date formats strictly
        try:
            start_dt = datetime.strptime(start_date, "%Y-%m-%d")
            end_dt = datetime.strptime(end_date, "%Y-%m-%d")
        except ValueError:
            return jsonify({"error": "Dates must follow the YYYY-MM-DD format."}), 400

        if start_dt >= end_dt:
            return jsonify({"error": "'start_date' must be earlier than 'end_date'."}), 400

        # Valid interval options supported by yfinance
        valid_intervals = ["1m", "2m", "5m", "15m", "30m", "60m", "90m",
                           "1h", "1d", "5d", "1wk", "1mo", "3mo"]
        if interval not in valid_intervals:
            return jsonify({
                "error": f"Invalid interval '{interval}'. Valid options: {valid_intervals}"
            }), 400

        ticker = get_ticker(symbol)
        hist = ticker.history(start=start_date, end=end_date, interval=interval)

        if hist.empty:
            return jsonify({
                "error": f"No historical data found for '{symbol}' in the given date range."
            }), 404

        # Format the index to string dates for JSON serialization
        hist.index = hist.index.strftime("%Y-%m-%d")

        records = []
        for date, row in hist.iterrows():
            records.append({
                "date": date,
                "open": round(float(row["Open"]), 4) if not pd.isna(row["Open"]) else None,
                "high": round(float(row["High"]), 4) if not pd.isna(row["High"]) else None,
                "low": round(float(row["Low"]), 4) if not pd.isna(row["Low"]) else None,
                "close": round(float(row["Close"]), 4) if not pd.isna(row["Close"]) else None,
                "volume": int(row["Volume"]) if not pd.isna(row["Volume"]) else None,
                "dividends": round(float(row.get("Dividends", 0)), 4),
                "stock_splits": float(row.get("Stock Splits", 0)),
            })

        # Summary statistics for the range
        closes = [r["close"] for r in records if r["close"] is not None]
        summary = {}
        if closes:
            summary = {
                "start_price": closes[0],
                "end_price": closes[-1],
                "period_return_pct": round(((closes[-1] - closes[0]) / closes[0]) * 100, 4),
                "highest_close": max(closes),
                "lowest_close": min(closes),
                "average_close": round(sum(closes) / len(closes), 4),
            }

        return jsonify({
            "symbol": symbol.upper(),
            "start_date": start_date,
            "end_date": end_date,
            "interval": interval,
            "total_records": len(records),
            "summary": summary,
            "data": records,
        }), 200

    except Exception as e:
        return jsonify({"error": f"An unexpected error occurred: {str(e)}"}), 500


# ══════════════════════════════════════════════════════════════
# ENDPOINT 4: Analytical Insights
# POST /analysis/<symbol>
# Body: { "start_date": "YYYY-MM-DD", "end_date": "YYYY-MM-DD", "interval": "1d" }
# ══════════════════════════════════════════════════════════════
@app.route("/analysis/<symbol>", methods=["POST"])
def analytical_insights(symbol):
    """
    Perform comprehensive technical and statistical analysis on a stock.
    Computes: returns, volatility, Sharpe ratio, RSI, MACD, Bollinger Bands,
    trend analysis, support/resistance levels, and actionable insights.
    """
    try:
        body = request.get_json(force=True, silent=True) or {}
        start_date = body.get("start_date")
        end_date = body.get("end_date")
        interval = body.get("interval", "1d")

        if not start_date or not end_date:
            return jsonify({
                "error": "Request body must include 'start_date' and 'end_date' in YYYY-MM-DD format."
            }), 400

        try:
            datetime.strptime(start_date, "%Y-%m-%d")
            datetime.strptime(end_date, "%Y-%m-%d")
        except ValueError:
            return jsonify({"error": "Dates must follow the YYYY-MM-DD format."}), 400

        ticker = get_ticker(symbol)
        hist = ticker.history(start=start_date, end=end_date, interval=interval)

        if hist.empty or len(hist) < 5:
            return jsonify({
                "error": "Insufficient historical data for analysis. Please use a wider date range."
            }), 404

        close = hist["Close"]
        high = hist["High"]
        low = hist["Low"]
        volume = hist["Volume"]

        # ── 1. Return Statistics ──────────────────────────────────
        daily_returns = close.pct_change().dropna()
        total_return_pct = round(((close.iloc[-1] - close.iloc[0]) / close.iloc[0]) * 100, 4)
        avg_daily_return = round(float(daily_returns.mean()) * 100, 4)
        best_day = round(float(daily_returns.max()) * 100, 4)
        worst_day = round(float(daily_returns.min()) * 100, 4)
        positive_days = int((daily_returns > 0).sum())
        negative_days = int((daily_returns < 0).sum())

        # ── 2. Volatility & Risk ──────────────────────────────────
        volatility_daily = round(float(daily_returns.std()) * 100, 4)
        # Annualise by multiplying by sqrt(252) — the number of trading days per year
        volatility_annual = round(float(daily_returns.std()) * np.sqrt(252) * 100, 4)

        # Sharpe Ratio (assuming risk-free rate of ~4.5% annually, i.e., ~0.0179% daily)
        risk_free_daily = 0.045 / 252
        excess_returns = daily_returns - risk_free_daily
        sharpe_ratio = None
        if daily_returns.std() != 0:
            sharpe_ratio = round(float(excess_returns.mean() / daily_returns.std()) * np.sqrt(252), 4)

        # Maximum Drawdown: largest peak-to-trough decline over the period
        rolling_max = close.cummax()
        drawdown = (close - rolling_max) / rolling_max
        max_drawdown = round(float(drawdown.min()) * 100, 4)

        # ── 3. Moving Averages ────────────────────────────────────
        ma20 = round(float(close.rolling(20).mean().iloc[-1]), 4) if len(close) >= 20 else None
        ma50 = round(float(close.rolling(50).mean().iloc[-1]), 4) if len(close) >= 50 else None
        ma200 = round(float(close.rolling(200).mean().iloc[-1]), 4) if len(close) >= 200 else None
        current_price = round(float(close.iloc[-1]), 4)

        # ── 4. RSI (Relative Strength Index, 14-period) ───────────
        # RSI measures momentum: >70 = overbought, <30 = oversold
        def compute_rsi(series, period=14):
            delta = series.diff()
            gain = delta.clip(lower=0).rolling(window=period).mean()
            loss = (-delta.clip(upper=0)).rolling(window=period).mean()
            rs = gain / loss
            return 100 - (100 / (1 + rs))

        rsi_series = compute_rsi(close)
        rsi = None
        if len(rsi_series.dropna()) > 0:
            rsi = round(float(rsi_series.iloc[-1]), 4)

        rsi_signal = "Neutral"
        if rsi is not None:
            if rsi >= 70:
                rsi_signal = "Overbought — potential sell/correction signal"
            elif rsi <= 30:
                rsi_signal = "Oversold — potential buy/reversal signal"
            elif rsi >= 55:
                rsi_signal = "Moderately bullish"
            elif rsi <= 45:
                rsi_signal = "Moderately bearish"

        # ── 5. MACD (12, 26, 9) ───────────────────────────────────
        # MACD = difference of two EMAs; signal = 9-day EMA of MACD
        ema12 = close.ewm(span=12, adjust=False).mean()
        ema26 = close.ewm(span=26, adjust=False).mean()
        macd_line = ema12 - ema26
        signal_line = macd_line.ewm(span=9, adjust=False).mean()
        macd_histogram = macd_line - signal_line

        macd_val = round(float(macd_line.iloc[-1]), 4)
        signal_val = round(float(signal_line.iloc[-1]), 4)
        histogram_val = round(float(macd_histogram.iloc[-1]), 4)

        macd_signal = "Neutral"
        if macd_val > signal_val:
            macd_signal = "Bullish — MACD above signal line"
            if len(macd_line) >= 2 and macd_line.iloc[-2] <= signal_line.iloc[-2]:
                macd_signal = "Bullish Crossover — potential buy signal"
        elif macd_val < signal_val:
            macd_signal = "Bearish — MACD below signal line"
            if len(macd_line) >= 2 and macd_line.iloc[-2] >= signal_line.iloc[-2]:
                macd_signal = "Bearish Crossover — potential sell signal"

        # ── 6. Bollinger Bands (20-period, 2 std devs) ────────────
        bb_period = min(20, len(close))
        bb_mid = close.rolling(bb_period).mean()
        bb_std = close.rolling(bb_period).std()
        bb_upper = bb_mid + (2 * bb_std)
        bb_lower = bb_mid - (2 * bb_std)

        bb_upper_val = round(float(bb_upper.iloc[-1]), 4) if not pd.isna(bb_upper.iloc[-1]) else None
        bb_mid_val = round(float(bb_mid.iloc[-1]), 4) if not pd.isna(bb_mid.iloc[-1]) else None
        bb_lower_val = round(float(bb_lower.iloc[-1]), 4) if not pd.isna(bb_lower.iloc[-1]) else None

        bb_bandwidth = None
        bb_position = None
        bb_signal = "Neutral"
        if bb_upper_val and bb_lower_val and bb_mid_val:
            bb_bandwidth = round((bb_upper_val - bb_lower_val) / bb_mid_val * 100, 4)
            bb_position = round((current_price - bb_lower_val) / (bb_upper_val - bb_lower_val) * 100, 2)
            if current_price >= bb_upper_val:
                bb_signal = "Price at upper band — overbought or strong uptrend"
            elif current_price <= bb_lower_val:
                bb_signal = "Price at lower band — oversold or strong downtrend"
            elif bb_position > 60:
                bb_signal = "Price in upper half of bands — bullish bias"
            else:
                bb_signal = "Price in lower half of bands — bearish bias"

        # ── 7. Volume Analysis ────────────────────────────────────
        avg_volume = round(float(volume.mean()), 0)
        recent_volume = int(volume.iloc[-1])
        volume_ratio = round(recent_volume / avg_volume, 4) if avg_volume > 0 else None
        volume_signal = "Normal volume"
        if volume_ratio:
            if volume_ratio > 1.5:
                volume_signal = "Above-average volume — strong conviction in recent move"
            elif volume_ratio < 0.5:
                volume_signal = "Below-average volume — weak conviction, potential consolidation"

        # ── 8. Support & Resistance (20-day rolling) ─────────────
        support = round(float(low.tail(20).min()), 4)
        resistance = round(float(high.tail(20).max()), 4)

        # ── 9. Trend Detection via Linear Regression ─────────────
        x = np.arange(len(close))
        slope, intercept = np.polyfit(x, close.values, 1)
        trend_direction = "Uptrend" if slope > 0 else "Downtrend"
        trend_strength = "Strong" if abs(slope) > (current_price * 0.001) else "Weak"
        overall_trend = f"{trend_strength} {trend_direction}"

        # ── 10. Composite Signal (majority vote of 5 indicators) ──
        bullish_signals = 0
        bearish_signals = 0

        if rsi is not None:
            if rsi < 50: bearish_signals += 1
            else: bullish_signals += 1
        if macd_val > signal_val: bullish_signals += 1
        else: bearish_signals += 1
        if slope > 0: bullish_signals += 1
        else: bearish_signals += 1
        if ma20 and current_price > ma20: bullish_signals += 1
        else: bearish_signals += 1
        if ma50 and current_price > ma50: bullish_signals += 1
        else: bearish_signals += 1

        total_signals = bullish_signals + bearish_signals
        bull_pct = round((bullish_signals / total_signals) * 100, 1) if total_signals > 0 else 50

        if bull_pct >= 70:
            composite_signal = "Strong Buy"
        elif bull_pct >= 55:
            composite_signal = "Buy"
        elif bull_pct >= 45:
            composite_signal = "Neutral / Hold"
        elif bull_pct >= 30:
            composite_signal = "Sell"
        else:
            composite_signal = "Strong Sell"

        # ── 11. Human-Readable Actionable Insights ────────────────
        insights = []

        if total_return_pct > 0:
            insights.append(f"The stock has gained {total_return_pct}% over the analysis period, indicating net appreciation.")
        else:
            insights.append(f"The stock has declined {abs(total_return_pct)}% over the analysis period, indicating net depreciation.")

        if volatility_annual > 40:
            insights.append(f"Annual volatility is high at {volatility_annual}% — suitable for risk-tolerant investors only.")
        elif volatility_annual < 15:
            insights.append(f"Annual volatility is low at {volatility_annual}% — the stock exhibits relatively stable price behaviour.")
        else:
            insights.append(f"Annual volatility is moderate at {volatility_annual}%.")

        if sharpe_ratio is not None:
            if sharpe_ratio > 1:
                insights.append(f"A Sharpe Ratio of {sharpe_ratio} indicates strong risk-adjusted returns above the risk-free rate.")
            elif sharpe_ratio > 0:
                insights.append(f"A Sharpe Ratio of {sharpe_ratio} suggests modest risk-adjusted returns.")
            else:
                insights.append(f"A negative Sharpe Ratio of {sharpe_ratio} suggests the stock has not compensated investors for its risk.")

        if max_drawdown < -20:
            insights.append(f"The maximum drawdown of {max_drawdown}% is significant — investors experienced notable capital risk during this period.")

        insights.append(f"RSI at {rsi}: {rsi_signal}.")
        insights.append(f"MACD Indicator: {macd_signal}.")
        insights.append(f"Bollinger Bands: {bb_signal}.")
        insights.append(f"Volume: {volume_signal}.")
        insights.append(f"Overall price trend: {overall_trend}.")
        insights.append(f"Key support level (20-day low): {support}. Key resistance level (20-day high): {resistance}.")
        insights.append(f"Composite signal based on {total_signals} indicators: {composite_signal} ({bull_pct}% bullish).")

        result = {
            "symbol": symbol.upper(),
            "analysis_period": {
                "start_date": start_date,
                "end_date": end_date,
                "interval": interval,
                "trading_days_analyzed": len(hist),
            },
            "price_summary": {
                "start_price": round(float(close.iloc[0]), 4),
                "end_price": current_price,
                "period_high": round(float(high.max()), 4),
                "period_low": round(float(low.min()), 4),
            },
            "return_statistics": {
                "total_return_pct": total_return_pct,
                "average_daily_return_pct": avg_daily_return,
                "best_single_day_pct": best_day,
                "worst_single_day_pct": worst_day,
                "positive_days": positive_days,
                "negative_days": negative_days,
                "win_rate_pct": round(positive_days / (positive_days + negative_days) * 100, 2),
            },
            "risk_metrics": {
                "daily_volatility_pct": volatility_daily,
                "annual_volatility_pct": volatility_annual,
                "sharpe_ratio": sharpe_ratio,
                "max_drawdown_pct": max_drawdown,
            },
            "technical_indicators": {
                "moving_averages": {
                    "ma_20": ma20,
                    "ma_50": ma50,
                    "ma_200": ma200,
                    "price_vs_ma20": "Above" if ma20 and current_price > ma20 else "Below",
                    "price_vs_ma50": "Above" if ma50 and current_price > ma50 else "Below",
                    "price_vs_ma200": "Above" if ma200 and current_price > ma200 else "Below",
                },
                "rsi": {
                    "value": rsi,
                    "signal": rsi_signal,
                },
                "macd": {
                    "macd_line": macd_val,
                    "signal_line": signal_val,
                    "histogram": histogram_val,
                    "signal": macd_signal,
                },
                "bollinger_bands": {
                    "upper": bb_upper_val,
                    "middle": bb_mid_val,
                    "lower": bb_lower_val,
                    "bandwidth_pct": bb_bandwidth,
                    "price_position_pct": bb_position,
                    "signal": bb_signal,
                },
            },
            "volume_analysis": {
                "average_volume": int(avg_volume),
                "most_recent_volume": recent_volume,
                "volume_ratio_vs_average": volume_ratio,
                "signal": volume_signal,
            },
            "trend_analysis": {
                "overall_trend": overall_trend,
                "support_level": support,
                "resistance_level": resistance,
            },
            "composite_analysis": {
                "bullish_indicators": bullish_signals,
                "bearish_indicators": bearish_signals,
                "bullish_percentage": bull_pct,
                "signal": composite_signal,
            },
            "actionable_insights": insights,
            "disclaimer": (
                "This analysis is for informational purposes only and does not constitute "
                "financial advice. Past performance is not indicative of future results."
            ),
        }

        return jsonify(result), 200

    except Exception as e:
        return jsonify({"error": f"An unexpected error occurred: {str(e)}"}), 500


# ══════════════════════════════════════════════════════════════
# Health Check Endpoint
# ══════════════════════════════════════════════════════════════
@app.route("/", methods=["GET"])
def health_check():
    return jsonify({
        "status": "running",
        "message": "Stock Analysis API is live.",
        "endpoints": {
            "company_info":        "GET  /company/<symbol>",
            "stock_market_data":   "GET  /stock/<symbol>",
            "historical_data":     "POST /historical/<symbol>",
            "analytical_insights": "POST /analysis/<symbol>",
        }
    }), 200


if __name__ == "__main__":
    app.run(debug=True, port=5000)
