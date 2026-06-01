# CEX Security Template

[![Security Scan](https://github.com/fee-191/security-template/actions/workflows/security.yml/badge.svg)](https://github.com/fee-191/security-template/actions/workflows/security.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> **Security tooling for a Vietnamese crypto exchange** — custom Semgrep rules, CI/CD pipeline, and Claude Code AI skills to enforce security standards before code reaches production.
>
> Built for a real production environment handling user funds, subject to Vietnamese fintech regulations (NĐ 356/2025, ATTT Cấp 4). Rule set informed by post-mortems of Mixin Network ($200M, 2023), Upbit ($50M, 2019), and Bybit ($1.5B, 2025).

---

## Bối cảnh — Tại sao dự án này tồn tại

Crypto exchange là môi trường có rủi ro cao nhất trong phần mềm: mọi lỗi logic trong balance, withdrawal, hay authentication đều có thể dẫn đến mất tài sản người dùng trực tiếp và không thể đảo ngược.

| Incident | Năm | Thiệt hại | Root cause |
|---|---|---|---|
| **Bybit** | 2025 | $1.5B | Supply chain — thư viện JS bị compromised qua phishing |
| **Mixin Network** | 2023 | $200M | Cloud DB compromise + JS code trên S3 bị thay |
| **Upbit** | 2019 | $50M | Hot wallet drain |

Template này phát hiện các pattern dẫn đến những incident trên — hardcoded secrets, eval injection, static nonce, float cho money, balance read without row lock — ngay tại thời điểm developer commit, không phải sau khi lên production.

**Compliance:** Sàn crypto tại Việt Nam (Nghị quyết 05/2025/NQ-CP) phải tuân thủ ATTT Cấp độ 4 và NĐ 356/2025 về bảo vệ dữ liệu cá nhân. Template tích hợp các yêu cầu này trực tiếp vào coding workflow.

---

## Highlights

| | |
|---|---|
| Semgrep rules | Custom rules cho CEX · 5 ngôn ngữ (Python, JS/TS, Kotlin, Swift, Java) |
| Test suite | 80 tests · 3 loại: block / warn-only / false-positive |
| CI/CD | **GitLab CI** (5 jobs) + **GitHub Actions** |
| AI integration | **Claude Code** skills, subagents, slash commands |
| Compliance | NĐ 356/2025 (PII), ATTT Cấp độ 4, Luật PCRT 2022 |

---

## Kiến trúc 4 lớp bảo vệ

| Lớp | Công cụ | Thời điểm | Mức độ |
|---|---|---|---|
| **1 — Pre-commit** | Gitleaks · Bandit · Semgrep (44 CEX rules) | Mỗi `git commit` (~5 giây) | Bắt buộc |
| **2 — CI/CD** | Gitleaks · Semgrep · Bandit · pip-audit | Mỗi MR/PR | Bắt buộc |
| **3 — AI scan** | Claude Code `/cex-security` | Trước MR, thủ công | Khuyến nghị |
| **4 — AI threat model** | Claude Code `/threat-model` | Trước khi code feature | Khuyến nghị |

Lớp 1 và 2 **không thể bypass** bằng cách bỏ qua commit hook (CI vẫn chạy độc lập).

---

## Rule coverage — 44 rules · 5 ngôn ngữ

| Ngôn ngữ | Số rules | Coverage |
|---|---|---|
| Python | 23 | SQLi, eval/exec injection, weak crypto, static nonce, balance race condition, hardcoded secrets, SHA-256 for password |
| JavaScript / TypeScript | 15 | Command injection, eval, weak hash, AES-ECB, static IV, float for money |
| Kotlin / Android | 8 | Weak hash, SQL injection, sensitive log, static IV, weak random, insecure SharedPreferences |
| Swift / iOS | 7 | Weak hash, insecure Keychain, sensitive log, static IV, weak random |
| Java | 5 | Shared với Kotlin rules |

### Severity policy

| Severity | Hành động |
|---|---|
| `ERROR` (CRITICAL) | Block commit + block MR/PR |
| `WARNING` (HIGH) | Warn only — dev thấy nhưng MR vẫn merge được |

---

## Cài đặt

### Clone standalone

```bash
git clone https://github.com/fee-191/security-template.git
cd security-template
bash scripts/test-ci-local.sh   # phải 80/80 PASS
```

### Thêm vào project làm git submodule

```bash
cd /path/to/your-project

# 1. Thêm submodule
git submodule add https://github.com/fee-191/security-template.git security

# 2. Commit
git add .gitmodules security
git commit -m "chore: add security template submodule"

# 3. Chạy setup (copy config files, cài pre-commit hooks)
bash security/scripts/setup-hooks.sh

# 4. Commit config
git add .
git commit -m "chore: add security template config files"
```

Kết quả: `✅ Setup hoàn tất — version 1.2.1`.

---

## Tích hợp CI/CD

### GitLab CI

```yaml
# Thêm vào .gitlab-ci.yml của project
include:
  - project: 'your-gitlab-group/security-template'
    ref: 'v1.2.1'
    file: '.gitlab-ci.yml'
```

5 jobs tự động: `security:gitleaks` · `security:semgrep-critical` · `security:semgrep-high` · `security:bandit` · `security:pip-audit`

Optional: set `SLACK_WEBHOOK_URL` trong CI variables để nhận alert khi CRITICAL finding block MR.

### GitHub Actions

Xem [`.github/workflows/security.yml`](.github/workflows/security.yml) — workflow tương đương, chạy tự động trên mỗi Pull Request.

---

## Claude Code AI Integration

Template cung cấp sẵn skills và agents cho Claude Code:

```
/cex-security              # scan uncommitted changes
/cex-security all          # scan toàn bộ codebase
/cex-security diff main    # scan diff với branch main
/threat-model <tính năng>  # phân tích STRIDE threat model trước khi code
```

Skill `cex-security-scan` (tại `.claude/skills/`) có thể dùng độc lập để review code trong conversation.

---

## Yêu cầu

| Thành phần | Phiên bản |
|---|---|
| Python | 3.10+ |
| Git | 2.30+ |
| pre-commit | 4.0+ |
| Node.js | 18+ *(chỉ cần cho Claude Code)* |

---

## Xử lý false positive

| Công cụ | Cách suppress |
|---|---|
| Semgrep | `# nosemgrep: rule-id` cuối dòng |
| Bandit | `# nosec BXXX` cuối dòng |
| Gitleaks | Thêm fingerprint vào `.gitleaksignore` |
| detect-secrets | `# pragma: allowlist secret` cuối dòng |

---

## Cập nhật template

```bash
cd security && git fetch --tags && git checkout <phiên-bản-mới>
cd .. && bash security/scripts/setup-hooks.sh
git add security && git commit -m "chore: update security template to <phiên-bản-mới>"
git push
```

---

## Tài liệu

| File | Nội dung |
|---|---|
| [`docs/guide.md`](docs/guide.md) | Hướng dẫn đầy đủ cho developer |
| [`docs/secure-checklist.md`](docs/secure-checklist.md) | 15-item checklist trước mỗi MR |
| [`docs/deck.html`](docs/deck.html) | Presentation — mở trực tiếp trên browser |
| [`CLAUDE.md`](CLAUDE.md) | Rules binding cho AI coding assistants |
| [`CHANGELOG.md`](CHANGELOG.md) | Lịch sử phiên bản |

---

[GitHub Issues](https://github.com/fee-191/security-template/issues) · **fee-191 · 2026**
