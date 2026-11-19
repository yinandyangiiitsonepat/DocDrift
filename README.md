[Philosophy](#-core-philosophy) • [Verified Status](#-whats-verified-vs-unverified) • [Quick Start](#-quick-start) • [Architecture](#-architecture) • [Limitations](#-known-limitations) • [Team](#-team-yin-and-yang)

## ☯ Core Philosophy

> \[!NOTE\]
> **Source code and documentation are two interdependent halves of every software system.**
>
> Active repositories merge code changes far faster than contributors update READMEs, API guides, and configuration tables. In GitHub's 2024 Open Source Survey, **86.7%** of respondents encountered incomplete or confusing documentation. Traditional CI pipelines fail on broken tests or syntax errors, yet silently pass when documentation becomes factually wrong. **DocDrift** flags documentation sentences invalidated by pull request code changes before they merge.

```
# Example: What DocDrift Catches on a Real PR (pallets/click @ 9f0bcdf)
- Parameter.human_readable_name
+ Parameter.spec

! [DRIFT FLAGGED] docs/upgrade-guides.md
! Sentence describing `Parameter.human_readable_name` was invalidated by this rename.

```

## ✅ What's Verified vs. Unverified

Everything listed below was **actually executed**, not just written:

| Component | Status | What Was Executed & Observed | Artifacts / Entrypoints | 
| ----- | ----- | ----- | ----- | 
| **Real End-to-End Run** |  | Ran against real Click commit `9f0bcdf` (`Parameter.human_readable_name` renamed to `Parameter.spec`). Correctly flagged the exact real doc sentence in `docs/upgrade-guides.md` describing the deprecation. | `run_real_pipeline.py`  `real_pipeline_report.json`  `example_real_click_diff.diff`  `example_real_click_doc.md` | 
| **Synthetic Fixtures** |  | Runs **4 synthetic cases** (exact match, fuzzy match, semantic-embedding fallback, and a safe refactor that must **NOT** be flagged) through `matcher.py` → `drift_classifier.py` → `cli.py`. All 4 produce the expected verdict. | `matching/demo_run.py` | 
| **Parser Unit Tests** |  | **6 synthetic tests** passing + **1 real-data test** (skipped by default unless environment variables are set — see docstring). | `parsing/test_parsers.py` | 
| **Ground-Truth Auto-Mining** |  | Mines `(code_commit, doc_commit)` pairs from git history via temporal proximity + shared-identifier overlap. Manually checked against `pallets/click`: produced mostly or entirely `FALSE` pairs even after tightening identifier filtering (`ground_truth.json` vs. `ground_truth_v2.json`). | `data/collect_ground_truth.py`  `data/verify_pairs.py` | 

> \[!WARNING\]
> **Known Ground-Truth Caveat:** Do **not** present auto-mined pairs from `collect_ground_truth.py` as verified real drift examples without manually checking each one with `data/verify_pairs.py` first.

## ⚡ Quick Start

```
# 1. Install dependencies (sentence-transformers is optional; falls back to n-gram cosine)
pip install -r requirements.txt

# 2. Run the real, verified Click commit example
python3 run_real_pipeline.py example_real_click_diff.diff example_real_click_doc.md --mock

# 3. Run the synthetic demo (all 4 fixture cases)
cd matching && python3 demo_run.py

# 4. Run parser unit tests
cd parsing && python3 test_parsers.py -v

# 5. Mine ground truth from a real repo (expect noise — verify before trusting)
python3 -m data.collect_ground_truth https://github.com/pallets/click \
  --max-commits 800 --emit-negatives 10 -o data/ground_truth.json
python3 -m data.verify_pairs -g data/ground_truth.json --export-md data/review.md

```

## 🏗 Architecture

### End-to-End Pipeline

```
flowchart LR
    A["🔍 Git Diff / PR<br/>(code_commit_sha)"] --> B["🧩 Code & Doc Parsers<br/>(AST + Markdown/reST)"]
    B --> C["🔗 Matcher<br/>(Exact / Fuzzy / Embeddings)"]
    C --> D["⚖️ Drift Classifier<br/>(Outdated / Correct / Uncertain)"]
    D --> E["🚨 Targeted CI Flags<br/>{file, line_range, reason}"]

    G["⛏️ Ground-Truth Miner<br/>collect_ground_truth.py"] --> H["👁️ Human Triage<br/>verify_pairs.py"]
    H --> I["📊 Benchmark Dataset<br/>ground_truth_verified.json"]
    I -.-> E

    style A fill:#14151c,stroke:#001aff,stroke-width:2px,color:#ffffff
    style B fill:#ffffff,stroke:#14151c,stroke-width:2px,color:#14151c
    style C fill:#ffffff,stroke:#14151c,stroke-width:2px,color:#14151c
    style D fill:#e6e9ff,stroke:#001aff,stroke-width:2px,color:#001aff
    style E fill:#001aff,stroke:#14151c,stroke-width:2px,color:#ffffff
    style G fill:#1e202c,stroke:#5c70ff,stroke-width:2px,color:#dbe1ff
    style H fill:#1e202c,stroke:#5c70ff,stroke-width:2px,color:#dbe1ff
    style I fill:#1e202c,stroke:#5c70ff,stroke-width:2px,color:#ffffff

```

### Repository Structure

```
docdrift/
├── data/
│   ├── collect_ground_truth.py   # mines (code_commit, doc_commit) pairs from git history
│   └── verify_pairs.py           # human-in-the-loop verification (REAL / FALSE triage)
├── parsing/
│   ├── code_diff_parser.py       # AST-based: diff -> symbol-change events
│   ├── doc_parser.py             # Markdown/reST -> doc units (sentence / table-row / list-item)
│   └── test_parsers.py           # unit & real-data parser tests
├── matching/
│   ├── matcher.py                # exact/fuzzy symbol match + semantic embedding fallback
│   ├── drift_classifier.py       # still correct / likely outdated / uncertain + reason
│   ├── cli.py                    # `docdrift check` - own fallback parsers + full CLI/CI flow
│   └── demo_run.py               # synthetic 4-case demo
└── run_real_pipeline.py          # wires real parsers directly into matcher + classifier

```

### Two Ways to Run the Pipeline

| Mode | Entrypoint | How It Works | Best For | 
| ----- | ----- | ----- | ----- | 
|  **CLI Fallback Mode** | `matching/cli.py check` | Self-contained; uses its own regex-based fallback parsers for diffs and docs. | Fast CI environments where you want zero extra dependencies. | 
|  **Full AST Pipeline** | `run_real_pipeline.py` | Uses the real AST-based `code_diff_parser.py` and structured `doc_parser.py` for precise symbol & line extraction. | Higher accuracy on Python-specific changes (renames, default values, removed symbols, CLI flag changes). | 

## ⚠️ Known Limitations

> \[!CAUTION\]
> **Upfront Engineering Transparency** — These are documented boundaries of the current implementation:

1. **Diff-only ("fragment") mode is less precise than full git mode.** Without access to the full before/after file (`sources_from_git`), the parser infers the enclosing function/class from the diff hunk header alone, which can misattribute a changed line to the wrong symbol in edge cases.

2. **Body-only logic changes produce zero events.** `code_diff_parser.py` only tracks signatures, defaults, removed symbols, and CLI flags — not changes inside function bodies. This is intentional to avoid false positives, but means some behavioral drift won't be caught.

3. **Auto-mined ground truth is noisy.** Temporal proximity + shared-token overlap produces false pairs even with strict identifier filtering (`CamelCase` / `snake_case` / `ALL_CAPS` / `--flags` only). Always manually verify with `verify_pairs.py` before using a mined pair as a demo claim.

4. **Positional CLI arguments are not tracked.** `click.argument` and positional `argparse` args are currently excluded — only `click.option` and `add_argument` flag changes are tracked.

## 👥 Team Yin and Yang

| Member | Role | Roll No. | Branch | 
| ----- | ----- | ----- | ----- | 
| **Manraj Singh** | IT | `12612031` | `IT` | 
| **Mayank Prajapati** | CSE | `12611069` | `CSE` | 
| **Ankit Bhichhar** | IT | `12612009` | `IT` | 




