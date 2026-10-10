# Decisions and open questions

## Decided during planning (change by editing this file and the affected doc)

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Manifest V3 only, DNR as the primary engine | blocking webRequest unavailable in both browsers; MV2 removed from Chrome |
| D2 | Support floor Safari 26, Chrome 128 | `requestMethods`/`initiatorDomains` landed in Safari 26; response-header conditions in Chrome 128 |
| D3 | WXT + TypeScript + Preact + Tailwind | cross-browser manifest generation, small bundles, owner's Tailwind familiarity |
| D4 | Condition tree normalised to DNF; one DNR rule per disjunct; cap 64 disjuncts per rule | only way to map arbitrary AND/OR onto flat DNR conditions while protecting the 5,000 rule budget |
| D5 | Header/body conditions handled by a clearly labelled two-tier fallback engine (observe+re-navigate; MAIN-world fetch/XHR wrapper) | DNR cannot see request headers or bodies; this is the same trade-off made by Requestly and similar tools |
| D6 | `optional_host_permissions` with per-rule runtime grants | matches Safari's mandatory per-site model and reduces Chrome Web Store review friction |
| D7 | No `storage.sync`, no remote code, no telemetry | Safari does not sync; Web Store policy; privacy |
| D8 | Generated Xcode project is committed to the repo | reproducible Safari builds; converter is run once |
| D9 | Safari build/test steps run on the owner's Mac via Remote Control | cloud threads have no macOS/Xcode |

## Open questions for the owner (answer in the thread; defaults in bold)

| # | Question | Options |
|---|----------|---------|
| Q1 | Safari distribution | **Both** App Store and notarized direct download / App Store only / direct only |
| Q2 | iOS support in v1 | **Yes, same codebase** (adds touch/width requirements) / macOS only for v1 |
| Q3 | Bundle identifier and product name | e.g. `com.<owner-domain>.redirecturls`, name "RedirectUrls" |
| Q4 | Apple Developer Program account exists? | needed from Phase 6; development before that works unsigned |
| Q5 | Response-header `modifyHeaders` on Safari | **Show as "Chrome only" in the UI** / hide entirely on Safari |
| Q6 | Arabic locale in v1 | **Yes** / later |
| Q7 | License | **MIT** / other |
| Q8 | Default branch and PR flow | **`main`, squash merge, one PR per phase deliverable** |

## Risks

| Risk | Mitigation |
|------|------------|
| Safari DNR rule limit far below 5,000 on iOS | measured in Phase 0; budget constant per platform; UI refuses over-budget saves |
| RE2 limits (no lookahead) make some negated URL conditions uncompilable | compiler reports them, routes to fallback, designer suggests `excluded*` style alternatives |
| Chrome Web Store review delays for `<all_urls>` + redirect | optional host permissions, detailed justification text prepared in Phase 6 |
| Safari service worker termination losing in-memory state | background kept stateless; all state in `storage` |
| MAIN-world injection blocked by strict CSP on some sites | Tier B marked best effort; failure surfaced in match log |
