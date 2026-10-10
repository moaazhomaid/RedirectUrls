# Phases

Each phase is sized for one Opus 5.5 development thread and ends in one or
more PRs to `main`. Within a thread: Sonnet 5.5 sub-agents for research and
writing content, Haiku 5.5 sub-agents for running commands, builds and
tests. **[Mac]** marks work that needs the owner's Mac (Remote Control
session or manual run); **[ASC]** marks work that can use the App Store
Connect web packager from a cloud thread.

## Dependency lanes

```
P0 bootstrap + probe harness
 ├─► Lane A (engine):   P1 core model + compiler ──► P2 background engine ──► P4 fallback engine
 ├─► Lane B (UI):       P3a rule list + import/export (after P1) ──► P3b logic designer + tester (after P2) ──► P5 popup + polish
 └─► Lane C (Safari):   P6a Xcode wrapper skeleton [Mac] (after P0) ──► P6b store readiness (after P5)
```

Lanes A and B can run in parallel threads once P1 has merged; P6a can run
as soon as P0 produces a Safari build.

| Phase | Size | Depends on | Thread brief (one line) |
|-------|------|------------|-------------------------|
| P0 | M | — | Bootstrap the monorepo, CI and the capability probe harness; record Safari and Chrome capability JSON |
| P1 | L | P0 | Implement the rule model, DNF normaliser, portable regex subset, evaluator and the DNR compiler with golden tests for both targets |
| P2 | M | P1 | Implement the background engine: storage, permissions, atomic DNR apply, budget, match log; Chrome e2e |
| P3a | M | P1 | Build the options app shell, rule list, ordering, import/export, presets, i18n scaffold |
| P3b | L | P2, P3a | Build the logic designer with live fidelity badges, validation and the rule tester |
| P4 | L | P2 | Implement the fallback engine tiers A and B with the isolated-world bridge and form interception |
| P5 | M | P3b, P4 | Popup, badge, blocked page, onboarding, accessibility and RTL pass |
| P6a | S | P0 | Generate and commit the Xcode wrapper, build script, run the Safari spike list [Mac] |
| P6b | M | P5, P6a | Store readiness: privacy manifest, listings, permission justifications, Arabic locale, release workflow, TestFlight build [ASC or Mac] |
| P7 | optional | P6b | Hardening and extras after first release |

## Phase 0 — Repo bootstrap and capability probe (size M)

Deliverables

- pnpm workspace, TypeScript strict, ESLint/Prettier, Vitest, WXT project
  with `chrome` and `safari` targets (`manifestVersion: 3`), Preact +
  Tailwind, `CLAUDE.md` with the conventions from `05-testing-ci.md` §5.5.
- CI workflow (lint, typecheck, test, build both targets, size report).
- `packages/core` skeleton: Zod `RuleSet` schema, `Capabilities` type.
- `packages/probe` harness (§5.2) and `packages/test-server`.
- Probe results for Chrome 155 committed as `core/src/capabilities/chrome.json`
  (run in CI with Playwright). Safari results need P6a **[Mac]**; until then
  `safari.json` carries the values from this plan marked `"source": "plan"`.
- ADRs for D1–D13 copied from `decisions.md`.

Acceptance: `pnpm build` produces loadable Chrome and Safari packages; CI
green; `chrome.json` committed; spike questions S1–S7 listed as issues.

## Phase 1 — Core model, normaliser, regex subset, evaluator, compiler (size L)

- Full `ConditionNode` / `Rule` / `Action` schemas, size limits (T1),
  `validateTarget()` (T4), migrations scaffold.
- DNF normaliser with property tests.
- Portable regex subset checker and `urlFilter` builder; escaping rules;
  the "trailing `^` or end" split for Safari.
- Evaluator `evaluate(tree, requestContext)`.
- Compiler to DNR for `chrome` and `safari` targets: leaf mapping table,
  host folding, band priorities, loop protection, budget calculator with
  Safari expansion formula, fidelity report, deterministic ids.
- Golden fixtures covering every leaf type, the worked example (§2.9), and
  at least one 64-disjunct refusal.

Acceptance: fixtures pass for both targets; property tests pass; `core` has
no browser imports (enforced by an ESLint rule).

## Phase 2 — Background engine and storage (size M)

- Storage adapter and `RuleStore`; `storage.onChanged` → normalise → compile
  → budget check → atomic `updateDynamicRules`; startup reconciliation (T11).
- Capability detection (`capabilities.ts`): static JSON + runtime probes.
- Permission manager: origin + destination patterns per rule, request on
  save, recompile on `permissions.onAdded/onRemoved`.
- Match log ring buffer (`declarativeNetRequestFeedback` when granted;
  `webRequest.onCompleted` correlation otherwise); badge counts via
  `setExtensionActionOptions`.
- Playwright e2e against the test server: redirect by host, method, query
  param, transform, regex substitution; block; modify request header; loop
  protection; budget refusal; permission revocation.

Acceptance: e2e green on Chrome in CI; Safari smoke when P6a exists **[Mac]**.

## Phase 3a — Options app shell, rule list, import/export (size M)

App shell, routing, rule list with ordering (drag + buttons), enable/disable,
duplicate, delete, search, budget meter, import/export (file and share
string, with the T10 redaction prompt), presets, settings, i18n scaffold,
dark mode, RTL-safe layout. Component tests.

## Phase 3b — Logic designer and rule tester (size L)

Nested groups with AND/OR/NOT, field/operator pickers, per-leaf and per-rule
fidelity badges from the compiler, regex validation (syntax + portable
subset), header-name validation against the Safari allowlist, DNF size
indicator, action editor with capture pickers and transform fields, loop
warning, "Test against URL" panel (evaluator; `testMatchOutcome` on Chrome).

Acceptance: component tests; Playwright flow "create the worked-example
rule in the designer → redirect happens → tester explains it"; works at
390 px width with touch ordering.

## Phase 4 — Fallback engine (size L, two PRs)

- PR 4a: Tier A (observe `webRequest` → evaluator → `tabs.update` /
  frame `location.replace` / blocked page), limitation copy in the UI.
- PR 4b: Tier B (`bridge.ts` isolated world holding rules and evaluating;
  `main-world.ts` shim for fetch, XHR, sendBeacon and form submit; nonce'd
  `CustomEvent` RPC; dynamic `registerContentScripts` only for granted hosts
  and only while fallback rules exist; custom request headers on Safari).
- E2E: request-header condition redirects an XHR; JSON-path body condition
  blocks a fetch; form-field condition redirects a POST navigation before it
  starts; custom header added on a page-initiated request.

Acceptance: e2e green on Chrome; Safari smoke for both tiers **[Mac]**.

## Phase 5 — Popup, badge, blocked page, onboarding, polish (size M)

Popup (global kill switch, per-site grant/revoke, rules active on this tab,
recent matches, open designer), blocked page with "allow once", first-run
onboarding, empty and error states, accessibility pass (keyboard, labels,
contrast), dark mode and RTL check, "redirect/block everything" confirmation
(T13).

## Phase 6a — Safari wrapper and spike run (size S) [Mac]

`xcrun safari-web-extension-packager` with `--copy-resources`, commit
`apps/safari`, `scripts/build-safari.sh`, SwiftUI host screen with extension
state check and app icon set, run S1–S7 and write `safari.json`, update the
capability matrix.

## Phase 6b — Store readiness and first release (size M) [ASC or Mac]

`strict_min_version`, `PrivacyInfo.xcprivacy`, App Store metadata, Chrome
Web Store listing and permission justifications, privacy policy page,
Arabic locale, release workflow, CHANGELOG, TestFlight build through the App
Store Connect packager (or Xcode archive and notarization per Q1), Chrome
Web Store submission.

## Phase 7 — Hardening and extras (optional, after first release)

Rule folders and tags, per-rule hit counters, temporary session rules,
tab-scoped rules on Chrome, rule sync for Safari through the host app's
iCloud key-value store, performance profiling with 1,000+ rules, Firefox
build check.
