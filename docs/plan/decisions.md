# Decisions and open questions

## Decided during planning (change by editing this file and the affected doc)

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Manifest V3 only, DNR as the enforcement engine | blocking webRequest unavailable in both browsers; MV2 removed from Chrome |
| D2 | Support floor Safari 26, Chrome 128 | `requestMethods`/`initiatorDomains` landed in Safari 26; response-header conditions in Chrome 128 |
| D3 | WXT + TypeScript + Preact + Tailwind, `manifestVersion: 3` forced for Safari | cross-browser manifest generation, small bundles, owner's Tailwind familiarity, WXT defaults Safari to MV2 |
| D4 | Condition tree normalised to DNF; one DNR rule per disjunct; cap 64 disjuncts per rule | only way to map arbitrary AND/OR onto flat DNR conditions while protecting the budget |
| D5 | Header and body conditions handled by a labelled two-tier fallback engine (observe + re-navigate; isolated-world evaluation with a MAIN-world shim for fetch/XHR/beacon/form) | DNR cannot see request headers or bodies; same trade-off as Requestly-class tools, with rule data kept out of the page |
| D6 | `optional_host_permissions` with per-rule runtime grants for origin and destination | Safari's mandatory per-site model (both hosts for a redirect); less Chrome Web Store friction |
| D7 | No `storage.sync`, no remote code, no telemetry | Safari does not sync; Web Store policy; privacy |
| D8 | Generated Xcode project committed with `--copy-resources`; build script syncs output | reproducible Safari builds; packager run once |
| D9 | Safari local testing on the owner's Mac via Remote Control; release packaging may use the App Store Connect web packager from a cloud thread | cloud threads have no Safari; the web packager removes the Mac requirement for TestFlight/App Store builds |
| D10 | Compiler emits only the portable regex subset (RE2 ∩ WebKit) and prefers `urlFilter` | one code path that both engines accept; Chrome's 1,000-regex and 2 KB limits |
| D11 | Host conditions are folded into the URL filter, never `requestDomains` | WebKit ignores `requestDomains` with `regexFilter` and expands it per domain otherwise |
| D12 | Precedence: rule order is the priority; band priorities `(N-i)*8+4`; modifyHeaders additive | matches user expectation and both browsers' resolution rules |
| D13 | Custom header names on Safari go through Tier B only; response-header modification is "unsupported here" on Safari | WebKit allowlist; response headers parsed but never applied |

## Open questions for the owner (answer in the thread; defaults in bold)

| # | Question | Options |
|---|----------|---------|
| Q1 | Safari distribution | **App Store (via App Store Connect packager) plus notarized direct download for macOS** / App Store only / direct only |
| Q2 | iOS support in v1 | **Yes, same codebase** / macOS only |
| Q3 | Bundle identifier and product name | e.g. `com.<owner-domain>.redirecturls`, name "RedirectUrls" |
| Q4 | Apple Developer Program account exists? | needed for the App Store Connect packager and TestFlight; unsigned local builds work without it |
| Q5 | Response-header modification on Safari | **Show as "Unsupported here" and skip** / hide the option on Safari |
| Q6 | Arabic locale in v1 | **Yes** / later |
| Q7 | License | **MIT** / other |
| Q8 | Default branch and PR flow | **`main`, squash merge, one PR per phase deliverable** |
| Q9 | Do you have a Mac with Xcode available for Remote Control sessions? | **Yes** (Phase 6a runs there) / No (Safari testing goes through TestFlight builds from the web packager, slower loop) |
| Q10 | Match log scope | **Session only, 200 entries** / persistent with a size cap |

## Risk register

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Safari expanded-rule budget hit by users with many methods × hosts | medium | medium | budget meter in the UI; host folding; warn above 16 disjuncts |
| Portable regex subset too restrictive for power users | medium | low | `matchesRegex` keeps full RE2 on Chrome with a "Chrome only" badge |
| Capture groups unsupported in WebKit filters (S1 fails) | low | medium | regex-substitution mode marked "Chrome only" on Safari; transform mode covers most cases |
| Cross-world `CustomEvent` asynchronous in Safari (S3) | medium | low | fetch/XHR unaffected; `sendBeacon` and sync XHR fall back to "no action" with a note |
| Chrome Web Store review delays for `<all_urls>` + redirect | medium | medium | optional host permissions; justification text prepared in P6b |
| App Store review questions about a "redirect" app | low | medium | clear listing copy; no data collection; blocked page and kill switch demonstrate user control |
| Service-worker termination mid-apply | low | medium | snapshot + reconciliation (T11) |
| MAIN-world shim conflicts with sites that freeze `fetch` | low | low | best effort by definition; match log shows "shim unavailable" |
