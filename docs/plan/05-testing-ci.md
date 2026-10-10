# 5. Testing, CI and release

## 5.1 Test pyramid

| Layer | Tool | What |
|-------|------|------|
| Unit (core) | Vitest | Zod schemas; DNF normaliser; portable-regex checker; urlFilter builder; compiler golden files per target (`*.chrome.dnr.json`, `*.safari.dnr.json`, `*.report.json`, `*.safari.expanded-count`); budget calculator; evaluator; migrations |
| Property (core) | fast-check | normaliser preserves the truth table; compiler output for DNR-compilable trees agrees with the evaluator on random requests; Safari expansion count equals the formula; every emitted regex passes the portable-subset checker |
| Unit (extension lib) | Vitest + `@webext-core/fake-browser` | storage adapter, permission flow (origin + destination), capability detection, tier selection, match-log ring buffer |
| Component | Vitest + `@testing-library/preact` | designer tree editing, badges, validation messages, import/export, tester panel |
| E2E Chrome | Playwright with `--load-extension` + `packages/test-server` | real DNR redirects (url, transform, regex substitution), block, modifyHeaders, loop protection, budget refusal, Tier A and Tier B flows, permission prompt stubbed |
| E2E Safari | `safaridriver` + XCUITest smoke on the owner's Mac (Remote Control) | extension loads; one DNR redirect; one header-allowlist rule; one Tier B custom-header rule; options page renders at phone width on the iOS simulator |
| Conformance | the probe harness (§5.2) | real per-browser capability facts, re-run on each new Safari/Chrome stable |

## 5.2 Capability probe harness (`packages/probe`)

A minimal extension that, when loaded, runs a list of probes and writes a
JSON report to `storage.local` and the console:

- for each DNR condition/action key: try `updateSessionRules` with a rule
  using it, record accept/reject and the error text;
- `isRegexSupported` over a fixed corpus of patterns (alternation, `\d`,
  counted quantifiers, groups, look-ahead) to confirm the portable subset;
- `webRequest` events: which `details` fields are populated for a POST
  navigation and an XHR against the test server;
- cross-world `CustomEvent` timing (S3);
- measured rule limit: binary-search `updateSessionRules` batches until the
  browser rejects, then remove them.

Output is committed as `core/src/capabilities/chrome.json` and
`safari.json`, with browser version and date. The compiler's golden tests
run against these files, so a capability change shows up as a test diff.

## 5.3 Local test server

`packages/test-server/` (Node + Hono): `/echo` (returns method, headers,
body as JSON), `/redirect-target`, `/slow`, `/post-form` (HTML form that
POSTs), `/beacon`, `/websocket`. Used by Playwright and the Safari smoke
test; also useful for the owner's manual tests.

## 5.4 CI (GitHub Actions)

`ci.yml` on every PR:

1. `pnpm install --frozen-lockfile`
2. `pnpm lint && pnpm typecheck`
3. `pnpm test` (unit, property, component)
4. `pnpm -F extension build` (chrome) and `build:safari`; upload both zips
   as artifacts
5. `pnpm e2e:chrome` (Playwright, headless Chromium with the extension)
6. Size report comment per entrypoint (iOS bundle discipline)

`release.yml` on tag `v*`: builds, zips the Chrome package for Web Store
upload, zips `.output/safari-mv3` for either packaging route, drafts a GitHub
release with a changelog from conventional commits. Uploading to App Store
Connect is a documented step (web packager or Xcode archive).

## 5.5 Conventions (to be copied into CLAUDE.md in Phase 0)

- Conventional commits; one PR per phase deliverable; PR template with
  Before/After.
- Every PR touching `core/compile` or `core/regex` adds or updates a golden
  fixture.
- No new permission without an ADR in `docs/adr/`.
- Never use `storage.sync`; never fetch remote code; never put rule data in
  the MAIN world.
- Dependencies pinned via the lockfile; `pnpm audit` in CI.
