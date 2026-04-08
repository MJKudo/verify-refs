# /verify-refs

A Claude Code command that verifies the consistency between references and manuscript claims using a 3-stage verification pipeline.

## The Problem

When AI assists in writing research manuscripts, references often contain "plausible inaccuracies" -- DOIs that exist, papers that are real, but manuscript claims that subtly diverge from the actual paper content. In our initial test, **3 out of 6 references had critical errors** (wrong title, DOI pointing to a completely different paper, slot-die paper cited as inkjet).

## How It Works

```
Stage 1: DOI Metadata Verification (WebSearch)
  Does the DOI actually point to the paper described in the manuscript?
  -> Catches title mismatches, author errors, wrong papers

Stage 2: Field-Specific Classification (keyword rules)
  Does the paper's methodology match the citation context?
  -> Catches method confusion (e.g., slot-die cited as inkjet)

Stage 3: Semantic Alignment (AI judgment + full-text retrieval)
  Does the manuscript's claim accurately reflect the paper's content?
  -> Catches numerical errors, misrepresented findings, severity assessment
```

Each reference gets a verdict: **PASS**, **REVIEW**, or **FAIL**.

The pipeline runs iteratively (Verify-Fix-Verify loop) until all references pass or the author has reviewed all flagged items.

## Install

```bash
# Download the command file
curl -sL https://raw.githubusercontent.com/YOUR_USERNAME/verify-refs/main/verify-refs.md \
  -o ~/.claude/commands/verify-refs.md
```

Or manually copy `verify-refs.md` to your `~/.claude/commands/` directory (global) or `your-project/.claude/commands/` (project-specific).

## Quick Start

1. **Copy the file:**
   ```bash
   cp verify-refs.md ~/.claude/commands/
   ```

2. **Edit field-specific keywords** (optional but recommended):
   Open `~/.claude/commands/verify-refs.md` and edit the "User Configuration: Field-Specific Keywords" section near the top. Define keyword groups for your field:
   ```
   Group A (your method): "CRISPR", "gene editing", "Cas9"
   Group B (confused with): "TALENs", "zinc finger"
   Group C (confused with): "base editing", "prime editing"
   ```
   If you skip this, Stage 2 is simply bypassed.

3. **Run in Claude Code:**
   ```
   /verify-refs manuscript.md
   ```

## What You Get

A structured verification report showing each reference's status across all 3 stages:

```
| Ref | Stage1 (DOI) | Stage2 (Field) | Stage3 (Semantic) | Verdict | Reason |
|-----|-------------|---------------|-------------------|---------|--------|
| [1] | PASS        | PASS          | PASS              | PASS    |        |
| [2] | PASS        | PASS          | REVIEW            | REVIEW  | data_mismatch: manuscript says 18.54%, paper says 17.55% |
| [3] | PASS        | REVIEW        | ---               | REVIEW  | method_mismatch: slot-die paper in inkjet section |
```

Plus:
- FAIL references with specific correction instructions
- REVIEW references with recommended actions
- Unsupported claims detection (numbers without citations)
- Iterative fix-verify loop until convergence

## Full-Text Retrieval

Stage 3 attempts to retrieve paper full text for more accurate verification, using a cascading fallback:

1. **Local PDF** (`./references/` or `./refs/` directory) -- highest reliability
2. **Publisher OA HTML** (via DOI redirect)
3. **Supplementary Information** (often freely available even for paywalled papers)
4. **Author self-archive** (ResearchGate, arXiv, institutional repositories)
5. **Abstract only** (from WebSearch results)
6. **Title only** (lowest precision, all uncertain verdicts become REVIEW)

To improve verification accuracy, place PDFs of your references in a `./references/` directory.

## Severity Assessment

When Stage 3 finds an inaccuracy, it assesses the severity:

| Severity | Meaning | Action |
|----------|---------|--------|
| **CRITICAL** | Core manuscript claim is invalidated | Verification loop stops immediately. Author must decide: rewrite, find alternative, or adjust tone |
| **SEVERE** | Section claim significantly weakened | Held for batch review after all references are checked |
| **MODERATE** | Individual claim needs correction | Standard REVIEW flow |
| **MINOR** | Formatting/metadata issue | Standard FAIL auto-correction flow |

## Customization

### Citation Format

The default format is IEEE-style `[N]`. If your manuscript uses a different format, edit the "User Configuration: Citation Format" section at the top of the file.

### Field Keywords

The "User Configuration: Field-Specific Keywords" section controls Stage 2 methodology classification. Examples for different fields:

**Biology (gene editing):**
```
Group A: "CRISPR", "Cas9", "guide RNA", "sgRNA"
Group B: "TALENs", "TALEN"
Group C: "zinc finger nuclease", "ZFN"
Group D: "base editing", "prime editing"
```

**Chemistry (synthesis):**
```
Group A: "electrochemical", "electrosynthesis"
Group B: "thermal", "pyrolysis"
Group C: "photochemical", "photocatalytic"
Group D: "mechanochemical", "ball milling"
```

**Machine Learning:**
```
Group A: "transformer", "attention mechanism"
Group B: "recurrent", "LSTM", "GRU"
Group C: "convolutional", "CNN"
Group D: "diffusion model", "denoising"
```

## Supported Formats

| Format | Manuscript | References | Citation Markers |
|--------|-----------|------------|-----------------|
| **Markdown** | `.md` | `[N] Author, "Title," ...` | `[N]` or `^[N]^` |
| **LaTeX** | `.tex` | `.bib` file or `\bibitem{}` | `\cite{}`, `\citep{}`, `\citet{}`, etc. |

LaTeX multi-file manuscripts (`\input{}`, `\include{}`) are supported. The command auto-detects the format from the file extension.

## How This Compares

| | verify-refs | academic-research-skills (2.2k stars) | SemanticCite (research) | CiteTrue/Citely (SaaS) |
|---|---|---|---|---|
| Focus | Reference verification only | Full research workflow (13 agents) | Citation semantic analysis | Citation checking |
| Setup | `cp` 1 file | 70+ files, 4 modules | Python + model download | Browser |
| Method confusion detection | Yes (Stage 2) | No | No | No |
| Full-text retrieval | 6-level fallback | Unknown | PDF-only | API-based |
| Fix-verify loop | Yes (up to 3 rounds) | Yes (2-stage review) | No | No |
| Severity escalation | CRITICAL/SEVERE/MODERATE/MINOR | No | 4-class (Supported/Partial/Unsupported/Uncertain) | Pass/Fail |
| License | MIT | CC BY-NC 4.0 | Research | Paid SaaS |
| Cost | Free (needs Claude Code) | Free (non-commercial) | Free (research) | Paid |

**verify-refs is the lightweight, single-file alternative** focused purely on reference-citation consistency. If you need a full academic writing pipeline, check out [academic-research-skills](https://github.com/Imbad0202/academic-research-skills).

## Requirements

- [Claude Code](https://claude.ai/code) (CLI, desktop app, or IDE extension)
- Internet connection (for WebSearch/WebFetch in Stage 1-3)
- Manuscript in Markdown (`.md`) or LaTeX (`.tex`) format

## License

MIT
