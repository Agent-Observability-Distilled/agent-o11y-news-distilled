# Design: Switch daily-digest from Copilot to opencode engine

## Problem

The daily-digest gh-aw workflow has two reliability issues:

1. **GitHub Copilot 429 rate limiting**: The `gpt-4o-mini` model (classified as "utility model") hits rate limits every 1-4 days. The agent retries 5 times over ~94s cumulative backoff, then exits. This causes ~75% of scheduled runs to fail.

2. **Narrow repo scope**: When the workflow succeeds, digests are constrained to only `Arize-ai/phoenix` and `anthropics/claude-code`. The rate limit likely prevents the agent from processing all 24 repos.

The project uses GitHub Agentic Workflows (gh-aw v0.81.6) with safe-outputs, AWF sandbox, and a two-stage prefetch+agent architecture.

## Constraints

- Must keep gh-aw for sandbox and safe-output guardrails
- Must keep the existing prefetch-data job (proven, reliable, no rate limit issues)
- Must use opencode-go subscription (no direct OpenAI/Anthropic API keys)
- Must preserve V1's agent prompt body (better signal-to-noise ratio than V2)

## Solution

Switch the gh-aw engine from `copilot` to `opencode`, routing LLM calls through the opencode-go subscription's OpenAI-compatible endpoint.

### How it works

Opencode Zen and Go both expose OpenAI-compatible endpoints:
- OpenCode Go: `https://opencode.ai/zen/go/v1/chat/completions`
- OpenCode Zen: `https://opencode.ai/zen/v1/chat/completions`

The gh-aw `opencode` engine (experimental) is described as "provider-agnostic, BYOK, 75+ models via provider/model format." The opencode CLI inside the AWF sandbox should use the opencode-go provider configured in the environment, making HTTP calls to `opencode.ai`.

The AWF firewall must allow egress to `opencode.ai` via `network.allowed`.

### Frontmatter changes (daily-digest-opencode.md)

```yaml
# FROM:
engine:
  id: copilot
  model: gpt-4o-mini
network: defaults

# TO:
engine:
  id: opencode
network:
  allowed: [defaults, opencode.ai]
```

Add `workflow_dispatch` with `dry_run` input for safe testing:

```yaml
on:
  schedule: daily
  workflow_dispatch:
    inputs:
      dry_run:
        description: "Skip issue creation — call noop instead"
        type: boolean
        default: false
```

Everything else (prefetch-data job, agent prompt body, safe-outputs, publishing contract) remains identical to V1.

### Test plan

1. Create `daily-digest-opencode.md` as a copy of `daily-digest.md`
2. Apply the engine + network + dry_run changes
3. `gh aw compile daily-digest-opencode --validate`
4. Trigger `workflow_dispatch` with `dry_run: true`
5. Check logs for:
   - Engine startup: does opencode engine initialize?
   - Network: does `opencode.ai` egress work?
   - LLM calls: does opencode-go routing succeed?
   - Digest quality: does the output match V1 signal-to-noise?
6. If successful: switch schedule to `daily-digest-opencode`
7. If failure: investigate log specifics, try env vars for explicit routing

### Fallback: Approach B

If the opencode engine in gh-aw cannot route through opencode-go, pivot to a hybrid approach:
- Keep gh-aw workflow for prefetch-data job
- Add a standard GitHub Actions job that runs opencode CLI directly (outside AWF sandbox)
- Opencode reads prefetched artifacts, generates digest Markdown
- Feed generated digest to gh-aw safe-outputs for publishing

## Unknowns

| Unknown | Mitigation |
|---|---|
| Does the opencode engine auto-detect opencode-go subscription? | Test with dry run; if not, pass explicit `OPENAI_BASE_URL` via `engine.env` |
| Does `network.allowed: [defaults, opencode.ai]` correctly allow API calls? | Test; if blocked, try `*.opencode.ai` wildcard |
| What authentication does opencode-go expect for API calls? | Check opencode config/env; may need API key as GitHub secret |
| Is the opencode engine stable enough for production scheduling? | Run multiple dry runs over several days before enabling schedule |

## Risks

- opencode engine is marked "experimental" in gh-aw v0.81.6
- AWF network config may block non-standard provider egress even with explicit allowlist
- OpenCode Go rate limits may also exist (but likely more generous than Copilot's utility-model tier)
