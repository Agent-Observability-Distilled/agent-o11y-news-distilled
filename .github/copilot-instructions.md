# Copilot instructions for this repository

## Commands

This repo does not have an application build or unit-test suite. The main verification path is **gh-aw** workflow validation and compilation.

```bash
# List workflows discovered from .github/workflows/*.md
gh aw list

# Validate every workflow
gh aw validate

# Validate a single workflow while iterating
gh aw validate daily-digest-v2

# Compile all workflow Markdown sources into .lock.yml outputs
gh aw compile

# Compile one workflow with extra validation
gh aw compile daily-digest-v2 --validate

# Debug a workflow run
gh aw logs daily-digest-v2
gh aw audit <run-id>
```

If you modify generated dependency manifests for workflows, update the source Markdown workflow instead and rerun:

```bash
gh aw compile --dependabot
```

## High-level architecture

This repository is a **GitHub Agentic Workflows (`gh aw`) repo**, not a conventional application codebase. The primary source files are the Markdown workflows in `.github/workflows/`.

- `.github/workflows/daily-digest.md` is the original single-stage digest workflow prompt.
- `.github/workflows/daily-digest-v2.md` is the more important workflow to understand before editing. It splits the job into:
  1. a GitHub Actions `prefetch-data` job that collects releases/issues from a fixed 24-repo shortlist with `gh` CLI and writes compact JSON artifacts under `/tmp/daily-digest-prefetch`
  2. an agent stage that reads `.aw/prefetch/daily-digest-metrics.json` and `.aw/prefetch/daily-digest-summary.json` first, then selectively inspects `daily-digest-cache.json` only for promising candidates before publishing results
- `.github/workflows/copilot-setup-steps.yml` prepares GitHub Copilot cloud runs by installing the `gh aw` extension.
- `.vscode/mcp.json` exposes `gh aw mcp-server` for local MCP usage.
- `.github/aw/actions-lock.json` pins external action SHAs used by the agentic workflow setup.

## Key conventions

- Treat `.github/workflows/*.md` as the **source of truth**. Generated `.lock.yml` files and generated dependency manifests are outputs of `gh aw compile`, not the place to hand-edit logic.
- The digest workflows are intentionally **GitHub-only**. Preserve the existing bias toward GitHub MCP / GitHub tooling and avoid replacing it with raw HTTP calls to GitHub when the same data is already available through GitHub tools.
- In `daily-digest-v2`, keep the **summary-first, cache-second** flow: read metrics and summary first, then use targeted `python3` extraction against `daily-digest-cache.json` only for shortlisted items.
- Preserve the current publishing contract for digest workflows: one digest issue at most, one follow-up comment at most, and direct safe-output calls (`create_issue`, `add_comment`, `noop`) rather than wrapper namespaces.
- Preserve the repo’s security posture when editing workflows: minimal permissions, explicit tooling, and action pinning via the checked-in lock data.
- The digest content is opinionated: it uses a fixed four-topic taxonomy, a fixed monitored repo shortlist, a hard telemetry relevance gate, and a score threshold of `>= 75`. Changes to those rules affect the workflow’s editorial behavior, not just formatting.
- If you add or reshape a workflow, keep the change self-contained in a single workflow Markdown file unless the repo already has an established shared component for that behavior.
