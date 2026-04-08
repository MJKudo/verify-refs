# Reference Verification Command (/verify-refs) v2.1

Verify the consistency between references and manuscript claims using a 3-stage pipeline.
Accepts manuscript file path as argument: $ARGUMENTS

---

## User Configuration: Citation Format

This command supports **Markdown** and **LaTeX** manuscripts. Auto-detection is based on
file extension: `.md` = Markdown mode, `.tex` = LaTeX mode. You can override by setting
the mode below.

**Format mode:** auto (based on file extension)

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

**If you skip this section (leave empty), Stage 2 will be skipped entirely,
and the pipeline proceeds directly from Stage 1 to Stage 3.**

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
- **Duplicate refs**: Same reference listed under different numbers

Output format:
```
## Structural Integrity Check
- In-text citations: [list]
- Reference list: [list]
- Orphan refs: [list or "none"]
- Missing refs: [list or "none"]
```

### Phase 3: Stage 1 -- DOI-based Metadata Verification

**Context length management:** If 4+ references, process in batches of 3 with intermediate output.

For each reference:

1. **WebSearch by DOI:** Search for "DOI: {DOI value}". If DOI is empty, search by "{title} {first author surname}".
2. **Retrieve metadata:** Extract actual paper metadata (title, author, journal, year) from search results. If abstract is available, retain for Stage 2-3.
3. **Compare:**
   - Title: Prioritize semantic match. Subtitle presence/absence, word order, colon delimiter differences are tolerated. Clearly different papers are FAIL.
   - Author: First author surname must match
   - Journal: Full name or abbreviation must match
   - Year: Exact match
4. **Verdict:**
   - Clearly different paper -> **FAIL**
   - DOI cannot be resolved -> **FAIL** (DOI_NOT_FOUND)
   - DOI field empty and unverifiable -> **REVIEW** (no_doi)
   - Author/journal/year mismatch -> **FAIL** (metadata_mismatch)
   - No WebSearch results -> **REVIEW** (search_unavailable)
   - All metadata matches -> **PASS**

**Output Stage 1 intermediate results:**
```
### Stage 1 Results (DOI Metadata Verification)
| Ref | Title | Author | Journal | Year | Verdict | Notes |
|-----|-------|--------|---------|------|---------|-------|
```

### Phase 4: Stage 2 -- Field-Specific Classification

**Only for Stage 1 PASS references.** **If no keyword groups are defined above, skip Stage 2 entirely and proceed to Stage 3.**

1. **Get abstract:** Use abstract from Stage 1 WebSearch if available. Otherwise, WebSearch "{DOI} abstract". If still unavailable, skip to Stage 3 title_only mode (REVIEW: abstract_unavailable).
2. **Keyword extraction:** Classify the paper's methodology based on the keyword groups defined above.
3. **Section identification:** Find the section heading (`##`) immediately before each citation marker. If cited in multiple sections, check each and use the strictest verdict.
4. **Verdict:**
   - Manuscript cites in Group A context but paper's method is Group B/C/D -> **REVIEW** (method_mismatch)
   - Abstract mentions 3+ method groups -> review article, skip Stage 2 (PASS as method_review_article)
   - No mismatch -> **PASS**

**Output Stage 2 intermediate results:**
```
### Stage 2 Results (Field Classification)
| Ref | Abstract | Detected Method | Citation Section | Verdict | Notes |
|-----|----------|----------------|-----------------|---------|-------|
```

### Phase 5: Stage 3 -- Semantic Alignment Verification

**Only for Stage 1-2 PASS references.**

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

**5.2b. Publisher OA HTML:**
- WebFetch the DOI redirect URL `https://doi.org/{DOI}`.
- Success -> **fulltext_mode (publisher_html)**
- 403/timeout -> next

**5.2c. Supplementary Information:**
- If the publisher landing page (from 5.2b) contains links with "supplementary", "supporting information", or "suppl", attempt to download the SI PDF.
- Use Bash: `curl -sL -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36" -o /tmp/ref{N}_si.pdf "{SI_URL}"`
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

Use the retrieved paper text to perform structured judgment:

```
Manuscript claim: [extracted citation text]
Paper content: [retrieved fulltext/abstract/title]

Judge the following:

(1) Accuracy: Does the manuscript accurately reflect the paper's content?
    ACCURATE / INACCURATE / UNCERTAIN — provide one sentence of evidence.

(2) If INACCURATE, assess severity:
    - Does the paper's conclusion contradict the manuscript's citation? (yes -> CRITICAL)
    - Is the paper's primary subject/method fundamentally different from the citation context? (yes -> CRITICAL)
    - How large is the numerical discrepancy? (large -> SEVERE, minor -> MODERATE)
    Rate as CRITICAL / SEVERE / MODERATE / MINOR with reasoning.
```

Verdict rules:
- ACCURATE -> **PASS**
- INACCURATE + CRITICAL -> **immediate escalation** (Phase 7)
- INACCURATE + SEVERE -> **SEVERE hold** (Phase 7)
- INACCURATE + MODERATE/MINOR -> **REVIEW**
- UNCERTAIN -> **REVIEW**

#### Step 5.4: Data Citation Verification (Numerical Cross-Check)

Extract all specific numbers from the manuscript and cross-check against paper text.

**[fulltext_mode] Full text / SI:**
- Number found and matches -> **PASS** (verified_{source_type})
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
| Ref | Source Mode | Citation Type | Verdict | Severity | Reason |
|-----|-----------|--------------|---------|----------|--------|
```

### Phase 6: Unsupported Claims Detection

Identify quantitative claims (numerical data, specific facts) in the manuscript that lack citation markers. Exclude:
- Widely known facts in the field
- Author's own experience/expertise (phrases like "we have", "our group", etc.)
- Explicitly labeled proprietary/internal data

### Phase 7: Severity Escalation

If Phase 5 contains CRITICAL or SEVERE findings, escalate before the final report.

#### CRITICAL -- Immediate Verification Loop Stop

Output impact analysis and ask the user for a decision:

```
*** CRITICAL FINDING -- Verification Loop Stopped ***

Finding: [specific discovery]
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

If no CRITICAL findings, present SEVERE findings after all references are checked:
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
Pipeline: 3-Stage v2.1

## 1. Structural Integrity
[Phase 2 results]

## 2. Verification Summary

| Ref | Stage1 (DOI) | Stage2 (Field) | Stage3 (Semantic) | Verdict | Reason |
|-----|-------------|---------------|-------------------|---------|--------|

Source modes: local_pdf / publisher_html / supplementary / self_archived / preprint / abstract / title_only

## 3. CRITICAL/SEVERE Findings
[Phase 7 results, or "None"]

## 4. FAIL References (Details)
[Each FAIL with mismatch details and correction instructions]

## 5. REVIEW References (Details)
[Each REVIEW with citation type, source mode, recommended action]

## 6. Unsupported Claims
[Phase 6 results]

## 7. Overall Verdict

- CRITICAL: N
- SEVERE: N
- FAIL: N
- REVIEW: N
- PASS: N

### Author Checklist
[ ] [CRITICAL/SEVERE] Decide response and apply corrections
[ ] [FAIL] Correct metadata (mandatory)
[ ] [REVIEW] Verify and record judgment
[ ] [PASS] No additional action (brief visual check recommended)
```

### Phase 9: Iterative Verification Loop

If any CRITICAL, SEVERE, FAIL, or REVIEW findings exist, run the fix-verify loop:

1. **CRITICAL:** After user decides, re-verify affected references from Phase 3.
2. **SEVERE:** After user decides, re-verify from Phase 4.
3. **FAIL:** Propose corrections using WebSearch metadata. Apply after user approval, re-verify.
4. **REVIEW:** Present details to user:
   - "Keep citation (add justification to manuscript)"
   - "Replace citation (search for alternative)"
   - "Remove citation"
   - "Provide PDF" (for data_not_in_abstract) -- user places PDF in ./references/, re-verified as local_pdf next round
5. **Convergence:** Loop ends when:
   - All PASS (ideal)
   - FAIL=0 and all REVIEW decided (practical)
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

- **Internet required.** This command depends on WebSearch, WebFetch, Read, Bash, and Grep tools. If WebSearch is unavailable, Phases 3-5 are skipped; only structural checks (Phase 1-2) and unsupported claims (Phase 6) are performed.
- Uncertain results default to REVIEW with manual verification recommendation.
- This command outputs verification reports and proposes corrections. Corrections are applied only after author approval.
- SI PDF download uses Bash curl with User-Agent header.
- PDF reading uses the Read tool (max 20 pages per call). Large PDFs use the pages parameter.
