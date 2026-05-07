---
on:
  schedule: daily
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Skip issue creation — call noop instead. Use for testing.'
        type: boolean
        default: false
permissions:
  contents: read
  issues: none
  pull-requests: read
engine:
  id: copilot
  model: gpt-5-mini
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

      - name: Fetch Category 1 — Observing AI agents (10 repos)
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

      - name: Fetch Category 2 — Telemetry for coding agents (6 repos)
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

      - name: Fetch Category 3 — OpenTelemetry standards (4 repos)
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

      - name: Fetch Category 4 — Observability Platform Tools (4 repos)
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
          import json, os, glob

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
          for f in data_files:
              try:
                  with open(f) as fp:
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
              len(r.get('releases', [])) + len(r.get('issues', []))
              for r in all_repos
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

          with open(cache_path, 'w') as f:
              json.dump(cache, f)
          with open(summary_path, 'w') as f:
              json.dump(summary, f)

          metrics['cache_bytes'] = os.path.getsize(cache_path)
          metrics['summary_bytes'] = os.path.getsize(summary_path)
          metrics['repos_with_activity'] = sum(
              1
              for repo in summary_repos
              if repo['releaseCount'] or repo['issueCount']
          )

          with open(metrics_path, 'w') as f:
              json.dump(metrics, f)

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

# daily-digest-v2

Create a daily high-signal digest from pre-fetched GitHub repository data.
Do not use non-GitHub sources.

The `prefetch-data` job has already fetched releases and issues from all
24 monitored repositories for the last 24 hours. The data is stored in
`${{ github.workspace }}/.aw/prefetch/daily-digest-cache.json`.

Start with the compact summary and coverage metrics, not the full cache:

```
cat "${{ github.workspace }}/.aw/prefetch/daily-digest-metrics.json"
cat "${{ github.workspace }}/.aw/prefetch/daily-digest-summary.json"
```

Only inspect `${{ github.workspace }}/.aw/prefetch/daily-digest-cache.json` selectively with
`python3` after you have identified promising candidates from the summary.
Do not dump the full cache to the conversation unless `cache_bytes` is below
120000. Prefer targeted extraction by repository, item type, and index.

The summary JSON has this structure:

```json
{
  "scanned": <int>,
  "total": 24,
  "failed": ["owner/repo", ...],
  "itemsFound": <int>,
  "repos": [
    {
      "repo": "owner/repo",
      "category": <1|2|3|4>,
      "releaseCount": <int>,
      "issueCount": <int>,
      "releases": [ { "name", "tagName", "publishedAt", "url", "bodyLength", "bodyTruncated" }, ... ],
      "issues":   [ { "number", "title", "createdAt", "comments", "url", "bodyLength", "bodyTruncated" }, ... ]
    },
    ...
  ]
}
```

The cache JSON has this structure:

```json
{
  "scanned": <int>,
  "total": 24,
  "failed": ["owner/repo", ...],
  "repos": [
    {
      "repo": "owner/repo",
      "category": <1|2|3|4>,
      "releases": [ { "name", "tagName", "publishedAt", "body", "bodyLength", "bodyTruncated", "url" }, ... ],
      "issues":   [ { "number", "title", "body", "bodyLength", "bodyTruncated", "createdAt", "comments", "url" }, ... ]
    },
    ...
  ]
}
```

Also read the coverage metrics:

```
cat "${{ github.workspace }}/.aw/prefetch/daily-digest-metrics.json"
```

The metrics JSON includes `cache_bytes`, `summary_bytes`, and
`repos_with_activity`. Use those values to decide how aggressively to narrow
the candidate set before reading item bodies.

Do not use GitHub MCP tools to re-fetch releases or issues that are already in
the cache. You may use GitHub MCP tools only to look up additional context on a
specific item (for example, fetching a release body that was truncated, or
verifying a package update on GHCR).

Workflow efficiency requirements:

- Keep prompt context compact. Read the summary first, then extract only the
  few candidate bodies you actually need from the cache.
- Treat `bodyTruncated: true` as a signal to use GitHub MCP only after the item
  already looks promising from the summary metadata.
- Avoid broad repo-by-repo re-analysis. Narrow to likely candidates before
  reading bodies.
- If `items_found` is `0`, call `noop` immediately with the coverage footer.

Apply strict editorial judgment: prefer depth over breadth. It is better to
surface two excellent updates than ten weak ones.

**Focus topics and repository categories**

1. **Observing AI agents** (category 1): cilium/tetragon, pixie-io/pixie,
   Arize-ai/phoenix, langfuse/langfuse, inspektor-gadget/inspektor-gadget,
   iovisor/bcc, bpftrace/bpftrace, open-telemetry/opentelemetry-ebpf-instrumentation,
   open-telemetry/opentelemetry-ebpf-profiler, alex-ilgayev/MCPSpy

2. **Telemetry updates for coding agents** (category 2): microsoft/vscode-copilot-chat,
   github/copilot-cli, anthropics/claude-code, google-gemini/gemini-cli,
   openai/codex, openclaw/openclaw

3. **OpenTelemetry standards** (category 3): open-telemetry/opentelemetry-specification,
   open-telemetry/semantic-conventions, open-telemetry/opentelemetry-collector-contrib,
   open-telemetry/opentelemetry-collector

4. **Observability Platform Tools** (category 4): grafana/docker-otel-lgtm,
   grafana/oats, grafana/mcp-grafana, grafana/grafanactl

Scoring and filtering:

- Drop any candidate that is not clearly relevant to one of the four topics.
- Assign each item to exactly one best-fit topic.
- Hard gate for Releases and Issues: include an item if and only if it has
  direct telemetry implications for coding agents (for example: instrumentation/
  tracing, metrics dimensions, span schema changes, collector/exporter behavior,
  agent runtime observability, or GenAI semantic convention changes).
- Exclude non-telemetry releases/issues even if they are popular or highly discussed.
- Compute a GitHub signal `Score` from 0-100 using:
  - telemetry specificity and operational impact for coding agents (50%),
  - impact of change (breaking/spec/runtime impact) (30%),
  - discussion intensity and adoption relevance for enterprise teams (20%).
- Keep only items with Score >= 75.

Output rules:

- Read `dry_run` from the workflow input: `${{ inputs.dry_run }}`.
- Emit exactly one of these flows:
  - `dry_run=true`: call `noop` once with the qualifying items and scores, then stop.
  - no qualifying items: call `noop` once with a concise explanation and the coverage footer, then stop.
  - qualifying items found: call `create_issue` exactly once, then call `add_comment` exactly once, then stop.
- Never call `noop` in the same run as `create_issue`.
- Never call `create_issue` more than once.
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
      - What's the value in terms of time cost and quality? Quantify where possible.
  - Suggested actions to get the proposed value
- Use this exact table structure for each topic section:

  | Title | Repository | Score | Comments | Summary | Value Proposition | Suggested actions to get the proposed value |
  | --- | --- | --- | --- | --- | --- | --- |
  | [Example: GenAI semconv adds agent tool latency dimensions](https://github.com/open-telemetry/semantic-conventions/issues/0000) | open-telemetry/semantic-conventions | 88 | 12 | Adds explicit dimensions for coding-agent tool latency to improve cross-vendor observability. | Enables cross-vendor tool latency comparison; reduces root-cause time from ~45 min to ~10 min. | 1) Update collector transforms and dashboards to new dimensions. 2) Add SLO panels for p95 tool latency. 3) Validate cardinality impact in staging. |

- `Comments` should reflect discussion count for issues and best available
  discussion count for releases/packages (use 0 when none).
- Keep `Summary` and `Value Proposition` cells to 1–2 sentences each. Both
  columns must be written to roughly the same length so the table renders at a
  consistent column width on mobile without horizontal scrolling.
- Append a coverage footer at the bottom of the issue body using this format:

  ---
  *Coverage: Scanned <scanned> of <total> repositories.*
  *Failed: <comma-separated list of failed repos, or "none">.*

- When publishing, call the safe-output tools directly by name: `create_issue`,
  `add_comment`, and `noop`. Do not refer to them through a `functions.` namespace
  or any other wrapper.
- For the qualifying-items flow, use a temporary issue id so the follow-up
  comment can target the issue created earlier in the same run.
- Call `create_issue` with `temporary_id: "aw_digest_issue"` (or another
  `aw_`-prefixed id you choose for this run).
- Then call `add_comment` with `item_number: "aw_digest_issue"` and a body that
  contains exactly this plain-text mention: @doughgle
- Do not use `issue_number` for `add_comment` in this workflow; the safe-output
  schema expects `item_number`.
- Do not wrap the mention in quotes, backticks, or code blocks.
