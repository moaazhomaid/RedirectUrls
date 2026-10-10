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
  plus the closely related block, modify headers, and allow (pass-through).

Rules are saved, individually enabled/disabled, ordered, importable and
exportable, and survive browser restarts.

## 1.2 User stories (acceptance basis for the phases)

| # | As a … | I want … | So that … |
|---|--------|----------|-----------|
| U1 | developer | to redirect `https://app.example.com/*` to `http://localhost:3000/*` keeping path and query | I test the production front end against a local backend |
| U2 | developer | to redirect only `POST`/`PUT` calls to `api.example.com` when the query has `env=staging` | I switch one environment without touching others |
| U3 | QA engineer | to send requests carrying header `X-Env: staging` to the staging host | I test header-driven routing (best effort, see §1.5) |
| U4 | QA engineer | to block a tracking pixel on one host and see a counter of blocked hits | I verify instrumentation |
| U5 | security tester | to add `Authorization`/`Cookie`/custom headers to requests to my own lab host | I reproduce authenticated flows (custom header names are Chrome-only via DNR, see §1.4) |
| U6 | any user | to group conditions with nested AND/OR/NOT and see which browser evaluates each exactly | I trust what the rule will do before saving |
| U7 | any user | to import/export rules as JSON and share a single rule as a string | I move rules between Safari and Chrome and between machines |
| U8 | any user | to test a URL (plus optional method/headers/body) against my rules | I debug a rule without reloading pages |

## 1.3 Non-goals (v1)

- Response body rewriting, mocking or script injection into pages.
- WebSocket or WebTransport frame interception (connection URL redirects are in scope as resource type `websocket`).
- Cross-device sync (Safari cannot sync extension storage; see D7). Export/import covers it.
- Firefox and Edge. The architecture does not preclude them (WXT builds both), but they are not tested or packaged in v1.
- Enterprise policy deployment.

## 1.4 Target browsers (verified 2026-10-10)

| Browser | Current stable | Source |
|---------|----------------|--------|
| Safari | **27.0** (released 2026-09-14, build 20625.1.29); 27.0.1 shipped 2026-09-28; 27.2 in beta | Apple Safari Release Notes index; The Apple Wiki |
| Chrome | **155** (155.0.8059.39/.40, stable desktop 2026-10-06); 156 in early-stable rollout since 2026-10-07 | Chrome Releases blog |

Support floor: **Safari 26** (first version with `requestMethods` and
`initiatorDomains` in DNR) and **Chrome 128** (response-header conditions).
Manifest V2 is gone from Chrome (disabled in 138, non-functional from 139,
removed from the Web Store on 2026-08-31), so the project is **Manifest V3
only**. Safari accepts MV2 and MV3; we ship MV3 to keep one code path.

## 1.5 Hard constraints that shape the design

1. **No blocking `webRequest` in either browser.** Chrome MV3 reserves
   `webRequestBlocking` for policy-installed extensions; Safari never
   implemented it (`BlockingResponse` unsupported). The only browser-level
   way to redirect, block or modify requests is **`declarativeNetRequest`
   (DNR)**.
2. **DNR matches on URL, method, resource type, initiator/request domains,
   first/third party, tab id (Chrome) and response headers (Chrome 128+).**
   It cannot see request headers or request bodies in either browser. Those
   conditions go to a fallback engine (`02-architecture.md` §2.7) and are
   labelled "best effort" in the UI.
3. **Safari DNR is a subset of Chrome DNR, implemented by converting each
   DNR rule into WebKit content-blocker rules.** Consequences verified in
   WebKit source (`_WKWebExtensionDeclarativeNetRequestRule.mm`,
   `URLFilterParser.cpp`, `WebExtensionConstants.h`):
   - one DNR rule expands to **one content-blocker rule per request method
     and per request domain** (multiplicative), plus one
     `ignore-following-rules` rule per excluded method/domain;
   - `requestDomains` is **silently ignored** when `regexFilter` is present;
   - the regex engine accepts only ASCII literals, `.`, bracket classes,
     `? * +`, a leading `^`, a trailing `$` and plain groups. **No `|`, no
     `\d \w \s \b`, no `{n,m}`, no look-around, no backreferences.**
   - `modifyHeaders` only accepts header names on a fixed allowlist of about
     100 standard names (`reference/safari-dnr-header-allowlist.json`);
     custom `X-*` names are rejected with "The header … is not recognized";
   - `modifyHeaders.responseHeaders` is parsed but never applied;
   - dynamic + session rules share one limit of **30,000**; the underlying
     content-blocker list caps at **150,000** expanded rules;
   - `redirect` supports `url`, `extensionPath`, `transform` and
     `regexSubstitution` (the last requires `regexFilter`).
4. **Safari extensions ship inside a native app.** Packaging is done either
   with `xcrun safari-web-extension-packager` (Xcode on a Mac) or with the
   web-based packager in App Store Connect, which needs no Mac. Running and
   debugging locally still needs Safari on a Mac or iPhone.
5. **Chrome Web Store** scrutinises `<all_urls>` plus DNR redirect or
   modifyHeaders. We use `optional_host_permissions` and request hosts at
   runtime, which is also the model Safari enforces (per-site consent, with
   both origin and destination needing access for a redirect).

## 1.6 Capability matrix (Safari 27 vs Chrome 155)

Legend: ✅ supported, ❌ not supported, ⚠️ partial / caveat.

### declarativeNetRequest

| Feature | Chrome 155 | Safari 27 | Notes |
|---------|-----------|-----------|-------|
| `updateDynamicRules` / `updateSessionRules` | ✅ | ✅ (15.4+) | user rules are dynamic rules |
| `getDynamicRules({filter})` | ✅ | ✅ (18.5+) | |
| `condition.urlFilter` | ✅ | ✅ | case-insensitive by default in both; Safari converts to regex (`\|\|` → `^[^:]+://+([^:/]+\.)?`, `^` → `[^a-zA-Z0-9_.%-]`, `*` → `.*`) |
| `condition.regexFilter` | ✅ RE2, ≤2 KB compiled, 1,000 regex rules per ruleset type | ⚠️ WebKit subset (see §1.5.3); no regex-count constant | compiler emits only the portable subset |
| `condition.requestMethods` / `excludedRequestMethods` | ✅ | ✅ (26+) | Safari: one expanded rule per method |
| `condition.resourceTypes` / `excludedResourceTypes` | ✅ | ✅ (one of the two, not both) | Safari lacks `object`, `csp_report`, `webbundle`, `webtransport`; omitted `resourceTypes` means "all but main_frame" in both |
| `condition.requestDomains` / `excludedRequestDomains` | ✅ | ⚠️ (16.4+) ignored with `regexFilter` | compiler folds host into the filter on Safari |
| `condition.initiatorDomains` / `excludedInitiatorDomains` | ✅ | ✅ (26+) | |
| `condition.domainType` | ✅ | ✅ | |
| `condition.responseHeaders` / `excludedResponseHeaders` | ✅ (128+) | ❌ | Chrome-only condition |
| `condition.tabIds` | ✅ (session rules) | ❌ | |
| `action.redirect.url` / `.extensionPath` / `.transform` | ✅ | ✅ (15.4+) | `transform` incl. `queryTransform`; `javascript:` rejected in both; Safari `url` must be http(s) |
| `action.redirect.regexSubstitution` | ✅ `\0`–`\9` | ✅ requires `regexFilter` **[spike: capture groups]** | |
| `action.modifyHeaders.requestHeaders` | ✅ set/append/remove (append allowlisted) | ⚠️ (16.4+) allowlisted header names only | |
| `action.modifyHeaders.responseHeaders` | ✅ | ❌ parsed, not applied | |
| `action.block` / `allow` / `upgradeScheme` / `allowAllRequests` | ✅ | ✅ (`allowAllRequests` main_frame only) | |
| Dynamic rule limit | 30,000 total, **5,000 unsafe** (redirect/modifyHeaders) | **30,000 shared** dynamic+session, 150,000 expanded | design budget in §2.6.5 |
| Regex rule limit | 1,000 per ruleset type | none exposed | |
| `isRegexSupported` | ✅ | ✅ | used at save time |
| `testMatchOutcome` | ✅ | ❌ | Chrome-only debugging aid |
| `getMatchedRules` / `declarativeNetRequestFeedback` | ✅ (unpacked only for `onRuleMatchedDebug`) | ✅ (15.4+) **[spike: dynamic rules]** | |
| `setExtensionActionOptions` (badge counts) | ✅ | ✅ (16.4+) | |

### webRequest (observation only)

| Feature | Chrome 155 | Safari 27 | Notes |
|---------|-----------|-----------|-------|
| `onBeforeRequest` (+ `requestBody`) | ✅ | ✅ (18.4+) **[spike: body populated?]** | non-blocking in both; Safari listeners must be synchronous |
| `onBeforeSendHeaders` (+ `requestHeaders`) | ✅ | ✅ (18.4+) | non-blocking |
| `onHeadersReceived` (+ `responseHeaders`, `statusCode`) | ✅ | ✅ (18.4+) | non-blocking |
| `webRequestBlocking` | ❌ (policy only) | ❌ | never available to us |

### scripting, content scripts, navigation

| Feature | Chrome 155 | Safari 27 |
|---------|-----------|-----------|
| `scripting.executeScript({world: "MAIN"})` | ✅ | ✅ (15.4+) |
| `scripting.registerContentScripts({world, persistAcrossSessions})` | ✅ | ✅ (16.4+) |
| `content_scripts[].world` in manifest | ✅ (111+) | ✅ (18+) |
| `webNavigation.onBeforeNavigate` / `onCommitted` | ✅ | ✅ |
| `tabs.update(tabId, {url})` | ✅ | ✅ |

### Background, storage, manifest

| Feature | Chrome 155 | Safari 27 |
|---------|-----------|-----------|
| `background.service_worker` | ✅ | ✅ (15.4+) |
| `storage.local` quota | 10 MB (unlimited with `unlimitedStorage`) | 10 MiB (`unlimitedStorage` supported 16+) |
| `storage.session` | ✅ | ✅ (16.4+), 10 MiB |
| `storage.sync` | ✅ (100 KiB) | ⚠️ accepted but **does not sync** |
| `optional_host_permissions` + `permissions.request()` | ✅ | ✅ (15.5+); requesting `<all_urls>` lets Safari prompt per visited site |
| `browser_specific_settings.safari.strict_min_version` | n/a | ✅ |
| `declarativeNetRequestWithHostAccess` | ✅ | ✅ (16+), required for redirect and modifyHeaders |
| `action.setBadgeText` | ✅ | ✅ (15.4+); `setBadgeBackgroundColor` has no effect |

## 1.7 What this means for the user-facing feature set

| Condition operand in the designer | Chrome | Safari | Engine |
|----------------------------------|--------|--------|--------|
| URL / scheme / host / port / path / query / query param / fragment | ✅ exact | ✅ exact | DNR |
| HTTP method | ✅ exact | ✅ exact (26+) | DNR |
| Resource type | ✅ exact | ✅ exact (minus `object`, `csp_report`) | DNR |
| Initiator domain, first vs third party | ✅ exact | ✅ exact (26+) | DNR |
| Response header key/value, status | ✅ exact (DNR) | ⚠️ best effort, documents only | DNR / Tier A |
| Request header key/value | ⚠️ best effort | ⚠️ best effort | Tier A (navigations) + Tier B (page requests) |
| Request body: form field, JSON path, raw substring/regex | ⚠️ best effort | ⚠️ best effort | Tier A (POST navigations) + Tier B (page requests, form submits) |
| Tab-scoped rule | ✅ | ❌ | DNR session rules |

| Action | Chrome | Safari |
|--------|--------|--------|
| Redirect (url, transform, regex substitution) | ✅ | ✅ |
| Block / allow / upgrade to HTTPS | ✅ | ✅ |
| Modify request headers, standard names | ✅ | ✅ |
| Modify request headers, custom names (`X-*`) | ✅ | ⚠️ Tier B only (page-initiated requests) |
| Modify response headers | ✅ | ❌ (no fallback possible) |

"Best effort" is defined precisely in `02-architecture.md` §2.7. The designer
shows a per-condition fidelity badge so the user always knows which engine
evaluates a condition in the browser they are running.
