# Building Agents Guide

![Good Robot vs Bad Robot — the difference is the harness](assets/banner.svg)

A practical, incident-backed guide to building autonomous AI agent infrastructure safely.

This is not theory. Every recommendation is backed by a real incident where the absence of that control caused damage, or by production practices from companies running agents at scale (Spotify, Stripe, Anthropic, OpenAI).

## Who This Is For

- Engineering leaders deploying autonomous agents for the first time
- Developers building agent harnesses (sandbox, gates, permissions, review)
- Anyone evaluating whether their agent infrastructure is production-ready
- Portfolio builders who want to demonstrate they can build agent infrastructure, not just talk about it

## Contents

### [Attack & Risk Index](attack-risk-index.md)
Every risk an autonomous agent poses, explained in depth. For each: what the attack is, a real incident, the defense, the test, and common mistakes. This is the reference encyclopedia behind the [Agent Production Readiness Checklist](https://github.com/brennhill/Delivery-Gap-Toolkit/blob/main/quality-correctness-gates/agent-production-checklist.md).

### How to Use the Checklist *(coming soon)*
The on-ramp. You just cloned a repo, you want to run an autonomous agent — here's the order of operations.

### How to Build an Agent Harness *(coming soon)*
Step-by-step from zero to a running sandboxed agent submitting PRs through gates with full observability.

## Related

- [Delivery Gap Toolkit](https://github.com/brennhill/Delivery-Gap-Toolkit) — the checklist, gate tooling, quick-starts, and AI policy templates
- [The Delivery Gap](https://aiaugmenteddevelopment.com) — the book on AI-assisted software delivery

## License

Apache 2.0. See [LICENSE](LICENSE).
