---
name: alpha-vantage
description: Stock prices, forex, crypto, economic indicators, and technical analysis via Alpha Vantage API
license: MIT
triggers:
  - stock price
  - fetch stock data
  - alpha vantage
  - stock market data
  - forex exchange rate
  - cryptocurrency price
  - economic indicator data
  - RSI MACD bollinger bands
  - technical analysis data
metadata:
  skill-author: gonzih
  data-sources:
    - www.alphavantage.co — equities, forex, crypto, economic indicators, technical indicators; free tier 25 req/day; paid plans available
  byok:
    - ALPHA_VANTAGE_API_KEY — required; free key at https://www.alphavantage.co/support/#api-key
  compatibility: claude-code>=1.0
---

# Alpha Vantage Financial Data Skill

## What it does
Retrieves financial market data from Alpha Vantage: daily/intraday equity prices, forex rates, cryptocurrency prices, US economic indicators (GDP, CPI, unemployment, treasury yields), and technical indicators (RSI, MACD, Bollinger Bands, VWAP, EMA, SMA).

## How to invoke
- "Get the daily closing price of AAPL for the last month"
- "Fetch EUR/USD forex rate data"
- "Get Bitcoin price history in USD"
- "Show me RSI for TSLA with 14-day period"
- "Fetch US unemployment rate data from FRED via Alpha Vantage"
- "Get MACD for NVDA stock"

## Key parameters
- `symbol` — ticker (e.g. `AAPL`, `MSFT`) or forex pair (`EUR`, `USD`)
- `function` — API function name (see list below)
- `interval` — `1min`, `5min`, `15min`, `30min`, `60min`, `daily`, `weekly`, `monthly`
- `outputsize` — `compact` (100 points) or `full` (20+ years)
- `time_period` — for technical indicators (default 14)
- `series_type` — `close`, `open`, `high`, `low`

## Rate limits
- Free tier: 25 requests/day, 5 requests/minute
- Premium: 75–1200 req/minute depending on plan
- Always check `Note` key in response for rate limit warnings

## Workflow steps
1. Set `ALPHA_VANTAGE_API_KEY` environment variable
2. Call desired function endpoint
3. Parse JSON response (watch for `Note` / `Information` error keys)
4. Convert to structured list or pandas/polars DataFrame

## Working code example

```python
import os
import json
import urllib.request
import urllib.parse
from typing import Optional

API_KEY = os.environ["ALPHA_VANTAGE_API_KEY"]
BASE_URL = "https://www.alphavantage.co/query"


def _av_request(params: dict) -> dict:
    params["apikey"] = API_KEY
    url = BASE_URL + "?" + urllib.parse.urlencode(params)
    with urllib.request.urlopen(url) as resp:
        data = json.loads(resp.read())
    if "Note" in data:
        raise RuntimeError(f"Rate limit hit: {data['Note']}")
    if "Information" in data:
        raise RuntimeError(f"API error: {data['Information']}")
    return data


# ── Equities ─────────────────────────────────────────────────────────────────

def get_daily_prices(symbol: str, outputsize: str = "compact") -> list[dict]:
    """TIME_SERIES_DAILY — daily OHLCV for a stock."""
    data = _av_request({
        "function": "TIME_SERIES_DAILY",
        "symbol": symbol,
        "outputsize": outputsize,
    })
    ts = data.get("Time Series (Daily)", {})
    return [
        {
            "date": date,
            "open": float(v["1. open"]),
            "high": float(v["2. high"]),
            "low": float(v["3. low"]),
            "close": float(v["4. close"]),
            "volume": int(v["5. volume"]),
        }
        for date, v in sorted(ts.items(), reverse=True)
    ]


def get_global_quote(symbol: str) -> dict:
    """GLOBAL_QUOTE — latest price snapshot."""
    data = _av_request({"function": "GLOBAL_QUOTE", "symbol": symbol})
    q = data.get("Global Quote", {})
    return {
        "symbol": q.get("01. symbol"),
        "price": float(q.get("05. price", 0)),
        "change": float(q.get("09. change", 0)),
        "change_pct": q.get("10. change percent", ""),
        "volume": int(q.get("06. volume", 0)),
        "latest_trading_day": q.get("07. latest trading day"),
        "prev_close": float(q.get("08. previous close", 0)),
    }


# ── Forex ─────────────────────────────────────────────────────────────────────

def get_fx_daily(from_currency: str, to_currency: str, outputsize: str = "compact") -> list[dict]:
    """FX_DAILY — daily forex OHLC."""
    data = _av_request({
        "function": "FX_DAILY",
        "from_symbol": from_currency,
        "to_symbol": to_currency,
        "outputsize": outputsize,
    })
    ts = data.get("Time Series FX (Daily)", {})
    return [
        {
            "date": date,
            "open": float(v["1. open"]),
            "high": float(v["2. high"]),
            "low": float(v["3. low"]),
            "close": float(v["4. close"]),
        }
        for date, v in sorted(ts.items(), reverse=True)
    ]


def get_fx_rate(from_currency: str, to_currency: str) -> dict:
    """CURRENCY_EXCHANGE_RATE — real-time exchange rate."""
    data = _av_request({
        "function": "CURRENCY_EXCHANGE_RATE",
        "from_currency": from_currency,
        "to_currency": to_currency,
    })
    r = data.get("Realtime Currency Exchange Rate", {})
    return {
        "from": r.get("1. From_Currency Code"),
        "to": r.get("3. To_Currency Code"),
        "rate": float(r.get("5. Exchange Rate", 0)),
        "last_refreshed": r.get("6. Last Refreshed"),
        "bid": float(r.get("8. Bid Price", 0)),
        "ask": float(r.get("9. Ask Price", 0)),
    }


# ── Crypto ────────────────────────────────────────────────────────────────────

def get_crypto_daily(symbol: str, market: str = "USD") -> list[dict]:
    """DIGITAL_CURRENCY_DAILY — daily crypto OHLCV."""
    data = _av_request({
        "function": "DIGITAL_CURRENCY_DAILY",
        "symbol": symbol,
        "market": market,
    })
    ts = data.get(f"Time Series (Digital Currency Daily)", {})
    return [
        {
            "date": date,
            "open": float(v.get(f"1a. open ({market})", 0)),
            "high": float(v.get(f"2a. high ({market})", 0)),
            "low": float(v.get(f"3a. low ({market})", 0)),
            "close": float(v.get(f"4a. close ({market})", 0)),
            "volume": float(v.get("5. volume", 0)),
        }
        for date, v in sorted(ts.items(), reverse=True)
    ]


# ── Economic Indicators ───────────────────────────────────────────────────────

ECONOMIC_FUNCTIONS = {
    "gdp": "REAL_GDP",
    "gdp_per_capita": "REAL_GDP_PER_CAPITA",
    "treasury_yield": "TREASURY_YIELD",
    "fed_funds_rate": "FEDERAL_FUNDS_RATE",
    "cpi": "CPI",
    "inflation": "INFLATION",
    "retail_sales": "RETAIL_SALES",
    "durables": "DURABLES",
    "unemployment": "UNEMPLOYMENT",
    "nonfarm_payroll": "NONFARM_PAYROLL",
}

def get_economic_indicator(indicator: str, interval: str = "annual") -> list[dict]:
    """Fetch a US economic indicator. interval: annual, quarterly, monthly, daily."""
    function = ECONOMIC_FUNCTIONS.get(indicator.lower(), indicator.upper())
    params = {"function": function}
    if indicator in ("treasury_yield",):
        params["interval"] = interval
        params["maturity"] = "10year"
    elif indicator in ("fed_funds_rate", "cpi", "retail_sales", "durables",
                       "unemployment", "nonfarm_payroll"):
        params["interval"] = interval
    else:
        params["interval"] = interval

    data = _av_request(params)
    series = data.get("data", [])
    return [{"date": d["date"], "value": float(d["value"])} for d in series if d["value"] != "."]


# ── Technical Indicators ──────────────────────────────────────────────────────

def get_rsi(symbol: str, interval: str = "daily", time_period: int = 14, series_type: str = "close") -> list[dict]:
    data = _av_request({
        "function": "RSI",
        "symbol": symbol,
        "interval": interval,
        "time_period": time_period,
        "series_type": series_type,
    })
    ts = data.get(f"Technical Analysis: RSI", {})
    return [{"date": d, "rsi": float(v["RSI"])} for d, v in sorted(ts.items(), reverse=True)]


def get_macd(symbol: str, interval: str = "daily") -> list[dict]:
    data = _av_request({
        "function": "MACD",
        "symbol": symbol,
        "interval": interval,
        "series_type": "close",
    })
    ts = data.get("Technical Analysis: MACD", {})
    return [
        {
            "date": d,
            "macd": float(v["MACD"]),
            "signal": float(v["MACD_Signal"]),
            "hist": float(v["MACD_Hist"]),
        }
        for d, v in sorted(ts.items(), reverse=True)
    ]


def get_bbands(symbol: str, interval: str = "daily", time_period: int = 20) -> list[dict]:
    """Bollinger Bands."""
    data = _av_request({
        "function": "BBANDS",
        "symbol": symbol,
        "interval": interval,
        "time_period": time_period,
        "series_type": "close",
        "nbdevup": 2,
        "nbdevdn": 2,
    })
    ts = data.get("Technical Analysis: BBANDS", {})
    return [
        {
            "date": d,
            "upper": float(v["Real Upper Band"]),
            "middle": float(v["Real Middle Band"]),
            "lower": float(v["Real Lower Band"]),
        }
        for d, v in sorted(ts.items(), reverse=True)
    ]


def get_vwap(symbol: str, interval: str = "60min") -> list[dict]:
    """VWAP — requires intraday interval."""
    data = _av_request({
        "function": "VWAP",
        "symbol": symbol,
        "interval": interval,
    })
    ts = data.get("Technical Analysis: VWAP", {})
    return [{"datetime": d, "vwap": float(v["VWAP"])} for d, v in sorted(ts.items(), reverse=True)]


# --- Example usage ---
if __name__ == "__main__":
    # Latest AAPL quote
    q = get_global_quote("AAPL")
    print(f"AAPL: ${q['price']:.2f} ({q['change_pct']})")

    # Daily prices
    prices = get_daily_prices("MSFT", outputsize="compact")
    print(f"MSFT last close: ${prices[0]['close']:.2f} on {prices[0]['date']}")

    # EUR/USD rate
    fx = get_fx_rate("EUR", "USD")
    print(f"EUR/USD: {fx['rate']:.4f}")

    # Bitcoin price
    btc = get_crypto_daily("BTC", "USD")
    print(f"BTC last close: ${btc[0]['close']:,.2f}")

    # US Unemployment rate
    unemp = get_economic_indicator("unemployment", interval="monthly")
    print(f"Unemployment ({unemp[0]['date']}): {unemp[0]['value']}%")

    # RSI for NVDA
    rsi = get_rsi("NVDA", time_period=14)
    print(f"NVDA RSI(14): {rsi[0]['rsi']:.2f}")
```

## Live Data Sources
- **Base URL**: `https://www.alphavantage.co/query`
- **Documentation**: https://www.alphavantage.co/documentation/
- **Free API key**: https://www.alphavantage.co/support/#api-key
- **Coverage**: 20+ years of US/global equity data, 50+ forex pairs, 500+ cryptocurrencies, all major US economic indicators
