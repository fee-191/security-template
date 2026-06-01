# Hướng dẫn phát triển template

> Dành cho người maintain và đóng góp cho template này. Bao gồm cấu trúc project, cách thêm rule mới, chạy test, và quy trình release.

## Dự án là gì

Security template cho team phát triển sàn giao dịch crypto (CEX). Tích hợp vào project dưới dạng git submodule, cung cấp 4 lớp bảo vệ tự động từ pre-commit đến CI/CD production.

**GitHub:** `https://github.com/fee-191/security-template`

---

## Trạng thái hiện tại — v1.2.1

| Thành phần | Trạng thái |
|---|---|
| Semgrep rules | Custom rules, 5 ngôn ngữ |
| Test suite | 80/80 PASSED |
| Languages | Python · JS/TS · Kotlin · Swift · Java |

---

## Cấu trúc repo

```
security-template/
├── .semgrep/rules/security.yml   # Custom Semgrep rules — core của template
├── scripts/
│   ├── test-ci-local.sh          # Chạy: bash scripts/test-ci-local.sh
│   └── setup-hooks.sh            # Setup cho project consumer mới
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
│   ├── guide.md                  # Hướng dẫn đầy đủ cho developer
│   ├── deck.html                 # Presentation (mở trực tiếp trên browser)
│   └── secure-checklist.md      # Checklist 15 mục trước MR
├── .gitlab-ci.yml                # GitLab CI pipeline
├── .github/workflows/security.yml  # GitHub Actions equivalent
├── CLAUDE.md                     # Rules binding cho AI coding assistant
└── CHANGELOG.md                  # Lịch sử phiên bản
```

---

## Kiến thức kỹ thuật quan trọng

### 1. Semgrep dùng `re.fullmatch`, không phải `re.search`

`metavariable-regex` apply `re.fullmatch` lên toàn bộ string của metavariable.

```yaml
# SAI — chỉ match khi keyword là suffix
regex: '(?i)(?:^|_)(secret|token)$'

# ĐÚNG — fullmatch-compatible, match bất kể prefix/suffix
regex: '(?i)(?:^|.*_)(secret|token)(?:_.*|$)'
```

Lỗi này không hiển thị error — rule chỉ âm thầm không fire. Verify bằng `python3 -c "import re; print(re.fullmatch(pattern, string))"`.

### 2. Test infrastructure — 3 loại hàm

```bash
expect_blocks    "$FILE" "rule-id" "desc"  # Rule PHẢI fire trong ERROR scan (CRITICAL)
expect_warns_only "$FILE" "rule-id" "desc" # Rule KHÔNG block ERROR, chỉ fire WARNING scan (HIGH)
expect_clean     "$FILE" "desc"            # 0 findings — kiểm tra false positive
```

`expect_warns_only` = 2 passes (1 ERROR scan + 1 WARNING scan).

### 3. Severity mapping

- `severity: ERROR` → CRITICAL → block commit và MR/PR
- `severity: WARNING` → HIGH → warn only

---

## Thêm rule mới

1. Viết pattern vào `.semgrep/rules/security.yml`
2. Validate: `semgrep --validate --config=.semgrep/rules/security.yml`
3. Thêm fixture vào `tests/ci/vuln_critical_*.` hoặc `vuln_high_*.*`
4. Thêm `expect_blocks` hoặc `expect_warns_only` vào `scripts/test-ci-local.sh`
5. Nếu `severity: WARNING`: thêm rule ID vào `should_be_warning` set trong TEST 6
6. Chạy `bash scripts/test-ci-local.sh` — phải pass 100%
7. Update CHANGELOG + VERSION + header comment trong `security.yml`

---

## Chạy test

```bash
pip install semgrep

# Full test suite — phải 100% trước khi push
bash scripts/test-ci-local.sh
```

---

## Setup local

```bash
git clone https://github.com/fee-191/security-template.git
cd security-template

pip install semgrep pre-commit
bash scripts/test-ci-local.sh
```

---

## Release checklist

- [ ] Test pass: `bash scripts/test-ci-local.sh`
- [ ] Cập nhật `VERSION`
- [ ] Thêm entry vào `CHANGELOG.md`
- [ ] Cập nhật header comment trong `security.yml`
- [ ] Tag: `git tag -a v1.x.x -m "v1.x.x"` → `git push origin --tags`

---

## Versioning

- **MINOR** — thêm rules mới hoặc thêm ngôn ngữ mới
- **PATCH** — bug fix, docs, cải tiến test
