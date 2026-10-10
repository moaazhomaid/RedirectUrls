# RedirectUrls — Implementation Plan

Status: **DRAFT, awaiting approval by Moaath** (planned 2026-10-10).
Do not start implementation phases until this document is marked APPROVED.

This folder is the hand-off point for the development threads. Each phase in
[`phases.md`](./phases.md) is written so that a fresh thread can pick it up
with only this folder as context.

| File | Purpose |
|------|---------|
| [`01-overview.md`](./01-overview.md) | Goals, scope, verified browser facts, capability matrix |
| [`02-architecture.md`](./02-architecture.md) | Repo layout, runtime architecture, rule model, DNR compiler, fallback engine |
| [`03-ui.md`](./03-ui.md) | Options page, logic designer, popup, UX rules |
| [`04-safari.md`](./04-safari.md) | Safari packaging, Xcode wrapper, distribution, Safari-specific limits |
| [`05-testing-ci.md`](./05-testing-ci.md) | Test strategy, CI, release process |
| [`phases.md`](./phases.md) | Ordered phases with deliverables and acceptance criteria |
| [`decisions.md`](./decisions.md) | Decisions made during planning and open questions for the owner |

Model split agreed for this project:

- Planning: Fable 5.1 (this document).
- Development and problem solving: Opus 5.5 threads, one per phase.
- Sub-agents that research, write or gather content: Sonnet 5.5.
- Sub-agents that run commands, builds, tests and other low-level tasks: Haiku 5.5.
