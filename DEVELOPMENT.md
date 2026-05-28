# Development Guide

> For contributors and maintainers of this template. Covers project structure, how to add rules, run tests, and release new versions.

## What This Project Is

Security template for crypto exchange (CEX) development teams. Integrates via git submodule into any project and provides 4 automated security layers — from pre-commit to production CI/CD.

**GitHub:** `https://github.com/fee-191/security-template`

---

## Current State — v1.2.1

| Component | Status |
|---|---|
| Semgrep rules | 44 rules, 5 languages |
| Test suite | **80/80 PASSED** |
| Languages | Python · JS/TS · Kotlin · Swift · Java |

---

## Repo Structure

```
security-template/
├── .semgrep/rules/security.yml   # 44 rules — the core of this project
├── scripts/
│   ├── test-ci-local.sh          # Run: bash scripts/test-ci-local.sh
│   └── setup-hooks.sh            # Setup for a new consumer project
├── tests/ci/
│   ├── vuln_critical.py          # Python CRITICAL fixtures
│   ├── vuln_high.py              # Python HIGH fixtures
│   ├── vuln_critical_js.ts       # JS/TS CRITICAL fixtures
│   ├── vuln_high_js.ts           # JS/TS HIGH fixtures
│   ├── vuln_critical_android.kt  # Kotlin CRITICAL fixtures
│   ├── vuln_high_android.kt      # Kotlin HIGH fixtures
│   ├── vuln_critical_ios.swift   # Swift CRITICAL fixtures
│   ├── vuln_high_ios.swift       # Swift HIGH fixtures
│   └── test_safe.py              # False positive check
├── docs/
│   ├── guide.md                  # Full developer guide
│   ├── deck.html                 # Presentation (open in browser)
│   └── secure-checklist.md      # Pre-MR checklist
├── .gitlab-ci.yml                # GitLab CI pipeline
├── .github/workflows/security.yml  # GitHub Actions equivalent
├── CLAUDE.md                     # AI assistant rules binding
└── CHANGELOG.md                  # Version history
```

---

## Critical Technical Notes

### 1. Semgrep uses `re.fullmatch`, not `re.search`

`metavariable-regex` applies `re.fullmatch` to the entire metavariable string.

```yaml
# WRONG — only matches if keyword is a suffix
regex: '(?i)(?:^|_)(secret|token)$'

# CORRECT — fullmatch-compatible, matches anywhere in string
regex: '(?i)(?:^|.*_)(secret|token)(?:_.*|$)'
```

### 2. Test infrastructure — 3 function types

```bash
expect_blocks    "$FILE" "rule-id" "desc"  # Rule MUST fire in ERROR scan (CRITICAL)
expect_warns_only "$FILE" "rule-id" "desc" # Rule MUST NOT block, only fire in WARNING scan (HIGH)
expect_clean     "$FILE" "desc"            # 0 findings — false positive check
```

`expect_warns_only` = 2 passes (1 ERROR scan + 1 WARNING scan).

### 3. Severity mapping

- `severity: ERROR` → CRITICAL → blocks commit and MR/PR
- `severity: WARNING` → HIGH → warns only

---

## Adding a New Rule

1. Write the pattern in `.semgrep/rules/security.yml`
2. Validate: `semgrep --validate --config=.semgrep/rules/security.yml`
3. Add fixture to `tests/ci/vuln_critical_*.` or `vuln_high_*.*`
4. Add `expect_blocks` or `expect_warns_only` to `scripts/test-ci-local.sh`
5. If `severity: WARNING`: add rule ID to `should_be_warning` set in TEST 6
6. Run `bash scripts/test-ci-local.sh` — must pass 100%
7. Update CHANGELOG + VERSION + header comment in `security.yml`

---

## Running Tests

```bash
# Install dependencies
pip install semgrep

# Run full test suite (must be 80/80 PASS before any push)
bash scripts/test-ci-local.sh
```

---

## Local Dev Setup

```bash
git clone https://github.com/fee-191/security-template.git
cd security-template

pip install semgrep pre-commit
bash scripts/test-ci-local.sh   # verify 80/80 PASS
```

---

## Release Checklist

- [ ] All tests pass: `bash scripts/test-ci-local.sh`
- [ ] `VERSION` file updated
- [ ] `CHANGELOG.md` entry added with date and changes
- [ ] Header comment in `security.yml` updated
- [ ] Git tag created: `git tag -a v1.x.x -m "v1.x.x"`
- [ ] Push tag: `git push origin --tags`

---

## Versioning Convention

- **MINOR** — new rules or new language coverage
- **PATCH** — bug fixes, doc updates, test improvements
