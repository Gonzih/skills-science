---
name: pubmed
description: Search PubMed and fetch NCBI records via E-utilities — abstracts, MeSH terms, citations
license: MIT
triggers:
  - search pubmed
  - find medical papers
  - look up pubmed abstracts
  - ncbi entrez search
  - fetch pubmed records
  - search biomedical literature
metadata:
  skill-author: gonzih
  data-sources:
    - eutils.ncbi.nlm.nih.gov — 35M+ PubMed records, GenBank, PMC; free; optional API key
  byok:
    - NCBI_API_KEY — raises rate limit from 3 req/s to 10 req/s (free at https://www.ncbi.nlm.nih.gov/account/)
  compatibility: claude-code>=1.0
---

# PubMed / NCBI E-utilities Skill

## What it does
Searches PubMed biomedical literature using NCBI E-utilities (esearch + efetch + einfo). Returns structured records with PMID, title, authors, abstract, journal, publication date, MeSH terms, and DOI. Supports all NCBI databases: pubmed, pmc, nucleotide, protein, gene, etc.

## How to invoke
- "Search PubMed for CRISPR-Cas9 off-target effects"
- "Find recent review articles about Alzheimer's treatment"
- "Look up PubMed abstract for PMID 34521426"
- "Search NCBI for papers about COVID-19 mRNA vaccines from 2021"

## Key parameters
- `query` — Entrez query string (supports field tags: `[Title]`, `[Author]`, `[MeSH]`, `[PDAT]`, `[DP]`)
- `max_results` — records to return (default 20, use `retmax`)
- `db` — NCBI database: `pubmed`, `pmc`, `nucleotide`, `protein`, `gene` (default: `pubmed`)
- `retmode` — return format: `xml`, `json`, `text`
- `sort` — `relevance`, `pub_date`, `Author`, `JournalName`

## Rate limits
- Without key: 3 requests/second
- With `NCBI_API_KEY`: 10 requests/second
- Always include `tool` and `email` params for compliance
- Bulk downloads: use NCBI FTP or EDirect instead

## Workflow steps
1. **esearch** — submit query, get list of PMIDs
2. **efetch** — fetch full records for PMIDs in XML format
3. Parse XML with ElementTree, extract structured fields
4. Return list of record dicts

## Working code example

```python
import os
import time
import urllib.request
import urllib.parse
import xml.etree.ElementTree as ET
from typing import Optional

NCBI_BASE = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils"
API_KEY = os.environ.get("NCBI_API_KEY", "")
TOOL = "skills-science"
EMAIL = "user@example.com"  # Replace with your email

def _ncbi_get(endpoint: str, params: dict) -> bytes:
    params.update({"tool": TOOL, "email": EMAIL})
    if API_KEY:
        params["api_key"] = API_KEY
    url = f"{NCBI_BASE}/{endpoint}?" + urllib.parse.urlencode(params)
    with urllib.request.urlopen(url) as resp:
        data = resp.read()
    # Respect rate limits
    time.sleep(0.11 if API_KEY else 0.34)
    return data


def search_pubmed(
    query: str,
    max_results: int = 20,
    db: str = "pubmed",
    sort: str = "relevance",
) -> list[str]:
    """Run esearch and return list of UIDs/PMIDs."""
    params = {
        "db": db,
        "term": query,
        "retmax": max_results,
        "sort": sort,
        "retmode": "json",
        "usehistory": "y",
    }
    data = _ncbi_get("esearch.fcgi", params)
    result = __import__("json").loads(data)
    ids = result["esearchresult"]["idlist"]
    return ids


def fetch_records(
    pmids: list[str],
    db: str = "pubmed",
) -> list[dict]:
    """Fetch full records for a list of PMIDs via efetch."""
    if not pmids:
        return []
    params = {
        "db": db,
        "id": ",".join(pmids),
        "retmode": "xml",
        "rettype": "abstract",
    }
    data = _ncbi_get("efetch.fcgi", params)
    return _parse_pubmed_xml(data)


def _parse_pubmed_xml(xml_data: bytes) -> list[dict]:
    root = ET.fromstring(xml_data)
    records = []

    for article in root.findall(".//PubmedArticle"):
        medline = article.find("MedlineCitation")
        if medline is None:
            continue

        pmid_el = medline.find("PMID")
        pmid = pmid_el.text if pmid_el is not None else ""

        art = medline.find("Article")
        if art is None:
            continue

        title_el = art.find("ArticleTitle")
        title = "".join(title_el.itertext()) if title_el is not None else ""

        # Abstract (may have multiple AbstractText sections)
        abstract_parts = []
        for ab in art.findall(".//AbstractText"):
            label = ab.attrib.get("Label", "")
            text = "".join(ab.itertext())
            if label:
                abstract_parts.append(f"{label}: {text}")
            else:
                abstract_parts.append(text)
        abstract = " ".join(abstract_parts)

        # Authors
        authors = []
        for auth in art.findall(".//Author"):
            last = auth.findtext("LastName", "")
            fore = auth.findtext("ForeName", "")
            if last:
                authors.append(f"{last} {fore}".strip())

        # Journal info
        journal = art.findtext(".//Journal/Title", "")
        pub_year = art.findtext(".//PubDate/Year", "")
        pub_month = art.findtext(".//PubDate/Month", "")
        volume = art.findtext(".//JournalIssue/Volume", "")
        issue = art.findtext(".//JournalIssue/Issue", "")

        # DOI
        doi = ""
        for eid in art.findall(".//ELocationID"):
            if eid.attrib.get("EIdType") == "doi":
                doi = eid.text

        # MeSH terms
        mesh_terms = [
            mh.findtext("DescriptorName", "")
            for mh in medline.findall(".//MeshHeading")
        ]

        records.append({
            "pmid": pmid,
            "title": title,
            "authors": authors[:10],
            "abstract": abstract[:800] + "..." if len(abstract) > 800 else abstract,
            "journal": journal,
            "pub_date": f"{pub_year} {pub_month}".strip(),
            "volume": volume,
            "issue": issue,
            "doi": doi,
            "mesh_terms": mesh_terms[:15],
            "pubmed_url": f"https://pubmed.ncbi.nlm.nih.gov/{pmid}/",
        })

    return records


def search_and_fetch(query: str, max_results: int = 10, db: str = "pubmed") -> list[dict]:
    """Convenience: search then fetch in one call."""
    pmids = search_pubmed(query, max_results=max_results, db=db)
    return fetch_records(pmids, db=db)


# --- Example usage ---
if __name__ == "__main__":
    # Search for CRISPR papers
    results = search_and_fetch(
        query="CRISPR Cas9 off-target effects[Title] AND Review[Publication Type]",
        max_results=5,
    )
    for r in results:
        print(f"PMID {r['pmid']}: {r['title'][:80]}")
        print(f"  Authors: {', '.join(r['authors'][:3])}")
        print(f"  Journal: {r['journal']} ({r['pub_date']})")
        if r['doi']:
            print(f"  DOI: https://doi.org/{r['doi']}")
        print(f"  MeSH: {', '.join(r['mesh_terms'][:5])}")
        print()
```

## Live Data Sources
- **esearch**: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi`
- **efetch**: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi`
- **einfo**: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/einfo.fcgi`
- **Documentation**: https://www.ncbi.nlm.nih.gov/books/NBK25497/
- **API key registration**: https://www.ncbi.nlm.nih.gov/account/
- **Entrez query syntax**: https://www.ncbi.nlm.nih.gov/books/NBK3837/
- **Coverage**: 35M+ PubMed records, dating to 1966
