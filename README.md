[README_Pre2Pubtracker_020526.md](https://github.com/user-attachments/files/31117878/README_Pre2Pubtracker_020526.md)
# Pre2Pub Retraction Tracker v2.2

Find journal publications corresponding to preprints, and optionally check whether retracted references were corrected in the published version.

## Usage Modes

### ① Basic — Preprint-to-publication linking

Upload any CSV with a `DOI` column (and optionally `Title`). The tool finds the corresponding journal publication using a 6-method linking cascade. No other files needed.

**Use case:** Any researcher tracking whether preprints have been published.

### ② Full — Retraction tracking

Upload preprints from the [Feet of Clay](https://dbrech.irit.fr/pls/apex/f?p=9999:31) detector (which flags preprints citing retracted references) along with [Retraction Watch](https://retractionwatch.com/) data. The tool links preprints to publications AND checks whether retracted references were corrected.

**Use case:** Research integrity studies tracking the propagation of retracted citations from preprints into journal articles.

## Quick DOI Check

The HTML tool includes a single-DOI checker at the top of the page. Paste any preprint DOI to run it through the full A→F cascade and see results immediately — useful for testing individual cases or verifying suspicious results.

## Linking Methods

The tool uses a cascade of six methods, each tried in order until a match is found:

| Method | Name | Description | Reliability |
|--------|------|-------------|-------------|
| **A** | bioRxiv API | Query bioRxiv/medRxiv API for linked publication | Ground truth |
| **B** | Crossref Relations | Check `is-preprint-of` / `update-to` relations | Ground truth |
| **C** | PubMed (Pre2Pub) | Title search + author matching per Langnickel et al. 2022 | High |
| **D** | Cabanac | Crossref bibliographic search per Cabanac et al. 2021 | Medium |
| **E** | Title-only | Jaccard ≥0.85 fallback, no author verification | Low (verify all) |
| **F** | Europe PMC | Curated preprint-publication links from Europe PMC | High |

**Date validation:** After any method finds a match, the tool verifies that the publication year is not before the preprint year. Matches where `pub_year < prep_year` are rejected as false positives.

### Method C: Pre2Pub Algorithm (Langnickel et al. 2022)

1. Search PubMed with stopwords-removed title (max 5 results)
2. Author matching uses **4 criteria** (match if ANY is satisfied):
   - More matched than unmatched pairs when iterating simultaneously
   - First 3 authors identical (ALL must match at same positions)
   - First AND last author identical at same positions
   - First AND last of preprint found ANYWHERE in journal list
3. Fallback: Author search → title Jaccard ≥0.80

**Reference:** Langnickel et al. (2022). Pre2Pub: An algorithm for tracking the path from preprint to journal. *JMIR*. [doi:10.2196/34072](https://doi.org/10.2196/34072)

### Method D: Cabanac Algorithm (Cabanac et al. 2021)

1. Search Crossref: `query.bibliographic` = title + first author (max 10 results)
2. **High confidence (D-hi):** Jaccard title similarity ≥0.80 alone
3. **Low confidence (D-lo):** Jaccard 0.25–0.80 + first author match via:
   - ORCID (if available), OR
   - Family name comparison (Levenshtein >0.85)

**Note:** The original paper used Jaccard ≥0.10 for low confidence. We raised to 0.25 for better precision, as the paper was developed on COVID preprints which had more specific titles.

**Reference:** Cabanac et al. (2021). Day-to-day discovery of preprint–publication links. *Scientometrics*. [doi:10.1007/s11192-021-03900-7](https://doi.org/10.1007/s11192-021-03900-7)

### Method F: Europe PMC Links

Europe PMC maintains curated links between preprints and their published versions. Method F queries the Europe PMC API as a final fallback after methods A–E.

**API:** `https://www.ebi.ac.uk/europepmc/webservices/rest/search?query=DOI:{preprint_doi}`

**Reliability:** High — links are curated by Europe PMC.

**Coverage note:** Europe PMC is biomedical-focused. Coverage is best for bioRxiv/medRxiv, partial for Research Square, and limited for arXiv, SSRN, and OSF-based servers.

## Data Sources

| Data | Primary source | Fallback |
|------|---------------|----------|
| **Metadata** (authors, dates, titles) | Crossref API | — |
| **Reference lists** | Crossref API (publisher-deposited) | OpenAlex API (PDF-extracted) |
| **Retraction data** | User-uploaded Retraction Watch CSV | — |
| **Preprint-publication links** | Methods A–F (see above) | — |

### OpenAlex Reference Fallback

When Crossref doesn't have a reference list for a publication (common with some publishers), the tool falls back to OpenAlex, which extracts references from PDFs for broader coverage.

The `refs_source` column in the output indicates "crossref" or "openalex". The dashboard shows a "Via OpenAlex" count.

## Author Comparison

### Name Matching
- **Last name:** Normalized (lowercase, non-letters removed)
- **Initials:** Extracted from given name
- **Match:** Same last name + at least one overlapping initial (if both have initials)
- **Uncertain:** Flagged when initials couldn't be compared (one/both missing)

Author metadata is retrieved from the Crossref API for both preprints and journal articles.

### Overlap Metrics
- **overlap_ratio:** `matched / max(prep_count, jour_count) × 100`
- **Categories:** same | added | removed | changed | no_overlap | no_data

## Timeline Classification

When was the cited reference retracted relative to the preprint/publication? Retraction dates come from the Retraction Watch CSV; preprint and publication dates from Crossref.

| Timeline | Description |
|----------|-------------|
| `retracted_before_preprint` | Retraction before preprint posted |
| `retracted_around_preprint` | Same month as preprint (month-level precision) |
| `retracted_before_pub` | Retracted between preprint and publication |
| `retracted_around_pub` | Same month as publication |
| `retracted_after_pub` | Retracted after publication (not authors' fault) |
| `ambiguous` | Year-only precision, same year — cannot determine |
| `unknown` | Missing dates |

**Lag days** only computed when both dates have full day precision (YYYY-MM-DD).

## Installation

### Python CLI

```bash
pip install requests pandas tqdm
python pre2pub_pipeline_v2.2.py --help
```

### HTML Tool

Open `pre2pub_tracker_v2.2.html` in a browser. No installation required.

## Usage

### HTML Tool

1. Open `pre2pub_tracker_v2.2.html` in browser
2. Upload files:
   - **Preprints CSV** (required): Any CSV with a `DOI` column (case-insensitive). Optionally `Title`, `Venue`, and `Cited Problematic Papers`.
   - **Retraction Watch CSV** (optional): For retraction checks. Contains retraction dates and DOIs.
   - **Feet of Clay Articles CSV** (optional): For cross-checking against the [Feet of Clay](https://dbrech.irit.fr/pls/apex/f?p=9999:31) journal articles database.
   - **PubPeer Batch CSV** (optional): For PubPeer flagging.
3. Enter email for API polite pool
4. Set batch size (default: 50)
5. Click "Run Pipeline"

### Python CLI

```bash
# Basic — just link preprints to publications
python pre2pub_pipeline_v2.2.py \
    --preprints  my_preprints.csv \
    --output     results.csv \
    --email      your@email.com

# Full — with retraction tracking
python pre2pub_pipeline_v2.2.py \
    --preprints  FoC_preprints.csv \
    --rw         retraction_watch.csv \
    --output     results.csv \
    --email      your@email.com

# All options
python pre2pub_pipeline_v2.2.py \
    --preprints  FoC_preprints.csv \
    --pps-pubs   FoC_journal_articles.csv \
    --rw         retraction_watch.csv \
    --pubpeer    pubpeer_batch.csv \
    --output     results.csv \
    --email      your@email.com \
    --limit      100 \
    --delay      0.35 \
    --checkpoint-interval 50

# Resume from checkpoint
python pre2pub_pipeline_v2.2.py \
    --preprints  FoC_preprints.csv \
    --rw         retraction_watch.csv \
    --output     results.csv \
    --resume     results_checkpoint.json
```

## Input Files

### Preprints CSV (required)

**Basic mode** — minimum columns:

| Column | Description |
|--------|-------------|
| DOI | Preprint DOI (case-insensitive column name) |

**Full mode** — additional columns from Feet of Clay:

| Column | Description |
|--------|-------------|
| DOI | Preprint DOI |
| Title | Preprint title (improves linking if Crossref metadata unavailable) |
| Cited Problematic Papers | Bullet (•) separated list of cited DOIs |
| Venue | Preprint server |

### Retraction Watch CSV (optional)

| Column | Description |
|--------|-------------|
| OriginalPaperDOI | DOI of retracted paper |
| RetractionDate | Date in M/D/YYYY format |
| RetractionNature | Type (retraction, correction, EoC) |
| Reason | Reason for retraction |

### Feet of Clay Articles CSV (optional)

| Column | Description |
|--------|-------------|
| DOI | DOI of journal article flagged by Feet of Clay |

## Output Columns

### Core Linking
| Column | Description |
|--------|-------------|
| preprint_doi | Input preprint DOI |
| journal_doi | Found journal publication DOI |
| source | Linking method (A, B, C, D-hi, D-lo, E, F) |
| preprint_date | Preprint posting date (from Crossref) |
| pub_date | Publication date (from Crossref) |
| lag_days | Days between preprint and publication |

### Reference Status (requires Retraction Watch CSV)
| Column | Description |
|--------|-------------|
| ref_status | still_cited / partial / corrected / no_refs / rejected_date |
| refs_source | Where refs came from: "crossref" or "openalex" |
| still_cited | List of retracted refs still in publication |
| corrected | List of retracted refs removed |
| timeline | When retraction occurred relative to preprint/pub |
| timeline_note | Additional precision info |

### Author Overlap
| Column | Description |
|--------|-------------|
| author_overlap | Category: same/added/removed/changed/no_overlap |
| author_overlap_ratio | Percentage overlap |
| author_matched | Number of preprint authors found in journal |
| author_uncertain | Matches without initial comparison |

### Cross-Checks (requires Feet of Clay Articles CSV)
| Column | Description |
|--------|-------------|
| in_foc | Is journal DOI in Feet of Clay articles file? |
| discrepancy | match / pps_has_other_issues / verify_link |
| pubpeer_preprint | Has PubPeer comments? |
| pubpeer_journal | Has PubPeer comments? |

## Dashboard (HTML Tool)

The HTML tool provides an interactive dashboard with:

- **Overview:** Processed / Published / Not Published counts with percentages
- **Reference Status:** Still Cited / Partial / Corrected / No Data / Via OpenAlex
- **Feet of Clay Cross-Check:** In FoC / Not in FoC / Match / Mismatch
- **Author Overlap:** Same / Added / Changed / No Overlap / Uncertain
- **Timeline:** Before Preprint / Between / After Pub / Ambiguous / Same Month
- **Linking Method:** Clickable A/B/C/D/E/F boxes with counts and percentages

Click any card to filter the results table. Export to CSV or generate summary reports.

## API Rate Limits

The tool respects API rate limits:
- **Crossref:** 0.35s between requests (polite pool with email)
- **PubMed:** 0.35s between requests
- **bioRxiv:** 0.25s between requests
- **Europe PMC:** 0.35s (generous limits — up to 25 req/s allowed)
- **OpenAlex:** 0.1s (generous limits)

Provide your email via `--email` (CLI) or the input field (HTML) for Crossref's polite pool.

## Changelog

### v2.2 (Current)
- **Two usage modes:** Basic (any preprint CSV) and Full (retraction tracking with FoC + RW)
- **Quick DOI Check:** Test a single DOI through the full pipeline
- **Date validation:** Rejects matches where publication year < preprint year
- **Method C:** Implemented all 4 Pre2Pub author criteria from Langnickel et al. 2022
- **Method C:** Reduced max results from 8 to 5 per paper
- **Method D:** Added ORCID matching per Cabanac et al. 2021
- **Method F:** Added Europe PMC curated links as final fallback
- **OpenAlex fallback:** When Crossref has no refs, tries OpenAlex
- **Author overlap:** Added quantitative metrics (ratio, counts, uncertain)
- **Timeline:** Added ambiguous category for year-only same-year cases
- **Timeline:** Added same-month handling with notes
- **Dashboard:** Shows percentages alongside counts
- **Data sources:** Documented in tool legend and tooltips
- **Renamed:** PPS → Feet of Clay (FoC) throughout
- **CSV columns:** `in_foc`, `foc_only_refs`, `refs_source`

### v2.0
- Fixed author normalization bug
- Fixed year comparison to allow same-year publications
- Added Crossref caching
- Added checkpoint/resume functionality

## License

MIT License

## Citation

If you use this tool in your research, please cite this github page.

And the underlying algorithms:

Langnickel et al. (2022). Pre2Pub. JMIR. doi:10.2196/34072
Cabanac et al. (2021). Scientometrics. doi:10.1007/s11192-021-03900-7

