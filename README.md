# ailock 🔐

> **Cryptographic lockfile for AI/LLM Python dependencies.**
> Like `package-lock.json`, but for `litellm`, `langchain`, `openai`, `transformers`, and friends.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![CI](https://github.com/midnightrunai/ailock/actions/workflows/ailock-ci.yml/badge.svg)](https://github.com/midnightrunai/ailock/actions/workflows/ailock-ci.yml)
[![Security](https://img.shields.io/badge/security-supply%20chain-red)](https://midnightrun.ai/ailock)
[![Known-Bad DB](https://img.shields.io/badge/known--bad%20DB-6%20entries-orange)](ailock/data/known-bad.json)
[![PyPI](https://img.shields.io/pypi/v/ailock)](https://pypi.org/project/ailock/)

---

## 🚨 Active Threat: TeamPCP Campaign (March 2026)

The supply chain campaign is active RIGHT NOW. Three ecosystems compromised in 8 days:

| Date | Package | Versions | CVE | SHA256 (malicious wheel) |
|------|---------|---------|-----|--------------------------|
| Mar 19 | trivy-action (GitHub) | 75 of 77 tags | CVE-2026-33634 | — |
| Mar 24 | **litellm** | **1.82.7, 1.82.8** | CVE-2026-33634 | `8395c326...` / `d2a0d5f5...` |
| Mar 27 | **telnyx** | **4.87.1, 4.87.2** | CVE-2026-33634 | `7321caa3...` / `cd081158...` |

`ailock audit` has all four malicious package hashes pre-loaded. Run it now.

---

## The Attack

On **March 24, 2026 at 10:39 UTC**, two versions of LiteLLM were published to PyPI by TeamPCP after they compromised maintainer credentials via a poisoned Trivy CI scanner.

**litellm 1.82.7** — malicious payload embedded in `proxy_server.py`. Triggers on import. Three-stage attack: credential harvest (SSH keys, .env files, AWS tokens, Kubernetes secrets) → lateral movement → persistent systemd backdoor. Exfiltrates to `models.litellm.cloud` as `tpcp.tar.gz`.

**litellm 1.82.8** — even worse: adds `litellm_init.pth` to `site-packages`. This fires on **every Python startup** — including `pip`, IDE language servers, anything. No import required.

On **March 27**, the stolen credentials from LiteLLM victims were used to compromise **telnyx** (4.87.1, 4.87.2). WAV steganography delivers platform-specific payloads from `83.142.209.203:8080`.

The version number matched. The package name matched. Standard `pip install --require-hashes` passed — because the attacker used **legitimate credentials** to publish malicious bytes. Only a pre-attack lockfile would have shown the hash change.

**`ailock verify` would have caught it in CI, before it hit prod.**

---

## Install

```bash
pip install ailock
```

## Quick Start

```bash
# 1. Generate a lockfile for your AI/LLM dependencies
ailock generate

# 2. Commit it
git add .ailock && git commit -m "chore: add ailock lockfile"

# 3. Verify in CI — fails loudly if ANY hash changed
ailock verify

# 4. Check against known-bad versions database
ailock audit
```

---

## How It Works

### `ailock generate`

Scans your `requirements.txt` / `pyproject.toml`, identifies AI/LLM ecosystem packages, fetches the SHA256 hashes from PyPI's JSON API for each pinned version, and writes a `.ailock` lockfile:

```json
{
  "schema_version": "1",
  "generated_at": "2024-11-15T09:41:22Z",
  "packages": {
    "litellm": {
      "version": "1.82.6",
      "hashes": [
        "sha256:a7c3f8e2b1d4..."
      ],
      "source": "pypi"
    },
    "langchain-core": {
      "version": "0.3.1",
      "hashes": [
        "sha256:f4e9a1b5c2d3..."
      ],
      "source": "pypi"
    }
  }
}
```

### `ailock verify`

Re-fetches hashes from PyPI and compares. If a package's hash changed since you ran `generate`, that package was **modified after it was released**.

```
✓  All 12 packages verified — integrity intact
```

```
🚨 SUPPLY CHAIN ATTACK DETECTED

1 package has been modified since your lockfile was generated.
DO NOT deploy. Investigate immediately.

┌─────────┬─────────┬────────────────────────────────────────────────────┐
│ Package │ Version │ Issue                                              │
├─────────┼─────────┼────────────────────────────────────────────────────┤
│ litellm │ 1.82.7  │ 2 new hashes added after lockfile was generated   │
└─────────┴─────────┴────────────────────────────────────────────────────┘
```

### `ailock audit`

Cross-checks your locked packages against the community `known-bad.json` database. Ships with the current TeamPCP campaign entries plus historical attacks:

| Package | Version | Severity | CVE | Attack |
|---------|---------|----------|-----|--------|
| litellm | 1.82.7  | CRITICAL | CVE-2026-33634 | Credential harvester + Kubernetes lateral movement |
| litellm | 1.82.8  | CRITICAL | CVE-2026-33634 | .pth persistence — fires without any import |
| telnyx  | 4.87.1  | CRITICAL | CVE-2026-33634 | WAV steganography payload delivery |
| telnyx  | 4.87.2  | CRITICAL | CVE-2026-33634 | WAV steganography payload delivery |
| ultralytics | 8.3.41 | HIGH | — | CI/CD pipeline cryptominer |

Real SHA256 hashes for all malicious wheels are included. The database is fetched fresh from GitHub on each run. Community PRs welcome — [known-bad.json](ailock/data/known-bad.json).

---

## GitHub Actions — One-Liner

```yaml
- name: Verify AI/LLM dependency integrity
  run: pip install ailock && ailock verify && ailock audit
```

Full workflow:

```yaml
name: Supply Chain Security

on: [push, pull_request]

jobs:
  ailock:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Install ailock
        run: pip install ailock

      - name: Generate lockfile (first run)
        # Only needed if .ailock doesn't exist yet
        run: |
          if [ ! -f .ailock ]; then
            ailock generate
            echo "Generated .ailock — commit this file to your repo"
            exit 1
          fi

      - name: Verify package integrity
        run: ailock verify

      - name: Audit against known-bad database
        run: ailock audit
```

---

## What ailock protects against

### ✅ Post-release replacement attacks (the LiteLLM scenario)

Package published clean → attacker gains PyPI access → replaces the wheel with malicious code under the same version number.

`ailock verify` catches this because the SHA256 changes.

### ✅ Known compromised versions

`ailock audit` cross-checks against a curated database of confirmed malicious versions. Catches packages you may have locked before the attack was public.

### ✅ Typosquats with pinned versions

If you accidentally pinned `langchian==0.1.0` instead of `langchain`, the lockfile records it — and `ailock audit` flags known typosquats.

### ❌ What it doesn't protect against

- Packages you haven't locked (unpin them at your peril)
- Brand new attacks not yet in the known-bad database
- Compromised packages where the attacker maintains the same hash (very hard but possible via pre-image attacks — SHA256 makes this computationally infeasible today)

---

## Supported Package Ecosystem

ailock automatically recognises 100+ AI/LLM packages including:

**LLM SDKs:** litellm, openai, anthropic, cohere, mistralai, groq, replicate, together
**LangChain:** langchain, langchain-core, langchain-community, langgraph, langsmith
**LlamaIndex:** llama-index, llama-index-core, llama-index-llms-*
**HuggingFace:** transformers, huggingface-hub, tokenizers, accelerate, peft, datasets
**Vector Stores:** chromadb, pinecone-client, weaviate-client, qdrant-client
**Frameworks:** haystack-ai, semantic-kernel, autogen, crewai, dspy
**Utilities:** tiktoken, sentence-transformers, instructor, outlines, guidance

Use `--all-packages` to lock your entire dependency tree.

---

## CLI Reference

```
Usage: ailock [OPTIONS] COMMAND [ARGS]...

  ailock — cryptographic lockfile for AI/LLM dependencies.

Commands:
  generate  Scan AI/LLM dependencies and write a cryptographic lockfile.
  verify    Verify installed packages against .ailock cryptographic hashes.
  audit     Cross-check packages against known-bad versions database.
```

### `ailock generate`

```
Options:
  -o, --output PATH     Output lockfile path.  [default: .ailock]
  -a, --all-packages    Include ALL packages (not just AI/LLM).
  -i, --from-installed  Use currently installed package versions.
  -f, --file PATH       Specific requirements file(s) to scan.
  --no-hashes           Skip PyPI hash fetching (faster but less secure).
```

### `ailock verify`

```
Options:
  -l, --lockfile PATH   Path to .ailock lockfile.  [default: .ailock]
  --fail-on-missing     Exit with error if any package not on PyPI.
  --json-output         Output results as JSON (for CI integration).
```

### `ailock audit`

```
Options:
  -l, --lockfile PATH   Path to .ailock lockfile.  [default: .ailock]
  --offline             Use only bundled database (no network call).
  --db-url TEXT         URL to known-bad JSON database.
  --json-output         Output results as JSON (for CI integration).
  --show-db             Print the full known-bad database and exit.
```

---

## Contributing to the Known-Bad Database

The `ailock/data/known-bad.json` file is community-maintained at **[github.com/midnightrunai/ailock](https://github.com/midnightrunai/ailock)**.

If you discover a compromised package version:

1. Fork [github.com/midnightrunai/ailock](https://github.com/midnightrunai/ailock)
2. Add an entry to `ailock/data/known-bad.json`
3. Include: package name, version, SHA256 hashes (if known), description, references
4. Open a PR

**Format:**

```json
{
  "package": "some-ai-package",
  "version": "1.2.3",
  "hashes": ["sha256:..."],
  "severity": "CRITICAL",
  "cve": "CVE-2024-XXXXX",
  "description": "What happened and why it's bad.",
  "reported_at": "2024-11",
  "references": ["https://..."]
}
```

---

## Why not just use pip's built-in hash checking?

`pip install --hash` works but requires you to:
1. Manually specify hashes in requirements.txt
2. Include hashes for **every** package (including transitive deps)
3. Re-generate everything on every update

ailock focuses specifically on the AI/LLM ecosystem, adds a curated known-bad database, and makes the workflow CI-native.

---

## License

MIT — see [LICENSE](LICENSE)

---

Built by [Midnight Run](https://midnightrun.ai) — the AI that builds while you sleep.
