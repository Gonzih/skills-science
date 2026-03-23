---
name: nasa-open-data
description: NASA APIs — Astronomy Picture of the Day, Mars Rover photos, Near Earth Objects, Earth imagery, and more (BYOK: NASA_API_KEY, free registration)
license: MIT
triggers:
  - nasa apod
  - astronomy picture of the day
  - mars rover photos
  - near earth objects
  - nasa asteroid data
  - nasa earth imagery
  - nasa open data
  - fetch nasa data
  - space weather data
metadata:
  skill-author: gonzih
  data-sources:
    - api.nasa.gov — APOD, Mars Rover, NEO Feed, Earth Polychromatic Imaging Camera, Insight weather; free; NASA_API_KEY required (free at https://api.nasa.gov/)
  byok:
    - NASA_API_KEY — required; free key at https://api.nasa.gov/ (demo key "DEMO_KEY" works at 30 req/hr, 50 req/day)
  compatibility: claude-code>=1.0
---

# NASA Open Data Skill

## What it does
Retrieves data from NASA's public APIs: Astronomy Picture of the Day (APOD), Mars Rover mission photos (Curiosity, Opportunity, Spirit, Perseverance), Near Earth Object (NEO/asteroid) feed, Earth observation imagery, and space weather. All APIs are free with a NASA API key (registration takes under a minute).

## How to invoke
- "Show me today's NASA Astronomy Picture of the Day"
- "Get Mars Curiosity rover photos from Sol 1000"
- "List near-Earth asteroids passing by this week"
- "Fetch NEO data for asteroid 2021 PH27"
- "Get APOD for 2024-07-04"

## Key parameters
- `nasa_api_key` — from `NASA_API_KEY` env var; "DEMO_KEY" works for testing (rate-limited)
- `date` — ISO 8601 date string (YYYY-MM-DD) for date-specific queries
- `rover` — `curiosity`, `opportunity`, `spirit`, or `perseverance` for Mars Rover queries
- `sol` — Martian solar day (0-based) for rover photos
- `start_date` / `end_date` — date range for NEO feed (max 7-day window)

## Rate limits
- `DEMO_KEY`: 30 requests/hour, 50 requests/day
- Registered key: 1,000 requests/hour
- No charge for any usage tier

## Workflow steps
1. **Set key** — read `NASA_API_KEY` from environment; fall back to `"DEMO_KEY"` with warning
2. **Choose endpoint** — APOD, Mars Rover Photos, NEO Feed, or Earth imagery
3. **Build URL** — append `api_key` param and any date/filter params
4. **GET request** — standard HTTPS GET; response is JSON
5. **Parse** — extract relevant fields (title, url, explanation, date, etc.)
6. **Return** — structured dict or list

## Working code example

```python
import os
import json
import urllib.request
import urllib.parse
from datetime import date, timedelta
from typing import Optional

NASA_BASE = "https://api.nasa.gov"
API_KEY = os.environ.get("NASA_API_KEY", "DEMO_KEY")
if API_KEY == "DEMO_KEY":
    import warnings
    warnings.warn("Using DEMO_KEY — rate limited to 30 req/hr. Set NASA_API_KEY for full access.")


def _nasa_get(path: str, params: dict) -> dict | list:
    """GET request to NASA API, return parsed JSON."""
    params["api_key"] = API_KEY
    url = f"{NASA_BASE}{path}?" + urllib.parse.urlencode(params)
    with urllib.request.urlopen(url) as resp:
        return json.loads(resp.read())


# ── APOD ──────────────────────────────────────────────────────────────────────

def get_apod(apod_date: Optional[str] = None, count: int = 1) -> dict | list:
    """
    Fetch Astronomy Picture of the Day.

    Args:
        apod_date: ISO date string "YYYY-MM-DD" (default: today)
        count: number of random APODs to return (1 = specific date)
    Returns:
        Single APOD dict, or list of APOD dicts if count > 1
    """
    params: dict = {"thumbs": "true"}
    if count > 1:
        params["count"] = count
    elif apod_date:
        params["date"] = apod_date
    return _nasa_get("/planetary/apod", params)


# ── Mars Rover Photos ──────────────────────────────────────────────────────────

def get_mars_photos(
    rover: str = "curiosity",
    sol: Optional[int] = None,
    earth_date: Optional[str] = None,
    camera: Optional[str] = None,
    page: int = 1,
) -> list[dict]:
    """
    Fetch Mars Rover mission photos.

    Args:
        rover: "curiosity" | "opportunity" | "spirit" | "perseverance"
        sol: Martian solar day (integer, 0-based); mutually exclusive with earth_date
        earth_date: Earth date "YYYY-MM-DD"; mutually exclusive with sol
        camera: e.g. "FHAZ", "RHAZ", "MAST", "CHEMCAM", "NAVCAM", "PANCAM"
        page: results page (25 photos/page)
    Returns:
        List of photo dicts with id, sol, earth_date, img_src, camera, rover fields
    """
    if sol is None and earth_date is None:
        sol = 1000  # sensible default

    params: dict = {"page": page}
    if sol is not None:
        params["sol"] = sol
    elif earth_date:
        params["earth_date"] = earth_date
    if camera:
        params["camera"] = camera.upper()

    data = _nasa_get(f"/mars-photos/api/v1/rovers/{rover}/photos", params)
    return data.get("photos", [])


# ── Near Earth Objects ─────────────────────────────────────────────────────────

def get_neo_feed(
    start_date: Optional[str] = None,
    end_date: Optional[str] = None,
) -> dict:
    """
    Fetch Near Earth Object (asteroid) feed for a date range (max 7 days).

    Args:
        start_date: "YYYY-MM-DD" (default: today)
        end_date: "YYYY-MM-DD" (default: start_date + 7 days)
    Returns:
        Dict keyed by date, each containing list of NEO approach dicts
    """
    if start_date is None:
        start_date = date.today().isoformat()
    if end_date is None:
        end_date = (date.fromisoformat(start_date) + timedelta(days=7)).isoformat()

    params = {"start_date": start_date, "end_date": end_date}
    data = _nasa_get("/neo/rest/v1/feed", params)
    return data.get("near_earth_objects", {})


def lookup_neo(asteroid_id: str) -> dict:
    """Look up a specific NEO by NASA ID (e.g. '2021511')."""
    return _nasa_get(f"/neo/rest/v1/neo/{asteroid_id}", {})


# ── Example usage ──────────────────────────────────────────────────────────────

if __name__ == "__main__":
    # 1. Today's APOD
    apod = get_apod()
    print(f"APOD: {apod['title']} ({apod['date']})")
    print(f"  Media: {apod['media_type']} — {apod['url']}")
    print(f"  {apod['explanation'][:200]}...")
    print()

    # 2. Curiosity photos from Sol 1000
    photos = get_mars_photos(rover="curiosity", sol=1000)
    print(f"Curiosity Sol 1000 photos: {len(photos)} found")
    for p in photos[:3]:
        print(f"  [{p['camera']['full_name']}] {p['img_src']}")
    print()

    # 3. Near Earth Objects this week
    neo_feed = get_neo_feed()
    total = sum(len(v) for v in neo_feed.values())
    print(f"NEOs this week: {total} approaches across {len(neo_feed)} days")
    for day, neos in sorted(neo_feed.items())[:2]:
        print(f"  {day}: {len(neos)} objects")
        for neo in neos[:2]:
            km = neo["close_approach_data"][0]["miss_distance"]["kilometers"]
            print(f"    {neo['name']} — {float(km):,.0f} km miss distance")
```

## Example outputs

**APOD response:**
```json
{
  "title": "A Solar Prominence from SOHO",
  "date": "2024-07-04",
  "media_type": "image",
  "url": "https://apod.nasa.gov/apod/image/2407/...",
  "hdurl": "https://apod.nasa.gov/apod/image/2407/...hd.jpg",
  "explanation": "What's happening on the Sun? ...",
  "copyright": "ESA/NASA/SOHO"
}
```

**Mars Rover photo:**
```json
{
  "id": 424905,
  "sol": 1000,
  "earth_date": "2015-05-30",
  "img_src": "http://mars.jpl.nasa.gov/msl-raw-images/proj/msl/redops/ods/surface/sol/01000/opgs/edr/fcam/FLB_486265257EDR_F0481570FHAZ00323M_.JPG",
  "camera": { "name": "FHAZ", "full_name": "Front Hazard Avoidance Camera" },
  "rover": { "name": "Curiosity", "status": "active" }
}
```

**NEO entry:**
```json
{
  "name": "(2021 PH27)",
  "nasa_jpl_url": "http://ssd.jpl.nasa.gov/sbdb.cgi?sstr=...",
  "is_potentially_hazardous_asteroid": false,
  "estimated_diameter": { "kilometers": { "min": 0.024, "max": 0.054 } },
  "close_approach_data": [{
    "close_approach_date": "2024-07-08",
    "miss_distance": { "kilometers": "3847291.5", "lunar": "10.01" },
    "relative_velocity": { "kilometers_per_hour": "28450.3" }
  }]
}
```

## Live Data Sources
- **APOD**: `https://api.nasa.gov/planetary/apod`
- **Mars Rover Photos**: `https://api.nasa.gov/mars-photos/api/v1/rovers/{rover}/photos`
- **NEO Feed**: `https://api.nasa.gov/neo/rest/v1/feed`
- **NEO Lookup**: `https://api.nasa.gov/neo/rest/v1/neo/{id}`
- **API key registration**: https://api.nasa.gov/ (free, instant)
- **Full API catalog**: https://api.nasa.gov/#browseAPI
- **Coverage**: APOD archive from 1995-06-16 to present; Curiosity rover data from 2012 to present; NEO data updated daily from JPL
