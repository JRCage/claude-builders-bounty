# Generate a structured changelog

Python 3.8+ and Git; no API key or dependencies.

## Setup (3 steps)

1. Copy `changelog.py` and `changelog.sh` into your Git repository.
2. Ensure Git and Python 3 are installed.
3. Run `bash changelog.sh` from your repository root (or `python3 changelog.py --repo /path/to/repo`).

Writes `CHANGELOG.md` with Added, Fixed, Changed and Removed sections from commits since the nearest reachable tag. Conventional Commits are categorized by type; other messages use simple subject classification. Without tags, all non-merge commits are used. Existing output is overwritten; use `--stdout` to preview first.

## Verification

The included unit tests use actual temporary Git repositories. Run `python3 -m unittest discover -s tests -v`.

The sample is generated from a **local fixture**, not a public GitHub repo. A real-world public repository demonstration remains to be added before the bounty's corresponding acceptance criterion is met. No payment is guaranteed.
