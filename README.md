# ClinVar Wrangler: A Variant Lookup Challenge

## What You're Building

A tool that accepts a disease name in plain English, maps it to associated genes, and returns the relevant ClinVar and dbSNP variants for those genes, ranked from most to least likely to be causative.

**The pipeline:**

```
"Huntington's disease"  →  [ HTT ]  →  ClinVar variants + dbSNP entries
```

**Input**: A disease name (e.g., `"achondroplasia"`, `"Huntington disease"`)

**Output**: For each associated gene, a ranked list of variants — most likely to cause the disease first — including at minimum: gene, rsid (if available), ClinVar variant ID, clinical significance, condition, and review status.

### Ranking

Results should be ordered from most to least likely to be causative for the queried disease. The ranking is yours to design, but it must be based on signals present in the data and defensible. Some signals worth considering:

- **Clinical significance**: Pathogenic and Likely pathogenic outrank VUS, which outranks Benign
- **Review status**: An expert panel assertion carries more weight than a single unreviewed submission
- **Submitter consensus**: More independent labs agreeing is stronger than one asserting
- **Conflicts**: A variant with conflicting significance calls should rank lower than one with consensus
- **Condition match**: A variant annotated specifically for the queried disease should rank above one annotated for a related but distinct condition

We are not specifying a formula. We want to see your reasoning — in the code, in comments, or in `NOTES.md`.

---

## Domain Background

This challenge is set in human genetics. You don't need a biology degree, but you need enough context to understand what the data represents. Read this section carefully.

### Genes and Variants

A **gene** is a segment of DNA that encodes a functional product (usually a protein). Humans have roughly 20,000 protein-coding genes, each identified by a standardized symbol (e.g., `FGFR3`, `HTT`, `SERPINA1`).

A **variant** is a position in the genome where some people have a different DNA sequence than the reference. Most variants are harmless. A small fraction alter how a gene's protein functions, and a subset of those cause or contribute to disease.

Variants are described in several ways:
- **HGVS notation**: a standardized string describing the exact change, e.g., `NM_000142.5(FGFR3):c.1138G>A (p.Gly380Arg)` means: in transcript NM_000142.5 of gene FGFR3, nucleotide position 1138 changed from G to A, resulting in amino acid 380 changing from Glycine to Arginine.
- **rsID**: a stable identifier assigned by dbSNP, e.g., `rs28931614`. These are the most portable cross-database identifiers.
- **VCV/RCV accession**: ClinVar's own identifiers for variant records.

Not all variants are single nucleotide changes. Some are insertions, deletions, or **repeat expansions** — where a short DNA sequence is repeated more times than normal. Huntington disease, for example, is caused by a CAG trinucleotide repeat in the HTT gene expanding beyond a threshold copy number.

### ClinVar

ClinVar is NCBI's database of reported relationships between human genetic variants and diseases. It was built to aggregate the clinical interpretations that diagnostic labs generate every day as part of patient care.

When a clinical lab sequences a patient and finds a variant, they assess it against available evidence: published literature, population databases, functional studies, family history. They classify it on a five-tier scale:

| Classification | Meaning |
|---------------|---------|
| Pathogenic | Strong evidence this variant causes disease |
| Likely pathogenic | Evidence supports causation, not yet definitive |
| Uncertain significance (VUS) | Evidence is insufficient or conflicting |
| Likely benign | Evidence supports this variant is not causative |
| Benign | Strong evidence this variant does not cause disease |

The lab then **submits** this interpretation to ClinVar, along with the condition they were assessing it for, their evidence, and their methodology. Each submission is called an **SCV** (submitted classification and variant).

Multiple labs may independently submit interpretations for the same variant. They may agree or disagree. ClinVar aggregates submissions and computes an overall **germline classification** for each variant, but this aggregate may mask nuance in the underlying submissions.

### ClinVar Data Hierarchy

```
Variation (VCV accession) — the specific genomic variant
  └── ReferenceClinVarAssertion (RCV accession) — variant + condition pair
       └── ClinVarAssertion (SCV accession) — a single lab's submission
            ├── ClinicalSignificance  (Pathogenic, VUS, Benign, etc.)
            ├── ModeOfInheritance     (may be absent)
            ├── Submitter             (lab name, org ID)
            ├── SubmissionDate
            ├── ReviewStatus          (maps to 0–4 stars)
            └── TraitSet              (the condition being annotated)
```

One variant can have RCVs for multiple conditions — because different labs may have submitted it in different disease contexts. Clinical significance and mode of inheritance live at the SCV level. The aggregate displayed on the VCV-level record is ClinVar's summary, which may not capture the nuance in the submissions underneath it.

### Review Stars

ClinVar assigns a 0–4 star review status to each variant record, based on the quality and source of the submissions:

| Stars | Meaning |
|-------|---------|
| 0 | No assertion criteria provided |
| 1 | Criteria provided, single submitter |
| 2 | Criteria provided, multiple submitters, no conflicts |
| 3 | Reviewed by expert panel |
| 4 | Practice guideline |

A 4-star variant has been evaluated by a recognized expert panel or is the subject of an official clinical practice guideline. This is the strongest possible evidence level in ClinVar.

### dbSNP and rsIDs

dbSNP is NCBI's database of all observed genetic variants, regardless of clinical significance. It assigns a stable numeric identifier — the **rsID** — to each unique variant position and allele. rsIDs look like `rs28931614`.

ClinVar variants almost always have an associated rsID, since the variant was presumably observed in some population before being clinically interpreted. The rsID is the most reliable way to cross-reference a ClinVar entry with population frequency data, other databases, and external tools.

Some variant types — particularly large repeat expansions and structural variants — may not have rsIDs.

### Mode of Inheritance

Mode of inheritance (MOI) describes the pattern by which a variant is transmitted through families:

- **Autosomal dominant (AD)**: One mutated copy is sufficient to cause disease
- **Autosomal recessive (AR)**: Two mutated copies required (one from each parent)
- **X-linked**: Variant is on the X chromosome; affects males and females differently
- **Mitochondrial**: Transmitted only through mothers

MOI is relevant to ranking because a variant's pathogenicity claim may be condition-specific. The same gene can have both dominant and recessive variants causing distinct diseases — and the MOI field is often absent, inconsistent, or expressed in non-standard terms across submissions.

### Disease-to-Gene Mapping

NCBI's **MedGen** database aggregates disease concepts from OMIM, MeSH, and other nomenclature systems and links them to associated genes. It's the most practical starting point for mapping a free-text disease name to a list of gene symbols.

Disease naming is not standardized. "Alpha-1 antitrypsin deficiency," "AAT deficiency," and "AATD" all refer to the same condition. MedGen handles many synonyms, but edge cases exist. Some disease names are ambiguous — "muscular dystrophy" refers to dozens of distinct conditions across many genes.

---

## Sample Data

The `sampledata/clinvar/` directory contains real ClinVar records fetched from NCBI for the three test genes. There are 269 records total, spanning the full range of clinical significance classifications and review star ratings.

**Working from sample data during development is essential.** The NCBI API has rate limits. Repeated live fetches during development will get you throttled and slow your iteration dramatically. Build and test your parsing, normalization, and ranking logic against the sample data first. Hit the live API only when you need to test the full pipeline end-to-end.

The sample data is in the same JSON format returned by the NCBI esummary API, so your parsing code will work identically against both.

---

## Test Diseases

We tested a range of diseases against the NCBI databases and selected these three as the best combination of verifiability, data coverage, and interesting edge cases. These are the diseases we will run your tool against.

| Disease | Gene | What to Expect |
|---------|------|----------------|
| Achondroplasia | FGFR3 | Single gene; the canonical variant (p.Gly380Arg) should rank at or near the top; multiple submitters, no conflicts |
| Huntington disease | HTT | Single gene; the canonical variant is a CAG repeat expansion, not a point mutation; highest possible ClinVar review status |
| Alpha-1 antitrypsin deficiency | SERPINA1 | Single gene; the classic Z allele has a conflicting classification in ClinVar — your output should reflect that conflict, not hide it |

Each tests something different. The first is a sanity check. The second tests whether you handle non-SNV variant types. The third tests whether you surface data quality problems honestly rather than papering over them.

---

## What We're Evaluating

**Does it work?** Run the test diseases. Does it return sensible results without falling over?

**Does the ranking make sense?** We will look at the top few results for each test disease and ask whether the ordering is defensible.

**Epistemic honesty.** When the gene mapping is ambiguous or a variant has conflicting significance calls, does the output say so? A tool that confidently returns one answer for an ambiguous input is worse than one that surfaces the ambiguity.

**Code structure.** The three pipeline stages — disease→gene, gene→variants, format output — should be cleanly separated. We'll want to discuss your design.

**The tradeoffs you made.** We'll ask about them.

---

## Submitting Your Work

1. Fork this repo to your own GitHub account
2. Push your implementation
3. Add a `NOTES.md` (one page max) describing:
   - The biggest data quality surprise you encountered
   - Any tradeoffs in your approach you want us to know about

We will run your tool against the test diseases and then have a conversation about your approach.

---

## Appendix: Useful References

- **ClinVar**: https://www.ncbi.nlm.nih.gov/clinvar/
- **ClinVar data format docs**: https://www.ncbi.nlm.nih.gov/clinvar/docs/help/
- **ClinVar review status**: https://www.ncbi.nlm.nih.gov/clinvar/docs/review_status/
- **NCBI Gene**: https://www.ncbi.nlm.nih.gov/gene/
- **MedGen**: https://www.ncbi.nlm.nih.gov/medgen/
- **dbSNP**: https://www.ncbi.nlm.nih.gov/snp/
- **NCBI E-utilities docs**: https://www.ncbi.nlm.nih.gov/books/NBK25499/
- **HPO browser**: https://hpo.jax.org/
