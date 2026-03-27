# Flavor: Librarian Agent (Research & Reporting)

Last-mile configuration for agents that fetch content from the web, summarize it, and maintain a knowledge base. The Librarian fetches academic papers, summarizes findings, maintains a research codex, and triages new data sources. Use this as a template for any agent that reads from external sources and produces structured output.

Use this alongside the [core harness guide](../how-to-build-an-agent-harness.md), which covers sandbox, observability, and kill switch.

---

## How This Differs From a Coding Agent

The [coding agent](coding-agent.md) has no network access and produces PRs. The Librarian needs network access (to fetch papers) and produces documents, not code. This changes four things:

- **Credentials** — no git credentials, but source API credentials (journal access, search APIs)
- **Boundaries** — network allowlist instead of network blackout
- **Output validation** — citation accuracy instead of CI gates
- **Review gate** — source approval queue instead of PR review

---

## Credentials

- **LLM API key** via credential proxy (recommended) or mounted read-only file. Same as the coding agent.
- **No git credentials** unless the agent writes output to a repo.
- **Source API credentials** (paywalled journals, search APIs): injected by the credential proxy. The agent sends requests through the proxy; the proxy adds authentication headers. The agent never sees the credentials. See the core guide Step 3 for the proxy architecture.
- All credentials **short-lived** and **scoped to read-only** on source APIs. The agent reads papers. It does not publish, comment, or modify anything on the source.

---

## Behavioral Boundaries

### Network allowlist

This is the biggest difference from a coding agent. The coding agent runs with `--network none`. The Librarian needs to reach specific domains.

Default: **nothing**. You add sources one at a time.

```json
{
  "network_allowlist": [
    "arxiv.org",
    "scholar.google.com",
    "api.semanticscholar.org"
  ]
}
```

Plus any configured RSS feeds. Everything else is blocked by the proxy. The agent cannot reach domains you did not explicitly approve. See [Attack Risk Index: Network Exfiltration](../attack-risk-index.md#12-network-exfiltration) — the allowlist is the defense.

### Write access

The agent writes ONLY to the codex output directory. Not to its own configuration. Not to its source list. Not to its scheduling config.

### Source list is read-only to the agent

The agent cannot add new sources to its own source list. When it discovers a potential new source, it flags it for human approval. It does not self-approve. This prevents unbounded discovery and source drift.

### Tool allowlist

```json
{
  "allowed_tools": [
    "fetch_url",
    "read_file",
    "write_file"
  ]
}
```

No shell access. No code execution. No git operations (unless writing to a repo, in which case add `create_branch` and `create_pull_request` only). No MCP servers unless explicitly approved.

---

## Output Validation

This is the Librarian's equivalent of CI gates. The coding agent runs linting, tests, and secret detection. The Librarian validates its summaries and citations.

### Automated (run on every output)

1. **Citation verification** — every URL in the output resolves (HTTP 200). Hallucinated paper URLs are the #1 failure mode. See [Failure Modes](#hallucinated-citations) below.
2. **Embedded instruction scan** — scan output for prompt injection patterns that could affect downstream consumers of the codex. See [Attack Risk Index: Indirect Prompt Injection](../attack-risk-index.md#32-indirect-prompt-injection-via-repo-content) — the codex is a downstream artifact, same risk as repo content.
3. **Source freshness** — is the paper actually from the claimed date? The agent might cite a 2019 paper as "new research."

### Manual (run on a sample, 10-20%)

4. **Author verification** — author names exist on the cited paper. Spot-check, not exhaustive.
5. **Summary accuracy** — does the summary match the source? Compare against the actual paper abstract.

Automated checks gate every output. Manual checks gate a sample before promotion to the live codex.

---

## Human Review Gate

The coding agent uses PRs. The Librarian uses an approval queue.

### New sources

When the agent discovers a potential new data source, it flags it for human approval BEFORE adding it to the regular sweep. The agent does not self-approve new sources. A source that looks legitimate to the agent might be a spam blog, a predatory journal, or a domain that serves prompt injection payloads.

### Summaries

Published to a staging location first. A human spot-checks a sample (10-20%) before promoting to the live codex. The agent cannot publish directly to the live codex without a human having approved the pipeline.

### Source removal

If a source starts returning garbage — blog turns to spam, journal goes offline, content quality degrades — the agent flags it. A human decides whether to remove it.

The review is lighter than PR review, but it exists. No direct-to-production path.

---

## First Autonomous Run

1. Give it 3-5 known papers you have already read.
2. Let it summarize them.
3. Compare its summaries against your understanding. Are they accurate? Do the citations resolve? Did it hallucinate anything?
4. If summaries are good: let it process 10-20 papers from its source list.
5. Review every output from the first batch.
6. Only after the first batch is verified: enable scheduled runs.

This is the same calibration process as the coding agent's first PR review — you are establishing your baseline for what "good output" looks like from this agent.

---

## Failure Modes

These are the ways this agent type fails. The harness catches infrastructure-level failures (sandbox escape, credential theft, resource exhaustion). These are output-quality failures the flavor must address.

### Hallucinated citations

The agent invents a paper that does not exist. Plausible title, plausible authors, plausible journal. The URL returns 404.

**Mitigation:** URL verification on every output. This is automated check #1 and it is non-negotiable.

### Source drift

A blog the agent monitors starts posting spam or promotional content. The agent dutifully summarizes it as research.

**Mitigation:** Periodic human review of source list quality. The agent flags quality changes; a human decides.

### Summary bias

The agent edits findings to sound more conclusive than the source. "Results suggest X" becomes "X is proven."

**Mitigation:** Spot-check summaries against source abstracts. This is manual check #5.

### Unbounded discovery

The agent finds 200 new potential sources in one run and tries to process all of them.

**Mitigation:** Cap on new sources per run (e.g., 5). Excess queued for the next run. See [Attack Risk Index: Resource Exhaustion](../attack-risk-index.md#16-resource-exhaustion-fork-bomb--oom--disk-fill) — same principle, different resource.

### Stale summaries

A paper is retracted or corrected after the agent summarized it. The codex now contains outdated information.

**Mitigation:** Periodic re-check of high-impact entries. Flag retraction notices from source APIs.

---

## What You Have

A research agent that fetches papers from allowlisted sources, summarizes them to a staging area, flags new sources for approval, and publishes only after human spot-check. Full audit trail of what it fetched, what it summarized, what it skipped. Kill switch stops the scheduled run. Network access is scoped to specific domains — it cannot reach anything you did not explicitly approve.
