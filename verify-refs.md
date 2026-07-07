# Reference Verification Command (/verify-refs) v3.0

Verify the consistency between references and manuscript claims using a 3-stage pipeline
backed by authoritative scholarly APIs.
Accepts manuscript file path as argument: $ARGUMENTS

**What changed in v3.0 (vs v2.1):** DOI/metadata verification now uses authoritative APIs
(Crossref, DOI content negotiation, Handle System) instead of generic web search; added
retraction/correction checking, multi-source cross-validation with a confidence score,
stricter author matching, verbatim-quote-with-location evidence for semantic judgments,
a 4-value alignment scale with confidence, OA full-text location via Unpaywall/OpenAlex,
a general topic-relevance fallback for Stage 2, defined duplicate-detection logic, a
numeric-source reliability tier, and an optional journal-legitimacy check. The overall
3-stage shape, cascading full-text retrieval, UNCERTAIN-default, and iterative fix loop
are unchanged.

---

## User Configuration: Citation Format

This command supports **Markdown** and **LaTeX** manuscripts. Auto-detection is based on
file extension: `.md` = Markdown mode, `.tex` = LaTeX mode. You can override by setting
the mode below.

**Format mode:** auto (based on file extension)

**Contact email (optional, but required for Unpaywall):** [edit here]
_Used as the `mailto`/`email` parameter (and User-Agent) for scholarly APIs. For Crossref
and OpenAlex it is optional — supplying it enters the "polite pool" (higher, more stable
rate limits). For **Unpaywall it is mandatory**: a request without `email=` returns HTTP 422
and fails. If this field is empty, skip Unpaywall (Phase 5.2b) and use OpenAlex
`best_oa_location` for OA discovery instead._

### Markdown Mode

**Reference list format (end of manuscript):**
```
[N] Author et al., "Title," *Journal*, Year. DOI: 10.xxxx/yyyy
```
Parse targets: Number `[N]`, first author surname, title (in quotes), journal (in italics), year, DOI

Supported formats: IEEE, APA, Vancouver, or similar numbered/author-year styles.

**In-text citation markers:** `^[N]^` or `[N]` — both patterns are searched.

### LaTeX Mode

**Reference source (in priority order):**
1. **BibTeX/BibLaTeX file (.bib):** Search for `.bib` files referenced via `\bibliography{}`
   or `\addbibresource{}` in the manuscript. Read with the Read tool.
   Parse each `@article{key, ...}` / `@inproceedings{key, ...}` entry to extract:
   author, title, journal/booktitle, year, doi fields.
2. **Inline \\bibitem:** If no `.bib` file, parse `\bibitem{key}` entries in
   `\begin{thebibliography}...\end{thebibliography}` environment.

**In-text citation markers:** All standard LaTeX citation commands are searched:
- `\cite{key}`, `\cite{key1,key2}`
- `\citep{key}`, `\citet{key}` (natbib)
- `\parencite{key}`, `\textcite{key}` (biblatex)
- `\autocite{key}`, `\cite[p.~42]{key}`

**BibTeX entry example:**
```bibtex
@article{hartman2011flow,
  author  = {Hartman, Ryan L. and McMullen, Jonathan P. and Jensen, Klavs F.},
  title   = {Deciding Whether To Go with the Flow},
  journal = {Angewandte Chemie International Edition},
  year    = {2011},
  volume  = {50},
  pages   = {7502--7519},
  doi     = {10.1002/anie.201004637}
}
```

**LaTeX-specific parsing notes:**
- Citation keys (e.g., `hartman2011flow`) are mapped to `.bib` entries for metadata lookup
- If a `.bib` entry lacks a `doi` field, search by title + author in Stage 1
- Multi-file manuscripts (`\input{}`, `\include{}`): read referenced `.tex` files to find all citations

---

## User Configuration: Field-Specific Keywords (Stage 2)

Stage 2 checks whether the cited paper's methodology matches the citation context
in your manuscript. Edit the keyword groups below for your research field.

**If you skip this section (leave empty), Stage 2 falls back to a general topic-relevance
check (see Phase 4) rather than being skipped entirely.**

### Keyword Groups

- **Group A (manuscript's primary method/topic):**
  _Example for perovskite solar cells: "inkjet", "ink-jet", "drop-on-demand", "DoD printing"_
  YOUR KEYWORDS: [edit here]

- **Group B (commonly confused method/topic 1):**
  _Example: "slot-die", "slot die", "slot-die coating"_
  YOUR KEYWORDS: [edit here]

- **Group C (commonly confused method/topic 2):**
  _Example: "blade coating", "doctor blade", "knife coating"_
  YOUR KEYWORDS: [edit here]

- **Group D (commonly confused method/topic 3):**
  _Example: "spin coating", "spin-coat"_
  YOUR KEYWORDS: [edit here]

### Stage 2 Classification Rule

If the manuscript cites a paper in the context of Group A, but the paper's
abstract primarily describes a method from Group B/C/D, the verdict is
**REVIEW (method_mismatch)**.

If the abstract mentions 3+ methods from different groups, classify as a
review article and skip Stage 2 (PASS as method_review_article).

---

## User Configuration: Optional Journal-Legitimacy Check (Phase 3b)

**Enable predatory/illegitimate journal check:** off (set to `on` to enable)

When `on`, Stage 1 adds a lightweight legitimacy signal for each reference's journal
(see Phase 3b). This is advisory only — a low signal is never an automatic FAIL, because
legitimacy lists are imperfect and contested. Off by default to avoid noise.

---

## API Reference (used throughout the pipeline)

All calls are made via Bash `curl` (already required for SI download). Append the contact
email as `mailto=` (Crossref/Unpaywall) or in the User-Agent when configured above.

- **DOI content negotiation (structured metadata, RA-agnostic):**
  `curl -sLH "Accept: application/vnd.citationstyles.csl+json" https://doi.org/{DOI}`
  Returns CSL-JSON (title, author[given/family], container-title, issued, type, DOI).
- **Crossref work record:** `curl -sL "https://api.crossref.org/works/{DOI}?mailto={email}"`
  (also exposes `is-referenced-by-count`, `update-to`, `relation` for retractions).
- **Crossref bibliographic search (no DOI):**
  `curl -sL "https://api.crossref.org/works?query.bibliographic={title}&query.author={surname}&rows=5&mailto={email}"`
- **DOI resolution check (authoritative):** `curl -sL "https://doi.org/api/handles/{DOI}"`
  Returns JSON with a `responseCode`: `1` = the DOI resolves; `100` = handle not found (does
  not exist); other codes = does not resolve cleanly. Judge by `responseCode`, not by any
  message string.
- **OpenAlex (cross-validation, OA location, author disambiguation/ORCID):**
  `curl -sL "https://api.openalex.org/works/doi:{DOI}?mailto={email}"` (fields:
  authorships[author.orcid], best_oa_location.pdf_url, primary_location). `mailto` is
  optional (polite pool).
- **Semantic Scholar (cross-validation):**
  `curl -sL "https://api.semanticscholar.org/graph/v1/paper/DOI:{DOI}?fields=title,authors,year,venue"`
  Unauthenticated access is heavily rate-limited; treat frequent 429s as normal degradation and fall through.
- **Unpaywall (OA full-text location):**
  `curl -sL "https://api.unpaywall.org/v2/{DOI}?email={email}"` (field: best_oa_location.url_for_pdf).
- **DataCite (datasets/software/some preprints):** `curl -sL "https://api.datacite.org/dois/{DOI}"`

If an API call fails (network error, non-JSON, timeout), do not hang: record the source as
unavailable and fall through to the next method. Missing internet degrades gracefully (see Notes).

---

## Execution Pipeline

### Phase 1: Manuscript Loading and Citation Mapping

1. Read the manuscript file specified by $ARGUMENTS. If empty, use `manuscript_draft.md`.
2. **Detect format mode:** If file extension is `.tex`, use LaTeX mode. If `.md`, use Markdown mode. For other extensions, inspect file content for `\documentclass` or `\begin{document}` (LaTeX) vs `#` headers (Markdown).
3. **Extract citations:**
   - Markdown: Search for `^[N]^` and `[N]` patterns.
   - LaTeX: Search for `\cite{key}`, `\citep{key}`, `\citet{key}`, `\parencite{key}`, `\textcite{key}`, `\autocite{key}`. Extract citation keys from `{key}` or `{key1,key2}`.
   - For LaTeX multi-file manuscripts: check for `\input{file}` and `\include{file}`, read those files too.
4. **Parse reference list:**
   - Markdown: Parse `[N] Author...` entries from the References section.
   - LaTeX: Read `.bib` file (from `\bibliography{}` or `\addbibresource{}`). Parse `@article`, `@inproceedings`, `@book`, `@misc` entries. If no `.bib` file, parse `\bibitem{key}` entries.
5. **Map citations to references:** Match in-text citation keys/numbers to reference list entries.
6. If no references are found, report "No references found in manuscript" and stop.

### Phase 2: Structural Integrity Check

Check and report:
- **Orphan refs**: In reference list but never cited in text
- **Missing refs**: Cited in text but not in reference list
- **Duplicate refs**: Same work listed under different numbers/keys. **Detection logic:**
  two entries are duplicates if (a) their DOIs are identical (case-insensitive), or (b) DOI
  is absent for one/both and the titles match after normalization (lowercase, strip
  punctuation/whitespace) with first-author surname and year also equal. Report each
  duplicate group with the numbers/keys involved.

Output format:
```
## Structural Integrity Check
- In-text citations: [list]
- Reference list: [list]
- Orphan refs: [list or "none"]
- Missing refs: [list or "none"]
- Duplicate refs: [group list or "none"]
```

### Phase 3: Stage 1 -- Authoritative Metadata Verification

**Context length management:** If 4+ references, process in batches of 3 with intermediate output.

For each reference:

1. **Resolve metadata authoritatively (in order; stop at first success):**
   - **If a DOI is present:** call DOI content negotiation (CSL-JSON) and/or Crossref
     `works/{DOI}`. Also call the Handle resolution check `doi.org/api/handles/{DOI}`.
     - Handle check `responseCode` is not `1` (e.g. `100` = handle not found), or content
       negotiation returns 404 -> the DOI does not exist -> **FAIL (DOI_NOT_FOUND)**. This is
       the authoritative test for the classic "well-formatted but fabricated DOI" failure;
       do not rely on web search for this.
   - **If no DOI:** Crossref bibliographic search by `query.bibliographic={title}` +
     `query.author={surname}`; take the top candidate.
   - **Fallback only if all API calls are unavailable:** WebSearch "DOI: {DOI}" or
     "{title} {first author surname}". Mark such references as `metadata_source: websearch`
     (lower confidence).
2. **Retrieve structured metadata:** title, full author list (family names), container-title
   (journal), issued year, type. If an abstract is returned (Crossref/OpenAlex), retain it for Stage 2-3.
3. **Compare (all must hold for PASS):**
   - **Title:** semantic match. Subtitle presence/absence, word order, colon delimiter
     differences are tolerated. A clearly different paper is FAIL.
   - **Authors:** compare the **full list of author surnames**, not only the first. Require
     the first author to match AND overall surname-list agreement (order-insensitive) above
     a high bar. Account for initials-vs-fullname, hyphenated/compound surnames, and
     family-name-order variation before declaring a mismatch. When available, use
     OpenAlex `authorships[].author.orcid` to confirm identity.
   - **Journal:** full name or standard abbreviation must match the container-title.
   - **Year:** exact match (tolerate online-first vs issue year by at most 1 year, noting it).
4. **Cross-validate across sources (P5):** query OpenAlex and Semantic Scholar for the same
   DOI/title. Compute a **confidence score** from source agreement:
   - 3/3 sources agree on title+first author -> confidence: high
   - 2/3 agree -> confidence: medium
   - only 1 source has the record, or sources disagree -> confidence: low (treat as REVIEW
     unless a hard FAIL already applies)
   - a work found in **no** authoritative source (only web search, or nowhere) -> **FAIL
     (not_in_any_index)** — a strong fabrication signal.
   - For DOI-less references, cross-validate by title+author search (Crossref
     `query.bibliographic`, OpenAlex title filter) rather than the DOI key. A `verified_no_doi`
     PASS (step 5) is exempt from the low-confidence downgrade when only one source indexes it.
5. **Verdict:**
   - DOI does not resolve -> **FAIL (DOI_NOT_FOUND)**
   - Found in no authoritative index -> **FAIL (not_in_any_index)**
   - Clearly different paper -> **FAIL (wrong_paper)**
   - Author/journal/year mismatch -> **FAIL (metadata_mismatch)** — specify which field
   - DOI field empty but metadata verified via title search -> **PASS (verified_no_doi)**
   - DOI field empty and unverifiable -> **REVIEW (no_doi)**
   - Sources unavailable / low confidence -> **REVIEW (low_confidence)**
   - All metadata matches with medium/high confidence -> **PASS**

**Output Stage 1 intermediate results:**
```
### Stage 1 Results (Authoritative Metadata Verification)
| Ref | Title | Authors | Journal | Year | Source(s) | Confidence | Verdict | Notes |
|-----|-------|---------|---------|------|-----------|-----------|---------|-------|
```

### Phase 3a: Retraction and Correction Check (P2)

**For every reference that resolves (any Stage 1 verdict except DOI_NOT_FOUND).**

1. From the Crossref `works/{DOI}` record, look for a retraction/correction signal in this
   order (Crossref's coverage of update relations is incomplete, so use all three):
   - **`updated-by[]`** entries whose `type` is `retraction`, `correction`, `erratum`, or
     `expression_of_concern`. This is the forward pointer ("this work was retracted/updated
     by ..."). Note: `update-to` is the *reverse* pointer that appears on the notice record,
     not on the retracted article — do not rely on it here.
   - The retrieved **`title`**: many retracted works carry a prefix such as `RETRACTED:`,
     `WITHDRAWN:`, `Retracted:`, or `Expression of Concern` even when the update relations
     are empty (verified for well-known retractions). Check for these strings.
   - Optionally the Crossref filter query `works?filter=doi:{DOI},update-type:retraction`
     (here `update-type` is a filter parameter *value*, not a field on the record).
2. **Verdict overlay (independent of Stage 1):**
   - Retraction / withdrawal found -> **RETRACTED** (escalates as CRITICAL in Phase 7 — a
     retracted work must not be cited as support without explicit justification)
   - Correction / erratum / expression of concern found -> **REVIEW (has_correction)** —
     note it; the cited claim may be affected
   - None found -> no overlay (this means "no signal found", not a guarantee of no retraction)

Record the retraction/correction status as a dedicated column in the final report.

### Phase 3b: Optional Journal-Legitimacy Check (P10)

**Only if enabled in User Configuration.** Advisory, never an automatic FAIL.

1. For each journal (container-title), check for a legitimacy signal: presence in DOAJ
   (`https://doaj.org/api/search/journals/{title}`) for OA journals, indexing in
   Crossref/OpenAlex `primary_location.source`, and whether the publisher/journal has
   plausible standard identifiers (ISSN).
2. **Verdict overlay:** if a journal has no discoverable indexing and no ISSN and matches
   no known legitimacy source -> **REVIEW (journal_legitimacy_uncertain)** with a note that
   the author should confirm the venue is not predatory (ICMJE advises against citing
   predatory/pseudo-journals). Never auto-FAIL on this signal alone.

### Phase 4: Stage 2 -- Field / Topic Relevance Classification

**Only for Stage 1 PASS references.**

**If keyword groups are defined:** use them (keyword-group rule below). **If keyword groups
are empty:** run the **general topic-relevance check** instead of skipping — since
"real DOI but topically irrelevant" is the single most common AI citation error, do not
skip relevance entirely.

1. **Get abstract:** Use abstract from Stage 1 (Crossref/OpenAlex) if available. Otherwise
   WebSearch "{DOI} abstract". If still unavailable, skip to Stage 3 title_only mode
   (REVIEW: abstract_unavailable).
2. **Section identification:** Find the section heading (`##`) immediately before each
   citation marker. If cited in multiple sections, check each and use the strictest verdict.
3. **Classify:**
   - **Keyword-group mode:** classify the paper's methodology by the keyword groups.
     - Manuscript cites in Group A context but paper's method is Group B/C/D -> **REVIEW (method_mismatch)**
     - Abstract mentions 3+ method groups -> review article, skip Stage 2 (PASS as method_review_article)
     - No mismatch -> **PASS**
   - **General topic-relevance mode (no keyword groups):** judge whether the paper's primary
     subject/method (from the abstract) is plausibly relevant to the citation context (the
     sentence + section where it is cited).
     - Paper's topic is clearly unrelated to the citation context -> **REVIEW (topic_mismatch)**
     - Plausibly relevant -> **PASS**
     - Cannot tell from abstract -> **REVIEW (relevance_uncertain)**

**Output Stage 2 intermediate results:**
```
### Stage 2 Results (Field / Topic Relevance)
| Ref | Abstract | Detected Topic/Method | Citation Section | Verdict | Notes |
|-----|----------|----------------------|-----------------|---------|-------|
```

### Phase 5: Stage 3 -- Semantic Alignment Verification

**For Stage 1-2 PASS references** (plus the Stage 2 `abstract_unavailable` case, which enters
Stage 3 in title_only mode and stays a REVIEW). Stage 1 metadata agreement does **not** imply
the paper supports the claim — Stage 3 is independent and must run even when Stage 1 is a clean PASS.

#### Step 5.1: Extract and Classify Citations

For each reference:
1. Extract sentences containing the citation marker, plus one sentence before and after (sentence boundary: period "." or "。" or newline).
2. Classify each citation as:
   - **Data citation**: Specific numerical values (efficiency, concentration, duration, etc.)
   - **Claim citation**: Findings, methods, conclusions from the paper
3. Also check tables in the manuscript for numerical data attributed to this reference.

#### Step 5.2: Paper Text Retrieval (Cascading Fallback)

Try in order. Stop at first success:

**5.2a. Local PDF:**
- Search `./references/` or `./refs/` directory for PDFs matching DOI, author surname, or title fragment.
- Found -> Read with Read tool (max 20 pages; use pages parameter for longer papers).
- Success -> **fulltext_mode (local_pdf)**

**5.2b. OA full text via Unpaywall / OpenAlex (preferred over scraping):**
- Query Unpaywall `v2/{DOI}?email={email}` (only if a contact email is configured — it is
  mandatory for Unpaywall) or OpenAlex `best_oa_location` for an OA PDF/HTML URL. If no
  email is set, use OpenAlex only.
- WebFetch the returned OA URL (or `https://doi.org/{DOI}` if none). On 403/timeout, do not
  retry indefinitely — move to next.
- Success -> **fulltext_mode (publisher_html / oa_repository)**

**5.2c. Supplementary Information:**
- If the landing page (from 5.2b) contains links with "supplementary", "supporting information", or "suppl", attempt to download the SI PDF.
- Use Bash: `curl -sL -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36" -o "./.verify-refs-cache/ref{N}_si.pdf" "{SI_URL}"`
  (the cache dir `./.verify-refs-cache/` is created if absent and is portable across OS; add it to `.gitignore`).
- Download success -> Read with Read tool. Prioritize numerical tables (Table S1, S2, etc.).
- Data citation values found in SI -> **supplementary_mode**
- Failure -> next

**5.2d. Author Self-Archive:**
- WebSearch: `"{paper title}" researchgate`, then `"{paper title}" arxiv`
- Found -> WebFetch full text.
- Success -> **fulltext_mode (self_archived)**. For arXiv, append **(preprint_version)**.
- Failure -> next

**5.2e. Abstract:**
- Use abstract from Stage 1-2 (max 300 words).
- Available -> **abstract_mode**
- Not available -> next

**5.2f. Title-only:**
- Use title only. Append **(title_only)** to results. UNCERTAIN auto-escalates to REVIEW.

#### Step 5.3: Claim Citation Verification

Use the retrieved paper text to perform structured judgment. **The judgment MUST be grounded
in the source: quote the exact supporting (or contradicting) sentence(s) verbatim from the
retrieved paper text, with location (page / section / figure / table).** A judgment with no
verbatim source quote is not permitted — output UNCERTAIN instead.

```
Manuscript claim: [extracted citation text]
Source evidence (verbatim quote from paper): "[exact sentence(s) from the retrieved text]"
Source location: [page / section / Table / Figure]

Judge the following:

(1) Alignment (4-value): does the source support the manuscript claim?
    SUPPORTED / PARTIALLY_SUPPORTED / UNSUPPORTED / UNCERTAIN
    - SUPPORTED: the quoted source text directly backs the claim.
    - PARTIALLY_SUPPORTED: the gist is backed but a detail (scope, magnitude, condition) differs.
    - UNSUPPORTED: the source does not back the claim, or contradicts it.
    - UNCERTAIN: the retrieved text is insufficient to decide.

(2) Confidence: high / medium / low.
    If confidence is low, or no verbatim quote could be extracted, force the result to UNCERTAIN.

(3) If UNSUPPORTED or PARTIALLY_SUPPORTED, assess severity:
    - The paper's conclusion contradicts the manuscript's citation -> CRITICAL
    - The paper's primary subject/method is fundamentally different from the citation context -> CRITICAL
    - Large numerical discrepancy -> SEVERE; minor discrepancy -> MODERATE/MINOR
    Rate as CRITICAL / SEVERE / MODERATE / MINOR with reasoning.
```

Verdict rules:
- SUPPORTED (medium/high confidence) -> **PASS**
- PARTIALLY_SUPPORTED -> severity MODERATE/MINOR -> **REVIEW**; if the differing detail is
  load-bearing (contradicts magnitude/scope of the claim) -> **SEVERE hold**
- UNSUPPORTED + CRITICAL -> **immediate escalation** (Phase 7)
- UNSUPPORTED + SEVERE -> **SEVERE hold** (Phase 7)
- UNSUPPORTED + MODERATE/MINOR -> **REVIEW**
- UNCERTAIN (incl. forced-low-confidence / title_only) -> **REVIEW**

Write the literal verdict into the Stage 3 table's `Verdict` column as one of `PASS`,
`REVIEW`, `SEVERE`, or `CRITICAL`; for SEVERE/CRITICAL also fill the `Severity` column.
These literals feed the Phase 8 counts directly.

#### Step 5.4: Data Citation Verification (Numerical Cross-Check)

Extract all specific numbers from the manuscript and cross-check against paper text.
**Reliability tier:** numbers taken from continuous body text are more reliable than numbers
read from tables, and both are more reliable than numbers read from figures/images. When a
match comes from an SI table or a figure, mark the PASS with `(low_reliability_source)` and
recommend a brief manual confirmation.

**[fulltext_mode] Full text / SI:**
- Number found and matches -> **PASS** (verified_{source_type}; add `(low_reliability_source)` if from SI table/figure)
- Number found but differs -> **REVIEW** (data_mismatch) — show both values
- Number not found in text -> **REVIEW** (data_not_found_in_fulltext)
- arXiv match -> **PASS** (verified_preprint, confirm with published version)

**[abstract_mode] Abstract:**
- Number found and matches -> **PASS** (verified_abstract)
- Number found but differs -> **REVIEW** (data_mismatch) — show both values
- Number not in abstract -> **REVIEW** (data_not_in_abstract) — "This value is not in the abstract. It may be in the paper body. Manual verification recommended."

**[title_only]:**
- Any data citation -> **REVIEW** (data_unverifiable_title_only)

**Output Stage 3 intermediate results:**
```
### Stage 3 Results (Semantic Verification)
| Ref | Source Mode | Citation Type | Alignment | Confidence | Verdict | Severity | Source Quote / Location | Reason |
|-----|-----------|--------------|-----------|-----------|---------|----------|------------------------|--------|
```

### Phase 6: Unsupported Claims Detection

Identify quantitative claims (numerical data, specific facts) in the manuscript that lack citation markers. Exclude:
- Widely known facts in the field
- Author's own experience/expertise (phrases like "we have", "our group", etc.)
- Explicitly labeled proprietary/internal data

### Phase 7: Severity Escalation

If Phase 3a produced a RETRACTED overlay, or Phase 5 contains CRITICAL or SEVERE findings, escalate before the final report.

#### RETRACTED / CRITICAL -- Immediate Verification Loop Stop

Output impact analysis and ask the user for a decision:

```
*** CRITICAL FINDING -- Verification Loop Stopped ***

Finding: [specific discovery; for retraction: "Ref[N] is RETRACTED (date/reason)"]
Reference: [Ref number]
Affected manuscript sections:
  - [Section]: [affected text]
  - [Table]: [affected data]

Impact analysis:
  This finding [invalidates/significantly weakens] the following claim:
  "[manuscript claim text]"

Options:
  A) Rewrite section -- remove/revise the affected claim
  B) Find alternative reference -- WebSearch for a replacement
  C) Tone adjustment -- change assertion to conditional language
  D) Author decides -- verify original paper directly
```

#### SEVERE -- Hold and Continue

If no CRITICAL/RETRACTED findings, present SEVERE findings after all references are checked:
```
[SEVERE] Ref[N]: [finding]
Impact: Section X.Y, [affected text]
Options: A) Tone adjustment  B) Alternative reference  C) Add clarification  D) Author decides
```

### Phase 8: Integrated Report

Compile all Stage results into a final report:

```markdown
# Reference Verification Report

Date: [date]
File: [file path]
References: [N]
Citations: [N] locations
Pipeline: 3-Stage v3.0 (authoritative APIs)

## 1. Structural Integrity
[Phase 2 results, including duplicate groups]

## 2. Verification Summary

| Ref | Stage1 (Metadata) | Confidence | Retraction | Stage2 (Relevance) | Stage3 (Semantic) | Verdict | Reason |
|-----|-------------------|-----------|-----------|--------------------|-------------------|---------|--------|

Source modes: local_pdf / oa_repository / publisher_html / supplementary / self_archived / preprint / abstract / title_only
Metadata sources: crossref / openalex / semantic_scholar / datacite / websearch(fallback)

## 3. CRITICAL/SEVERE/RETRACTED Findings
[Phase 7 results, or "None"]

## 4. FAIL References (Details)
[Each FAIL with mismatch details and correction instructions]

## 5. REVIEW References (Details)
[Each REVIEW with citation type, source mode, confidence, recommended action]

## 6. Unsupported Claims
[Phase 6 results]

## 7. Overall Verdict

- CRITICAL: N
- RETRACTED: N
- SEVERE: N
- FAIL: N
- REVIEW: N
- PASS: N

### Author Checklist
[ ] [RETRACTED] Remove or explicitly justify citing the retracted work (mandatory)
[ ] [CRITICAL/SEVERE] Decide response and apply corrections
[ ] [FAIL] Correct metadata (mandatory)
[ ] [REVIEW] Verify and record judgment
[ ] [PASS] No additional action (brief visual check recommended; low_reliability_source items warrant a look)
```

### Phase 9: Iterative Verification Loop

If any RETRACTED, CRITICAL, SEVERE, FAIL, or REVIEW findings exist, run the fix-verify loop:

1. **RETRACTED/CRITICAL:** After user decides, re-verify affected references from Phase 3.
2. **SEVERE:** After user decides, re-verify from Phase 4.
3. **FAIL:** Propose corrections using authoritative-API metadata. Apply after user approval, re-verify.
4. **REVIEW:** Present details to user:
   - "Keep citation (add justification to manuscript)"
   - "Replace citation (search for alternative)"
   - "Remove citation"
   - "Provide PDF" (for data_not_in_abstract) -- user places PDF in ./references/, re-verified as local_pdf next round
5. **Convergence:** Loop ends when:
   - All PASS (ideal)
   - FAIL=0, RETRACTED=0, and all REVIEW decided (practical)
   - Max 3 rounds (remaining issues listed for manual resolution)
6. **Round diff display:**
   ```
   === Round N Results (vs Round N-1) ===
   [1] FAIL -> PASS  (correction applied)
   [2] REVIEW(data_not_in_abstract) -> PASS(verified_local_pdf)  (PDF provided)
   Remaining: FAIL=X, REVIEW=Y
   ```
7. **Final check:** After loop completion, request author's 30-second per-reference visual confirmation.

---

## Notes

- **Internet required.** This command depends on Bash (curl), WebSearch, WebFetch, Read, and Grep. Authoritative metadata uses Crossref / DOI content negotiation / Handle API / OpenAlex / Semantic Scholar / Unpaywall via curl; WebSearch is only a fallback when APIs are unavailable. If all network access is unavailable, Phases 3-5 are skipped; only structural checks (Phase 1-2) and unsupported claims (Phase 6) are performed.
- **API etiquette:** set the contact email in User Configuration to enter the polite pool (higher rate limits). On any API error (non-JSON, 4xx/5xx, timeout), do not hang — record the source as unavailable and fall through.
- **Verifier grounding:** semantic judgments require a verbatim source quote with location; without one, the result is UNCERTAIN (REVIEW). Metadata agreement never substitutes for semantic verification.
- Uncertain results default to REVIEW with a manual verification recommendation.
- This command outputs verification reports and proposes corrections. Corrections are applied only after author approval.
- SI/temp files are written under `./.verify-refs-cache/` (portable; add to `.gitignore`). PDF reading uses the Read tool (max 20 pages per call; use the pages parameter for large PDFs).
