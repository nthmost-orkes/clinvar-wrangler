# ClinVar Wrangler: A Variant Inheritance Resolver Challenge

> **For the candidate**: Yes, this README is long. No, that's not a mistake. Yes, you should absolutely paste it into your favorite LLM. That's the point. What matters is what you do *after* the LLM gives you its first response — specifically, the part where you discover that the LLM was confidently wrong about something.

---

## Background: Why ClinVar Is Both Invaluable and Infuriating

ClinVar is NCBI's archive of reports about relationships between human genetic variants and phenotypes. If you're building anything in genomics, clinical genetics, or variant interpretation pipelines, you will eventually have to deal with ClinVar. And when you do, you will encounter a set of data quality issues that are simultaneously completely understandable (the data is crowdsourced from thousands of labs with different standards, workflows, and definitions of "good enough") and genuinely maddening to work with programmatically.

This challenge is designed around one specific category of those issues: **mode of inheritance (MOI) conflicts**.

### What Is Mode of Inheritance?

Mode of inheritance describes how a genetic variant is transmitted through families and what it takes (one copy? two?) to produce a clinical effect. The major categories are:

- **Autosomal dominant (AD)**: One mutated copy causes disease. The variant "wins" over the normal copy.
- **Autosomal recessive (AR)**: Two mutated copies are needed for disease. Heterozygous carriers are typically unaffected.
- **X-linked dominant (XLD)**: Mutation on the X chromosome, acts dominantly. Affects both sexes, usually more severely in males.
- **X-linked recessive (XLR)**: Mutation on X chromosome, acts recessively. Males (hemizygous) are typically affected; female carriers usually are not.
- **Mitochondrial**: Maternally inherited. Affects both sexes; transmitted only through mothers.
- **Semidominant / Incomplete penetrance**: Heterozygotes show some effect but less than homozygotes. Used for genes where dosage really matters.
- **Digenic**: Two mutations in two different genes are required for disease. Rare. Annoying to annotate.

The Human Phenotype Ontology (HPO) provides standardized terms for these:

| HPO ID | Label |
|--------|-------|
| HP:0000006 | Autosomal dominant inheritance |
| HP:0000007 | Autosomal recessive inheritance |
| HP:0001417 | X-linked inheritance (ambiguous — avoid if possible) |
| HP:0001419 | X-linked recessive inheritance |
| HP:0001423 | X-linked dominant inheritance |
| HP:0001427 | Mitochondrial inheritance |
| HP:0001426 | Multifactorial inheritance |
| HP:0001442 | Somatic mutation (not really MOI but appears in the field) |
| HP:0001450 | Y-linked inheritance |
| HP:0003743 | Genetic anticipation |

ClinVar submissions can include MOI annotations. They can also omit them entirely. Or include them in a format nobody else uses. Or use the same HPO ID to mean subtly different things in context. This is the problem.

---

## The Specific Mess You'll Be Working With

When you query ClinVar for a specific RSID, you may receive any of the following situations:

**Case 1**: A single submission with a single MOI.  
*Frequency*: Rare for clinically important variants. Common for novel/rare variants.

**Case 2**: Multiple submissions from different labs, all agreeing on the same MOI.  
*Frequency*: Moderately common for well-studied variants. Lucky you.

**Case 3**: Multiple submissions with conflicting MOIs.  
*Frequency*: Common enough that you need a strategy.

**Case 4**: Multiple submissions where some have MOI and some don't.  
*Frequency*: Very common. The "Not provided" problem.

**Case 5**: The same variant appearing under multiple ClinVar variant IDs (VCV/RCV accessions) due to variant merging history, each set with their own submissions.  
*Frequency*: Common enough to bite you if you don't check.

**Case 6**: Submissions where the `mode_of_inheritance` field contains plain English rather than HPO terms, or worse, HPO terms *that are wrong*.  
*Frequency*: Regular. Labs copy-paste from old submissions.

**Case 7**: The same lab submitting the same variant multiple times over years, with different MOI each time.  
*Frequency*: More common than you'd hope.

**Case 8**: MOI that is condition-specific — the same variant causes AD disease in heterozygotes and a more severe AR disease in homozygotes — and different submissions are annotating different conditions.  
*Frequency*: Rare but important, and your conflict detector will light up on these incorrectly if you're not careful.

The delightful thing is that ClinVar knows about cases 1-8 and still doesn't give you a clean way to resolve them. That's your job now.

---

## The Tool You're Building

You will build a Python command-line tool called `clinvar_wrangler` that resolves these conflicts and produces a structured, honest report.

### Core Functionality

**Input**: One or more gene symbols OR one or more RSIDs (or both, mixed)

**Output**: A structured report (JSON required, CSV optional) containing, for each variant processed:

| Field | Type | Description |
|-------|------|-------------|
| `rsid` | string | The rs number, e.g. `"rs113993960"` |
| `variant_id` | string | ClinVar variant ID (VCV accession preferred over RCV) |
| `gene` | string | Gene symbol |
| `all_moi_claims` | list[str] | All unique raw MOI strings seen across all submissions |
| `normalized_moi_claims` | list[str] | MOI terms normalized to HPO IDs |
| `resolved_moi` | string or null | Your best determination, as HPO ID |
| `resolved_moi_label` | string or null | Human-readable label for resolved_moi |
| `resolution_method` | string | How you resolved: `"unanimous"`, `"majority"`, `"highest_star"`, `"single_submission"`, `"unresolvable"`, `"no_moi_data"` |
| `confidence` | float | 0.0 to 1.0 |
| `num_submissions` | int | Total number of ClinVar SCV submissions considered |
| `num_submitters` | int | Number of unique submitting organizations |
| `conflicts_detected` | bool | Were there meaningful MOI conflicts after normalization? |
| `conflict_detail` | string or null | Human-readable description of the conflict, if any |
| `max_review_stars` | int | Highest ClinVar review star rating seen among submissions (0-4) |
| `notes` | list[str] | Any caveats, warnings, or flags worth surfacing |

### The Resolution Algorithm

Here's the hard part: you must implement a resolution algorithm. We are deliberately *not* specifying exactly what it should be. That's part of the evaluation. But it must handle all of these cases:

**Unanimous after normalization**: All submissions that provide MOI data agree, after normalization → confidence ~0.95, method `"unanimous"`

**Supermajority (≥80% after normalization)**: → confidence ~0.75, method `"majority"`, `conflicts_detected=True`

**Simple majority (>50%)**: → confidence ~0.50, method `"majority"`, `conflicts_detected=True`, meaningful conflict detail

**Tied or worse**: No clear winner → `resolved_moi=null`, method `"unresolvable"`, confidence low

**Single high-quality submission (4-star review status)**: → method `"highest_star"`, confidence ~0.80 even with one submission (the review process is rigorous)

**All submissions say "Not provided"**: → method `"no_moi_data"`, confidence 0.0, `resolved_moi=null`

**Conflicting-but-reconcilable (normalization resolves it)**: AD vs "Autosomal dominant inheritance (HP:0000006)" → same thing after normalization, `conflicts_detected=False`

**Genuinely irreconcilable (AR vs AD from reputable labs)**: → `conflicts_detected=True`, detailed conflict note, low confidence

### What "Not Provided" Means

Submissions where MOI is absent or explicitly "Not provided" / "Not applicable" should:
- Be counted in `num_submissions` (they exist)
- NOT count toward or against any MOI claim
- Be noted if they constitute the majority of submissions (that's useful information)

### Caching

NCBI rate limits will make your development experience painful if you re-fetch the same data repeatedly. Implement a simple disk cache. It doesn't need to be fancy — even just serializing fetched results to JSON files keyed by RSID is fine. Make it optional via a CLI flag (`--no-cache` to bypass). The cache can be stale — that's acceptable for this challenge.

---

## Using metapub

This challenge requires the `metapub` Python library to access ClinVar data. metapub wraps NCBI's E-utilities APIs and gives you Python objects for the results.

```bash
pip install metapub
```

metapub requires an NCBI API key for reasonable rate limits. Without a key, NCBI allows 3 requests/second. With a free API key (register at https://www.ncbi.nlm.nih.gov/account/), you get 10 requests/second. You want the key.

```bash
export NCBI_API_KEY="your_key_here"
```

### ClinVar Access via metapub

The `ClinVarFetcher` class is your primary interface. The basics look something like:

```python
from metapub import ClinVarFetcher

fetcher = ClinVarFetcher()

# By RSID
result = fetcher.get_clinvar_data_by_rsid("rs80357906")

# By gene — this returns a dict of results
results = fetcher.get_clinvar_variants_by_gene("BRCA1")
```

But here's the thing: **you need to verify this API for yourself with the actual installed version of metapub**. The above is plausible but may not be exactly right. Check `help(ClinVarFetcher)` and look at the actual class methods. The metapub ClinVar API is not as well-documented as the PubMed API, and the method names and return types have changed between versions.

What you get back from ClinVar fetching will be Python objects wrapping the underlying XML. Understanding the structure of those objects is part of the challenge.

### Important metapub / NCBI E-utilities Notes

**These are not theoretical. They will bite you in real time.**

**1. Rate limiting is real.**

Even with an API key, NCBI throttles aggressively during business hours and during high-traffic periods. Your code must handle HTTP 429 responses gracefully with exponential backoff — not just a `time.sleep(1)` but actual backoff with jitter. Something like:

```python
import time
import random

def fetch_with_retry(fetch_fn, *args, max_retries=5, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return fetch_fn(*args)
        except Exception as e:
            if "429" in str(e) or "Too Many Requests" in str(e):
                delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
                time.sleep(delay)
            else:
                raise
    raise RuntimeError(f"Max retries exceeded for {args}")
```

**2. E-utilities search syntax is finicky.**

`"BRCA1"[Gene Name]` is different from `"BRCA1"[Gene Symbol]`. One returns variants with BRCA1 as the gene name field. The other uses a different ClinVar index. Experiment with both and compare result counts. The ClinVar search field tags are documented at https://www.ncbi.nlm.nih.gov/clinvar/docs/help/ — read that page, it will save you hours.

**3. The ClinVar XML format is not what you think.**

ClinVar's efetch endpoint returns XML. The schema has evolved significantly and there are currently two formats in play: the "full" format (recommended) and the "vcv" format. The metapub library may parse one or both. Understanding what's actually in the objects you get back requires either reading the metapub source or printing and inspecting actual results.

**4. Variant IDs are unstable.**

ClinVar periodically merges variant records. An RSID you fetched last year may now map to a different VCV accession. The old VCV may still exist but redirect. Your code should handle this gracefully rather than crashing or silently returning stale data.

**5. Not all RSIDs are in ClinVar.**

Some RSIDs are dbSNP entries with no clinical significance submissions whatsoever. Your code should handle this case cleanly (return a result with `resolution_method="no_moi_data"` and a note explaining why) rather than throwing an unhandled exception.

**6. The `mode_of_inheritance` field is inconsistent at the object level.**

Depending on the metapub version and the specific variant record, what you access as `mode_of_inheritance` on a result object may be: a string, a list of strings, `None`, an empty string, or in some records, a nested structure. Write defensive code and test what you actually receive before writing code that assumes a specific type.

**7. Batch operations exist and you should use them.**

If you're fetching 50+ variants, use NCBI's epost/efetch batching rather than hitting the API 50 times in sequence. metapub may support this directly or you may need to drop down to the underlying requests layer. Batching makes you a good citizen of the NCBI API and makes your tool dramatically faster.

---

## Test Cases

These are real RSIDs with known characteristics in ClinVar. We have verified them as of early 2025, but ClinVar data changes — if a "conflict" case now appears unanimous, it may have been resolved by new expert submissions.

### Group A: Clean Cases

Start here. These should work without any special conflict-handling logic.

| RSID | Gene | Expected MOI | Notes |
|------|------|-------------|-------|
| `rs80357906` | BRCA1 | Autosomal dominant | Classic BRCA1 pathogenic variant; many high-quality submissions |
| `rs28897696` | CFTR | Autosomal recessive | Well-characterized CF-causing variant |
| `rs121912578` | MECP2 | X-linked dominant | Rett syndrome; should be clear-cut |

Your resolver should return `confidence > 0.8` and `conflicts_detected=False` for all three.

### Group B: Conflict Cases

This is the point of the exercise. These should surface conflicts in your resolver.

| RSID | Gene | What's Messy | What to Expect |
|------|------|-------------|-------|
| `rs387906304` | COL3A1 | Mix of AD submissions and "Not specified" / "Not provided" | Conflicts_detected should be False after filtering non-MOI entries; but if your filter is wrong, you'll cry wolf |
| `rs121918166` | MYBPC3 | Some submissions say AD, some AR | MYBPC3 can genuinely be both depending on zygosity and condition; your resolver needs to decide whether to flag or explain |
| `rs193922376` | KCNQ4 | Same MOI, different HPO term variants | Your normalizer should collapse these; if it doesn't, you'll see fake conflicts |
| `rs267607140` | NF1 | Very high submission count, wide range of submitter quality | Neurofibromatosis; lots of submissions, lots of variability in MOI formatting |

### Group C: Edge Cases

Will your code crash? Will it handle these gracefully and informatively?

| RSID | Expected Behavior |
|------|------------------|
| `rs80359550` | Should resolve correctly despite multiple VCV IDs |
| `rs1064793474` | Novel/rare variant; may have 0 or 1 submissions; should not crash |
| `rs762551` | High-frequency population variant; ClinVar data may be sparse |
| `FAKE000000` | Invalid RSID; should return a clear error result, not an exception |

---

## Architecture Requirements

- **Language**: Python 3.10+
- **Required library**: `metapub` (any recent version, but note the API notes above)
- **CLI interface**: `argparse` or `click` (your choice)
- **Output**: JSON (required), CSV (bonus)
- **Logging**: Use Python's `logging` module. INFO-level logs should tell you what's happening. DEBUG-level should tell you everything. No print statements in library code.
- **Error handling**: Failed fetches for individual RSIDs should not crash the entire run. Log the failure and continue. Include the failure in the output with an appropriate status.

### Expected Directory Structure

We expect something close to:

```
clinvar-wrangler/
├── README.md
├── requirements.txt
├── clinvar_wrangler/
│   ├── __init__.py
│   ├── fetcher.py        # metapub integration, caching, retry logic
│   ├── normalizer.py     # MOI term normalization and HPO mapping
│   ├── resolver.py       # conflict resolution logic
│   └── cli.py            # CLI entry point
├── tests/
│   ├── test_normalizer.py
│   └── test_resolver.py
├── sample_data/
│   └── test_rsids.txt    # one RSID per line
└── sample_output/
    └── (keep any output you generate during development)
```

### Usage Examples

Your tool should work like this:

```bash
# Single gene
python -m clinvar_wrangler --gene BRCA1 --output brca1_results.json

# Multiple genes
python -m clinvar_wrangler --genes BRCA1,BRCA2,TP53 --output multi_gene.json

# By RSID list
python -m clinvar_wrangler --rsids rs80357906,rs28897696,rs121912578 --output results.json

# From file
python -m clinvar_wrangler --rsid-file sample_data/test_rsids.txt --output results.json

# With CSV output
python -m clinvar_wrangler --gene CFTR --output cftr.json --csv cftr.csv

# Skip cache
python -m clinvar_wrangler --gene CFTR --output cftr.json --no-cache

# Verbose
python -m clinvar_wrangler --gene NF1 --output nf1.json --log-level DEBUG
```

---

## What We're Evaluating

### 1. Does It Work? (table stakes)

Run your tool against the Group A test cases. Do you get sensible output? Do the Group B cases surface conflicts appropriately? Do the Group C edge cases fail gracefully?

### 2. Epistemic Honesty

This is the most important dimension. When your resolver genuinely can't determine a confident MOI, does it say so clearly — with a useful explanation of *why*? We are deeply suspicious of tools that always return a confident answer. The correct output for an irreconcilable conflict is:

```json
{
  "resolved_moi": null,
  "resolution_method": "unresolvable",
  "confidence": 0.1,
  "conflicts_detected": true,
  "conflict_detail": "3 submissions claim AD (HP:0000006); 2 submissions claim AR (HP:0000007); no consensus possible; condition-stratification might resolve but is out of scope"
}
```

Not:
```json
{
  "resolved_moi": "HP:0000006",
  "confidence": 0.9
}
```

### 3. Normalization Quality

How well does your normalizer handle the mess in Appendix B? Does "AD" and "Autosomal dominant inheritance" and "HP:0000006" all collapse to the same canonical form? Does "de novo" get handled appropriately rather than silently dropped or misclassified?

### 4. Code Structure

- Is fetching logic clearly separated from resolution logic from normalization logic?
- Are edge cases handled explicitly with comments explaining why?
- Is the code readable to someone who didn't write it?
- Does the resolution algorithm have a clear, auditable decision tree?

### 5. Effective Tool Use

We will ask you to walk through your development process. We expect you used LLM tools. That's fine — it's part of what we're evaluating. The questions we'll ask:

- Where did the LLM give you incorrect information about the metapub API?
- How did you discover and fix it?
- What did you have to figure out from actual documentation or trial-and-error rather than from LLM output?
- What was the most surprising thing you found when you actually ran the code against ClinVar?

There are no wrong answers here, but "I just ran what the LLM gave me" is a red flag.

### 6. Testing

We don't expect 100% coverage. We do expect:
- Tests for the normalizer (pure function, easy to test, important to get right)
- Tests for the resolver logic (use mocked/fixed input data, don't hit the network in tests)
- At least one integration test (or documented manual test run) against real ClinVar data

### 7. The Tradeoffs You Made

We want to know what choices you made and why. Inline comments in the code are fine:

```python
# Using majority vote rather than star-weighted vote here because ClinVar star
# ratings aren't normalized across submitter types — a 4-star expert panel
# submission and a 4-star clinical lab submission have different evidentiary weights
# and we don't have a principled way to quantify that difference.
```

---

## Appendix A: ClinVar Data Model (Simplified)

ClinVar's hierarchy is:

```
Variation (VCV accession) — the specific genomic variant
  └── ReferenceClinVarAssertion (RCV accession) — variant + condition pair
       └── ClinVarAssertion (SCV accession) — a single lab's submission
            ├── ClinicalSignificance  (Pathogenic, VUS, Benign, etc.)
            ├── ModeOfInheritance     (may be absent, may be a list)
            ├── Submitter             (lab name, org ID)
            ├── SubmissionDate
            ├── ReviewStatus          (maps to 0-4 stars)
            └── TraitSet              (the condition being annotated)
```

When you fetch by RSID, you're looking up the VCV. That VCV contains all the RCVs (one per condition). Each RCV contains one or more SCVs (one per submission). The SCV level is where MOI lives. If you're only looking at VCV-level aggregate fields, you may be missing per-condition specificity.

**The per-condition subtlety**: A variant can cause different diseases in different zygosity states. MYBPC3 can cause:
- Hypertrophic cardiomyopathy (AD when heterozygous — the common presentation)
- A severe early-onset cardiomyopathy (AR when homozygous — very rare, few submissions)
- Various conflicting claims about dilated cardiomyopathy

If you naively aggregate MOI across all conditions for this gene, you see AD + AR, flag it as a conflict, and you're technically correct but practically misleading. A smarter approach stratifies by condition first. We are not requiring condition-stratification for this challenge. We *are* requiring that your conflict notes acknowledge this possibility when AR and AD both appear, rather than just saying "CONFLICT: AR vs AD."

---

## Appendix B: MOI Term Normalization Reference

Labs use wildly inconsistent terminology. Here is a non-exhaustive list of strings you may see in the `mode_of_inheritance` field, all meaning (or approximately meaning) Autosomal Dominant:

```
"Autosomal dominant"
"Autosomal dominant inheritance"
"autosomal dominant"
"Autosomal Dominant"
"AD"
"HP:0000006"
"Autosomal dominant (HP:0000006)"
"Dominant"
"Heterozygous"               ← technically incorrect usage of this field
"de novo"                    ← describes mechanism, not pattern; appears here anyway
"Autosomal dominant, de novo"
"Autosomal dominant inheritance with variable expressivity"
```

And these mean (approximately) Autosomal Recessive:
```
"Autosomal recessive"
"Autosomal recessive inheritance"
"AR"
"HP:0000007"
"Autosomal recessive (HP:0000007)"
"Recessive"
"Homozygous"                 ← again, technically wrong field usage
"Compound heterozygous"      ← describes genotype, implies AR, appears in MOI field
```

And these are not MOI at all, and should be excluded from your analysis (but counted as "no MOI data"):
```
"Not provided"
"Not applicable"
"Unknown"
""                           ← empty string
"not specified"
"N/A"
"See conditions"             ← found in wild ClinVar data, utterly unhelpful
```

And these are edge cases that require judgment:
```
"Autosomal dominant with reduced penetrance"  → HP:0000006 + penetrance caveat
"Semidominant"                                → HP:0001426 (or HP:0000006 + note)
"X-linked"                                    → HP:0001417 (ambiguous, flag it)
"Digenic dominant"                            → no clean HPO mapping
"Oligogenic"                                  → no clean HPO mapping
"Somatic mutation"                            → HP:0001442 (not really MOI)
"Sporadic"                                    → not MOI; often confused with de novo
```

Your normalizer must handle all of the above. We will test it with variants from the NF1 and MYBPC3 gene panels, which have particularly creative MOI annotation histories.

---

## Appendix C: Why LLMs Will Mislead You About This

We wrote this challenge partly to illustrate something important about the limits of LLM-assisted development in data-heavy domains. This is not a knock on LLMs — it's a description of when you need to be the one driving.

**The metapub ClinVar API has changed over library versions.** The LLM's training data spans old and new metapub releases. Code it generates may use method names that existed in 2021 but were renamed in 2023. It may use return-value attribute names that no longer exist. Check the installed version.

**ClinVar's own data model has evolved.** The XML schema changed significantly around 2022. Example code and documentation written before that change uses field names and XPath expressions that may return nothing on current data. If your LLM is drawing on StackOverflow answers from 2020, it will give you code that parses ghosts.

**LLMs hallucinate HPO term IDs.** If you ask an LLM for the HPO identifier for "autosomal dominant inheritance," it may give you a plausible-sounding ID that is not the actual HPO term. Always verify HPO term IDs against https://hpo.jax.org/ directly.

**LLMs don't know about NCBI's ongoing data curation.** The model cannot tell you that rs387906304 currently maps to three VCV IDs due to a merge that happened in early 2025, or that the submission history for rs267607140 was reorganized when a major submitter updated their pipeline. This is live data; the LLM's knowledge of it is frozen and partial.

**LLMs will invent ClinVar API endpoints.** NCBI's E-utilities are well-documented but specific. An LLM may confidently suggest a URL or parameter that does not exist, or that returns a different result than described. Always run the call and inspect the actual response before writing code that depends on its structure.

All of this is expected. This is why software engineers still have jobs in the LLM era. The challenge is not "don't use LLMs." The challenge is: use them for what they're good at (generating boilerplate, explaining concepts, drafting normalization logic) while knowing when you need to read the actual docs, run the actual code, and look at the actual data.

---

## Appendix D: Useful References

- **ClinVar homepage**: https://www.ncbi.nlm.nih.gov/clinvar/
- **ClinVar data format docs**: https://www.ncbi.nlm.nih.gov/clinvar/docs/help/
- **ClinVar FTP data**: https://ftp.ncbi.nlm.nih.gov/pub/clinvar/ (if you want bulk data instead of API)
- **metapub GitHub**: https://github.com/metapub/metapub
- **metapub docs**: https://metapub.readthedocs.io/
- **NCBI E-utilities docs**: https://www.ncbi.nlm.nih.gov/books/NBK25499/
- **HPO browser**: https://hpo.jax.org/
- **ClinVar review status / star ratings**: https://www.ncbi.nlm.nih.gov/clinvar/docs/review_status/

---

## Getting Started

```bash
# 1. Clone this repo
git clone https://github.com/nthmost-orkes/clinvar-wrangler.git
cd clinvar-wrangler

# 2. Set up your Python environment
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. Get an NCBI API key (free, takes 2 minutes)
# https://www.ncbi.nlm.nih.gov/account/
export NCBI_API_KEY="your_key_here"

# 4. Start with Group A test cases
python -m clinvar_wrangler --rsids rs80357906 --output test_a1.json
cat test_a1.json

# 5. Build from there
```

Work through Group A, then Group B, then Group C. By the time your tool handles all of them without crashing and with honest output, you'll have a complete submission.

---

## Submitting Your Work

1. Fork this repo to your own GitHub account
2. Push your implementation
3. Include any sample output files you generated during development in `sample_output/`
4. Add a `NOTES.md` (brief, no more than a page) describing:
   - The biggest data quality surprise you encountered
   - One place where LLM output misled you and how you caught it
   - Any tradeoffs in your resolution algorithm you want us to know about

We will review your code, run it against the test cases, and then have a conversation about your approach and what you learned.

Good luck.

---

*This challenge was designed for senior software engineering candidates working in bioinformatics, clinical data systems, or genomics tooling. The evaluation is less about whether you got every edge case perfect and more about how you reason about uncertainty, use external tools, and handle the gap between a confident LLM response and actual working code.*
