# Daily Digest OpenCode Engine Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate daily-digest from Copilot engine to OpenCode engine (via gh-aw definition-based third-party engine) to bypass Copilot 429 rate limits, using opencode-go subscription.

**Architecture:** Upgrade gh-aw v0.81.6 → v0.86.1 (which removes built-in opencode engine, replaces with import-based definition system), create `shared/opencode.md` engine definition file, copy V1 workflow to `daily-digest-opencode.md`, add engine import + env vars for opencode-go OpenAI-compatible routing, compile, and test with dry run.

**Tech Stack:** gh-aw v0.86.1 CLI, OpenCode CLI v1.2.14 (installed via npm by engine definition), opencode-go (OpenAI-compatible endpoint at `opencode.ai/zen/go/v1`), GitHub Actions, AWF sandbox v0.27.11

**Spec:** `docs/superpowers/specs/2026-08-09-daily-digest-opencode-engine-design.md`
**Guide:** https://github.github.com/gh-aw/guides/third-party-agent/
**Release:** https://github.com/github/gh-aw/releases/tag/v0.86.1

## Global Constraints

- Must keep gh-aw for sandbox and safe-output guardrails
- Must keep the prefetch-data job untouched (460+ lines of bash + Python from V1)
- Must preserve V1's agent prompt body and scoring/output format
- Must route LLM calls through opencode-go (OpenAI-compatible endpoint at `opencode.ai/zen/go/v1`)
- Must use `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` as credential secret (set to opencode-go key)
- Workflow must produce `.lock.yml` via `gh aw compile --validate`
- No manual edits to `.lock.yml` — it is generated output only
- The opencode engine definition is marked `experimental: true` — expect compile warnings

## Prerequisite

Before starting tasks, create a GitHub Actions secret in the repository:

- **Name**: `OPENCODE_GO_API_KEY`
- **Value**: Your opencode-go API key/token
- **Path**: Repository → Settings → Secrets and variables → Actions → New repository secret

---

### Task 0: Upgrade gh-aw from v0.81.6 to v0.86.1

**Files:**
- Modify: `.github/workflows/daily-digest.lock.yml` (recompiled)
- Modify: `.github/workflows/daily-digest-v2.lock.yml` (recompiled)

**Interfaces:**
- Consumes: Current gh-aw v0.81.6 installation, existing compiled workflows
- Produces: gh-aw v0.86.1, all workflows recompiled and validated

- [ ] **Step 1: Check current version**

```bash
gh aw --version
```

Expected: Shows current version (v0.81.6).

- [ ] **Step 2: Upgrade gh-aw CLI**

```bash
gh extension upgrade github/gh-aw
```

Expected: Downloads and installs v0.86.1 (or latest). Verify:

```bash
gh aw --version
```

Expected: v0.86.1 or later.

- [ ] **Step 3: Run gh aw fix to apply any codemods for breaking changes**

```bash
gh aw fix --write
```

Expected: Lists any fixes applied. Review the output. If any files are modified, review the diff:

```bash
git diff
```

- [ ] **Step 4: Recompile and validate ALL existing workflows**

v0.86.0 removed the built-in opencode engine. Existing workflows that use `engine.id: copilot` should still be fine, but the AWF sandbox version and action SHAs need updating.

```bash
gh aw compile --validate
```

Expected: All existing workflows compile with 0 errors. Warnings about deprecated fields are acceptable but should be reviewed.

If the v1 `daily-digest.md` workflow fails to compile, check the error and run `gh aw fix` again.

- [ ] **Step 5: Run full validation**

```bash
gh aw validate
```

Expected: All workflows pass.

- [ ] **Step 6: Commit the upgraded lock files**

```bash
git add .github/workflows/*.lock.yml .github/aw/actions-lock.json
git commit -m "chore: upgrade gh-aw v0.81.6 -> v0.86.1, recompile all workflows"
```

---

### Task 1: Create OpenCode engine definition file

**Files:**
- Create: `.github/workflows/shared/opencode.md`

**Interfaces:**
- Consumes: None (standalone definition file)
- Produces: `shared/opencode.md` — declarative engine definition telling gh-aw how to install, configure, and invoke OpenCode CLI

The engine definition is a Markdown file with YAML frontmatter. It describes:
- How to install OpenCode CLI (`npm install opencode-ai@1.2.14`)
- What config to write (`opencode.jsonc` with permissions + `autoupdate: false`)
- How to invoke it (`opencode run --print-logs --log-level DEBUG`)
- What network domains to allow (`opencode.ai`, `models.dev`, etc.)
- How to map credentials (`universal-llm-consumer` reads standard env vars)

- [ ] **Step 1: Create the shared directory**

```bash
mkdir -p .github/workflows/shared
```

- [ ] **Step 2: Write the engine definition file**

Write to `.github/workflows/shared/opencode.md`:

```markdown
---
engine:
  id: opencode
  display-name: OpenCode
  description: OpenCode CLI with headless mode and multi-provider LLM support
  runtime-id: opencode
  experimental: true
  behaviors:
    secret-strategy: universal-llm-consumer
    capabilities:
      max-turns: true
    manifest:
      files:
        - opencode.jsonc
        - AGENTS.md
      path-prefixes:
        - .opencode/
    network:
      defaults:
        - host.docker.internal
        - github.com
        - raw.githubusercontent.com
        - registry.npmjs.org
        - opencode.ai
        - models.dev
      provider-domains:
        copilot: api.githubcopilot.com
        anthropic: api.anthropic.com
        openai: api.openai.com
    installation:
      package-manager: npm
      package-name: opencode-ai
      version: "1.2.14"
      step-name: Install OpenCode
      binary-name: opencode
      include-node-setup: true
      cooldown: true
      verify-command: opencode --version
      verify-step-name: Verify OpenCode CLI installation
      docs-url: https://opencode.ai/docs
    config-file:
      path: opencode.jsonc
      step-name: Write OpenCode Config
      content: |-
        {
          "agent": {
            "build": {
              "permission": {
                "bash": "allow",
                "edit": "allow",
                "read": "allow",
                "glob": "allow",
                "grep": "allow",
                "webfetch": "allow",
                "websearch": "allow",
                "external_directory": "allow"
              }
            }
          },
          "autoupdate": false
        }
      merge-strategy: json-merge
    execution:
      command-name: opencode
      args:
        - run
        - --print-logs
        - --log-level
        - DEBUG
      step-name: Execute OpenCode CLI
      model-env-var: OPENCODE_MODEL
      mcp-config-env-var: GH_AW_MCP_CONFIG
      write-timestamp: true
      provider-env-mode: universal-llm-consumer
    mcp:
      config-path: opencode.jsonc
---
```

- [ ] **Step 3: Verify the file structure**

```bash
cat .github/workflows/shared/opencode.md | head -5
```

Expected: Shows YAML frontmatter with `engine:\n  id: opencode`.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/shared/opencode.md
git commit -m "feat: add OpenCode engine definition for gh-aw third-party agent support"
```

---

### Task 2: Create daily-digest-opencode.md workflow

**Files:**
- Create: `.github/workflows/daily-digest-opencode.md` (copy of `daily-digest.md` with modifications)

**Interfaces:**
- Consumes: `.github/workflows/daily-digest.md` (entire file), `.github/workflows/shared/opencode.md` (engine definition)
- Produces: `.github/workflows/daily-digest-opencode.md` with opencode engine + dry_run + opencode-go env vars

The workflow keeps V1's prefetch-data job and agent prompt body unchanged. Only the frontmatter is modified:
1. Add `imports: [shared/opencode.md]` to load the engine definition
2. Set `engine: opencode` (bare string, per third-party engine convention)
3. Add `engine.env` with opencode-go credentials
4. Add `dry_run` input to `workflow_dispatch`
5. Remove `model` field (not needed — engine definition handles it)

- [ ] **Step 1: Copy the V1 workflow**

```bash
cp .github/workflows/daily-digest.md .github/workflows/daily-digest-opencode.md
```

- [ ] **Step 2: Replace the entire frontmatter (lines 1-25 of the new file)**

The current V1 frontmatter is:

```yaml
---
on:
  schedule: daily
  workflow_dispatch:
permissions:
  contents: read
  issues: none
  pull-requests: read
engine:
  id: copilot
  model: gpt-4o-mini
network: defaults
tools:
  bash:
    - cat
    - python3
  github:
    toolsets: [default]
    min-integrity: none
safe-outputs:
  create-issue:
    max: 1
  add-comment:
    max: 1
    target: "*"
```

Replace with:

```yaml
---
on:
  schedule: daily
  workflow_dispatch:
    inputs:
      dry_run:
        description: "Skip issue creation — call noop for testing"
        type: boolean
        default: false
permissions:
  contents: read
  issues: none
  pull-requests: read
engine: opencode
imports:
  - shared/opencode.md
engine.env:
  OPENCODE_MODEL: "openai/deepseek-v4-pro"
  OPENAI_API_KEY: ${{ secrets.OPENCODE_GO_API_KEY }}
  OPENAI_BASE_URL: "https://opencode.ai/zen/go/v1"
tools:
  bash:
    - cat
    - python3
  github:
    toolsets: [default]
    min-integrity: none
safe-outputs:
  create-issue:
    max: 1
  add-comment:
    max: 1
    target: "*"
```

The `prefetch-data` job and everything after the frontmatter separator (`---`) stays unchanged.

Use Edit tool on `.github/workflows/daily-digest-opencode.md`:

```
oldString: ---
on:
  schedule: daily
  workflow_dispatch:
permissions:
  contents: read
  issues: none
  pull-requests: read
engine:
  id: copilot
  model: gpt-4o-mini
network: defaults
tools:
  bash:
    - cat
    - python3
  github:
    toolsets: [default]
    min-integrity: none
safe-outputs:
  create-issue:
    max: 1
  add-comment:
    max: 1
    target: "*"

newString: ---
on:
  schedule: daily
  workflow_dispatch:
    inputs:
      dry_run:
        description: "Skip issue creation — call noop for testing"
        type: boolean
        default: false
permissions:
  contents: read
  issues: none
  pull-requests: read
engine: opencode
imports:
  - shared/opencode.md
engine.env:
  OPENCODE_MODEL: "openai/deepseek-v4-pro"
  OPENAI_API_KEY: ${{ secrets.OPENCODE_GO_API_KEY }}
  OPENAI_BASE_URL: "https://opencode.ai/zen/go/v1"
tools:
  bash:
    - cat
    - python3
  github:
    toolsets: [default]
    min-integrity: none
safe-outputs:
  create-issue:
    max: 1
  add-comment:
    max: 1
    target: "*"
```

- [ ] **Step 3: Add dry_run handling to agent prompt body**

Find the `Output rules:` section near the end of the file (inside the markdown body, after the second `---` separator). Insert dry_run instructions before it:

```
oldString: Output rules:

- If no items qualify after filtering, do nothing and do not create an
  issue.
- Otherwise create one issue titled "Daily Digest - <date>".

newString: Before publishing, check the dry_run workflow input: `${{ inputs.dry_run }}`.
Use exactly one of these three flows:

- If `dry_run` is `true`: score and filter items normally, then call `noop`
  once with the qualifying items, their scores, and the coverage footer. Do
  not call `create_issue` or `add_comment`. Stop after noop.
- If `dry_run` is `false` (or not set) and no items qualify: call `noop`
  once with a concise explanation and the coverage footer. Stop after noop.
- If `dry_run` is `false` (or not set) and items qualify: proceed with
  `create_issue` and `add_comment` as described below.

Never call `noop` in the same run as `create_issue`.
Never call `create_issue` more than once.

Output rules:

- If no items qualify after filtering, do nothing and do not create an
  issue.
- Otherwise create one issue titled "Daily Digest - <date>".
```

- [ ] **Step 4: Verify the changes**

```bash
rg -n "opencode\|dry_run\|imports:\|shared/opencode\|OPENCODE_MODEL\|OPENAI_BASE_URL\|OPENCODE_GO_API_KEY" .github/workflows/daily-digest-opencode.md
```

Expected matches:
- `imports:` in frontmatter
- `shared/opencode.md` in frontmatter
- `engine: opencode` in frontmatter
- `OPENCODE_MODEL` in frontmatter
- `OPENAI_BASE_URL` in frontmatter
- `OPENCODE_GO_API_KEY` secret reference in frontmatter
- `dry_run` in both frontmatter (`inputs.dry_run`) and agent prompt (`${{ inputs.dry_run }}`)

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/daily-digest-opencode.md
git commit -m "feat: add daily-digest-opencode workflow with OpenCode engine + opencode-go routing"
```

---

### Task 3: Compile and validate the workflow

**Files:**
- Create: `.github/workflows/daily-digest-opencode.lock.yml` (generated by compiler)
- Create/Modify: `.github/aw/actions-lock.json` (action pins, may be updated by compiler)

**Interfaces:**
- Consumes: Task 2's `.github/workflows/daily-digest-opencode.md`, Task 1's `shared/opencode.md`
- Produces: `.github/workflows/daily-digest-opencode.lock.yml`

- [ ] **Step 1: Compile the new workflow with strict validation**

```bash
gh aw compile daily-digest-opencode --validate
```

Expected: Compilation succeeds. May show warning like `experimental engine 'opencode' — test thoroughly before production use`. If the compilation fails with an error about the opencode import, run:

```bash
gh aw fix --write
gh aw compile daily-digest-opencode --validate
```

(v0.86.1 has `gh aw fix` diagnostics specifically for missing opencode/crush imports.)

- [ ] **Step 2: Inspect the generated lock file for engine correctness**

```bash
head -5 .github/workflows/daily-digest-opencode.lock.yml
```

Expected: The `gh-aw-metadata` line should show `"agent_id":"opencode"` (not `"copilot"`). The manifest should list `shared/opencode.md` as an import.

- [ ] **Step 3: Verify the lock file is valid YAML**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/daily-digest-opencode.lock.yml')); print('Valid YAML')"
```

Expected: `Valid YAML`

- [ ] **Step 4: Run full validation on all workflows**

```bash
gh aw validate
```

Expected: All workflows pass (or warnings only). Pay attention to any errors about the new workflow.

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/daily-digest-opencode.lock.yml .github/aw/actions-lock.json
git commit -m "feat: compile daily-digest-opencode workflow"
```

---

### Task 4: Manual verification — dry run test

**No file changes.** This is a manual operational step requiring the `OPENCODE_GO_API_KEY` secret to exist in the repository.

- [ ] **Step 1: Push changes to GitHub**

```bash
git push
```

- [ ] **Step 2: Confirm the secret exists**

Go to repository Settings → Secrets and variables → Actions. Verify `OPENCODE_GO_API_KEY` is present.

- [ ] **Step 3: Trigger the workflow with dry_run enabled**

```bash
gh workflow run daily-digest-opencode.md -f dry_run=true
```

- [ ] **Step 4: Monitor the run**

```bash
gh run watch
```

Expected: Run completes. Note the run ID for log inspection.

- [ ] **Step 5: Check logs for OpenCode engine and opencode-go routing evidence**

```bash
gh aw logs daily-digest-opencode
```

Look for these markers in the logs:

| What to check | Expected signal |
|---|---|
| **Engine startup** | Log shows "Install OpenCode" step running `npm install opencode-ai@1.2.14` |
| **CLI verification** | "Verify OpenCode CLI installation" step shows `opencode --version` output |
| **Config write** | "Write OpenCode Config" step writes `opencode.jsonc` |
| **Execution** | "Execute OpenCode CLI" step runs `opencode run --print-logs --log-level DEBUG` |
| **Model routing** | `OPENCODE_MODEL=openai/deepseek-v4-pro` is set in the environment |
| **API endpoint** | `OPENAI_BASE_URL=https://opencode.ai/zen/go/v1` is set |
| **Network egress** | No blocks to `opencode.ai` domain |
| **LLM completion** | Digest is generated (scoring + filtering applied) |
| **Dry run behavior** | Calls `noop` with qualifying items and scores, does NOT call `create_issue` |

- [ ] **Step 6: If the run fails, diagnose**

| Failure mode | Likely cause | Fix |
|---|---|---|
| OpenCode CLI installation fails | npm registry not reachable | Check `network.defaults` in engine definition includes `registry.npmjs.org` |
| `opencode --version` fails | Wrong Node.js version | Engine definition has `include-node-setup: true` — should auto-setup Node |
| opencode-go auth error (401/403) | Invalid API key or wrong env var | Verify `OPENCODE_GO_API_KEY` secret value; check if `OPENAI_API_KEY` is the correct env var name for opencode-go |
| Connection refused to opencode.ai | Firewall blocking the domain | `opencode.ai` is in engine definition's `network.defaults` — verify in compiled lock.yml |
| Model not found | Wrong `OPENCODE_MODEL` value | Try different model format: `deepseek-v4-pro`, `openai/deepseek-v4-pro`, or check available models via opencode-go API |
| Digest quality poor | Model not suited for scoring/filtering | Try a different model (e.g., check what opencode-go supports for classification tasks) |
| No output / silent failure | opencode run command issue | Check if `--print-logs` + `--log-level DEBUG` flags work with the installed version |

- [ ] **Step 7: If the dry run succeeds, review the digest quality**

Check the workflow logs for the `noop` output — it should list qualifying items with scores. Verify:
- Items from all 4 categories and multiple repos (not just phoenix/Claude Code)
- Scores are reasonable (>= 75 threshold)
- Summary and Value Proposition cells are populated

---

### Task 5: Enable the schedule

**No file changes needed.** The `schedule: daily` trigger is already in the workflow frontmatter.

- [ ] **Step 1: Disable the old V1 schedule**

If `daily-digest.md` is still active on schedule, rename it to disable (or remove `schedule: daily` from its `on:` block):

```bash
mv .github/workflows/daily-digest.md .github/workflows/daily-digest.md.disabled
```

If you rename, recompile to remove the disabled workflow from the registry:

```bash
gh aw compile
```

- [ ] **Step 2: Verify the schedule trigger on the new workflow**

```bash
rg -A3 "^on:" .github/workflows/daily-digest-opencode.md
```

Expected: `schedule: daily` is present.

- [ ] **Step 3: Commit and push**

```bash
git add -A
git commit -m "chore: enable daily-digest-opencode schedule, disable old V1 Copilot workflow"
git push
```

- [ ] **Step 4: Monitor the first scheduled run**

The schedule fires daily at ~08:00 UTC. After the first run:

```bash
gh run list --workflow daily-digest-opencode.md --limit 5
gh aw logs daily-digest-opencode
```

Expected:
- Run succeeds (green checkmark)
- Digest issue created in the repository with @doughgle mention
- Digest covers all 4 categories and multiple repos (not just phoenix/Claude Code)

- [ ] **Step 5: Monitor for 3-5 consecutive days**

The Copilot 429 issue happened every 1-4 days. Monitor for 5 days to confirm opencode-go avoids rate limiting:

```bash
# Check recent runs
gh run list --workflow daily-digest-opencode.md --limit 10 --status success
gh run list --workflow daily-digest-opencode.md --limit 10 --status failure
```

Expected: 5 consecutive successes with no 429 errors.

---

## Fallback: If OpenCode engine fails → Approach B

If Task 4 or Task 5 reveals the OpenCode engine cannot route through opencode-go, execute this fallback:

1. **Revert the workflow**:
   ```bash
   rm .github/workflows/daily-digest-opencode.md .github/workflows/daily-digest-opencode.lock.yml
   mv .github/workflows/daily-digest.md.disabled .github/workflows/daily-digest.md
   gh aw compile --validate
   ```

2. **Create new plan for Approach B**: gh-aw prefetch-data job + separate standard GitHub Actions job running opencode CLI directly (outside AWF sandbox, using opencode-go natively). This bypasses the engine definition entirely at the cost of losing automatic safe-outputs for the publishing step.

---

## Self-Review

**1. Spec coverage:**
- Engine switch (copilot → opencode): Task 2 ✅
- Network egress (opencode.ai): Handled by engine definition defaults in Task 1 ✅
- Dry run input + prompt handling: Task 2 ✅
- Prefetch job unchanged: Verified by copy-from-V1 approach ✅
- V1 prompt preserved: Only dry_run block added ✅
- gh-aw upgrade: Task 0 ✅
- Engine definition file: Task 1 ✅
- Compile and validate: Task 3 ✅
- Dry run test: Task 4 ✅
- Enable schedule: Task 5 ✅
- Fallback to Approach B: Documented ✅

**2. Placeholder scan:** No TBD, TODO, or vague instructions. All steps have exact commands. All edits have exact oldString/newString. ✅

**3. Type consistency:** YAML config change — no types to verify. ✅
