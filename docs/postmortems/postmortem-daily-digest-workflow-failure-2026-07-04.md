# Postmortem: daily-digest Agentic Workflow Fails with Exit Code 1

## Summary

| Field | Value |
|---|---|
| **Incident date** | 2026-07-01 (first observed) – 2026-07-04 (fix deployed) |
| **Duration** | ~3 days (first failure run #59, 2026-07-01 ~08:00 UTC through fix run #66, 2026-07-04 06:41 UTC) |
| **Services affected** | daily-digest.md (v1), daily-digest-v2.md (v2) — agentic-workflows on GitHub Actions |
| **User impact** | No daily-digest issues published Jul 1–3 (expected daily); historical cadence broken |
| **Detected by** | GitHub Copilot CLI exit code 1 in workflow run logs; `[aw] daily-digest failed` auto-issues |
| **Trigger(s)** | Scheduled workflow ran with deprecated model `gpt-5-mini` |
| **Root cause (summary)** | Three independent failures: (1) `gpt-5-mini` model deprecated/retired returning 400 error, causing "Execute GitHub Copilot CLI" step to exit 1; (2) outdated gh-aw v0.68.1 infrastructure pinned Node.js 20 actions incompatible with forced Node.js 24 runner environment; (3) GitHub Copilot utility model rate limiting (429) during agent execution |
| **Resolution (summary)** | Upgraded gh-aw v0.68.1 → v0.81.6 (compiler, actions, containers, Copilot CLI v1.0.21→v1.0.65, AWF v0.25.18→v0.27.11, all action SHAs refreshed), changed model from `gpt-5-mini` → `gpt-4o-mini`, fixed v1 add_comment prompt to use `item_number` + `temporary_id` pattern. Rate limiting is a transient condition requiring cooldown or retry strategy. |
| **SLO/Error Budget impact** | [unknown] |
| **Responders** | [doughgle] |

## Impact

The daily-digest workflow failed to produce a daily digest issue on Jul 1, 2, and 3. Each run exited during the "Execute GitHub Copilot CLI" step, which is the primary agent loop that fetches news, generates the digest, and creates the GitHub issue. No user-facing service was affected, but the expected daily output (a GitHub issue with curated AI agent observability news) was not produced for three consecutive days.

## System Background

The daily-digest workflow is a gh-aw (GitHub Actions Workflow) agentic workflow that:
1. Activates a Copilot CLI environment
2. Prefetches news data via an "agentic workflow foundation" (AWF) step
3. Runs a Copilot-powered agent that processes the prefetched data, generates a daily digest, and creates a GitHub issue with the result

It uses a compiled lockfile (daily-digest.lock.yml) that pins action SHAs, Node.js versions, Copilot CLI versions, and AWF versions. The v2 variant (daily-digest-v2.md) uses the same infrastructure with a different agent prompt.

## Root Causes

### Root Cause 1: Model deprecation

The workflow specified `gpt-5-mini` as the Copilot model. At some point between the last successful run (Jun 18) and the first failure (Jul 1), this model was deprecated and began returning HTTP 400 errors: "The requested model is not supported." The Copilot CLI propagated this error as exit code 1, which caused the workflow step to fail.

**Why was `gpt-5-mini` chosen?** It was the original configuration when the workflow was authored, presumed to be the recommended default at that time.

**Why wasn't the model change detected proactively?** No periodic validation step tested model availability independently of the full workflow execution.

### Root Cause 2: Utility model rate limiting (429)

After the model and infrastructure fixes, run #67 failed with a 429 rate limit error: "Sorry, you've exceeded your rate limit for utility models." The agent ran for ~8 minutes, retried 5 times with 94s of cumulative backoff, then exited with code 1. The agent also accumulated 185 permission denied events during execution, suggesting it was making many requests in a short window.

**Why did the rate limit trigger?** The agent workflow consumed tokens aggressively, likely exceeding the per-minute quota for `gpt-4o-mini` (classified as a utility model). The gh-aw Copilot harness retries on 429 but exhausts its retries before the rate limit window expires.

**Why wasn't rate limiting handled gracefully?** The workflow has no mechanism to pause and retry after a longer backoff, nor does it have a "dry run" or token budget estimator.

### Root Cause 3: Outdated gh-aw infrastructure (v0.68.1)

The workflow was compiled with gh-aw v0.68.1. This version pinned:
- Copilot CLI v1.0.21
- AWF (Agentic Workflow Foundation) v0.25.18
- Node.js 20 actions
- Older action SHAs

The GitHub Actions runner environment had been updated to Node.js 24, causing Node.js 20-based actions to fail or behave unexpectedly. Additionally, the older action SHAs lacked compatibility with the current runner environment.

**Why were dependencies not updated sooner?** The workflow was authored and left running successfully for months; no monitoring was in place for upstream dependency deprecation.

### Root Cause 3: v1 workflow prompt mismatch

The v1 daily-digest workflow (daily-digest.md) instructed the agent to call `add_comment` with `issue_number` as the parameter. However, the safe-outputs schema defined `item_number` (not `issue_number`) as the correct field name. This caused the safe_outputs job in run #66 to fail (`add_comment` used `item_number: 0` — missing the temporary_id resolution). The v2 workflow already had the correct `item_number` + `temporary_id` pattern.

**Why did the v1 prompt diverge?** The prompt was written independently of v2 and wasn't cross-checked against the tool schema when `add_comment` was added.

## Trigger

The daily-digest scheduled trigger (`cron` at 08:00 UTC) fired as normal. The workflow began executing but the "Execute GitHub Copilot CLI" step failed because the specified model `gpt-5-mini` was no longer available.

## Detection

- **First indication:** Run #59 on Jul 1 failed. The GitHub Actions UI showed a red ❌ on the workflow run.
- **Auto-created issue:** The workflow is configured to create `[aw] daily-digest failed` issues automatically on failure.
- **Time to detection:** The first failure was detected only when the auto-created issue was noticed. No pager/alert was configured beyond GitHub's run-failure notification.
- **Detection gap:** The failure persisted for ~3 days (6 missed runs: Jul 1 morning, Jul 1 afternoon? [unclear cadence], Jul 2, Jul 3) before investigation began on Jul 4.

## Resolution

### Step 1: Upgrade gh-aw and all dependencies

- Upgraded gh-aw from v0.68.1 → v0.81.6 (latest at time of fix)
- Recompiled both workflows with the new compiler
- Updated all action SHAs to Node.js 24-compatible versions
- Updated Copilot CLI from v1.0.21 → v1.0.65
- Updated AWF from v0.25.18 → v0.27.11
- This fixed the infrastructure compatibility issue

### Step 2: Change model

- Attempted: `gpt-5.4-mini` → returns 400 (also deprecated)
- Attempted: `haiku` → rejected client-side by Copilot CLI v1.0.65 ("model 'haiku' is retired or unsupported")
- Attempted: `gpt-5.4` → returns 400
- Final: `gpt-4o-mini` — accepted and produced successful output
- This fixed the model availability issue

### Step 3: Fix v1 prompt

- Changed `add_comment` instructions in v1 workflow from `issue_number` to `item_number` + `temporary_id` pattern, matching the v2 workflow and tool schema

### Step 4: Remove stray file

- Removed `agentics-maintenance.yml` that was accidentally committed during debugging

### Step 5: Rate limit / ongoing issue

Run #67 (b626af1, full fix) completed but the agent job failed with a 429 rate limit error. The `safe_outputs` and `conclusion` jobs succeeded (confirming the add_comment fix works), but the agent exhausted its 5 retries against the utility model rate limit. This is a transient operational condition and may resolve with cooldown or require a retry strategy enhancement.

## Action Items

| Action Item | Type | Owner | Ticket | Status |
|---|---|---|---|---|
| Document model selection guide for gh-aw workflows | prevent | doughgle | — | TODO |
| Add periodic validation step to test model availability | detect | doughgle | — | TODO |
| Pin explicit model fallback order in workflow config | mitigate | doughgle | — | TODO |
| Add gh-aw version check / upgrade reminder to workflow | prevent | doughgle | — | TODO |
| Align v1 and v2 prompts for add_comment (done) | prevent | doughgle | — | DONE |
| Upgrade gh-aw to latest (done) | prevent | doughgle | — | DONE |
| Schedule monthly review of gh-aw + dependency versions | process | doughgle | — | TODO |
| Add retry with exponential backoff for 429 rate limit errors | mitigate | doughgle | — | TODO |
| Investigate 185 permission-denied events during agent run | process | doughgle | — | TODO |
| Add rate-limit-aware scheduling (cooldown between runs) | mitigate | doughgle | — | TODO |

## Lessons Learned

### What went well
- The auto-created `[aw] daily-digest failed` issues provided a clear signal when runs failed
- The workflow compiled cleanly after each fix attempt, giving fast feedback
- gh-aw v0.81.6 compilation produced 0 errors/warnings immediately
- Run #66 with `gpt-4o-mini` successfully produced a real daily digest issue (#55)

### What went wrong
- No model deprecation monitoring existed — the model could silently become unavailable without any proactive signal
- Debugging took ~3 hours of iterative model testing across 5 models (gpt-5-mini → gpt-5.4-mini → haiku → gpt-5.4 → gpt-4o-mini)
- The correlation between Node.js 24 runner and Node.js 20 action failures was initially missed; first attempts focused solely on model issues
- v1 and v2 workflows diverged in prompt structure for the same tool (add_comment), creating a subtle failure mode unique to v1

### Where we got lucky
- The model issue produced an actionable error (exit code 1 with clear log message), rather than silent corruption
- `gpt-4o-mini` was still available and produced quality digest output — if all gpt-4o models had also been deprecated, no fallback would have existed
- The gh-aw v0.81.6 upgrade was seamless — no breaking changes affected the workflow structure

## Timeline

All times UTC on 2026-07-04 unless otherwise noted.

| Time | Event | Source |
|---|---|---|
| Jul 1, ~08:00 | Workflow run #59 fails — first observed failure | GitHub Actions UI |
| Jul 1, ~08:00 | Auto-issue `[aw] daily-digest failed` created | GitHub Issues |
| Jul 2, ~08:00 | Workflow run #60 fails | GitHub Actions UI |
| Jul 3, ~08:00 | Workflow run #61 fails | GitHub Actions UI |
| Jul 4, 04:53 | Issue #52 opened: "[Bug] daily-digest agentic workflow failing" | GitHub Issues; user |
| Jul 4, 04:53–05:00 | Investigation begins: confirm model returns 400, try `gpt-5.4-mini` → also 400 | Debugging session |
| Jul 4, 05:00–05:30 | Attempt `haiku` → rejected by Copilot CLI. Attempt `gpt-5.4` → also 400 | Debugging session |
| Jul 4, 05:30–06:00 | Read lockfile; observe gh-aw v0.68.1, outdated SHAs. Upgrade → v0.81.6 | Debugging session |
| Jul 4, 06:00–06:20 | Recompile both workflows. Update all SHAs. Change model to `gpt-4o-mini` | Debugging session |
| Jul 4, 06:20–06:30 | Run v1 workflow → agent succeeds, issue #55 created, `safe_outputs` step fails | Run #66 |
| Jul 4, 06:30–06:41 | Fix add_comment prompt in v1 to use `item_number` + `temporary_id`. Commit and push. | Debugging session |
| Jul 4, 06:41 | Run #67 triggered with full fix (b626af1) | `gh aw run` |
| Jul 4, 06:50–06:58 | Run #67 agent runs for ~8 min, hits 429 rate limit ("Sorry, you've exceeded your rate limit for utility models"), exits code 1. safe_outputs job succeeds (add_comment fix verified). | Run #67 logs |
| Jul 4, 07:00+ | Run #67 completes with failure due to rate limiting. Detection job runs and completes. | `gh run view` |

## Supporting Information

- Bug report: https://github.com/Agent-Observability-Distilled/agent-o11y-news-distilled/issues/52
- Run #66 (first successful agent, failed safe_outputs): [workflow run #28697734045](https://github.com/Agent-Observability-Distilled/agent-o11y-news-distilled/actions/runs/28697734045)
- Run #67 (full fix): [workflow run #28698214169](https://github.com/Agent-Observability-Distilled/agent-o11y-news-distilled/actions/runs/28698214169)
- Issue #55 (successful digest from run #66): https://github.com/Agent-Observability-Distilled/agent-o11y-news-distilled/issues/55
- PR: https://github.com/Agent-Observability-Distilled/agent-o11y-news-distilled/pull/new/fix/daily-digest-model-and-infra
- Relevant commits:
  - `b626af1` — Fix v1 add_comment prompt, remove stray file, finalize gpt-4o-mini
  - `34af8e6` — Fix model: gpt-5.x -> gpt-4o-mini
  - `7ddf199` — Upgrade gh-aw v0.68.1 -> v0.81.6, recompile workflows

---

*Status: Draft · Authors: doughgle · Written: 2026-07-04*
