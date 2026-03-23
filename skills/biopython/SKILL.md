---
name: biopython
description: Biological sequence parsing, NCBI Entrez queries, remote BLAST, and pairwise alignment using Biopython
license: MIT
triggers:
  - parse fasta sequence
  - biopython
  - blast sequence
  - parse genbank file
  - ncbi entrez biopython
  - translate dna sequence
  - pairwise sequence alignment
  - fetch gene sequence from ncbi
metadata:
  skill-author: gonzih
  data-sources:
    - blast.ncbi.nlm.nih.gov — remote BLAST against nr/nt/swissprot; free; no key
    - eutils.ncbi.nlm.nih.gov — Entrez queries for GenBank/PubMed/Protein; free; optional NCBI_API_KEY
  byok:
    - NCBI_API_KEY — raises NCBI rate limit from 3 req/s to 10 req/s (optional)
    - ENTREZ_EMAIL — required by NCBI ToS to identify your queries
  compatibility: claude-code>=1.0
---

# Biopython Skill

## What it does
Uses Biopython to parse FASTA/GenBank/FASTQ sequences, query NCBI Entrez databases, run remote BLAST searches, perform pairwise sequence alignment, and translate DNA to protein. Works with sequences from files or direct NCBI downloads.

## How to invoke
- "Parse this FASTA file and show sequence lengths"
- "Fetch the nucleotide sequence for gene NM_005228 from NCBI"
- "BLAST this protein sequence against SwissProt"
- "Translate this CDS to protein"
- "Align these two sequences with pairwise2"

## Key parameters
- `seq_file` — path to FASTA/GenBank/FASTQ file
- `accession` — NCBI accession (e.g. `NM_005228`, `P12931`)
- `sequence` — raw sequence string
- `program` — BLAST program: `blastn`, `blastp`, `blastx`, `tblastn`
- `database` — BLAST database: `nr`, `nt`, `swissprot`, `refseq_rna`
- `db` — Entrez database: `nucleotide`, `protein`, `gene`, `pubmed`

## Rate limits
- NCBI Entrez: 3 req/s without key, 10 req/s with NCBI_API_KEY
- Remote BLAST: variable, typically 30–120s per job; limited concurrent jobs
- Always set `Entrez.email` — required by NCBI terms of service

## Workflow steps
1. Install biopython: `pip install biopython`
2. Set `Entrez.email` before any NCBI calls
3. Use `SeqIO.parse()` for file parsing or `Entrez.efetch()` for downloads
4. For BLAST: submit with `NCBIWWW.qblast()`, parse with `NCBIXML.read()`
5. For alignment: use `PairwiseAligner` (Bio.Align) — modern API

## Working code example

```python
import os
import time
from io import StringIO, BytesIO

from Bio import SeqIO, Entrez, Seq
from Bio.Blast import NCBIWWW, NCBIXML
from Bio import Align

# REQUIRED by NCBI Terms of Service
Entrez.email = os.environ.get("ENTREZ_EMAIL", "your_email@example.com")
if os.environ.get("NCBI_API_KEY"):
    Entrez.api_key = os.environ["NCBI_API_KEY"]


# ── FASTA / GenBank parsing ───────────────────────────────────────────────────

def parse_fasta(filepath: str) -> list[dict]:
    """Parse a FASTA file and return sequence info."""
    records = []
    for rec in SeqIO.parse(filepath, "fasta"):
        records.append({
            "id": rec.id,
            "description": rec.description,
            "length": len(rec.seq),
            "sequence": str(rec.seq),
            "gc_content": round(
                (str(rec.seq).upper().count("G") + str(rec.seq).upper().count("C"))
                / len(rec.seq) * 100, 2
            ) if len(rec.seq) > 0 else 0,
        })
    return records


def parse_genbank(filepath: str) -> list[dict]:
    """Parse a GenBank file and extract annotations + features."""
    records = []
    for rec in SeqIO.parse(filepath, "genbank"):
        features = []
        for feat in rec.features:
            if feat.type in ("CDS", "gene", "mRNA", "rRNA", "tRNA"):
                gene_name = feat.qualifiers.get("gene", [""])[0]
                product = feat.qualifiers.get("product", [""])[0]
                features.append({
                    "type": feat.type,
                    "gene": gene_name,
                    "product": product,
                    "location": str(feat.location),
                })
        records.append({
            "id": rec.id,
            "name": rec.name,
            "description": rec.description,
            "length": len(rec.seq),
            "organism": rec.annotations.get("organism", ""),
            "accession": rec.annotations.get("accessions", [""])[0],
            "features_count": len(features),
            "features": features[:20],
        })
    return records


# ── NCBI Entrez ───────────────────────────────────────────────────────────────

def fetch_nucleotide(accession: str) -> dict:
    """Fetch a nucleotide record from NCBI by accession number."""
    with Entrez.efetch(db="nucleotide", id=accession, rettype="gb", retmode="text") as handle:
        record = SeqIO.read(handle, "genbank")
    time.sleep(0.4)
    return {
        "id": record.id,
        "name": record.name,
        "description": record.description,
        "length": len(record.seq),
        "organism": record.annotations.get("organism", ""),
        "sequence_preview": str(record.seq[:100]),
        "cds_count": sum(1 for f in record.features if f.type == "CDS"),
    }


def search_ncbi(query: str, db: str = "nucleotide", max_results: int = 10) -> list[str]:
    """Search NCBI and return list of accession IDs."""
    with Entrez.esearch(db=db, term=query, retmax=max_results) as handle:
        result = Entrez.read(handle)
    time.sleep(0.4)
    return result["IdList"]


def fetch_protein(accession: str) -> dict:
    """Fetch protein sequence from NCBI."""
    with Entrez.efetch(db="protein", id=accession, rettype="fasta", retmode="text") as handle:
        record = SeqIO.read(handle, "fasta")
    time.sleep(0.4)
    return {
        "id": record.id,
        "description": record.description,
        "length": len(record.seq),
        "sequence": str(record.seq),
    }


# ── Sequence operations ───────────────────────────────────────────────────────

def translate_cds(dna_sequence: str, table: int = 1) -> str:
    """Translate a DNA coding sequence to protein."""
    seq = Seq.Seq(dna_sequence.upper().replace(" ", ""))
    # Pad to multiple of 3 if needed
    remainder = len(seq) % 3
    if remainder:
        seq = seq[:-remainder]
    return str(seq.translate(table=table, to_stop=True))


def reverse_complement(dna_sequence: str) -> str:
    """Return the reverse complement of a DNA sequence."""
    return str(Seq.Seq(dna_sequence).reverse_complement())


def compute_gc(sequence: str) -> float:
    """GC content as a percentage."""
    s = sequence.upper()
    gc = s.count("G") + s.count("C")
    return round(gc / len(s) * 100, 2) if s else 0.0


# ── Pairwise alignment ────────────────────────────────────────────────────────

def align_sequences(seq1: str, seq2: str, mode: str = "global") -> dict:
    """
    Pairwise sequence alignment using Bio.Align.PairwiseAligner.
    mode: 'global' (Needleman-Wunsch) or 'local' (Smith-Waterman)
    """
    aligner = Align.PairwiseAligner()
    aligner.mode = mode
    aligner.match_score = 2
    aligner.mismatch_score = -1
    aligner.open_gap_score = -2
    aligner.extend_gap_score = -0.5

    alignments = aligner.align(seq1.upper(), seq2.upper())
    best = next(iter(alignments))
    return {
        "score": best.score,
        "mode": mode,
        "aligned_length": best.length,
        "identity_pct": round(
            sum(a == b for a, b in zip(str(best[0]), str(best[1]))) / best.length * 100, 2
        ),
        "alignment_str": str(best)[:500],
    }


# ── Remote BLAST ──────────────────────────────────────────────────────────────

def blast_sequence(
    sequence: str,
    program: str = "blastp",
    database: str = "swissprot",
    hitlist_size: int = 10,
) -> list[dict]:
    """
    Run remote BLAST against NCBI.
    program: blastn, blastp, blastx, tblastn, tblastx
    database: nr, nt, swissprot, refseq_protein, refseq_rna, pdb
    Note: may take 30-120 seconds.
    """
    result_handle = NCBIWWW.qblast(program, database, sequence, hitlist_size=hitlist_size)
    blast_records = NCBIXML.read(result_handle)

    hits = []
    for alignment in blast_records.alignments[:hitlist_size]:
        hsp = alignment.hsps[0]  # best HSP
        hits.append({
            "title": alignment.title[:100],
            "accession": alignment.accession,
            "length": alignment.length,
            "score": hsp.score,
            "e_value": hsp.expect,
            "identity_pct": round(hsp.identities / hsp.align_length * 100, 1),
            "query_start": hsp.query_start,
            "query_end": hsp.query_end,
            "subject_start": hsp.sbjct_start,
            "subject_end": hsp.sbjct_end,
        })
    return hits


# --- Example usage ---
if __name__ == "__main__":
    # Translate a simple CDS
    dna = "ATGAAAGCAATTTTCGTACTGAAAGGTTTTGTTGGTTTTCTTGCCATGAGC"
    protein = translate_cds(dna)
    print(f"DNA: {dna}")
    print(f"Protein: {protein}")
    print(f"GC content: {compute_gc(dna)}%")
    print()

    # Fetch a nucleotide record from NCBI (EGFR mRNA)
    rec = fetch_nucleotide("NM_005228")
    print(f"Fetched: {rec['id']} — {rec['description'][:60]}")
    print(f"  Organism: {rec['organism']}, Length: {rec['length']} bp, CDS count: {rec['cds_count']}")
    print()

    # Pairwise alignment
    seq_a = "MKTLLLTLVVVTIVCLDLGAVGNLSQMAQDIGPPTGPAFMQDMSSFYNTAATAQAMAPTSS"
    seq_b = "MKTLLLTLVVVTIVCLDLGAVGNLSQMAQDIGPPTGPAFMQDMSSFYNTAATAQAMAPTSA"
    aln = align_sequences(seq_a, seq_b, mode="global")
    print(f"Alignment score: {aln['score']}, Identity: {aln['identity_pct']}%")
```

## Live Data Sources
- **NCBI Entrez**: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/`
- **Remote BLAST**: `https://blast.ncbi.nlm.nih.gov/blast/Blast.cgi`
- **Biopython docs**: https://biopython.org/wiki/Documentation
- **Biopython tutorial**: https://biopython.org/DIST/docs/tutorial/Tutorial.html
- **NCBI databases**: nucleotide, protein, gene, pubmed, taxonomy, structure
