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

## The Data Sources

**NCBI** provides disease-to-gene mapping, variant data, and stable identifiers across several linked databases — Gene, ClinVar, dbSNP, MedGen, and others — all accessible via the same E-utilities API.

**ClinVar** holds clinical assertions about variants: which labs think a variant is pathogenic, what disease it causes, how it's inherited. Each submission is from a specific lab and may disagree with others.

**dbSNP** assigns stable rs numbers to variants. ClinVar variants usually have an associated rsID — dbSNP provides the identifier; ClinVar provides the clinical interpretation.

---

## The Problems You'll Encounter

### Disease name → gene is not clean

"Huntington's disease", "Huntington disease", and "HD" all refer to the same condition but may return different results depending on which NCBI index you query. MeSH terms, OMIM entries, and free-text gene descriptions don't share a unified namespace. A query that works for one disease may fail silently for another.

Some diseases map to a single gene. Some map to dozens. Some names are ambiguous — "muscular dystrophy" encompasses many distinct conditions with different gene sets. Your tool should surface ambiguity to the user rather than silently picking one interpretation.

### ClinVar has multiple submissions per variant, often conflicting

For any given variant, you may have submissions from 1 lab or 50. They may agree on pathogenicity, or not. The same variant can appear under multiple ClinVar accessions due to historical merges. The aggregate classification at the variant level is ClinVar's attempt to summarize all submissions — it may not capture the nuance underneath.

See Appendix A for the ClinVar data hierarchy and Appendix B for the range of MOI strings you'll actually see in the wild.

### Not all ClinVar entries have rsIDs

Repeat expansions, structural variants, and copy-number variants often lack rsIDs. Some entries predate systematic dbSNP integration. Handle missing rsIDs gracefully.

### NCBI rate-limits the API

Free access allows 3 requests/second; a free API key raises that to 10. Even with a key, you'll get throttled. Handle it without crashing.

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

If you surface MOI data, be aware of the range of strings you'll actually see. All of the following appear in real ClinVar submissions and nominally mean "Autosomal dominant":

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
- **NCBI E-utilities docs**: https://www.ncbi.nlm.nih.gov/books/NBK25499/
- **HPO browser**: https://hpo.jax.org/
- **ClinVar review status**: https://www.ncbi.nlm.nih.gov/clinvar/docs/review_status/
