# @gonzih/skills-science

Scientific data access skills for [Claude Code](https://claude.ai/code).

## Skills included

| Skill | Description | Auth |
|-------|-------------|------|
| [arxiv](skills/arxiv/SKILL.md) | Search arXiv preprints via Atom API | None |
| [pubmed](skills/pubmed/SKILL.md) | Search PubMed/NCBI via E-utilities | Optional: `NCBI_API_KEY` |
| [alpha-vantage](skills/alpha-vantage/SKILL.md) | Stock, forex, crypto & economic data | `ALPHA_VANTAGE_API_KEY` |
| [biopython](skills/biopython/SKILL.md) | Sequence parsing, BLAST, Entrez queries | None (public NCBI) |
| [rdkit-chemistry](skills/rdkit-chemistry/SKILL.md) | Cheminformatics, fingerprints, ChEMBL | None |
| [astropy](skills/astropy/SKILL.md) | Coordinates, cosmology, FITS I/O, Vizier | None |
| [openalexander](skills/openalexander/SKILL.md) | 250M+ academic works via OpenAlex API | None (polite pool) |
| [fred-economics](skills/fred-economics/SKILL.md) | 800K+ FRED economic time series | `FRED_API_KEY` |
| [polars-data](skills/polars-data/SKILL.md) | Lazy/streaming DataFrames, Parquet I/O | None |
| [pytorch-ml](skills/pytorch-ml/SKILL.md) | Model training, mixed precision, checkpointing | None |

## Installation

```bash
npm install @gonzih/skills-science
```

Then add to your Claude Code configuration or use directly via the skills directory.

## Usage

Each skill lives in `skills/<name>/SKILL.md`. Claude Code discovers skills automatically from installed packages.

## License

MIT
