---
name: openalexander
description: Search 250M+ academic works, authors, institutions, and citation graphs via the OpenAlex API
license: MIT
triggers:
  - openalex
  - openalexander
  - academic paper search
  - citation graph
  - research works api
  - find academic authors
  - institution publication count
  - topic concept search
  - scholarly literature search
metadata:
  skill-author: gonzih
  data-sources:
    - api.openalex.org — 250M+ works, 90M+ authors, 110K+ institutions, 65K+ concepts; free; no auth required
  byok:
    - none — but add email to User-Agent for polite pool (10 req/s vs 100K/day anonymous)
  compatibility: claude-code>=1.0
---

# OpenAlex Academic Literature Skill

## What it does
Queries the OpenAlex API for academic works, authors, institutions, concepts, and venues. Covers 250M+ scholarly works across all disciplines. Supports full-text search, filtering by year/type/OA status, author/institution lookup, citation graph traversal, and concept-based navigation.

## How to invoke
- "Search OpenAlex for papers about protein folding"
- "Find publications by Geoffrey Hinton"
- "Get the top 10 most cited papers on machine learning from 2020"
- "Look up MIT's publication stats on OpenAlex"
- "Find papers citing this DOI"
- "What papers reference 'attention is all you need'?"

## Key parameters
- `query` — full-text search string
- `filter` — OpenAlex filter expression (e.g. `publication_year:2023,type:article`)
- `sort` — `cited_by_count:desc`, `publication_date:desc`, `relevance_score:desc`
- `per_page` — results per page (max 200)
- `page` — pagination (or use `cursor` for deep pagination)
- `email` — include in User-Agent for polite pool access (10 req/s)

## Rate limits
- Anonymous: 100K requests/day, burst limited
- Polite pool (with email in User-Agent): 10 req/s sustained
- No API key required

## Workflow steps
1. Build request URL for `https://api.openalex.org/{entity}`
2. Set `User-Agent: mailto:your@email.com` header for polite pool
3. Parse JSON response, extract `results` array and `meta` for pagination
4. Navigate related entities via OpenAlex IDs in response

## Working code example

```python
import json
import urllib.request
import urllib.parse
from typing import Optional

OPENALEX_BASE = "https://api.openalex.org"
# Use your email for polite pool (10 req/s instead of rate-limited anonymous)
CONTACT_EMAIL = "user@example.com"  # Replace with your email

HEADERS = {
    "User-Agent": f"skills-science/1.0 (mailto:{CONTACT_EMAIL})",
    "Accept": "application/json",
}


def _oa_get(entity: str, params: dict = None) -> dict:
    params = params or {}
    url = f"{OPENALEX_BASE}/{entity}?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers=HEADERS)
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())


# ── Works (papers) ────────────────────────────────────────────────────────────

def search_works(
    query: str,
    filters: dict = None,
    sort: str = "relevance_score:desc",
    per_page: int = 10,
    page: int = 1,
) -> dict:
    """
    Search OpenAlex works (papers).

    filter examples:
        {"publication_year": "2023", "type": "article", "is_oa": "true"}
        {"cited_by_count": ">100", "concepts.id": "C154945302"}  # C154945302 = ML
    """
    params = {
        "search": query,
        "sort": sort,
        "per_page": per_page,
        "page": page,
    }
    if filters:
        params["filter"] = ",".join(f"{k}:{v}" for k, v in filters.items())

    data = _oa_get("works", params)
    meta = data.get("meta", {})
    return {
        "total_count": meta.get("count", 0),
        "page": page,
        "per_page": per_page,
        "results": [_format_work(w) for w in data.get("results", [])],
    }


def _format_work(w: dict) -> dict:
    return {
        "id": w.get("id", "").replace("https://openalex.org/", ""),
        "doi": w.get("doi"),
        "title": w.get("title"),
        "publication_year": w.get("publication_year"),
        "publication_date": w.get("publication_date"),
        "type": w.get("type"),
        "cited_by_count": w.get("cited_by_count", 0),
        "is_oa": w.get("open_access", {}).get("is_oa", False),
        "oa_url": w.get("open_access", {}).get("oa_url"),
        "authors": [
            {
                "name": a.get("author", {}).get("display_name"),
                "id": a.get("author", {}).get("id", "").replace("https://openalex.org/", ""),
                "institution": (a.get("institutions") or [{}])[0].get("display_name") if a.get("institutions") else None,
            }
            for a in (w.get("authorships") or [])[:8]
        ],
        "venue": (w.get("primary_location") or {}).get("source", {}).get("display_name") if w.get("primary_location") else None,
        "concepts": [c.get("display_name") for c in (w.get("concepts") or [])[:5]],
        "abstract_preview": _reconstruct_abstract(w.get("abstract_inverted_index"), max_words=60),
        "openalex_url": f"https://openalex.org/{w.get('id', '').replace('https://openalex.org/', '')}",
    }


def _reconstruct_abstract(inverted_index: Optional[dict], max_words: int = 100) -> str:
    """OpenAlex stores abstracts as inverted index {word: [positions]}."""
    if not inverted_index:
        return ""
    word_list = [""] * (max(pos for positions in inverted_index.values() for pos in positions) + 1)
    for word, positions in inverted_index.items():
        for pos in positions:
            if pos < len(word_list):
                word_list[pos] = word
    abstract = " ".join(w for w in word_list if w)
    words = abstract.split()
    return " ".join(words[:max_words]) + ("..." if len(words) > max_words else "")


def get_work_by_doi(doi: str) -> dict:
    """Fetch a single work by DOI."""
    doi_encoded = urllib.parse.quote(doi, safe="")
    data = _oa_get(f"works/https://doi.org/{doi_encoded}")
    return _format_work(data)


def get_citations(work_id: str, per_page: int = 10) -> dict:
    """Get works that cite a given work (by OpenAlex ID like 'W2741809807')."""
    return search_works(
        query="",
        filters={"cites": work_id},
        sort="cited_by_count:desc",
        per_page=per_page,
    )


def get_references(work_id: str) -> list[dict]:
    """Get the reference list of a work."""
    data = _oa_get(f"works/{work_id}")
    refs = data.get("referenced_works", [])
    return [r.replace("https://openalex.org/", "") for r in refs]


# ── Authors ───────────────────────────────────────────────────────────────────

def search_authors(name: str, per_page: int = 5) -> list[dict]:
    """Search for authors by name."""
    data = _oa_get("authors", {"search": name, "per_page": per_page})
    return [_format_author(a) for a in data.get("results", [])]


def _format_author(a: dict) -> dict:
    return {
        "id": a.get("id", "").replace("https://openalex.org/", ""),
        "name": a.get("display_name"),
        "orcid": a.get("orcid"),
        "works_count": a.get("works_count", 0),
        "cited_by_count": a.get("cited_by_count", 0),
        "h_index": a.get("summary_stats", {}).get("h_index"),
        "i10_index": a.get("summary_stats", {}).get("i10_index"),
        "affiliations": [
            aff.get("institution", {}).get("display_name")
            for aff in (a.get("affiliations") or [])[:3]
        ],
        "top_concepts": [c.get("display_name") for c in (a.get("x_concepts") or [])[:5]],
        "openalex_url": f"https://openalex.org/{a.get('id', '').replace('https://openalex.org/', '')}",
    }


def get_author_works(author_id: str, per_page: int = 10) -> dict:
    """Get works by an author (OpenAlex ID like 'A5023888391')."""
    return search_works(
        query="",
        filters={"author.id": author_id},
        sort="cited_by_count:desc",
        per_page=per_page,
    )


# ── Institutions ──────────────────────────────────────────────────────────────

def search_institutions(name: str, per_page: int = 5) -> list[dict]:
    """Search for institutions by name."""
    data = _oa_get("institutions", {"search": name, "per_page": per_page})
    return [
        {
            "id": i.get("id", "").replace("https://openalex.org/", ""),
            "name": i.get("display_name"),
            "type": i.get("type"),
            "country_code": i.get("country_code"),
            "ror": i.get("ror"),
            "works_count": i.get("works_count", 0),
            "cited_by_count": i.get("cited_by_count", 0),
            "h_index": i.get("summary_stats", {}).get("h_index"),
            "top_concepts": [c.get("display_name") for c in (i.get("x_concepts") or [])[:5]],
        }
        for i in data.get("results", [])
    ]


# ── Concepts ──────────────────────────────────────────────────────────────────

def search_concepts(name: str, per_page: int = 5) -> list[dict]:
    """Search for academic concepts/topics."""
    data = _oa_get("concepts", {"search": name, "per_page": per_page})
    return [
        {
            "id": c.get("id", "").replace("https://openalex.org/", ""),
            "name": c.get("display_name"),
            "level": c.get("level"),  # 0=root, 5=very specific
            "description": c.get("description"),
            "works_count": c.get("works_count", 0),
            "cited_by_count": c.get("cited_by_count", 0),
            "wikidata": c.get("wikidata"),
        }
        for c in data.get("results", [])
    ]


# --- Example usage ---
if __name__ == "__main__":
    # Search for recent transformer papers
    results = search_works(
        query="transformer self-attention neural language model",
        filters={"publication_year": "2023", "type": "article"},
        sort="cited_by_count:desc",
        per_page=5,
    )
    print(f"Found {results['total_count']:,} works. Top {len(results['results'])}:")
    for w in results["results"]:
        authors = ", ".join(a["name"] for a in w["authors"][:2] if a["name"])
        print(f"  [{w['publication_year']}] {w['cited_by_count']} cites — {w['title'][:70]}")
        print(f"    By: {authors} | OA: {w['is_oa']}")
    print()

    # Author lookup
    authors = search_authors("Yoshua Bengio", per_page=1)
    if authors:
        a = authors[0]
        print(f"Author: {a['name']} (h-index: {a['h_index']}, {a['works_count']} works)")
        print(f"  Concepts: {', '.join(a['top_concepts'])}")
    print()

    # Institution lookup
    insts = search_institutions("MIT", per_page=1)
    if insts:
        i = insts[0]
        print(f"Institution: {i['name']} ({i['country_code']}) — {i['works_count']:,} works")
```

## Live Data Sources
- **API base**: `https://api.openalex.org/`
- **Documentation**: https://docs.openalex.org/
- **Coverage**: 250M+ works, 90M+ authors, 110K+ institutions, 65K+ concepts
- **Data dump**: https://docs.openalex.org/download-all-data/openalex-snapshot (free S3 snapshot)
- **Entity types**: works, authors, institutions, concepts, venues, publishers, funders
