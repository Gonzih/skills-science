---
name: arxiv
description: Search and retrieve arXiv preprints via the Atom XML API — no API key required
license: MIT
triggers:
  - search arxiv
  - find papers on arxiv
  - look up arxiv preprints
  - fetch arxiv results
  - arxiv search for
metadata:
  skill-author: gonzih
  data-sources:
    - export.arxiv.org — 2M+ preprints across physics, math, CS, biology, economics; free; no auth
  byok:
    - none
  compatibility: claude-code>=1.0
---

# arXiv Search Skill

## What it does
Searches arXiv.org preprint repository using the official Atom API. Returns structured results including title, authors, abstract, submission date, categories, and PDF links. Covers all arXiv subject areas: cs.*, math.*, physics.*, q-bio.*, econ.*, stat.*, and more.

## How to invoke
- "Search arxiv for transformer attention mechanisms"
- "Find recent papers by Yann LeCun on arxiv"
- "Look up arxiv papers in cs.LG about diffusion models from 2024"

## Key parameters
- `query` — free-text search query (supports field prefixes: `ti:`, `au:`, `abs:`, `cat:`)
- `max_results` — number of results (default 10, max 2000 per request)
- `start` — offset for pagination
- `sort_by` — `relevance`, `lastUpdatedDate`, or `submittedDate`
- `sort_order` — `descending` or `ascending`
- `category` — e.g. `cs.LG`, `quant-ph`, `math.CO`

## Rate limits
- No key required
- Recommended: max 1 request every 3 seconds
- Bulk access: use OAI-PMH harvesting interface instead

## Workflow steps
1. Construct query URL with `http://export.arxiv.org/api/query`
2. Parse Atom XML response with `xml.etree.ElementTree`
3. Extract entries: id, title, authors, summary, published, updated, categories, links
4. Return structured list of paper dicts

## Working code example

```python
import urllib.request
import urllib.parse
import xml.etree.ElementTree as ET
from datetime import datetime

ARXIV_API = "http://export.arxiv.org/api/query"
ATOM_NS = "http://www.w3.org/2005/Atom"
ARXIV_NS = "http://arxiv.org/schemas/atom"

def search_arxiv(
    query: str,
    max_results: int = 10,
    start: int = 0,
    sort_by: str = "relevance",
    sort_order: str = "descending",
    category: str = None,
) -> list[dict]:
    """
    Search arXiv preprints.

    Examples:
        search_arxiv("attention is all you need", max_results=5)
        search_arxiv("au:Hinton AND ti:deep learning", sort_by="submittedDate")
        search_arxiv("diffusion models", category="cs.LG", max_results=20)
    """
    q = query
    if category:
        q = f"({query}) AND cat:{category}"

    params = {
        "search_query": q,
        "start": start,
        "max_results": max_results,
        "sortBy": sort_by,
        "sortOrder": sort_order,
    }
    url = ARXIV_API + "?" + urllib.parse.urlencode(params)

    with urllib.request.urlopen(url) as resp:
        xml_data = resp.read()

    root = ET.fromstring(xml_data)
    results = []

    for entry in root.findall(f"{{{ATOM_NS}}}entry"):
        arxiv_id = entry.find(f"{{{ATOM_NS}}}id").text.split("/abs/")[-1]
        title = entry.find(f"{{{ATOM_NS}}}title").text.strip().replace("\n", " ")
        summary = entry.find(f"{{{ATOM_NS}}}summary").text.strip().replace("\n", " ")
        published = entry.find(f"{{{ATOM_NS}}}published").text
        updated = entry.find(f"{{{ATOM_NS}}}updated").text

        authors = [
            a.find(f"{{{ATOM_NS}}}name").text
            for a in entry.findall(f"{{{ATOM_NS}}}author")
        ]

        categories = [
            c.attrib.get("term")
            for c in entry.findall(f"{{{ATOM_NS}}}category")
        ]

        pdf_url = None
        for link in entry.findall(f"{{{ATOM_NS}}}link"):
            if link.attrib.get("title") == "pdf":
                pdf_url = link.attrib.get("href")

        results.append({
            "arxiv_id": arxiv_id,
            "title": title,
            "authors": authors,
            "abstract": summary[:500] + "..." if len(summary) > 500 else summary,
            "published": published[:10],
            "updated": updated[:10],
            "categories": categories,
            "pdf_url": pdf_url,
            "abs_url": f"https://arxiv.org/abs/{arxiv_id}",
        })

    return results


def fetch_paper_by_id(arxiv_id: str) -> dict:
    """Fetch a single paper by arXiv ID (e.g. '1706.03762' or '2301.00001v2')."""
    params = {"id_list": arxiv_id, "max_results": 1}
    url = ARXIV_API + "?" + urllib.parse.urlencode(params)
    with urllib.request.urlopen(url) as resp:
        xml_data = resp.read()
    results = _parse_entries(ET.fromstring(xml_data))
    return results[0] if results else None


def _parse_entries(root):
    results = []
    for entry in root.findall(f"{{{ATOM_NS}}}entry"):
        if entry.find(f"{{{ATOM_NS}}}title") is None:
            continue
        arxiv_id = entry.find(f"{{{ATOM_NS}}}id").text.split("/abs/")[-1]
        results.append({
            "arxiv_id": arxiv_id,
            "title": entry.find(f"{{{ATOM_NS}}}title").text.strip(),
            "authors": [
                a.find(f"{{{ATOM_NS}}}name").text
                for a in entry.findall(f"{{{ATOM_NS}}}author")
            ],
            "abstract": entry.find(f"{{{ATOM_NS}}}summary").text.strip(),
            "published": entry.find(f"{{{ATOM_NS}}}published").text[:10],
            "categories": [
                c.attrib.get("term")
                for c in entry.findall(f"{{{ATOM_NS}}}category")
            ],
            "abs_url": f"https://arxiv.org/abs/{arxiv_id}",
        })
    return results


# --- Example usage ---
if __name__ == "__main__":
    # Search for recent diffusion model papers in machine learning
    papers = search_arxiv(
        query="score-based diffusion generative models",
        max_results=5,
        sort_by="submittedDate",
        category="cs.LG",
    )
    for p in papers:
        print(f"[{p['published']}] {p['arxiv_id']}: {p['title']}")
        print(f"  Authors: {', '.join(p['authors'][:3])}")
        print(f"  URL: {p['abs_url']}")
        print()

    # Fetch specific paper (Attention Is All You Need)
    paper = fetch_paper_by_id("1706.03762")
    print(f"Fetched: {paper['title']}")
```

## Live Data Sources
- **API endpoint**: `http://export.arxiv.org/api/query`
- **Documentation**: https://info.arxiv.org/help/api/index.html
- **OAI-PMH bulk harvest**: `http://export.arxiv.org/oai2`
- **Subject categories**: https://arxiv.org/category_taxonomy
- **Coverage**: 2M+ papers since 1991, new submissions daily
