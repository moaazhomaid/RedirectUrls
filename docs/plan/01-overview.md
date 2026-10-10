# 1. Overview, scope and verified facts

## 1.1 Product goal

A browser extension for **Safari (current stable) and Chrome (current stable)**
that lets a user create **multiple redirect rules**. Each rule has:

- a **condition tree** built in a visual logic designer with `AND` / `OR`
  (and `NOT`) groups, nested to any depth;
- **condition operands** covering all traffic attributes: URL (full, scheme,
  host, port, path, query parameters, fragment), HTTP method, request headers
  (key and value), request payload/body, resource type, initiator/referrer,
  response headers and status (where the browser allows it);
- an **action**: redirect (static URL, regex substitution, URL transform),
  plus the closely related block, modify headers, and pass-through/allow.

The rules must be saved, enabled/disabled individually, ordered, importable and
exportable, and must survive browser restarts.

## 1.2 Target browsers (verified 2026-10-10)

| Browser | Current stable | Source |
|---------|----------------|--------|
| Safari | **27.0** (released 2026-09-14, build 20625.1.29); 27.0.1 shipped 2026-09-28 | Apple Safari Release Notes index; The Apple Wiki |
| Chrome | **155** (155.0.8059.39/.40, stable desktop 2026-10-06); 156 is in early-stable rollout since 2026-10-07 | Chrome Releases blog, October 2026 |

Support floor (recommended): **Safari 26+** (Safari 26 added
`requestMethods` and `initiatorDomains` to DNR, which the rule compiler
depends on) and **Chrome 128+** (response-header conditions). Manifest V2 is
dead in Chrome (disabled from Chrome 138, non-functional from 139, removed
from the Web Store 2026-08-31), so the project is **Manifest V3 only**.

## 1.3 Hard constraints that shape the design

These were checked against MDN browser-compat-data (`mdn/browser-compat-data`,
`webextensions/api/*.json`, fetched 2026-10-10), Apple's Safari release notes,
and Chrome's developer docs.

1. **No blocking `webRequest` anywhere we ship.** Chrome MV3 reserves
   `webRequestBlocking` for policy-installed extensions; Safari has never
   supported `webRequestBlocking` or `BlockingResponse`. The only
   browser-level way to redirect/block/modify requests is
   **`declarativeNetRequest` (DNR)**.
2. **DNR can only match on URL, method, resource type, initiator/request
   domains, first/third-party, tab id (Chrome) and response headers
   (Chrome 128+).** It cannot match on request headers or request body in
   either browser. Those conditions need a fallback engine (section 2.6 of
   the architecture doc) and must be clearly labelled "best effort" in the UI.
3. **Safari DNR is a subset of Chrome DNR.** See the matrix below.
4. **Safari extensions ship inside a native app** built with Xcode and signed
   with an Apple Developer account. There is no "load unpacked" for end users.
5. **Chrome Web Store** scrutinises `<all_urls>` host permissions plus DNR
   redirect/modifyHeaders. We use `optional_host_permissions` and request
   hosts at runtime, which is also the model Safari enforces (per-site
   consent).

## 1.4 Capability matrix (Safari 26/27 vs Chrome 155)

Legend: ✅ supported, ❌ not supported, ⚠️ partial / caveat.

### declarativeNetRequest

| Feature | Chrome 155 | Safari 27 | Notes |
|---------|-----------|-----------|-------|
| `updateDynamicRules` / `updateSessionRules` | ✅ | ✅ (15.4+) | user rules are dynamic rules |
| `getDynamicRules({filter})` | ✅ | ✅ (18.5+) | |
| `condition.urlFilter` | ✅ | ✅ | case-insensitive by default in both |
| `condition.regexFilter` | ✅ RE2, ≤2 KB compiled, 1,000 regex rules per ruleset | ⚠️ supported; no `MAX_NUMBER_OF_REGEX_RULES` constant exposed; must probe with `isRegexSupported()` | |
| `condition.requestMethods` / `excludedRequestMethods` | ✅ | ✅ (26+) | |
| `condition.resourceTypes` | ✅ | ✅ | |
| `condition.requestDomains` / `excludedRequestDomains` | ✅ | ✅ (16.4+) | |
| `condition.initiatorDomains` / `excludedInitiatorDomains` | ✅ | ✅ (26+) | |
| `condition.domainType` | ✅ | ✅ | |
| `condition.responseHeaders` / `excludedResponseHeaders` | ✅ (128+) | ❌ | Chrome-only condition |
| `condition.tabIds` | ✅ (session rules) | ❌ | |
| `action.redirect.url` / `.regexSubstitution` / `.transform` / `.extensionPath` | ✅ | ✅ (15.4+) | `URLTransform` incl. `queryTransform` ✅ in Safari |
| `action.modifyHeaders.requestHeaders` | ✅ | ✅ (16.4+) | |
| `action.modifyHeaders.responseHeaders` | ✅ | ❌ | Chrome-only |
| `action.block` / `allow` / `upgradeScheme` | ✅ | ✅ | |
| `MAX_NUMBER_OF_DYNAMIC_RULES` | 30,000 (5,000 "unsafe": redirect/modifyHeaders) | ❌ constant; Safari exposes `MAX_NUMBER_OF_DYNAMIC_AND_SESSION_RULES` (shared pool) | treat **5,000 total** as the design cap, measure Safari's real value in Phase 0 |
| `testMatchOutcome` | ✅ | ❌ | Chrome-only debugging aid |
| `getMatchedRules` / `declarativeNetRequestFeedback` | ✅ | ✅ (15.4+) | |
| `onRuleMatchedDebug` | ✅ (unpacked only) | ❌ | |

### webRequest (observation only)

| Feature | Chrome 155 | Safari 27 | Notes |
|---------|-----------|-----------|-------|
| `onBeforeRequest` (+ `requestBody`) | ✅ | ✅ (18.4+; `requestBody` details present) | non-blocking in both; Safari: sync listeners only |
| `onBeforeSendHeaders` (+ `requestHeaders`) | ✅ | ✅ (18.4+) | non-blocking |
| `onHeadersReceived` (+ `responseHeaders`, `statusCode`) | ✅ | ✅ (18.4+) | non-blocking |
| `webRequestBlocking` | ❌ (policy only) | ❌ | never available to us |

### scripting / content scripts

| Feature | Chrome 155 | Safari 27 |
|---------|-----------|-----------|
| `scripting.executeScript({world: "MAIN"})` | ✅ | ✅ (15.4+) |
| `scripting.registerContentScripts({world, persistAcrossSessions})` | ✅ | ✅ (16.4+) |
| `content_scripts[].world` in manifest | ✅ (111+) | ⚠️ verify in Phase 0; use `registerContentScripts` as the portable path |

### Background, storage, manifest

| Feature | Chrome 155 | Safari 27 |
|---------|-----------|-----------|
| `background.service_worker` | ✅ | ✅ (15.4+) |
| `background.scripts` (event page) | ❌ in MV3 | ✅ |
| `storage.local` / `storage.session` | ✅ | ✅ |
| `storage.sync` | ✅ (102 KB quota) | ⚠️ accepted but **does not sync**; behaves as local |
| `optional_host_permissions` + `permissions.request()` | ✅ | ✅ (required model) |
| `browser_specific_settings.safari` (`strict_min_version`) | n/a | ✅ |
| `declarativeNetRequestWithHostAccess` | ✅ | ✅ (16+) — required for redirect in Safari |

## 1.5 What this means for the user-facing feature set

| Condition operand in the designer | Chrome | Safari | Engine |
|----------------------------------|--------|--------|--------|
| URL / scheme / host / port / path / query / fragment | ✅ exact | ✅ exact | DNR |
| HTTP method | ✅ exact | ✅ exact (26+) | DNR |
| Resource type (document, xhr, script, image, …) | ✅ exact | ✅ exact | DNR |
| Initiator / referrer domain, first vs third party | ✅ exact | ✅ exact (26+) | DNR |
| Response header key/value, status | ✅ exact (DNR) | ⚠️ best effort (observe + re-navigate for documents only) | DNR / fallback |
| Request header key/value | ⚠️ best effort | ⚠️ best effort | fallback |
| Request payload / body (form fields, JSON path, raw substring/regex) | ⚠️ best effort | ⚠️ best effort | fallback |
| Tab id / current tab only | ✅ | ❌ | DNR session rules |

"Best effort" is defined precisely in `02-architecture.md` §2.6. The designer
shows a per-condition fidelity badge so the user always knows which engine
will evaluate a condition in the browser they are running.
