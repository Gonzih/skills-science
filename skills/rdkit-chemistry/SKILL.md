---
name: rdkit-chemistry
description: Cheminformatics with RDKit — SMILES parsing, molecular fingerprints, Tanimoto similarity, Lipinski rules, ChEMBL lookup
license: MIT
triggers:
  - rdkit
  - smiles molecule
  - molecular fingerprint
  - tanimoto similarity
  - lipinski rule of five
  - chembl compound lookup
  - cheminformatics
  - drug likeness
  - molecular weight calculation
metadata:
  skill-author: gonzih
  data-sources:
    - www.ebi.ac.uk/chembl — 2.4M+ bioactive compounds, targets, assays; free; no auth required
  byok:
    - none
  compatibility: claude-code>=1.0
---

# RDKit Chemistry Skill

## What it does
Uses RDKit for cheminformatics: parse SMILES strings, compute molecular descriptors (MW, LogP, HBD, HBA, TPSA), generate Morgan/MACCS fingerprints, calculate Tanimoto similarity between molecules, assess Lipinski Rule-of-Five drug-likeness, and query the ChEMBL REST API for compound data.

## How to invoke
- "Calculate the molecular weight and LogP of aspirin"
- "Compute Tanimoto similarity between these two SMILES"
- "Check Lipinski rule-of-five for this compound"
- "Generate Morgan fingerprint for caffeine"
- "Look up ibuprofen on ChEMBL"
- "Find compounds similar to this SMILES in ChEMBL"

## Key parameters
- `smiles` — SMILES string (e.g. `CC(=O)Oc1ccccc1C(=O)O` for aspirin)
- `radius` — Morgan fingerprint radius (default 2)
- `n_bits` — fingerprint bit length (default 2048)
- `chembl_id` — ChEMBL compound ID (e.g. `CHEMBL25`)
- `query` — compound name for ChEMBL search

## Rate limits
- ChEMBL REST API: ~10 req/s, no key required
- RDKit: local computation, no rate limits

## Workflow steps
1. Install: `pip install rdkit` (or `conda install -c conda-forge rdkit`)
2. Parse SMILES with `Chem.MolFromSmiles()`
3. Compute descriptors with `Descriptors` module
4. Generate fingerprints with `AllChem.GetMorganFingerprintAsBitVect()`
5. Compute similarity with `DataStructs.TanimotoSimilarity()`
6. Query ChEMBL REST API for external data

## Working code example

```python
import json
import urllib.request
import urllib.parse
from typing import Optional

from rdkit import Chem, DataStructs
from rdkit.Chem import AllChem, Descriptors, rdMolDescriptors
from rdkit.Chem.FilterCatalog import FilterCatalog, FilterCatalogParams

CHEMBL_API = "https://www.ebi.ac.uk/chembl/api/data"


# ── Molecule parsing & validation ─────────────────────────────────────────────

def parse_smiles(smiles: str) -> Optional[object]:
    """Parse a SMILES string and return an RDKit Mol object, or None if invalid."""
    mol = Chem.MolFromSmiles(smiles)
    if mol is None:
        raise ValueError(f"Invalid SMILES: {smiles}")
    return mol


def smiles_to_canonical(smiles: str) -> str:
    """Canonicalize a SMILES string."""
    mol = parse_smiles(smiles)
    return Chem.MolToSmiles(mol)


# ── Molecular descriptors ─────────────────────────────────────────────────────

def compute_descriptors(smiles: str) -> dict:
    """
    Compute common 2D molecular descriptors.
    Returns MW, LogP, HBD, HBA, TPSA, RotBonds, rings, heavy_atom_count.
    """
    mol = parse_smiles(smiles)
    return {
        "smiles_canonical": Chem.MolToSmiles(mol),
        "molecular_weight": round(Descriptors.ExactMolWt(mol), 4),
        "mol_weight_avg": round(Descriptors.MolWt(mol), 4),
        "logP": round(Descriptors.MolLogP(mol), 4),
        "hbd": rdMolDescriptors.CalcNumHBD(mol),       # H-bond donors
        "hba": rdMolDescriptors.CalcNumHBA(mol),       # H-bond acceptors
        "tpsa": round(Descriptors.TPSA(mol), 4),       # Topological polar surface area
        "rotatable_bonds": rdMolDescriptors.CalcNumRotatableBonds(mol),
        "aromatic_rings": rdMolDescriptors.CalcNumAromaticRings(mol),
        "heavy_atom_count": mol.GetNumHeavyAtoms(),
        "ring_count": rdMolDescriptors.CalcNumRings(mol),
        "formal_charge": Chem.GetFormalCharge(mol),
        "stereo_centers": len(Chem.FindMolChiralCenters(mol, includeUnassigned=True)),
    }


# ── Lipinski Rule-of-Five ─────────────────────────────────────────────────────

def lipinski_ro5(smiles: str) -> dict:
    """
    Assess drug-likeness using Lipinski's Rule of Five.
    Oral bioavailability: MW<=500, LogP<=5, HBD<=5, HBA<=10.
    """
    desc = compute_descriptors(smiles)
    violations = []
    if desc["molecular_weight"] > 500:
        violations.append(f"MW={desc['molecular_weight']:.1f} > 500")
    if desc["logP"] > 5:
        violations.append(f"LogP={desc['logP']:.2f} > 5")
    if desc["hbd"] > 5:
        violations.append(f"HBD={desc['hbd']} > 5")
    if desc["hba"] > 10:
        violations.append(f"HBA={desc['hba']} > 10")

    return {
        "drug_like": len(violations) == 0,
        "violations": violations,
        "violation_count": len(violations),
        "mw": desc["molecular_weight"],
        "logP": desc["logP"],
        "hbd": desc["hbd"],
        "hba": desc["hba"],
        "tpsa": desc["tpsa"],
        "verdict": "Pass" if len(violations) == 0 else f"Fail ({len(violations)} violation(s))",
    }


# ── Fingerprints ──────────────────────────────────────────────────────────────

def morgan_fingerprint(smiles: str, radius: int = 2, n_bits: int = 2048):
    """Compute Morgan (circular) fingerprint as a bit vector."""
    mol = parse_smiles(smiles)
    fp = AllChem.GetMorganFingerprintAsBitVect(mol, radius=radius, nBits=n_bits)
    return fp


def maccs_fingerprint(smiles: str):
    """Compute MACCS keys fingerprint (166 bits, interpretable)."""
    mol = parse_smiles(smiles)
    return AllChem.GetMACCSKeysFingerprint(mol)


def rdkit_fingerprint(smiles: str, n_bits: int = 2048):
    """Compute RDKit topological fingerprint."""
    mol = parse_smiles(smiles)
    return Chem.RDKFingerprint(mol, fpSize=n_bits)


# ── Similarity ────────────────────────────────────────────────────────────────

def tanimoto_similarity(smiles1: str, smiles2: str, fp_type: str = "morgan") -> float:
    """
    Compute Tanimoto (Jaccard) similarity between two molecules.
    fp_type: 'morgan', 'maccs', 'rdkit'
    """
    fp_fn = {"morgan": morgan_fingerprint, "maccs": maccs_fingerprint, "rdkit": rdkit_fingerprint}
    fp_func = fp_fn.get(fp_type, morgan_fingerprint)
    fp1 = fp_func(smiles1)
    fp2 = fp_func(smiles2)
    return round(DataStructs.TanimotoSimilarity(fp1, fp2), 4)


def find_most_similar(query_smiles: str, library: list[str], top_n: int = 5) -> list[dict]:
    """Find the most similar molecules from a library to the query."""
    query_fp = morgan_fingerprint(query_smiles)
    results = []
    for smi in library:
        try:
            sim = DataStructs.TanimotoSimilarity(query_fp, morgan_fingerprint(smi))
            results.append({"smiles": smi, "tanimoto": round(sim, 4)})
        except Exception:
            pass
    return sorted(results, key=lambda x: x["tanimoto"], reverse=True)[:top_n]


# ── ChEMBL REST API ───────────────────────────────────────────────────────────

def chembl_get(endpoint: str, params: dict = None) -> dict:
    params = params or {}
    params["format"] = "json"
    url = f"{CHEMBL_API}/{endpoint}?" + urllib.parse.urlencode(params)
    with urllib.request.urlopen(url) as resp:
        return json.loads(resp.read())


def chembl_search_compound(name: str, limit: int = 5) -> list[dict]:
    """Search ChEMBL for compounds by name."""
    data = chembl_get("molecule", {
        "pref_name__icontains": name,
        "limit": limit,
    })
    results = []
    for mol in data.get("molecules", []):
        props = mol.get("molecule_properties") or {}
        results.append({
            "chembl_id": mol.get("molecule_chembl_id"),
            "name": mol.get("pref_name"),
            "smiles": (mol.get("molecule_structures") or {}).get("canonical_smiles"),
            "mw": props.get("mw_freebase"),
            "alogp": props.get("alogp"),
            "hbd": props.get("hbd"),
            "hba": props.get("hba"),
            "tpsa": props.get("psa"),
            "ro5_violations": props.get("num_ro5_violations"),
            "max_phase": mol.get("max_phase"),  # clinical trial phase
        })
    return results


def chembl_fetch_compound(chembl_id: str) -> dict:
    """Fetch a single compound by ChEMBL ID."""
    data = chembl_get(f"molecule/{chembl_id}")
    props = data.get("molecule_properties") or {}
    structs = data.get("molecule_structures") or {}
    return {
        "chembl_id": data.get("molecule_chembl_id"),
        "name": data.get("pref_name"),
        "smiles": structs.get("canonical_smiles"),
        "inchi": structs.get("standard_inchi"),
        "inchikey": structs.get("standard_inchi_key"),
        "mw": props.get("mw_freebase"),
        "alogp": props.get("alogp"),
        "hbd": props.get("hbd"),
        "hba": props.get("hba"),
        "tpsa": props.get("psa"),
        "ro5_violations": props.get("num_ro5_violations"),
        "max_phase": data.get("max_phase"),
        "molecule_type": data.get("molecule_type"),
    }


# --- Example usage ---
if __name__ == "__main__":
    # Aspirin
    aspirin = "CC(=O)Oc1ccccc1C(=O)O"
    ibuprofen = "CC(C)Cc1ccc(cc1)C(C)C(=O)O"
    caffeine = "Cn1cnc2c1c(=O)n(c(=O)n2C)C"

    print("=== Aspirin descriptors ===")
    desc = compute_descriptors(aspirin)
    for k, v in desc.items():
        print(f"  {k}: {v}")

    print("\n=== Lipinski Ro5 — Ibuprofen ===")
    ro5 = lipinski_ro5(ibuprofen)
    print(f"  Verdict: {ro5['verdict']}")
    print(f"  MW: {ro5['mw']:.1f}, LogP: {ro5['logP']:.2f}, HBD: {ro5['hbd']}, HBA: {ro5['hba']}")

    print("\n=== Tanimoto similarity ===")
    sim = tanimoto_similarity(aspirin, ibuprofen, fp_type="morgan")
    print(f"  Aspirin vs Ibuprofen (Morgan): {sim}")
    sim2 = tanimoto_similarity(aspirin, caffeine, fp_type="morgan")
    print(f"  Aspirin vs Caffeine (Morgan): {sim2}")

    print("\n=== ChEMBL lookup — ibuprofen ===")
    results = chembl_search_compound("ibuprofen", limit=3)
    for r in results:
        print(f"  {r['chembl_id']}: {r['name']} | Phase: {r['max_phase']} | Ro5 violations: {r['ro5_violations']}")
```

## Live Data Sources
- **ChEMBL API**: `https://www.ebi.ac.uk/chembl/api/data/`
- **ChEMBL docs**: https://chembl.gitbook.io/chembl-interface-documentation/web-services/chembl-data-web-services
- **RDKit documentation**: https://www.rdkit.org/docs/
- **RDKit GitHub**: https://github.com/rdkit/rdkit
- **PubChem REST API** (alternative): `https://pubchem.ncbi.nlm.nih.gov/rest/pug/`
- **Coverage**: ChEMBL 2.4M+ bioactive molecules, 14K+ targets, 20M+ activities
