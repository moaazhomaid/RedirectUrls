# Phases

Each phase is sized for one Opus 5.5 development thread and ends in a PR to
`main`. Within a thread: Sonnet 5.5 sub-agents for research/writing content,
Haiku 5.5 sub-agents for running commands, builds and tests. Phases marked
**[Mac]** need the owner's Mac (Remote Control session or manual run).

## Phase 0 — Repo bootstrap and capability spike  (≈1 PR, + Mac spike)

Deliverables

- pnpm workspace, TypeScript strict, ESLint/Prettier, Vitest, WXT project
  with `chrome` and `safari` targets, Preact + Tailwind, `CLAUDE.md` with the
  conventions from `05-testing-ci.md`.
- CI workflow (lint, typecheck, test, build both targets).
- `packages/core` skeleton with Zod `RuleSet` schema and `Capabilities` type.
- `scripts/build-safari.sh` and the generated Xcode project under
  `apps/safari/` **[Mac]**.
- Spike extension (throwaway code under `spikes/`) that logs, on both
  browsers, the answers to `04-safari.md` §4.4 and Chrome equivalents.
  Results are written back into `01-overview.md` §1.4 and `decisions.md`.

Acceptance: `pnpm build` produces loadable Chrome and Safari packages; CI green;
spike results recorded.

## Phase 1 — Core rule model, DNF normaliser, compiler  (≈1–2 PRs)

- Full `ConditionNode` / `Rule` / `Action` schemas with migrations scaffold.
- DNF normaliser with property tests (truth-table equivalence).
- In-JS evaluator (`evaluate(tree, requestContext)`), used by tests, the
  rule tester and the fallback engine.
- Compiler to DNR with golden fixtures for Chrome and Safari capability sets,
  including: urlFilter merging, regex folding, `excluded*` for NOT, rule id
  hashing, priority, loop detection with anti-loop allow rules, budget check,
  fidelity report.

Acceptance: 100% of fixtures pass for both targets; evaluator and compiler
agree on random inputs for DNR-compilable trees (property test).

## Phase 2 — Background engine and storage  (≈1 PR)

- Storage adapter, `RuleStore`, `storage.onChanged` → compile → atomic
  `updateDynamicRules`.
- Capability detection module.
- Permission manager (`optional_host_permissions`, request on save,
  re-apply on `permissions.onAdded/onRemoved`).
- Match log ring buffer in `storage.session` (DNR feedback on Chrome;
  `webRequest.onCompleted` correlation elsewhere).
- Playwright e2e: redirect by host, by method, by query param, block, modify
  request header, transform redirect, loop protection, budget rejection.

Acceptance: e2e green on Chrome in CI; manual smoke on Safari **[Mac]**.

## Phase 3 — Options page: rule list and logic designer  (≈2 PRs)

- PR 3a: app shell, rule list, enable/disable, ordering, import/export,
  settings, presets, i18n scaffold.
- PR 3b: logic designer (nested groups, AND/OR/NOT, field/operator pickers,
  regex validation with re2js, fidelity badges, action editor with capture
  placeholders, "Test against URL" panel using the evaluator and, on Chrome,
  `testMatchOutcome`).

Acceptance: component tests; Playwright flow "create rule in designer → see
redirect happen"; works at 390 px width (iOS) with touch-friendly reordering.

## Phase 4 — Fallback engine (headers and body)  (≈2 PRs)

- PR 4a: Tier A (observe `webRequest` → evaluator → `tabs.update` for
  navigations), with honest in-UI limitation text.
- PR 4b: Tier B (MAIN-world fetch/XHR/sendBeacon wrapper, isolated-world
  bridge with nonce, dynamic `registerContentScripts` only for granted hosts
  and only while fallback rules exist).
- E2E against the test server for: request-header condition redirects an
  XHR; body JSON-path condition blocks a fetch; POST form field condition
  redirects a navigation.

Acceptance: e2e green on Chrome; Safari smoke **[Mac]** for both tiers.

## Phase 5 — Popup, badge, onboarding, polish  (≈1 PR)

- Popup: global toggle, per-site grant, rules active on this tab, recent
  matches, open designer.
- Badge counts (Chrome native, Safari manual).
- First-run onboarding, empty states, error states (regex unsupported,
  budget exceeded, host not granted).
- Accessibility pass (keyboard, labels, contrast), dark mode, RTL check.

## Phase 6 — Safari app wrapper and store readiness  (≈1 PR + Mac work)

- SwiftUI host app screen, extension state check, app icon set.
- `strict_min_version`, privacy manifest (`PrivacyInfo.xcprivacy`), App
  Store metadata draft, Chrome Web Store listing draft and permission
  justifications, privacy policy page (static, in `docs/`).
- Arabic locale.
- Release workflow, version bump script, CHANGELOG.
- **[Mac]** Archive, notarize or submit per the owner's distribution decision.

## Phase 7 — Hardening (optional, after first release)

- Rule sharing links, rule groups/folders, per-rule hit counters, session
  (temporary) rules, tab-scoped rules on Chrome, dark-launch telemetry
  (opt-in, local only), performance profiling with 1,000+ rules.
