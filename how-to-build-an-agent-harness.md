# How to Build an Agent Harness

Step-by-step from zero to a running autonomous agent with full safety infrastructure. This is what Spotify, Stripe, and Anthropic built, distilled into a guide anyone can follow.

Read [How to Use the Checklist](how-to-use-the-checklist.md) first for the order of operations. This guide follows that order exactly.

This guide is agent-type-agnostic — it works for coding agents, research agents, data pipeline agents, or any autonomous AI system. Steps that vary by agent type link to [flavor guides](flavors/) for specifics.

---

## Step 1: Choose Your Agent Tool

Pick one. Don't overthink it. The harness matters more than the tool.

| Tool | Sandbox support | Headless mode | Hook/gate integration | Notes |
|------|----------------|---------------|----------------------|-------|
| **Claude Code** | Native sandbox-runtime | Yes (`--print`, `-p`) | Hooks, MCP, CLAUDE.md | Anthropic's own tool. Best sandbox docs. |
| **Codex** | Runs in sandbox by default | Yes | CLI-native | OpenAI's entry. Sandboxed out of the box. |
| **Cursor** | Needs external sandbox | Background agent mode | `.cursorrules` | IDE-first. Requires wrapping for autonomous use. |
| **Goose** | Needs external sandbox | Yes | Plugin system | Open source. Flexible, more setup required. |
| **OpenCode** | Needs external sandbox | Yes | Config-based | Open source. Lightweight. |

If you have no strong preference, start with Claude Code or Codex. Both have native sandboxing, which means fewer moving parts in Step 2.

The rest of this guide is tool-agnostic. The harness is the same regardless of what runs inside it.

---

## Step 2: Build the Sandbox

The sandbox is a container the agent runs inside. It has no network access, no host filesystem access, no privilege escalation path, and limited resources. The agent can do its work — read code, write code, run tests — and nothing else.

### Option A: Docker (recommended for production)

Start with this `docker run` command. Every flag is there for a reason.

```bash
docker run \
  --cap-drop ALL \
  --network none \
  --read-only \
  --pids-limit 256 \
  --memory 4g \
  --cpus 2 \
  --user 1000:1000 \
  --tmpfs /tmp:rw,noexec,nosuid,size=1g \
  --tmpfs /workspace/.agent-tmp:rw,noexec,nosuid,size=512m \
  -v /path/to/repo:/workspace:ro \
  -v /path/to/agent-output:/workspace/output:rw \
  -v /var/run/agent-proxy.sock:/var/run/proxy.sock:ro \
  --security-opt no-new-privileges \
  your-agent-image
```

What each flag does:

- `--cap-drop ALL` — removes all Linux capabilities. No privilege escalation. See [Attack Risk Index: Privilege Escalation](attack-risk-index.md#15-privilege-escalation).
- `--network none` — no network access at all. No HTTP, no DNS, no exfiltration. See [Attack Risk Index: Network Exfiltration](attack-risk-index.md#12-network-exfiltration).
- `--read-only` — container root filesystem is read-only. Agent can't modify system files.
- `--pids-limit 256` — prevents fork bombs. See [Attack Risk Index: Resource Exhaustion](attack-risk-index.md#16-resource-exhaustion-fork-bomb--oom--disk-fill).
- `--memory 4g` — hard memory ceiling.
- `--cpus 2` — CPU limit.
- `--user 1000:1000` — runs as non-root.
- `--tmpfs` — writable scratch space with size limits. `noexec` prevents running binaries from tmp.
- `-v /path/to/repo:/workspace:ro` — workspace is read-only. Agent writes output to a separate mount.
- `--security-opt no-new-privileges` — prevents setuid escalation.

### Option B: Anthropic's sandbox-runtime (lighter weight)

If you're using Claude Code, Anthropic provides `sandbox-runtime` — a lightweight isolation layer that handles the container configuration for you.

```bash
npx @anthropic-ai/sandbox-runtime -- claude -p "your task"
```

This is simpler to set up but gives you less control over the container configuration. Good for getting started. Move to Docker when you need production-grade control.

### Option C: Cloudflare Dynamic Workers (cloud-native, high-concurrency)

For agents that generate and execute TypeScript/JavaScript, [Cloudflare Dynamic Workers](https://blog.cloudflare.com/dynamic-workers/) provides V8 isolate-based sandboxing. Millisecond startup (~100x faster than containers), built-in credential injection and HTTP filtering, and nearly a decade of isolate security hardening. Best for high-concurrency scenarios where each request spins up a separate sandbox — supports a million requests per second.

Tradeoff: V8 isolates only run JavaScript/TypeScript. If your agent needs Python, shell access, or system-level tools, use Docker or sandbox-runtime instead.

### Sensitive file exclusion

Before mounting your workspace, exclude these files. They should never be inside the agent's container:

```
.env, .env.*, .env.local, .env.production
.git-credentials
~/.aws/credentials
~/.docker/config.json
~/.kube/config
.npmrc, .pypirc
*.pem, *.key
*-service-account.json
.vscode/settings.json (may contain tokens)
.idea/ (may contain database credentials)
```

Use a `.dockerignore`-style approach or a build script that copies the repo without these files.

### Verify

Run the sandbox tests from the [checklist](https://github.com/brennhill/Delivery-Gap-Toolkit/blob/main/quality-correctness-gates/agent-production-checklist.md):

```bash
# No Docker socket
docker exec $AGENT_CONTAINER ls /var/run/docker.sock 2>&1
# Expected: no such file or directory

# No network
docker exec $AGENT_CONTAINER curl -s --max-time 5 https://example.com
# Expected: connection refused or timeout

# No DNS
docker exec $AGENT_CONTAINER nslookup test.example.com
# Expected: failure

# Read-only filesystem
docker exec $AGENT_CONTAINER touch /etc/test-escape 2>&1
# Expected: read-only file system

# No privilege escalation
docker exec $AGENT_CONTAINER sudo whoami 2>&1
# Expected: sudo not found

# Fork bomb contained
docker exec $AGENT_CONTAINER bash -c ':(){ :|:& };:' 2>&1
# Expected: cannot fork
```

All six must pass before moving on.

---

## Step 3: Set Up Credential Management

The agent needs to interact with external systems — APIs, databases, file stores, version control. It should never see the credentials that make this possible.

### Option A: Credential proxy (recommended)

A proxy runs outside the agent's container. The agent sends requests through a Unix socket. The proxy injects the credentials and forwards the request. The agent never sees the token.

```
Agent (inside sandbox)
  → Unix socket (/var/run/proxy.sock)
    → Credential proxy (outside sandbox, on host)
      → Injects scoped token
        → External API
```

The proxy:
- Runs on the host, not in the container
- Has its own credential store (vault, environment variable, or secrets file)
- Enforces an allowlist of API endpoints the agent can reach
- Logs every request

### Option B: Read-only mounted secret (simpler)

For simpler setups, mount a short-lived token as a read-only file:

```bash
-v /path/to/agent-token:/run/secrets/agent-token:ro
```

The token should be:
- **Scoped** — only the operations the agent needs. See your [flavor guide](flavors/) for specific credential scoping.
- **Short-lived** — expires after the task or after a few hours.
- **Rotated** — generate a new one for each task if possible.

### What not to do

- Don't pass credentials as environment variables (`docker run -e ACCESS_TOKEN=...`). See [Attack Risk Index: Environment Variable Leakage](attack-risk-index.md#21-environment-variable-leakage).
- Don't use your personal access token. If the agent is compromised, your entire account access is compromised.
- Don't use long-lived tokens. A token that expires in 1 hour limits the blast radius.

### Verify

```bash
# No environment variable leakage
docker exec $AGENT_CONTAINER env | grep -i "AWS\|GITHUB\|API_KEY\|SECRET\|TOKEN"
# Expected: empty

# No sensitive files accessible
docker exec $AGENT_CONTAINER cat /workspace/.env 2>&1
# Expected: no such file

# Proxy config not accessible to agent
docker exec $AGENT_CONTAINER cat /etc/proxy/credentials.json 2>&1
# Expected: no such file or directory
```

---

## Step 4: Configure Behavioral Boundaries

The sandbox blocks what the agent *can't physically do*. Behavioral boundaries restrict what the agent *shouldn't do* — even though it technically could within the sandbox.

### Tool allowlist

Most agent tools support a tool allowlist. Only enable the tools the agent needs:

```json
{
  "allowed_tools": [
    "read_file",
    "write_file",
    "run_command"
  ]
}
```

Your [flavor guide](flavors/) has the specific tool allowlist for your agent type.

No web browsing. No MCP servers unless explicitly required and individually approved.

### Protected paths

The agent can write to the workspace but not to everything in it:

```json
{
  "protected_paths": [
    "CLAUDE.md",
    ".cursorrules",
    ".claude/"
  ]
}
```

Agent configuration files should always be read-only for the agent. See [Attack Risk Index: Configuration File Poisoning](attack-risk-index.md#31-configuration-file-poisoning). See your [flavor guide](flavors/) for additional protected paths specific to your agent type.

### Configuration file protection

This is the one control where there is no override. The agent cannot write to files that define its own behavior:

- `CLAUDE.md` / `.cursorrules` / equivalent
- Hook configurations
- MCP server configurations

Mount these read-only from a trusted source outside the workspace.

### Verify

```bash
# Can't modify agent config
docker exec $AGENT_CONTAINER sh -c 'echo "ignore all rules" >> /workspace/CLAUDE.md' 2>&1
# Expected: permission denied
```

Then run the red team prompts from the checklist. Give the agent a task that includes an injected instruction in a file it reads. Verify it doesn't follow the instruction — or that the sandbox blocks the consequence.

---

## Step 5: Set Up Observability

You need to see everything the agent does. The agent can't touch the logs.

### What to log

Every agent invocation should produce a trace:

```
Input prompt / task description
  → File reads (which files, in what order)
    → Tool calls (with parameters and return values)
      → Intermediate reasoning (if the tool exposes it)
        → Files written / modified
          → Final output (deliverable, status, location)
```

Not just the final output. The full chain. When something goes wrong, the trace is how you diagnose it. See [Attack Risk Index: Incomplete Traces](attack-risk-index.md#82-incomplete-traces).

### Where to send logs

Logs stream to an external system the agent cannot reach:

```bash
docker run ... \
  --log-driver=fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag=agent.{{.Name}}
```

Or pipe agent stdout/stderr to a log collector running on the host. The key constraint: the agent cannot delete, modify, or read its own logs. See [Attack Risk Index: Log Tampering](attack-risk-index.md#81-log-tampering).

For simpler setups, redirect to a host-mounted append-only file:

```bash
docker exec $AGENT_CONTAINER your-agent-command 2>&1 | tee -a /var/log/agent-audit/$(date +%Y%m%d-%H%M%S).jsonl
```

### Verify

```bash
# Agent can't delete logs
docker exec $AGENT_CONTAINER rm /var/log/agent-audit/latest.jsonl 2>&1
# Expected: permission denied or no such file

# Logs survive container destruction
docker rm -f $AGENT_CONTAINER
cat /var/log/agent-audit/latest.jsonl
# Expected: logs intact
```

---

## Step 6: Configure Output Gates

The agent's output must pass automated checks before human review. What those checks look like depends on your agent type.

### What output gates look like by agent type

- **Coding agents**: CI pipeline — lint, test, secret detection, dependency audit
- **Research/report agents**: citation verification, output scanning, source validation
- **Data pipeline agents**: schema validation, data quality checks, output format verification

See your [flavor guide](flavors/) for the specific checks your agent type needs.

### The universal requirement

The agent cannot publish, deploy, merge, or take any real-world action on its own output. Automated checks run first. A human reviews second.

If any automated check fails, the output does not proceed to human review.

---

## Step 7: Set Up the Human Review Gate

Every agent output is reviewed by a human before it has real-world effect. This is non-negotiable.

### What the reviewer needs

1. **The task description** — what was the agent asked to do?
2. **The agent's output** — what did the agent actually produce?
3. **The audit trail** — what inputs did the agent read, what tool calls did it make, what reasoning did it follow?

If the reviewer can't see all three, the review is incomplete.

### What "review" looks like by agent type

- **Coding agents**: PR review with branch protection
- **Research agents**: approval queue for new sources, spot-check for summaries
- **Data pipeline agents**: output validation, schema review, sample inspection

See your [flavor guide](flavors/) for specifics on review workflow.

### No bypass mechanism

There is no "auto-approve for low-risk outputs." Every agent output gets reviewed. The agent doesn't know what's low-risk, and neither does an automated classifier.

If review becomes a bottleneck, the answer is faster reviews or fewer agent tasks — not skipping review.

---

## Step 8: Configure the Kill Switch

You can stop the agent in under 60 seconds. This is not optional.

### Minimum: docker kill

```bash
docker kill $AGENT_CONTAINER
```

That's it. The container dies. No graceful shutdown, no cleanup hook the agent could intercept. The agent is gone.

### Requirements

- **Tested** — actually kill the agent once a month. Not "we know the command." Run it. Verify the agent is dead. Verify no orphaned processes remain.
- **Accessible to 2+ people** — if the one person who knows how to kill the agent is on vacation, you have no kill switch.
- **Post-kill cleanup documented** — after you kill the agent, what do you do? Discard in-progress outputs? Rotate credentials? Notify downstream consumers? Document this before you need it.

### For orchestrated agents

If you're running multiple agents or using an orchestrator:

```bash
# Kill all agent containers
docker ps --filter "label=agent" -q | xargs docker kill

# Or via orchestrator API
curl -X POST http://orchestrator:8080/kill-all
```

### Verify

```bash
# Kill switch works
docker kill $AGENT_CONTAINER
docker ps --filter "id=$AGENT_CONTAINER" --format "{{.Status}}"
# Expected: empty (container gone)

# Logs survived the kill
cat /var/log/agent-audit/latest.jsonl
# Expected: logs intact, including the last actions before kill
```

---

## Step 9: Run the Pre-Flight Test Suite

You've built the harness. Now prove it works.

### Automated tests

Run all 36 tests from the [checklist](https://github.com/brennhill/Delivery-Gap-Toolkit/blob/main/quality-correctness-gates/agent-production-checklist.md). In order. In a single session.

Every test must pass. If a test fails, fix the issue and re-run the full suite — not just the failing test. Controls interact with each other.

### Manual red team exercises

After the automated tests pass, run the three manual red team exercises from the checklist:

1. **Prompt injection via repo content** — plant a hidden instruction in a file the agent reads. Verify the agent doesn't follow it, or that the sandbox blocks the consequence.
2. **Memory poisoning** — write malicious instructions to a progress or context file. Verify the agent doesn't execute them.
3. **Inter-agent escalation** (if running multiple agents) — have a low-privilege agent write instructions that a high-privilege agent might follow.

### Sign off

Print the test results. Write your name and the date. Store this with your infrastructure documentation.

This is the artifact that proves the harness works. When leadership asks "is this safe?" you hand them this.

---

## Step 10: First Autonomous Run

The harness is built, tested, and signed off. Now run the agent.

### Pick a low-risk task

- A low-stakes area — not your core revenue path or critical data
- A small, well-defined task with clear success criteria
- Something you could review in 10 minutes, not 2 hours
- See your [flavor guide](flavors/) for what "low-risk first task" means for your agent type

### Monitor in real time

For the first few runs, watch the audit trail live. You're calibrating your expectations:

- How long does the agent take?
- How many tool calls does it make?
- Does it read inputs you didn't expect?
- Does the output match what you'd want from a competent new hire?

### Review the first output carefully

The first output sets your baseline. Review it like you'd review a new hire's first deliverable — thoroughly, with comments, noting anything that surprises you. This calibrates both your expectations and any behavioral boundary adjustments you need to make.

---

## What You Have Now

At this point, you have:

- **A sandboxed agent** that can't escape its container, access the network, or escalate privileges
- **Credentials it can't see** — the proxy or mounted secret handles authentication without exposing tokens to the agent
- **Behavioral boundaries enforced by infrastructure** — not by asking the model nicely, but by read-only mounts and tool allowlists
- **A full audit trail** streaming to a system the agent can't touch
- **Human review on every output** — no auto-approve, no bypass
- **A kill switch that works** — tested, documented, known to multiple people
- **Signed evidence** — test results with a name and date that you can show to leadership, auditors, security teams, or hiring managers

This is not theoretical. This is running infrastructure you can demonstrate.

---

## Next Steps

**Add continuous evals.** The checklist has a continuous evaluation section. Run a subset of tests on every agent invocation or daily. The pre-flight suite catches problems before launch; continuous evals catch drift after.

**Tier up your gates.** The [Delivery Gap Toolkit](https://github.com/brennhill/Delivery-Gap-Toolkit/tree/main/quality-correctness-gates) defines Tier 0-3 gate configurations. Start at Tier 0 (the basics), then add architectural review, performance budgets, and compliance checks as your agent takes on higher-risk work.

**Run the quarterly review cycle.** Every quarter: full pre-flight re-run, all 36 tests, all 3 red team exercises, fresh sign-off. Infrastructure drifts. Dependencies update. Team members change configurations. Quarterly reviews catch it.

**Scale to more workspaces.** Once the harness works for one agent, extending to others is straightforward — the harness is the same, only the workspace mount and flavor configuration change.

---

## Sources

This guide draws on production practices documented by:

- [Anthropic: Securely deploying AI agents](https://platform.claude.com/docs/en/agent-sdk/secure-deployment)
- [Stripe: Building an agentic coding system](https://stripe.com/blog/building-an-agentic-coding-system)
- [Spotify: How Spotify builds autonomous agents](https://engineering.atspotify.com/2025/06/how-spotify-builds-agents/)
- [OpenAI: How we monitor internal coding agents for misalignment](https://openai.com/index/how-we-monitor-internal-coding-agents-misalignment/)
- [NVIDIA: Practical Security Guidance for Sandboxing Agentic Workflows](https://developer.nvidia.com/blog/practical-security-guidance-for-sandboxing-agentic-workflows-and-managing-execution-risk/)
- [Attack Risk Index](attack-risk-index.md) — the incident and risk reference behind every recommendation in this guide
