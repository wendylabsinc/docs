# Contributing to WendyOS

Thank you for contributing to WendyOS! This document covers the development workflows, CI pipelines, and guidelines for contributing to the project.

## CI / GitHub Actions

### Docs Update Workflow (`.github/workflows/docs-update.yml`)

The docs-update workflow automatically proposes documentation changes whenever a pull request is merged. It is driven by Claude (Anthropic) and reflects the changes in the PR diff.

**Key behaviours (as of PR #588):**

- **Full doc sweep:** Instead of mapping changed source paths to specific doc sections, the workflow now reads *all* Markdown files under `docs/` up to a 50,000-character budget. Files are collected in sorted order; collection stops when the budget is reached.
- **File-tree context:** A compact file tree (max 3 levels deep) is included in the prompt so the model can reason about the overall documentation structure.
- **Larger context window:** `max_tokens` has been raised from `8096` to `16,000` to accommodate more thorough updates.
- **Diff formatting:** The raw diff is now wrapped in a fenced ` ```diff ``` ` code block in the prompt, and the truncation limit has been raised from 20,000 to 30,000 characters.
- **Docs truncation limit:** The current-docs context limit has been raised from 20,000 to 50,000 characters.
- **Path safety:** The workflow uses `Path.relative_to()` (raising `ValueError` on traversal) instead of a string-prefix check, and additionally rejects any output path whose suffix is not `.md`.
- **Directory creation:** Output directories are created automatically (`mkdir -p`) so the model can introduce new documentation pages in new subdirectories.
- **Created vs. Updated logging:** The workflow now prints `Created:` for new files and `Updated:` for existing files.
- **Empty-diff guard:** If the diff is empty, the workflow exits early rather than attempting a model call.
- **Prompt wording:** The system prompt has been updated to instruct the model to be thorough, touch every affected page, create new files where needed, and not delete existing documentation unless the diff explicitly removes a feature.

### Security Review Workflow (`.github/workflows/security-review.yml`)

A new **AI Security Review** workflow was added in PR #588. It runs on every pull request targeting `main` and posts a structured security and compliance report as a PR comment.

#### Trigger

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]
```

Concurrent runs for the same PR are cancelled automatically (`cancel-in-progress: true`).

#### What it does

1. **Fetches the PR diff** using the GitHub CLI (`gh pr diff`).
2. **Invokes Claude** (`claude-sonnet-4-6`, `max_tokens=8096`) with a detailed system prompt covering security and compliance analysis.
3. **Writes a Markdown report** (`review.md`) to the workspace.
4. **Posts or updates a PR comment** headed `## AI Security Review`. If a previous security review comment already exists on the PR it is edited in place; otherwise a new comment is created.

#### Security analysis scope

The model analyses the diff for:

- Injection vulnerabilities (SQL, command, LDAP, XPath, template, etc.)
- Authentication and authorisation flaws
- Sensitive data exposure (secrets, PII, tokens hardcoded or logged)
- Cryptographic weaknesses
- Input validation and output encoding issues
- Security misconfiguration (overly broad permissions, debug flags, unsafe defaults)
- Insecure dependencies or version pins
- Race conditions and TOCTOU vulnerabilities
- Path traversal and SSRF
- Denial-of-service vectors (unbounded loops, large allocations, missing rate limits)

#### Compliance frameworks checked

| Framework | Controls evaluated |
|---|---|
| **SOC 2** | CC6, CC7, CC8, CC9, A1, C1, P-series |
| **ISO/IEC 27001:2022** | A.8 (technological controls), A.9 (access control), A.12 (logging & monitoring) |
| **PCI DSS v4.0** | Req 3, 4, 6, 10 — flagged only when the diff touches payment/card data |
| **GDPR / Privacy** | Lawful basis, PII logging, data-subject rights |
| **HIPAA** | PHI encryption, access controls — flagged only when the diff touches health data |
| **NIST SP 800-53 / CSF 2.0** | AC, AU, IA, SC, SI control families |

Each finding is rated **CRITICAL**, **HIGH**, **MEDIUM**, **LOW**, or **INFORMATIONAL** and tagged with the relevant standard(s) (e.g. `[SOC2-CC6] [ISO27001-A.8]`).

#### Report structure

1. One-paragraph executive summary
2. Findings table: `| Severity | Standards | File | Line(s) | Title |`
3. Detailed section per finding with description, quoted snippet, remediation advice, and controls violated
4. Compliance summary listing which frameworks were checked and whether violations were found
5. Explicit "no findings" statement if the diff looks safe

#### Permissions

The job requests only `contents: read` and `pull-requests: write`. It only runs on PRs from within the same repository (`github.event.pull_request.head.repo.full_name == github.repository`), preventing untrusted forks from triggering the workflow.

#### Prompt injection mitigation

PR title, body, and diff are wrapped in `<untrusted_pr_content>` tags, and the system prompt explicitly instructs the model to ignore any instructions embedded within that content.

#### Required secrets

| Secret | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Authenticates calls to the Anthropic API |

No additional secrets are required; the `GH_TOKEN` is provided automatically by GitHub Actions.
