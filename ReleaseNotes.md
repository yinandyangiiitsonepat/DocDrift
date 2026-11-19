DocDrift is an automated Continuous Integration check designed to catch documentation that becomes stale, incomplete, or factually wrong after a pull request modifies source code—before changes get merged.

It pairs an AST-based code diff analyzer and structured documentation parser with hybrid symbol/semantic matching and a drift classifier to emit targeted {file, line_range, reason} flags in CI. Alongside the detector, DocDrift includes a standalone data and evaluation pipeline (collect_ground_truth.py, verify_pairs.py, and evaluate.py) to mine (code_commit, doc_commit) pairs from git history, triage candidates manually, and measure file/line recall, precision, and false-positive rates on unrelated PRs.

Verified end-to-end against real open-source history (including a pallets/click parameter rename catch) as well as synthetic fixtures and unit tests, while documenting upfront limitations such as body-only logic changes and noisy auto-mined pairs prior to human verification.
