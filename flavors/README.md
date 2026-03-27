# Flavors

The [core harness guide](../how-to-build-an-agent-harness.md) covers universal infrastructure: sandbox, observability, kill switch. Every agent needs these regardless of what it does.

Flavors cover the last mile — the parts that vary by agent type:

- What credentials does the agent need?
- What does the tool allowlist look like?
- What does "output validation" mean for this kind of agent?
- What does the human review gate look like in practice?

The core guide tells you to set up CI gates. A flavor tells you *which* gates, with *what* configuration, for *this* kind of agent.

## Available Flavors

| Flavor | For agents that... | Status |
|--------|-------------------|--------|
| [Coding Agent](coding-agent.md) | Write code, submit PRs, interact with git and CI | Available |
| [Librarian Agent](librarian-agent.md) | Fetch, summarize, and triage research | Available |

## How to Use

1. Build the full harness using the core guide (Steps 1-2, 5, 8-9 are universal).
2. Pick the flavor closest to your agent type.
3. Use the flavor to fill in the agent-specific details: credentials, boundaries, gates, review process.

If no flavor matches exactly, use one as a template and adapt. The structure is the same — credentials, boundaries, gates, review — even if the specifics differ.
