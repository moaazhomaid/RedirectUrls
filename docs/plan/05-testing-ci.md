# 5. Testing, CI and release

## 5.1 Test pyramid

| Layer | Tool | What |
|-------|------|------|
| Unit (core) | Vitest | Zod schemas, DNF normaliser, compiler golden files (`fixtures/*.designer.json` → `*.chrome.dnr.json`, `*.safari.dnr.json`, `*.report.json`), in-JS evaluator, migrations, regex validation |
| Unit (extension lib) | Vitest + `@webext-core/fake-browser` | storage adapter, permission flow, capability detection, fallback engine decision logic |
| Component | Vitest + `@testing-library/preact` | designer tree editing, badge rendering, import/export |
| E2E Chrome | Playwright with `--load-extension` + a local test server (Node, serves endpoints that echo method/headers/body) | real DNR redirects, header modification, Tier A and Tier B fallbacks, permission prompts stubbed |
| E2E Safari | `safaridriver` + XCUITest smoke on the owner's Mac (Remote Control) | extension loads, one DNR redirect works, one fallback redirect works, options page renders |
| Property tests | fast-check | DNF normaliser never changes truth table of the tree (compare evaluator results on random inputs); compiler output stays under budget |

## 5.2 Local test server

`packages/test-server/` (Node + Hono): routes `/echo` (returns method,
headers, body as JSON), `/redirect-target`, `/slow`, `/post-form`. Used by
Playwright and by the Safari smoke test.

## 5.3 CI (GitHub Actions)

`ci.yml` on every PR:

1. `pnpm install --frozen-lockfile`
2. `pnpm lint && pnpm typecheck`
3. `pnpm test` (unit + component + property)
4. `pnpm -F extension build` (chrome) and `build:safari` → upload both zips as artifacts
5. `pnpm e2e:chrome` (Playwright, headless Chromium with extension; uses the pre-installed Chromium where available)
6. Size report comment (bundle size per entrypoint) to catch bloat for iOS.

`release.yml` on tag `v*`: builds, zips the Chrome package for Web Store
upload, zips the `safari-mv3` output for the Mac build step, drafts a GitHub
release with changelog from conventional commits. Xcode archive/notarize is a
documented manual step (or a self-hosted macOS runner later).

## 5.4 Conventions (to be copied into CLAUDE.md in Phase 1)

- Conventional commits; one PR per phase deliverable; PR template with
  Before/After.
- Every PR that touches `core/compile` adds or updates a golden fixture.
- No new permission without an ADR in `docs/adr/`.
- Never use `storage.sync`, never fetch remote code.
