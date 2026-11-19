### Click for the [README](./README.md).


# DocDrift Release Notes

## Overview

**DocDrift** is an automated Continuous Integration (CI) check designed to catch documentation that becomes stale, incomplete, or factually wrong after a pull request modifies source code—before changes get merged.

## What's Included

### Core Drift Detector
- **AST-Based Code Diff Analyzer & Structured Documentation Parser:** Parses source code changes and structured documentation to map relationships between code and docs.
- **Hybrid Symbol/Semantic Matching & Drift Classifier:** Combines symbol-level and semantic matching to classify documentation drift accurately.
- **Actionable CI Flags:** Emits targeted `{file, line_range, reason}` flags during CI runs so authors and reviewers can address discrepancies prior to merge.

### Standalone Data & Evaluation Pipeline
DocDrift includes a standalone pipeline for building ground truth and benchmarking detector performance:
- `collect_ground_truth.py`: Mines `(code_commit, doc_commit)` candidate pairs from git history.
- `verify_pairs.py`: Supports manual triage and verification of mined candidate pairs.
- `evaluate.py`: Measures file/line recall, precision, and false-positive rates on unrelated pull requests.

## Verification & Known Limitations

- **End-to-End Verification:** Validated against real open-source history—including catching a parameter rename in `pallets/click`—as well as synthetic fixtures and unit tests.
- **Known Limitations:**
  - **Body-only logic changes:** Function or method body changes that do not alter signatures or documented interfaces may not be flagged as drift.
  - **Auto-mined pair noise:** Automatically mined `(code_commit, doc_commit)` pairs from git history can be noisy prior to manual human verification with `verify_pairs.py`.
