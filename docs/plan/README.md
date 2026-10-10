# RedirectUrls — Implementation Plan (v1.1)

Status: **DRAFT, awaiting approval by Moaath**.
Do not start implementation phases until this document is marked APPROVED.

This folder is the hand-off point for the development threads. Each phase in
[`phases.md`](./phases.md) is written so that a fresh thread can pick it up
with only this folder as context. Everything stated as a fact about Safari or
Chrome was checked against a primary source on 2026-10-10; the sources are
listed in [`reference/sources.md`](./reference/sources.md). Statements that
still need a live browser to confirm are marked **[spike]** and collected in
`04-safari.md` §4.6.

| File | Purpose |
|------|---------|
| [`01-overview.md`](./01-overview.md) | Goals, user stories, non-goals, verified browser facts, capability matrix |
| [`02-architecture.md`](./02-architecture.md) | Stack, repo layout, runtime, rule model, precedence semantics, DNR compiler, fallback engine, worked example |
| [`03-ui.md`](./03-ui.md) | Options page, logic designer, popup, validation and copy rules |
| [`04-safari.md`](./04-safari.md) | Packaging (Xcode and App Store Connect), distribution, Safari-specific limits, spike list |
| [`05-testing-ci.md`](./05-testing-ci.md) | Test pyramid, capability probe harness, CI, release |
| [`06-security.md`](./06-security.md) | Threat model and security requirements |
| [`phases.md`](./phases.md) | Ordered phases, dependency lanes, sizes, acceptance criteria |
| [`decisions.md`](./decisions.md) | Decisions made, open questions for the owner, risk register |
| [`reference/`](./reference/) | Sources, Safari header allowlist, other data the code will embed |

## How the pieces fit

```
          Options page (logic designer)            Popup / badge
                     │  save RuleSet                      │ per-site grant, toggle
                     ▼                                    ▼
   ┌───────────────────────────── Background (service worker) ──────────────────────────┐
   │ RuleStore ──► DNF normaliser ──► Compiler ──► { dnrRules, fallbackRules, report }   │
   │                                                │                 │                 │
   │                                                ▼                 ▼                 │
   │                              declarativeNetRequest       Fallback engine           │
   │                              (exact: URL, method,        (best effort: request     │
   │                               resource type, initiator,   headers, body; response  │
   │                               response headers on Chrome) headers on Safari)       │
   └────────────────────────────────────────────────────────────────┬───────────────────┘
                                                                     │ registerContentScripts
                                                   bridge.ts (isolated world, evaluates rules)
                                                   main-world.ts (fetch / XHR / beacon / form shim)
```

## Model split agreed for this project

- Planning: Fable 5.1 (this document).
- Development and problem solving: Opus 5.5 threads, one per phase.
- Sub-agents that research, write or gather content: Sonnet 5.5.
- Sub-agents that run commands, builds, tests and other low-level tasks: Haiku 5.5.

## Changelog

| Version | Date | Change |
|---------|------|--------|
| 1.0 | 2026-10-10 | First draft |
| 1.1 | 2026-10-10 | Verified Safari DNR internals from WebKit source (30,000 shared rule limit, 150,000 content-blocker ceiling, rule expansion per method and domain, `requestDomains` ignored when `regexFilter` is set, regex subset, header allowlist, `regexSubstitution` supported). Added precedence semantics, portable regex subset, Safari expansion budget, worked example, form-submission interception, isolated-world evaluation, security threat model, capability probe harness, dependency lanes, App Store Connect web packager (no Mac needed to package), corrected the CSP claim, renamed converter to packager. |
