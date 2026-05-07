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
  model: gpt-5-mini
network: defaults
tools:
  github:
    toolsets: [default]
    min-integrity: none
safe-outputs:
  create-issue:
    max: 1
  add-comment:
    max: 1
    target: "*"
---

# daily-digest

Create a daily high-signal digest from public GitHub repositories only.
Do not use non-GitHub sources.
Use GitHub MCP tools for GitHub repository data collection.
Do not fetch `api.github.com`, GitHub HTML pages, or other GitHub-hosted
content over raw HTTP when the same data is available through GitHub MCP
tools.

Research recent activity from these sources only:

1. GitHub Releases
2. GitHub Packages (especially GHCR image/package updates)
3. GitHub Issues

Fetch efficiency and API hygiene:

- When re-checking release metadata, use conditional requests with
  `If-None-Match` (ETag) and `If-Modified-Since` to avoid repeated full
  payload downloads.
- If the API returns `304 Not Modified`, skip that repo for release
  content processing in this run.
- Prefer metadata-first retrieval and only fetch expanded content when a
  release or issue passes the telemetry relevance gate.
- If a GitHub MCP request fails for a repository, skip that repository and
  continue; do not fall back to raw HTTP requests against GitHub APIs.
Apply strict editorial judgment: prefer depth over breadth. It is better
to surface two excellent updates than ten weak ones.

Use the last 24 hours of activity and evaluate updates only from this
shortlist of public repositories.

**Focus topics and repository shortlist**

1. **Observing AI agents** (monitoring, snooping, eBPF for agent
   workloads)
  - cilium/tetragon
  - pixie-io/pixie
  - Arize-ai/phoenix
  - langfuse/langfuse
  - inspektor-gadget/inspektor-gadget
  - iovisor/bcc
  - bpftrace/bpftrace
  - open-telemetry/opentelemetry-ebpf-instrumentation
  - open-telemetry/opentelemetry-ebpf-profiler
  - alex-ilgayev/MCPSpy

2. **Telemetry updates for coding agents** (GitHub Copilot, Claude Code,
   Gemini CLI, OpenAI Codex, OpenClaw)
  - microsoft/vscode-copilot-chat
  - github/copilot-cli
  - anthropics/claude-code
  - google-gemini/gemini-cli
  - openai/codex
  - openclaw/openclaw

3. **OpenTelemetry standards** (specification and Semantic Conventions
   for GenAI)
  - open-telemetry/opentelemetry-specification
  - open-telemetry/semantic-conventions
  - open-telemetry/opentelemetry-collector-contrib
  - open-telemetry/opentelemetry-collector

4. **Observability Platform Tools**
  - grafana/docker-otel-lgtm
  - grafana/oats
  - grafana/mcp-grafana
  - grafana/grafanactl

Scoring and filtering:

- Drop any candidate that is not clearly relevant to one of the four
  topics.
- Assign each item to exactly one best-fit topic.
- Hard gate for Releases and Issues: include an item if and only if it
  has direct telemetry implications for coding agents (for example:
  instrumentation/tracing, metrics dimensions, span schema changes,
  collector/exporter behavior, agent runtime observability, or GenAI
  semantic convention changes).
- Exclude non-telemetry releases/issues even if they are popular or
  highly discussed.
- Compute a GitHub signal `Score` from 0-100 using:
  - telemetry specificity and operational impact for coding agents (50%),
  - impact of change (breaking/spec/runtime impact) (30%),
  - discussion intensity and adoption relevance for enterprise teams
    (20%).
- Keep only items with Score >= 75.

Output rules:

- If no items qualify after filtering, do nothing and do not create an
  issue.
- Otherwise create one issue titled "GitHub Digest - <date>".
- Use one Markdown table per topic that has qualifying items.
- Keep the same table format for each row:
  - Title (linked)
  - Repository
  - Score
  - Comments
  - Summary
  - Value Proposition 
      - What does it enable people to do?
      - What's the value in terms of time cost and quality? Quantify it where possible e.g. startup time reduced from 60s to 5s
  - Suggested actions to get the proposed value
- Use this exact table structure for each topic section:

  | Title | Repository | Score | Comments | Summary | Value Proposition | Suggested actions to get the proposed value |
  | --- | --- | --- | --- | --- | --- | --- |
  | [Example: GenAI semconv adds agent tool latency dimensions](https://github.com/open-telemetry/semantic-conventions/issues/0000) | open-telemetry/semantic-conventions | 88 | 12 | Adds explicit dimensions for coding-agent tool latency to improve cross-vendor observability. | Enables cross-vendor tool latency comparison; reduces root-cause time from ~45 min to ~10 min. | 1) Update collector transforms and dashboards to new dimensions. 2) Add SLO panels for p95 tool latency. 3) Validate cardinality impact in staging. |
  | Example: New GHCR image release for eBPF observability agent | cilium/tetragon | 82 | 5 | New release of Tetragon with improved eBPF-based observability features for AI agent workloads. | Enables deeper visibility into AI agent behavior with lower overhead; reduces troubleshooting time by ~30%. | 1) Test new release in staging. 2) Update production deployment to new version. 3) Monitor for improvements in observability and troubleshooting efficiency. |

- `Comments` should reflect discussion count for issues and best
  available discussion count for releases/packages (use 0 when none).
- Keep `Summary` and `Value Proposition` cells to 1–2 sentences each. Both columns must be written to roughly the same length so the table renders at a consistent column width on mobile without horizontal scrolling. Do not truncate meaning to match length; instead write each cell with the same level of detail and density as the other.
- When publishing, call the safe-output tools directly by name:
  `create_issue`, `add_comment`, and `noop`. Do not refer to them through
  a `functions.` namespace or any other wrapper.
- If no items qualify, call `noop` with a concise explanation instead of
  drafting issue content.
- After creating the issue, add exactly one issue comment containing a plain-text mention: @doughgle
- Because this workflow runs on schedule/dispatch (no triggering issue), for `add_comment` you MUST set `issue_number` to the issue number returned by the same-run `create_issue` output.
- Do NOT provide `item_number` for this workflow.
- Do not wrap the mention in quotes, backticks, or code blocks.
