# Contributing to ailock

Thanks for wanting to make the AI/LLM supply chain safer. Here's how to help.

## Reporting Known-Bad Packages

The most impactful thing you can do is submit entries to the `known-bad.json` database.

**To report a compromised package:**

1. Fork this repo
2. Edit `ailock/data/known-bad.json`
3. Add your entry following this schema:

```json
{
  "package": "package-name",
  "version": "1.2.3",
  "hashes": ["sha256:..."],
  "severity": "CRITICAL|HIGH|MEDIUM|LOW",
  "cve": "CVE-2024-XXXXX",
  "description": "What happened and what it does",
  "attack_type": "post-release_replacement|typosquat|ci_cd_compromise|confusable_name",
  "reported_at": "YYYY-MM",
  "indicators": ["Observable behavior 1", "Observable behavior 2"],
  "references": ["https://..."],
  "remediation": "What to do if affected"
}
```

4. Open a PR with the title: `known-bad: add <package>==<version> (<attack_type>)`

**Severity guide:**

| Severity | When to use |
|----------|-------------|
| CRITICAL | Active exploitation, data exfiltration, remote code execution |
| HIGH     | Confirmed malicious, significant impact |
| MEDIUM   | Suspicious but unconfirmed, or limited impact |
| LOW      | Historical/theoretical, low impact |

**We'll merge your PR quickly** — catching these attacks before they spread is the goal.

## Developing

```bash
# Clone
git clone https://github.com/midnightrunai/ailock
cd ailock

# Set up
pip install -e ".[dev]"

# Run tests
pytest

# Run ailock itself
ailock --help
```

## Running the Tests

```bash
pytest tests/
pytest tests/ -v  # verbose
pytest tests/test_audit.py  # specific file
```

## Code Style

- We use `ruff` for linting and `black` for formatting
- Run `black .` and `ruff check .` before submitting PRs
- Type hints are appreciated but not required

## Adding AI/LLM Package Patterns

If you know of an AI/LLM package that ailock doesn't detect, add it to the `AI_PACKAGE_PATTERNS` in `ailock/core/constants.py`:

```python
AI_PACKAGE_PATTERNS = [
    # ...existing patterns...
    r"your-new-package",  # add your pattern here
]
```

## License

By contributing, you agree your contributions are MIT licensed.
