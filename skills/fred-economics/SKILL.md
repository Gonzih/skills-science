---
name: fred-economics
description: Access 800K+ FRED economic time series — GDP, CPI, unemployment, interest rates, M2, treasury yields
license: MIT
triggers:
  - fred economic data
  - federal reserve data
  - gdp data
  - inflation cpi data
  - unemployment rate data
  - interest rate data
  - fred time series
  - treasury yield
  - money supply data
  - economic indicator fred
metadata:
  skill-author: gonzih
  data-sources:
    - fred.stlouisfed.org — 800K+ economic time series from 100+ sources (BLS, BEA, Census, Fed); free; API key required
  byok:
    - FRED_API_KEY — required; free registration at https://fred.stlouisfed.org/docs/api/api_key.html
  compatibility: claude-code>=1.0
---

# FRED Economics Skill

## What it does
Retrieves economic time series from the Federal Reserve Bank of St. Louis FRED database. Access 800K+ series covering US and international economic data: GDP, CPI, unemployment, fed funds rate, treasury yields, M2 money supply, housing starts, trade balance, and much more. Supports fetching series data, searching for series, and getting series metadata.

## How to invoke
- "Get US GDP data from FRED for the last 10 years"
- "Fetch the federal funds rate series from FRED"
- "Show me CPI inflation data monthly since 2015"
- "Get the 10-year treasury yield series"
- "Search FRED for unemployment series"
- "Fetch M2 money supply data"

## Key FRED series IDs
| Series | ID | Frequency | Description |
|--------|-----|-----------|-------------|
| Real GDP | `GDPC1` | Quarterly | Chained 2017 dollars |
| Nominal GDP | `GDP` | Quarterly | Current dollars |
| CPI All Items | `CPIAUCSL` | Monthly | Urban consumers |
| Core CPI | `CPILFESL` | Monthly | Excl. food & energy |
| Fed Funds Rate | `FEDFUNDS` | Monthly | Effective rate |
| Unemployment | `UNRATE` | Monthly | Civilian rate % |
| 10-yr Treasury | `DGS10` | Daily | Constant maturity |
| 2-yr Treasury | `DGS2` | Daily | Constant maturity |
| M2 Money Supply | `M2SL` | Monthly | Billions USD |
| Nonfarm Payrolls | `PAYEMS` | Monthly | Thousands |
| PCE Inflation | `PCEPI` | Monthly | Personal consumption |
| Housing Starts | `HOUST` | Monthly | Thousands of units |
| Trade Balance | `BOPGSTB` | Monthly | Goods & services |
| NASDAQ | `NASDAQCOM` | Daily | Composite index |
| VIX | `VIXCLS` | Daily | CBOE volatility |

## Rate limits
- Free tier: no explicit rate limit, but be reasonable (~120 req/minute)
- API key is free, required for all requests
- CSV endpoint available without key for simple series

## Workflow steps
1. Set `FRED_API_KEY` environment variable
2. Use `/series/observations` endpoint with series ID
3. Parse JSON response, handle missing values (`.`)
4. Optionally convert to pandas/polars for analysis

## Working code example

```python
import os
import json
import urllib.request
import urllib.parse
from datetime import datetime, date
from typing import Optional

FRED_BASE = "https://api.stlouisfed.org/fred"
API_KEY = os.environ.get("FRED_API_KEY", "")


def _fred_get(endpoint: str, params: dict) -> dict:
    if not API_KEY:
        raise RuntimeError("FRED_API_KEY environment variable not set. "
                           "Get a free key at: https://fred.stlouisfed.org/docs/api/api_key.html")
    params["api_key"] = API_KEY
    params["file_type"] = "json"
    url = f"{FRED_BASE}/{endpoint}?" + urllib.parse.urlencode(params)
    with urllib.request.urlopen(url) as resp:
        return json.loads(resp.read())


# ── Series data ───────────────────────────────────────────────────────────────

def get_series(
    series_id: str,
    observation_start: Optional[str] = None,
    observation_end: Optional[str] = None,
    frequency: Optional[str] = None,
    aggregation_method: str = "avg",
    units: str = "lin",
) -> list[dict]:
    """
    Fetch observations for a FRED series.

    Args:
        series_id: FRED series ID (e.g. 'UNRATE', 'GDPC1', 'FEDFUNDS')
        observation_start: start date 'YYYY-MM-DD' (default: earliest available)
        observation_end: end date 'YYYY-MM-DD' (default: latest available)
        frequency: aggregate to lower frequency — 'd','w','bw','m','q','sa','a'
        aggregation_method: 'avg', 'sum', 'eop' (end of period)
        units: 'lin' (levels), 'chg' (change), 'pch' (% change), 'pca' (% change annual rate),
               'cch' (compounded annual rate of change), 'cca', 'log'
    """
    params = {"series_id": series_id, "units": units, "aggregation_method": aggregation_method}
    if observation_start:
        params["observation_start"] = observation_start
    if observation_end:
        params["observation_end"] = observation_end
    if frequency:
        params["frequency"] = frequency

    data = _fred_get("series/observations", params)
    observations = []
    for obs in data.get("observations", []):
        value_str = obs.get("value", ".")
        if value_str == ".":
            continue  # missing value
        observations.append({
            "date": obs["date"],
            "value": float(value_str),
        })
    return observations


def get_series_info(series_id: str) -> dict:
    """Get metadata for a FRED series."""
    data = _fred_get("series", {"series_id": series_id})
    sers = data.get("seriess", [{}])[0]
    return {
        "id": sers.get("id"),
        "title": sers.get("title"),
        "observation_start": sers.get("observation_start"),
        "observation_end": sers.get("observation_end"),
        "frequency": sers.get("frequency"),
        "units": sers.get("units"),
        "seasonal_adjustment": sers.get("seasonal_adjustment"),
        "last_updated": sers.get("last_updated"),
        "notes": (sers.get("notes") or "")[:300],
    }


def search_series(query: str, limit: int = 10, order_by: str = "popularity") -> list[dict]:
    """Search FRED for series matching a query."""
    data = _fred_get("series/search", {
        "search_text": query,
        "limit": limit,
        "order_by": order_by,
        "sort_order": "desc",
    })
    return [
        {
            "id": s.get("id"),
            "title": s.get("title"),
            "frequency": s.get("frequency_short"),
            "units": s.get("units_short"),
            "last_updated": s.get("last_updated"),
            "observation_start": s.get("observation_start"),
            "observation_end": s.get("observation_end"),
        }
        for s in data.get("seriess", [])
    ]


# ── Convenience functions ─────────────────────────────────────────────────────

def get_latest_value(series_id: str) -> dict:
    """Get the most recent observation for a series."""
    obs = get_series(series_id)
    if not obs:
        return {}
    latest = obs[-1]
    info = get_series_info(series_id)
    return {
        "series_id": series_id,
        "title": info.get("title"),
        "date": latest["date"],
        "value": latest["value"],
        "units": info.get("units"),
        "frequency": info.get("frequency"),
    }


def get_yoy_change(series_id: str, year: int = None) -> list[dict]:
    """
    Compute year-over-year percent changes for a series.
    Uses FRED's built-in pch unit for annual rates.
    """
    obs = get_series(series_id, units="pc1")  # pc1 = % change from a year ago
    return obs


def get_fred_release_calendar(release_id: int) -> dict:
    """Get a FRED data release schedule."""
    data = _fred_get("release/dates", {"release_id": release_id, "limit": 10, "sort_order": "desc"})
    return {
        "release_id": release_id,
        "recent_dates": [d["date"] for d in data.get("release_dates", [])[:5]],
    }


# ── CSV endpoint (no key required for basic access) ───────────────────────────

def get_series_csv(series_id: str) -> list[dict]:
    """
    Fetch series data via the CSV graph endpoint — no API key required.
    Limited to basic series data.
    """
    url = f"https://fred.stlouisfed.org/graph/fredgraph.csv?id={series_id}"
    with urllib.request.urlopen(url) as resp:
        lines = resp.read().decode("utf-8").strip().split("\n")

    results = []
    for line in lines[1:]:  # skip header
        parts = line.split(",")
        if len(parts) == 2 and parts[1] != ".":
            try:
                results.append({"date": parts[0], "value": float(parts[1])})
            except ValueError:
                pass
    return results


# --- Example usage ---
if __name__ == "__main__":
    # Get current unemployment rate
    unemp = get_latest_value("UNRATE")
    print(f"Unemployment: {unemp['value']}% (as of {unemp['date']})")

    # Get Fed Funds Rate last 12 months
    fedfunds = get_series("FEDFUNDS", observation_start="2023-01-01")
    print(f"\nFed Funds Rate — recent readings:")
    for obs in fedfunds[-5:]:
        print(f"  {obs['date']}: {obs['value']:.2f}%")

    # GDP quarterly data
    gdp = get_series("GDPC1", observation_start="2020-01-01")
    print(f"\nReal GDP (last 5 quarters, bil. chained 2017$):")
    for obs in gdp[-5:]:
        print(f"  {obs['date']}: ${obs['value']:,.1f}B")

    # 10-yr Treasury yield
    dgs10 = get_series("DGS10", observation_start="2024-01-01")
    print(f"\n10-yr Treasury (recent): {dgs10[-1]['value']:.2f}% on {dgs10[-1]['date']}")

    # Search for CPI series
    results = search_series("consumer price index urban", limit=3)
    print(f"\nFRED series matching 'CPI urban':")
    for s in results:
        print(f"  {s['id']}: {s['title']} ({s['frequency']})")

    # M2 money supply (CSV, no key needed)
    m2 = get_series_csv("M2SL")
    print(f"\nM2 Money Supply (latest via CSV): ${m2[-1]['value']:,.1f}B on {m2[-1]['date']}")
```

## Live Data Sources
- **API base**: `https://api.stlouisfed.org/fred/`
- **CSV endpoint**: `https://fred.stlouisfed.org/graph/fredgraph.csv?id=SERIES_ID`
- **API documentation**: https://fred.stlouisfed.org/docs/api/fred/
- **Free API key**: https://fred.stlouisfed.org/docs/api/api_key.html
- **Coverage**: 800K+ series from BLS, BEA, Census Bureau, Federal Reserve, ECB, World Bank, and more
- **ALFRED** (vintage data): https://alfred.stlouisfed.org/ for real-time data vintages
