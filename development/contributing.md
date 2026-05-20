# Contributing to WendyOS

Thank you for contributing to WendyOS! This document covers the development workflows, CI pipelines, and guidelines for contributing to the project.

## CI / GitHub Actions

### Docs Update Workflow (`.github/workflows/docs-update.yml`)

The docs-update workflow automatically proposes documentation changes whenever a pull request is merged. It is driven by Claude (Anthropic) and reflects the changes in the PR diff.

**Key behaviours:**

- **Relevance-ranked doc selection:** Markdown files under `docs/` are scored by keyword overlap between their path/content and the PR title and diff. Files are ranked by relevance descending before the 50,000-character budget is applied, so the most pertinent documentation fills the context window rather than alphabetically-first files.
- **File-tree context:** A compact file tree (max 3 levels deep) is included in the prompt so the model can reason about the overall documentation structure.
- **Larger context window:** `max_tokens` is set to `16,000`.
- **Diff formatting:** The raw diff is wrapped in a fenced ` ```diff ``` ` code block in the prompt, truncated at 30,000 characters.
- **Docs truncation limit:** The current-docs context limit is 50,000 characters, presented to the model as "most relevant first".
- **Source repo context:** The name of the source repository is included in the prompt.
- **Path safety:** The workflow uses `Path.relative_to()` (raising `ValueError` on traversal) and rejects any output path whose suffix is not `.md`.
- **Protected files:** `THREAT_MODEL.md` and `threat-model.md` are never read as context and never written by this workflow. Threat model files are managed by a dedicated security workflow.
- **Directory creation:** Output directories are created automatically (`mkdir -p`) so the model can introduce new documentation pages in new subdirectories.
- **Created vs. Updated logging:** The workflow prints `Created:` for new files and `Updated:` for existing files.
- **Empty-diff guard:** If the diff is empty, the workflow exits early rather than attempting a model call.
- **Prompt wording:** The system prompt instructs the model to make only minimal, surgical edits to keep docs accurate — updating only content that is directly wrong or missing as a result of the diff, writing in timeless present tense, and never inventing or removing features not present in the diff.

### VSCode Extension Update Workflow (`.github/workflows/vscode-update.yml`)

The vscode-update workflow automatically proposes changes to the `wendylabsinc/wendy-vscode` extension whenever a CLI command file is merged to `main`. It is driven by Claude (Anthropic) and analyses the CLI diff against the current extension source.

**Trigger:** Fires on merged pull requests that touch files under `go/internal/cli/commands/**`. PRs labelled `ai-suggestion` are skipped to prevent recursive loops.

**Key behaviours:**

- **Relevance-ranked file selection:** TypeScript, JSON, and Markdown files in the `wendy-vscode` repository are scored by keyword overlap between their path/content and the PR title and diff. Files are ranked by relevance descending before a 50,000-character budget is applied.
- **File-tree context:** A compact file tree (max 3 levels deep) of the extension repository is included in the prompt.
- **Context window:** `max_tokens` is set to `16,000`.
- **Diff formatting:** The raw CLI diff is wrapped in a fenced ` ```diff ``` ` code block, truncated at 30,000 characters.
- **Source repo context:** The name of the source repository and the originating PR number and title are included in the prompt.
- **Path safety:** The workflow uses `Path.relative_to()` (raising `ValueError` on traversal) and only writes files with `.ts`, `.json`, or `.md` extensions.
- **Protected files and directories:** `package-lock.json` is never written. Files inside `dist/`, `node_modules/`, or `.git/` are never read or written.
- **Directory creation:** Output directories are created automatically so the model can introduce new files in new subdirectories.
- **Created vs. Updated logging:** The workflow prints `Created:` for new files and `Updated:` for existing files.
- **Empty-diff guard:** If the diff is empty, the workflow exits early rather than attempting a model call.
- **No-op guard:** If Claude determines the extension does not need updating (no `<description>` block and no `<file>` blocks in the response), no PR is opened.
- **Output format:** Claude is instructed to output a `<description>` block with a human-readable explanation and `<file path="...">…
