# ClinVar Wrangler: A Variant Lookup Challenge

## What You're Building

A CLI or web tool that accepts a disease name in plain English, maps it to associated genes, and returns the relevant ClinVar and dbSNP variants for those genes.

**The pipeline:**

```
"Huntington's disease"  →  [ HTT ]  →  ClinVar variants + dbSNP entries
```

**Input**: A disease name as a string (e.g., `"cystic fibrosis"`, `"Huntington disease"`, `"achondroplasia"`)

**Output**: For each associated gene, a list of variants with:

| Field | Description |
|-------|-------------|
| `gene` | Gene symbol |
| `rsid` | dbSNP rs number (if available) |
| `clinvar_id` | ClinVar variant ID (VCV accession) |
| `clinical_significance` | Pathogenic, VUS, Benign, etc. |
| `condition` | The condition this submission annotates |
| `mode_of_inheritance` | MOI, if provided |
| `review_stars` | ClinVar review status (0–4) |
| `submitter_count` | Number of submitting organizations |

---

## The Data Sources

**NCBI Gene** is where disease-to-gene mapping lives. You can query it with free-text disease names and get back associated gene records.

**ClinVar** holds clinical assertions about variants — which labs think a variant is pathogenic, what disease it causes, how it's inherited. Each submission is from a specific lab and may disagree with others.

**dbSNP** assigns stable rs numbers to variants. ClinVar variants almost always have an associated rsID. They are the same underlying variant — dbSNP provides the identifier and population-level data; ClinVar provides the clinical interpretation.

All three are accessible via NCBI's E-utilities API, wrapped by the `metapub` Python library.

---

## The Problems You'll Encounter

### Disease name → gene is not clean

"Huntington's disease", "Huntington disease", and "HD" all refer to the same condition but may return different results depending on which NCBI index you're searching. MeSH terms, OMIM entries, and free-text gene descriptions don't share a unified namespace. A search that works for one disease may fail silently for another.

Some diseases map to a single gene. Some map to dozens. Some names are ambiguous — "muscular dystrophy" encompasses many distinct conditions with different gene sets. Your tool needs to handle all of these without crashing, and should surface ambiguity to the user rather than silently picking one interpretation.

### ClinVar has multiple submissions per variant, often conflicting

For any given variant, you may have submissions from 1 lab or 50. They may agree on pathogenicity, or not. The `mode_of_inheritance` field — when present — may use HPO terms, free text, abbreviations, or nothing at all. The same variant can appear under multiple ClinVar accessions due to historical merges.

See Appendix A for the full ClinVar data hierarchy and Appendix B for the range of MOI strings you'll actually see in the wild.

### Not all genes have ClinVar entries; not all ClinVar entries have rsIDs

Novel variants, structural variants, and copy-number variants often lack rsIDs. Some ClinVar entries predate systematic dbSNP integration. Your tool should handle missing rsIDs gracefully.

### NCBI rate limits are real

Without an API key, NCBI allows 3 requests/second. With a free key, 10/second. Even with a key, you'll get throttled during high-traffic periods. Your tool needs to handle this without crashing.

---

## Using metapub

```bash
pip install metapub
```

metapub wraps NCBI's E-utilities and provides Python objects for the results. The `ClinVarFetcher` class is your primary interface for variant data. For disease-to-gene mapping you'll likely need to work with NCBI Gene or OMIM search via eutils directly.

```bash
export NCBI_API_KEY="your_key_here"   # get one free at https://www.ncbi.nlm.nih.gov/account/
```

The metapub ClinVar API is less documented than the PubMed API. Verify method names and return types against the installed version — `help(ClinVarFetcher)` and reading the source are your friends here. The return types for fields like `mode_of_inheritance` are inconsistent across records (string, list, None, empty string) and have changed across metapub versions.

---

## Test Diseases

These are the diseases we'll use to verify your tool works. They were chosen because they have short, well-known gene lists and enough ClinVar data to be interesting but not overwhelming.

| Disease | Expected Gene(s) | Why It's Useful |
|---------|-----------------|-----------------|
| Achondroplasia | FGFR3 | Single gene, 2–3 classic pathogenic variants, easy to visually verify |
| Huntington disease | HTT | Single gene, triplet repeat, canonical example |
| Phenylketonuria | PAH | Single gene, many variants — tests pagination/volume handling |
| Marfan syndrome | FBN1 (primarily) | Well-characterized, moderate variant count |
| Cystic fibrosis | CFTR | Many variants; F508del should dominate pathogenic results |

We will also try at least one ambiguous or multi-gene disease during the interview.

---

## Architecture Requirements

- **Language**: Python 3.10+
- **Required library**: `metapub`
- **Interface**: CLI (`argparse` or `click`) or web (Flask, FastAPI, Streamlit — your choice)
- **Output**: Human-readable by default; JSON available via flag or endpoint
- **Logging**: Use Python's `logging` module; no bare print statements in library code
- **Error handling**: A failed lookup for one gene should not crash the entire run

---

## Usage

A CLI tool might look like:

```bash
python -m clinvar_wrangler "achondroplasia"
python -m clinvar_wrangler "cystic fibrosis" --format json
python -m clinvar_wrangler "Huntington disease" --sig pathogenic
```

A web tool should accept the disease name as input and render the results readably. The exact interface is up to you.

---

## What We're Evaluating

**Does it work?** Run the test diseases. Does it return sensible results? Does it handle a disease with many variants without falling over?

**Epistemic honesty.** When the gene mapping is ambiguous or a variant has conflicting significance calls, does the output say so? A tool that confidently returns one answer for an ambiguous input is worse than one that surfaces the ambiguity.

**Code structure.** The three pipeline stages — disease→gene, gene→variants, format output — should be cleanly separated. We'll want to discuss your design.

**The tradeoffs you made.** We'll ask about them. Inline comments explaining a choice are a good sign.

---

## Submitting Your Work

1. Fork this repo to your own GitHub account
2. Push your implementation
3. Add a `NOTES.md` (one page max) describing:
   - The biggest data quality surprise you encountered
   - Any tradeoffs in your approach you want us to know about

We will run your tool against the test diseases and then have a conversation about your approach.

---

## Appendix A: ClinVar Data Model (Simplified)

```
Variation (VCV accession) — the specific genomic variant
  └── ReferenceClinVarAssertion (RCV accession) — variant + condition pair
       └── ClinVarAssertion (SCV accession) — a single lab's submission
            ├── ClinicalSignificance  (Pathogenic, VUS, Benign, etc.)
            ├── ModeOfInheritance     (may be absent, may be a list)
            ├── Submitter             (lab name, org ID)
            ├── SubmissionDate
            ├── ReviewStatus          (maps to 0–4 stars)
            └── TraitSet              (the condition being annotated)
```

When you fetch by gene or RSID, you get VCVs. Each VCV contains RCVs (one per condition). Each RCV contains SCVs (one per submitting lab). Clinical significance and MOI live at the SCV level — aggregate fields at the VCV level are ClinVar's attempt to summarize, which may or may not reflect the nuance below.

---

## Appendix B: MOI Term Normalization Reference

If you surface MOI data, be aware of the range of strings you'll actually see in the field. All of the following appear in real ClinVar submissions and nominally mean "Autosomal dominant":

```
"Autosomal dominant"
"Autosomal dominant inheritance"
"autosomal dominant"
"AD"
"HP:0000006"
"Autosomal dominant (HP:0000006)"
"Dominant"
"Heterozygous"
"de novo"
"Autosomal dominant, de novo"
```

And these mean "Autosomal recessive":
```
"Autosomal recessive"
"AR"
"HP:0000007"
"Recessive"
"Homozygous"
"Compound heterozygous"
```

And these are not MOI at all:
```
"Not provided"
"Not applicable"
"Unknown"
""
"not specified"
"See conditions"
```

---

## Appendix C: Useful References

- **ClinVar**: https://www.ncbi.nlm.nih.gov/clinvar/
- **ClinVar data format docs**: https://www.ncbi.nlm.nih.gov/clinvar/docs/help/
- **NCBI Gene**: https://www.ncbi.nlm.nih.gov/gene/
- **dbSNP**: https://www.ncbi.nlm.nih.gov/snp/
- **metapub GitHub**: https://github.com/metapub/metapub
- **metapub docs**: https://metapub.readthedocs.io/
- **NCBI E-utilities docs**: https://www.ncbi.nlm.nih.gov/books/NBK25499/
- **HPO browser**: https://hpo.jax.org/
- **ClinVar review status**: https://www.ncbi.nlm.nih.gov/clinvar/docs/review_status/
