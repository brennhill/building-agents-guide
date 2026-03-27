# How to Use the Checklist

The [Agent Production Readiness Checklist](https://github.com/brennhill/Delivery-Gap-Toolkit/blob/main/quality-correctness-gates/agent-production-checklist.md) is your gate between "we have an agent" and "we have an agent we can defend." This guide tells you how to use it.

---

## Before You Start

You need four things:

1. **A workspace** the agent will operate in. A repo, a data directory, a project folder — whatever the agent needs access to.
2. **An AI agent tool** that supports autonomous operation — Claude Code, Cursor, Codex, Goose, or similar.
3. **An output review process.** For coding agents, this means CI/CD. For other agents, this means whatever validation pipeline checks the agent's output before it has real-world effect.
4. **2-3 hours** of uninterrupted time. This is infrastructure work, not a quick install.

If you're missing any of these, fix that first. The checklist assumes they exist.

---

## The Order of Operations

Teams get this backwards. They install the agent, let it run, then bolt on security after the first incident. Do it the other way:

1. **Sandbox** — build the container the agent will run in. No network, no host access, no privilege escalation. The agent doesn't exist yet. You're building the box it will live in.
2. **Permissions** — set up credential management. The agent will need to interact with external systems. It should never see the credentials that make that possible.
3. **Behavioral boundaries** — configure what the agent can and cannot do. Tool allowlists, protected paths, read-only configuration files.
4. **Observability** — set up logging that streams outside the container. You need to see everything the agent does, and the agent can't touch the logs.
5. **Gates** — Automated checks run on every agent output. The agent cannot publish or deploy directly. Human review is required.
6. **Kill switch** — you can stop the agent in under 60 seconds. Two people know how.
7. **The agent** — now you install and configure the agent inside the harness you just built.

The harness comes first. The agent comes last. If you're tempted to skip ahead, read the [Attack Risk Index](attack-risk-index.md) — every entry is what happens when you skip a step.

---

## How to Read the Checklist

The checklist is organized into sections that mirror the order of operations above: Sandbox Isolation, Permission & Credential Management, Behavioral Boundaries, Observability & Audit Trail, CI/CD Gates, Human Review Gate, Kill Switch, and Continuous Evaluation.

Each section has two things:

1. **Checkboxes** — reminders of what to configure. These are for your planning. Check them off as you go.
2. **Runnable tests** — bash commands you execute against your agent's container. These are the real checklist. A checked box with a failing test means nothing. A passing test means the control actually works.

The tests are the source of truth. If every test passes, your harness is solid. If a test fails, it doesn't matter how many boxes are checked.

---

## The Pre-Flight Test Suite

Before your first autonomous run, run all 36 core tests from the checklist. Every one.

Do this in a single session. Print the results. The checklist includes a sign-off section — put your name and the date on it. This is not ceremony for ceremony's sake. When someone asks "who verified the agent infrastructure?" you need an answer.

If any test fails, stop. Fix it. Re-run the full suite, not just the one that failed. Controls interact — fixing a sandbox escape might break a permission test.

The three manual red team exercises are separate from the 36 automated tests. Do them after the automated tests pass. They test whether the agent follows injected instructions, which can't be verified with a bash one-liner.

---

## After Launch

The pre-flight test suite is not a one-time event.

- **Continuous evals** — the checklist has a section on this. Run a subset of tests on every agent invocation or daily, depending on your risk tolerance.
- **Monthly kill switch test** — actually kill the agent. Verify the kill switch works, the cleanup procedure works, and at least two people can execute it. Don't just check the box.
- **Quarterly full re-run** — all 36 tests, all 3 red team exercises, fresh sign-off. Infrastructure drifts. Dependencies update. New team members change configurations. Quarterly re-runs catch the drift.

---

## Common Mistakes

**Jumping straight to the agent.** You install Claude Code, let it run autonomously on your main repo, and plan to "add security later." Later never comes, or it comes after the agent pushes credentials to a public branch.

**Checking boxes without running tests.** The checklist has checkboxes because they're easy to scan. But a checkbox that says "sandbox configured" means nothing if the sandbox test fails. Run the test.

**"We'll add security later."** Security is not a feature you add. It's the harness the agent runs inside. Without it, you don't have an autonomous agent — you have an unmonitored process with your credentials.

**Running the pre-flight once and never again.** Your infrastructure changes. Your Docker base image updates. Someone adds an environment variable. The pre-flight suite catches regressions. Run it quarterly at minimum.

**Treating the checklist as optional for "low-risk" tasks.** The agent doesn't know what's low-risk. It has access to whatever you gave it access to. A "low-risk" task on a repo with production credentials is a high-risk deployment.

---

## Next Steps

- Read the [Attack Risk Index](attack-risk-index.md) to understand *why* each control exists
- Follow [How to Build an Agent Harness](how-to-build-an-agent-harness.md) for the step-by-step implementation
- Open the [checklist](https://github.com/brennhill/Delivery-Gap-Toolkit/blob/main/quality-correctness-gates/agent-production-checklist.md) and start with Step 1: Sandbox
