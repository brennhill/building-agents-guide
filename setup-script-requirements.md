# Setup Script Requirements — The Consumer Path

This document specifies the requirements for a setup script that gets a user from zero to a running, locked-down agent harness in under an hour. The script is the "consumer path" — for people who want the harness without understanding every control. The professional path is the guide + checklist.

**This script does not exist yet. This document is the spec for building it.**

---

## Prerequisites the Script Must Handle

- **Detect OS:** Linux or Mac. Error clearly on Windows with `"Windows is not supported. Use WSL2 or a Linux VM."`
- **Check for Docker:** If missing, offer to install (`apt-get` on Linux, `brew` on Mac). If install fails, clear error with manual install link.
- **Check for git:** If missing, error with install instructions.
- **Prompt for LLM API key:** `"Which provider? (anthropic/openai/other)"` → prompt for key → validate with a test API call → store securely (not in env vars).
- **Check for agent tool:** If missing, recommend Claude Code or Codex with install command.

---

## What the Script Creates

Directory structure:

```
.agent-harness/
├── config.yaml          # Capability config (default: everything off)
├── kill-agent.sh        # Kill script
├── run-agent.sh         # Launcher (always runs pre-flight)
├── preflight.sh         # Pre-flight test runner
├── audit-logs/          # Log destination (host-side, agent can't touch)
├── output/              # Agent output directory
├── rollback/            # Rollback copies of generated configs
└── .gitignore           # Ignore logs, API keys, secrets
```

---

## Capability Config Format (config.yaml)

```yaml
# Agent Capabilities Configuration
# DEFAULT: Everything is off. Uncomment to enable.
# WARNING: Dangerous combinations trigger confirmation prompts.

agent:
  tool: "claude-code"       # claude-code, codex, goose, opencode, cursor
  workspace: "/path/to/workspace"

# Network access (DEFAULT: OFF — agent has no internet)
# network:
#   enabled: false
#   allowlist: []           # Domains the agent can reach
#                           # e.g., ["api.github.com", "arxiv.org"]

# Write access outside container (DEFAULT: OFF)
# write_external:
#   enabled: false
#   paths: []               # Host paths the agent can write to
#                           # e.g., ["/path/to/output"]

# Credentials (DEFAULT: NONE)
# secrets:
#   method: "none"          # "none", "mounted_file", "credential_proxy"
#   path: ""                # Path to secret file or proxy socket

# Human review gate (DEFAULT: ON)
# human_review:
#   enabled: true
#   method: "pr"            # "pr" (coding), "approval_queue" (non-code), "manual_check"

# Tools the agent can use (DEFAULT: NONE beyond read/write)
# tools:
#   allowlist: []           # e.g., ["git", "npm", "python3", "fetch_url"]

# Resource limits
resource_limits:
  memory: "4g"
  cpus: 2
  disk: "1g"
  timeout_minutes: 30
  max_iterations: 50
  pids_limit: 256

# Audit logging (DEFAULT: ON)
audit:
  enabled: true
  destination: "local"      # "local", "s3", "loki", "datadog"
  retention_days: 90

# Kill switch (DEFAULT: ON, cannot be disabled)
kill_switch:
  enabled: true             # This line is ignored. Kill switch is always on.
```

---

## Dangerous Combination Warnings

The script parses config at launch and checks for these combinations. If triggered, display a WARNING and require explicit `y` confirmation:

1. **`write_external.enabled: true` AND `human_review.enabled: false`**
   → `"DANGER: Agent can modify external systems with NO human review. This means the agent's output takes effect without anyone checking it. ARE YOU SURE? (y/N)"`

2. **`network.enabled: true` AND `secrets.method != "none"`**
   → `"DANGER: Agent has network access AND can read credentials. It could exfiltrate secrets to any allowed domain. ARE YOU SURE? (y/N)"`

3. **`network.enabled: true` AND `write_external.enabled: true`**
   → `"DANGER: Agent can reach the internet AND write to external systems. Combined with any vulnerability, this is full compromise. ARE YOU SURE? (y/N)"`

4. **`audit.enabled: false` AND (`write_external.enabled: true` OR `network.enabled: true`)**
   → `"DANGER: Agent can modify systems or reach the network with NO audit trail. If something goes wrong, you won't know what happened. ARE YOU SURE? (y/N)"`

5. **`resource_limits.timeout_minutes > 60` AND kill switch (always on, but warn about long runs)**
   → `"WARNING: Agent can run for over an hour. Monitor the audit trail. Confirm? (y/N)"`

6. **Count of enabled capabilities >= 5**
   → `"WARNING: You've enabled many capabilities on this agent. Each one expands what the agent can do if compromised. Review each one. Continue? (y/N)"`

---

## Launcher Behavior (run-agent.sh)

Every launch follows this sequence:

1. Parse `config.yaml` — validate format, error on invalid YAML with line number.
2. Check for dangerous combinations — prompt if triggered, abort on `N`.
3. Run `preflight.sh` against current config — all tests must pass.
4. If any pre-flight test fails: show which test, show expected vs actual, STOP. Do not launch.
5. If all pass: build Docker run command from config, launch agent in container.
6. Stream audit logs to `audit-logs/` directory.
7. On completion/timeout/kill: destroy container, preserve logs and output.
8. Print summary: task status, duration, token usage (if available), log location.

---

## Pre-Flight Test Runner (preflight.sh)

Runs a subset of the 36 tests from the checklist, appropriate to the current config:

- **Always run:** sandbox tests (network, filesystem, privilege escalation, resource limits)
- **If `secrets` configured:** run credential tests
- **If `network` enabled:** verify allowlist works (can reach allowed domains, can't reach others)
- **If `write_external` enabled:** verify write paths are correct and scoped
- **If `audit` enabled:** verify logs are being captured and agent can't touch them

Output: pass/fail per test, total score, timestamp. Save to `audit-logs/preflight-YYYY-MM-DD.log`.

---

## Kill Script (kill-agent.sh)

```bash
#!/bin/bash
# Kill all agent containers immediately
docker ps --filter "label=agent-harness" -q | xargs -r docker kill
echo "All agent containers killed."
echo "Check audit-logs/ for the final state."
echo "Next steps:"
echo "  1. Review the last audit log entry"
echo "  2. Check for any partial output in output/"
echo "  3. If the agent had write_external access, verify external state"
```

---

## Gate Installation (Language-Aware)

The script detects the primary language from the workspace:

- `package.json` or `tsconfig.json` → TypeScript/JavaScript
- `pyproject.toml` or `setup.py` or `requirements.txt` → Python
- `go.mod` → Go
- `pom.xml` or `build.gradle` → JVM

Then offers to install Tier 0 gates from the Delivery Gap Toolkit quick-starts. For unknown languages: offer to web search for best options, warn user the results are not curated.

---

## What the Script Does NOT Do

- Does not install the agent tool itself (user picks their own).
- Does not configure the agent's task or prompt (that's the agent's domain).
- Does not set up CI/CD (assumes it exists for coding agents, or uses the approval queue for non-code agents).
- Does not handle Windows.
- Does not guarantee security — it provides a strong default that the user is responsible for maintaining.

---

## Testing the Script Itself

Before releasing the setup script, it must be tested on:

- [ ] Fresh Ubuntu 24.04 (no Docker pre-installed)
- [ ] Fresh macOS (no Docker pre-installed)
- [ ] Machine with Docker already installed
- [ ] Machine with existing `.agent-harness/` directory (upgrade path)
- [ ] Invalid config.yaml (should error clearly)
- [ ] Every dangerous combination (should trigger appropriate warning)
- [ ] Pre-flight failure (should stop launch with clear error)
