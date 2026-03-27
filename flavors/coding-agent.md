# Flavor: Coding Agent

Last-mile configuration for agents that write code, submit PRs, and interact with git/CI. Use this alongside the [core harness guide](../how-to-build-an-agent-harness.md), which covers sandbox, observability, and kill switch.

---

## Credentials

The agent needs to push branches and create PRs. It should never see the credentials that make this possible.

**If using a credential proxy** (recommended): the proxy runs outside the container, injects git auth, and the agent sends requests through a Unix socket. The agent never touches the token.

**If using a mounted secret**: mount a read-only token at `/run/secrets/github-token`.

Either way, the token must be:

- **Scoped** to create branch, create PR, read repo. Nothing else.
- **Short-lived** — 1 hour max. Expires after the task completes.
- **Rotated** — generate a new token for each task.
- **Never a developer's personal access token.** If the agent is compromised, a scoped bot token limits the blast radius. A developer PAT compromises everything that developer can access.

See the core guide Step 3 for the full credential proxy architecture and verification tests.

---

## Behavioral Boundaries

### Tool allowlist

Only enable the tools the agent needs:

```json
{
  "allowed_tools": [
    "read_file",
    "write_file",
    "run_terminal_command",
    "create_branch",
    "create_pull_request"
  ]
}
```

No web browsing. No MCP servers unless explicitly approved and individually reviewed.

### Protected paths

The agent can write to the workspace but not to everything in it:

```json
{
  "protected_paths": [
    ".github/workflows/",
    "CLAUDE.md",
    ".cursorrules",
    ".claude/",
    "*.config.js",
    "*.config.ts"
  ]
}
```

Why these paths matter:

- `.github/workflows/` — the agent could disable its own CI gates. See [Attack Risk Index: Configuration File Poisoning](../attack-risk-index.md#31-configuration-file-poisoning).
- `CLAUDE.md` / `.cursorrules` / `.claude/` — agent configuration files. An agent that can rewrite its own instructions can override every other boundary.
- `*.config.js` / `*.config.ts` — build configurations that affect what gets deployed.

Mount these read-only from a trusted source outside the workspace. This is the one control where there is no override.

---

## Output Gates

These run in CI on every agent PR, same pipeline as human PRs. No shortcuts.

1. **Linting** — agent code meets the same style standards as human code.
2. **Tests** — the existing test suite passes.
3. **Secret detection** — TruffleHog, Gitleaks, or equivalent scans the diff for credentials. See [Attack Risk Index: Secret Exfiltration via PR Diff](../attack-risk-index.md#41-secret-exfiltration-via-pr-diff).
4. **Dependency audit** — `npm audit`, `pip-audit`, `cargo-audit`. Catches malicious or vulnerable packages the agent might introduce. See [Attack Risk Index: Dependency Poisoning](../attack-risk-index.md#42-dependency-poisoning-via-agent-output).

If any gate fails, the PR cannot merge. Same as for human PRs.

### Branch protection

```yaml
# GitHub branch protection (via settings or gh CLI)
# main branch:
#   - Require pull request reviews: 1+
#   - Require status checks to pass: all CI checks
#   - Do not allow bypassing the above settings
#   - Restrict who can push: not the agent's token
```

The agent pushes to feature branches. It creates a PR. It cannot push to main or merge its own PR.

### Label agent PRs

```bash
gh pr create --label "agent-generated" --title "..." --body "..."
```

This lets reviewers immediately see what they're looking at. It also lets you track agent PR metrics later: merge rate, rework rate, time-to-review.

For the full gate setup including Tier 0-3 configurations, see the [Delivery Gap Toolkit quick-starts](https://github.com/brennhill/Delivery-Gap-Toolkit/tree/main/quality-correctness-gates).

---

## Human Review Gate

Every agent PR is reviewed by a human before merge. No exceptions.

### What the reviewer needs

1. **The task description** — what was the agent asked to do?
2. **The full diff** — what did the agent actually change?
3. **The audit trail** — what files did it read, what tool calls did it make, what reasoning did it follow?

If the reviewer can't see all three, the review is incomplete.

### Enforcement

The agent's token cannot merge PRs. The merge action requires a human. Verify this:

```bash
# Using the agent's scoped token:
gh pr merge $PR_NUMBER --merge 2>&1
# Expected: denied
```

### No bypass mechanism

There is no "auto-merge for low-risk changes." There is no "skip review for test-only PRs." The agent does not know what is low-risk, and neither does an automated classifier.

If review becomes a bottleneck, the answer is faster reviews or fewer agent PRs — not skipping review.

---

## First Autonomous Run

The harness is built and tested. Now run the agent on a real task.

### Pick the right target

- A **Tier 1 service** — not your core revenue path
- A **well-tested area** — high test coverage, so the test gate actually catches regressions
- A **small, well-defined task** — not "refactor the auth system"
- Something a developer would review in 10 minutes, not 2 hours

### Monitor in real time

For the first few runs, watch the audit trail live:

- How long does the agent take?
- How many tool calls does it make?
- Does it read files you didn't expect?
- Does the output match what you'd want from a junior developer?

### Review the first PR like a new hire's first PR

Thoroughly. With comments. Note anything that surprises you. This calibrates your expectations and identifies behavioral boundary adjustments you need to make.

---

## What You Have

The agent submits PRs through the same pipeline as humans — same CI gates, same review requirement, same branch protection. The difference: it runs in a sandbox with scoped credentials and a kill switch.

No special path. No reduced scrutiny. The same bar, enforced by infrastructure.
