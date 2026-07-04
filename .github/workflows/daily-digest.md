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
jobs:
  prefetch-data:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      PREFETCH_DIR: /tmp/daily-digest-prefetch
    steps:
      - name: Create output directory
        run: mkdir -p "${PREFETCH_DIR}"

      - name: Set time window
        run: echo "SINCE=$(date -u -d '24 hours ago' '+%Y-%m-%dT%H:%M:%SZ')" >> "$GITHUB_ENV"

      - name: Fetch Category 1 - Observing AI agents (10 repos)
        run: |
          SEARCH_DATE=$(date -u -d '24 hours ago' '+%Y-%m-%d')
          for REPO in \
            "cilium/tetragon" \
            "pixie-io/pixie" \
            "Arize-ai/phoenix" \
            "langfuse/langfuse" \
            "inspektor-gadget/inspektor-gadget" \
            "iovisor/bcc" \
            "bpftrace/bpftrace" \
            "open-telemetry/opentelemetry-ebpf-instrumentation" \
            "open-telemetry/opentelemetry-ebpf-profiler" \
            "alex-ilgayev/MCPSpy"
          do
            OUT="${PREFETCH_DIR}/$(echo "${REPO}" | tr '/' '_').json"
            RELEASES=$(gh api "repos/${REPO}/releases?per_page=20" \
              --jq "[.[] | select(.published_at >= \"${SINCE}\")]" 2>/dev/null) \
              || RELEASES="[]"
            ISSUES=$(gh issue list -R "${REPO}" --state all \
              --search "created:>${SEARCH_DATE}" --limit 30 \
              --json number,title,body,createdAt,comments,url 2>/dev/null) \
              || ISSUES="[]"
            printf '%s' "{\"repo\":\"${REPO}\",\"category\":1,\"releases\":${RELEASES},\"issues\":${ISSUES}}" \
              > "${OUT}" \
              || echo "FAILED:${REPO}" >> "${PREFETCH_DIR}/failures.log"
          done

      - name: Fetch Category 2 - Telemetry for coding agents (6 repos)
        run: |
          SEARCH_DATE=$(date -u -d '24 hours ago' '+%Y-%m-%d')
          for REPO in \
            "microsoft/vscode-copilot-chat" \
            "github/copilot-cli" \
            "anthropics/claude-code" \
            "google-gemini/gemini-cli" \
            "openai/codex" \
            "openclaw/openclaw"
          do
            OUT="${PREFETCH_DIR}/$(echo "${REPO}" | tr '/' '_').json"
            RELEASES=$(gh api "repos/${REPO}/releases?per_page=20" \
              --jq "[.[] | select(.published_at >= \"${SINCE}\")]" 2>/dev/null) \
              || RELEASES="[]"
            ISSUES=$(gh issue list -R "${REPO}" --state all \
              --search "created:>${SEARCH_DATE}" --limit 30 \
              --json number,title,body,createdAt,comments,url 2>/dev/null) \
              || ISSUES="[]"
            printf '%s' "{\"repo\":\"${REPO}\",\"category\":2,\"releases\":${RELEASES},\"issues\":${ISSUES}}" \
              > "${OUT}" \
              || echo "FAILED:${REPO}" >> "${PREFETCH_DIR}/failures.log"
          done

      - name: Fetch Category 3 - OpenTelemetry standards (4 repos)
        run: |
          SEARCH_DATE=$(date -u -d '24 hours ago' '+%Y-%m-%d')
          for REPO in \
            "open-telemetry/opentelemetry-specification" \
            "open-telemetry/semantic-conventions" \
            "open-telemetry/opentelemetry-collector-contrib" \
            "open-telemetry/opentelemetry-collector"
          do
            OUT="${PREFETCH_DIR}/$(echo "${REPO}" | tr '/' '_').json"
            RELEASES=$(gh api "repos/${REPO}/releases?per_page=20" \
              --jq "[.[] | select(.published_at >= \"${SINCE}\")]" 2>/dev/null) \
              || RELEASES="[]"
            ISSUES=$(gh issue list -R "${REPO}" --state all \
              --search "created:>${SEARCH_DATE}" --limit 30 \
              --json number,title,body,createdAt,comments,url 2>/dev/null) \
              || ISSUES="[]"
            printf '%s' "{\"repo\":\"${REPO}\",\"category\":3,\"releases\":${RELEASES},\"issues\":${ISSUES}}" \
              > "${OUT}" \
              || echo "FAILED:${REPO}" >> "${PREFETCH_DIR}/failures.log"
          done

      - name: Fetch Category 4 - Observability Platform Tools (4 repos)
        run: |
          SEARCH_DATE=$(date -u -d '24 hours ago' '+%Y-%m-%d')
          for REPO in \
            "grafana/docker-otel-lgtm" \
            "grafana/oats" \
            "grafana/mcp-grafana" \
            "grafana/grafanactl"
          do
            OUT="${PREFETCH_DIR}/$(echo "${REPO}" | tr '/' '_').json"
            RELEASES=$(gh api "repos/${REPO}/releases?per_page=20" \
              --jq "[.[] | select(.published_at >= \"${SINCE}\")]" 2>/dev/null) \
              || RELEASES="[]"
            ISSUES=$(gh issue list -R "${REPO}" --state all \
              --search "created:>${SEARCH_DATE}" --limit 30 \
              --json number,title,body,createdAt,comments,url 2>/dev/null) \
              || ISSUES="[]"
            printf '%s' "{\"repo\":\"${REPO}\",\"category\":4,\"releases\":${RELEASES},\"issues\":${ISSUES}}" \
              > "${OUT}" \
              || echo "FAILED:${REPO}" >> "${PREFETCH_DIR}/failures.log"
          done

      - name: Aggregate prefetch data
        run: |
          python3 << 'PYEOF'
          import glob
          import json
          import os

          RELEASE_BODY_LIMIT = 4000
          ISSUE_BODY_LIMIT = 2500

          def compact_text(value, limit):
              text = (value or '').strip()
              truncated = len(text) > limit
              if truncated:
                  text = text[:limit].rstrip() + '\n...[truncated]'
              return text, len(value or ''), truncated

          def compact_release(release):
              body, body_length, body_truncated = compact_text(
                  release.get('body'), RELEASE_BODY_LIMIT
              )
              return {
                  'name': release.get('name') or release.get('tag_name') or '',
                  'tagName': release.get('tag_name') or release.get('tagName') or '',
                  'publishedAt': release.get('published_at') or release.get('publishedAt') or '',
                  'body': body,
                  'bodyLength': body_length,
                  'bodyTruncated': body_truncated,
                  'url': release.get('html_url') or release.get('url') or '',
              }

          def compact_issue(issue):
              body, body_length, body_truncated = compact_text(
                  issue.get('body'), ISSUE_BODY_LIMIT
              )
              comments = issue.get('comments', 0)
              if isinstance(comments, dict):
                  comments = comments.get('totalCount', 0)
              return {
                  'number': issue.get('number'),
                  'title': issue.get('title') or '',
                  'body': body,
                  'bodyLength': body_length,
                  'bodyTruncated': body_truncated,
                  'createdAt': issue.get('createdAt') or issue.get('created_at') or '',
                  'comments': comments,
                  'url': issue.get('url') or '',
              }

          prefetch_dir = os.environ['PREFETCH_DIR']
          data_files = sorted(glob.glob(os.path.join(prefetch_dir, '*.json')))
          all_repos = []
          for path in data_files:
              try:
                  with open(path) as fp:
                      raw_repo = json.load(fp)
                      releases = [
                          compact_release(release)
                          for release in raw_repo.get('releases', [])
                      ]
                      issues = [
                          compact_issue(issue)
                          for issue in raw_repo.get('issues', [])
                      ]
                      all_repos.append({
                          'repo': raw_repo.get('repo'),
                          'category': raw_repo.get('category'),
                          'releases': releases,
                          'issues': issues,
                      })
              except Exception:
                  pass

          failures = []
          failures_log = os.path.join(prefetch_dir, 'failures.log')
          if os.path.exists(failures_log):
              with open(failures_log) as fp:
                  for line in fp:
                      line = line.strip()
                      if line.startswith('FAILED:'):
                          failures.append(line[7:])

          total = 24
          scanned = len(all_repos)
          items_found = sum(
              len(repo.get('releases', [])) + len(repo.get('issues', []))
              for repo in all_repos
          )

          summary_repos = []
          for repo in all_repos:
              summary_repos.append({
                  'repo': repo.get('repo'),
                  'category': repo.get('category'),
                  'releaseCount': len(repo.get('releases', [])),
                  'issueCount': len(repo.get('issues', [])),
                  'releases': [
                      {
                          'name': release.get('name'),
                          'tagName': release.get('tagName'),
                          'publishedAt': release.get('publishedAt'),
                          'url': release.get('url'),
                          'bodyLength': release.get('bodyLength'),
                          'bodyTruncated': release.get('bodyTruncated'),
                      }
                      for release in repo.get('releases', [])
                  ],
                  'issues': [
                      {
                          'number': issue.get('number'),
                          'title': issue.get('title'),
                          'createdAt': issue.get('createdAt'),
                          'comments': issue.get('comments'),
                          'url': issue.get('url'),
                          'bodyLength': issue.get('bodyLength'),
                          'bodyTruncated': issue.get('bodyTruncated'),
                      }
                      for issue in repo.get('issues', [])
                  ],
              })

          cache = {
              'scanned': scanned,
              'total': total,
              'failed': failures,
              'repos': all_repos,
          }
          summary = {
              'scanned': scanned,
              'total': total,
              'failed': failures,
              'itemsFound': items_found,
              'repos': summary_repos,
          }
          metrics = {
              'scanned': scanned,
              'total': total,
              'failed': failures,
              'items_found': items_found,
          }

          cache_path = os.path.join(prefetch_dir, 'daily-digest-cache.json')
          summary_path = os.path.join(prefetch_dir, 'daily-digest-summary.json')
          metrics_path = os.path.join(prefetch_dir, 'daily-digest-metrics.json')

          with open(cache_path, 'w') as fp:
              json.dump(cache, fp)
          with open(summary_path, 'w') as fp:
              json.dump(summary, fp)

          metrics['cache_bytes'] = os.path.getsize(cache_path)
          metrics['summary_bytes'] = os.path.getsize(summary_path)
          metrics['repos_with_activity'] = sum(
              1
              for repo in summary_repos
              if repo['releaseCount'] or repo['issueCount']
          )

          with open(metrics_path, 'w') as fp:
              json.dump(metrics, fp)

          print(
              f"Scanned {scanned}/{total} repos, {len(failures)} failed, "
              f"{items_found} items found, cache={metrics['cache_bytes']} bytes, "
              f"summary={metrics['summary_bytes']} bytes"
          )
          PYEOF

      - name: Upload prefetch artifact
        uses: actions/upload-artifact@v4
        with:
          name: daily-digest-prefetch
          path: ${{ env.PREFETCH_DIR }}/
          retention-days: 1

steps:
  - name: Download prefetch data
    uses: actions/download-artifact@v4
    with:
      name: daily-digest-prefetch
      path: ${{ github.workspace }}/.aw/prefetch
---

# daily-digest

Create a daily high-signal digest from public GitHub repositories only.
Do not use non-GitHub sources.
Use the pre-fetched GitHub repository data in this workflow first. Use GitHub
MCP tools only for targeted follow-up on a small number of shortlisted items
when the cached data is insufficient.
Do not fetch `api.github.com`, GitHub HTML pages, or other GitHub-hosted
content over raw HTTP when the same data is available through GitHub tools.

Research recent activity from these sources only:

1. GitHub Releases
2. GitHub Packages (especially GHCR image/package updates)
3. GitHub Issues

The `prefetch-data` job has already fetched releases and issues from all
24 monitored repositories for the last 24 hours. The data is stored in:

- `${{ github.workspace }}/.aw/prefetch/daily-digest-metrics.json`
- `${{ github.workspace }}/.aw/prefetch/daily-digest-summary.json`
- `${{ github.workspace }}/.aw/prefetch/daily-digest-cache.json`

Start with the compact summary and coverage metrics, not the full cache:

```
cat "${{ github.workspace }}/.aw/prefetch/daily-digest-metrics.json"
cat "${{ github.workspace }}/.aw/prefetch/daily-digest-summary.json"
```

Only inspect `${{ github.workspace }}/.aw/prefetch/daily-digest-cache.json`
selectively with `python3` after you have identified promising candidates from
the summary. Do not dump the full cache into the conversation unless
`cache_bytes` is below `120000`. Prefer targeted extraction by repository,
item type, and index.

Fetch efficiency and API hygiene:

- Prefer metadata-first retrieval and only fetch expanded content when a
  release or issue passes the telemetry relevance gate.
- If cached metadata is enough to reject an item, reject it without further
  expansion.
- If a targeted GitHub MCP request fails for a shortlisted item, skip that item
  and continue; do not fall back to raw HTTP requests against GitHub APIs.
- When re-checking release metadata, use conditional requests with
  `If-None-Match` (ETag) and `If-Modified-Since` to avoid repeated full
  payload downloads.
- If the API returns `304 Not Modified`, skip that repo for release
  content processing in this run.
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
- Otherwise create one issue titled "Daily Digest - <date>".
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
- Because this workflow runs on schedule/dispatch (no triggering issue), call `add_comment` with `item_number` set to `"aw_digest_issue"` and a body that contains exactly the plain-text mention.
- Call `create_issue` with `temporary_id: "aw_digest_issue"` so the follow-up comment can target the issue.
- Do not wrap the mention in quotes, backticks, or code blocks.
