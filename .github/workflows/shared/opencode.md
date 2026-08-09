---
runtimes:
  node:
    version: "22"
pre-agent-steps:
  - name: Install OpenCode CLI
    run: |
      npm install -g "opencode-ai@$GH_AW_ENGINE_VERSION"
      opencode --version
engine:
  id: opencode
  version: "1.18.15"
  display-name: OpenCode
  description: OpenCode CLI with headless mode and multi-provider LLM support
  experimental: true
  mcp: false
  provider:
    name: github
  auth:
    - role: api-key
      secret: OPENCODE_GO_API_KEY
  behaviors:
    secret-strategy: universal-llm-consumer
    supported-env-var-keys:
      - OPENAI_API_KEY
      - OPENAI_BASE_URL
      - OPENCODE_MODEL
      - XDG_DATA_HOME
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
        - api.github.com
        - objects.githubusercontent.com
        - opencode.ai
        - models.dev
      provider-domains:
        copilot: api.githubcopilot.com
        anthropic: api.anthropic.com
        openai: api.openai.com
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
      write-timestamp: true
      provider-env-mode: universal-llm-consumer
      env:
        XDG_DATA_HOME: /tmp/opencode-data
        OPENAI_BASE_URL: "https://opencode.ai/zen/go/v1"
    harness-script: |
      const { spawnSync } = require("child_process");
      const { readFileSync } = require("fs");

      const [, ...commandArgs] = process.argv.slice(2);

      const log = message => process.stderr.write(`[opencode-harness] ${message}\n`);

      try {
        if (!process.env.OPENAI_API_KEY && process.env.SECRET_OPENCODE_GO_API_KEY) {
          process.env.OPENAI_API_KEY = process.env.SECRET_OPENCODE_GO_API_KEY;
        }

        const promptPath = process.env.GH_AW_PROMPT;
        if (!promptPath) throw new Error("GH_AW_PROMPT is not set");
        const prompt = readFileSync(promptPath, "utf8");

        const result = spawnSync("opencode", [...commandArgs, prompt], {
          encoding: "utf8",
          cwd: process.env.GITHUB_WORKSPACE,
          env: process.env,
          stdio: "inherit",
        });
        if (result.error) throw result.error;
        if (result.status !== 0) process.exitCode = result.status;
      } catch (error) {
        log(error instanceof Error ? error.message : String(error));
        process.exitCode = 1;
      }
    log-parser: |
      function parseLog(logContent) {
        const lines = logContent.split('\n');
        const markdown = [];
        const logEntries = [];
        let mcpFailures = [];
        let maxTurnsHit = false;
        let turnCount = 0;

        for (const line of lines) {
          logEntries.push({
            timestamp: new Date().toISOString(),
            level: 'info',
            message: line
          });
          if (/turn/i.test(line)) turnCount++;
        }

        return {
          markdown: [`**Turns:** ${turnCount}`].join(' · '),
          logEntries,
          mcpFailures,
          maxTurnsHit
        };
      }
---
