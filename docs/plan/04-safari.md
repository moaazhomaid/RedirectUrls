# 4. Safari specifics

## 4.1 Packaging options

| Route | Needs | Produces | When to use |
|-------|-------|----------|-------------|
| **Command line**: `xcrun safari-web-extension-packager .output/safari-mv3 --project-location apps/safari --app-name RedirectUrls --bundle-identifier <reverse-dns> --swift --copy-resources --no-open` (the tool was renamed from `safari-web-extension-converter`) | Xcode on a Mac | Xcode project with macOS + iOS targets; re-run with `--rebuild-project` to add platforms | development loop, local debugging, notarized direct download |
| **App Store Connect web packager** (Xcode Cloud tab → Safari Web Extension Packager → upload the extension zip) | Apple Developer Program, any browser, **no Mac** | macOS and iOS apps (iOS app also runs on iPadOS and visionOS) delivered to TestFlight / App Store; compute time counts against the 25 Xcode Cloud hours per month | release builds and TestFlight testing from cloud threads |

Plan: the Xcode project is generated once and **committed**; later builds
copy fresh extension output into it (`scripts/build-safari.sh` runs
`pnpm -F extension build:safari` then `rsync` into
`apps/safari/Shared (Extension)/Resources`). Release builds can go through
either route; the App Store Connect route lets a cloud thread ship a
TestFlight build without the owner's Mac.

WXT: `wxt build -b safari` with `manifestVersion: 3` set in `wxt.config.ts`
(WXT defaults Safari to MV2). Output in `.output/safari-mv3/`.

## 4.2 Local development loop (Mac)

1. Safari → Settings → Developer → "Allow unsigned extensions".
2. Run the macOS target from Xcode; enable the extension in Safari Settings →
   Extensions; grant site access from the toolbar badge.
3. Web Inspector → Develop → Web Extension Background Content for the
   service worker console. Safari 27 reports uncaught exceptions and
   unhandled rejections from extension scripts.
4. iOS: run the iOS target on a simulator or device; enable in Settings →
   Apps → Safari → Extensions.

These steps need macOS; cloud threads cannot run them, so the Safari
build/test tasks in `phases.md` are written for a Remote Control session on
the owner's Mac or for the owner to run manually.

## 4.3 Distribution

| Channel | Requirement |
|---------|-------------|
| Mac App Store / iOS App Store | Apple Developer Program (US$99/yr), App Store review, privacy nutrition labels (the extension collects no data), `PrivacyInfo.xcprivacy` |
| Outside the App Store (macOS only) | Developer ID signing + notarization (Xcode route); users open the .app once and enable the extension |
| Beta | TestFlight (both routes) |

Decision needed from the owner: see `decisions.md` Q1 and Q9.

## 4.4 Behaviour the code must handle on Safari

- **Per-site permissions are mandatory.** `host_permissions` do not grant
  access at install. Use `optional_host_permissions` + `permissions.request`;
  requesting `<all_urls>` makes Safari prompt per visited site. Handle
  `permissions.onAdded/onRemoved` by recompiling, because Safari drops rules
  for hosts without access.
- **Redirect needs access to both origin and destination hosts** and
  `declarativeNetRequestWithHostAccess`.
- **Rule expansion** (§2.6.5): methods × domains per DNR rule; budget on the
  expanded count; never emit `requestDomains`.
- **Regex subset** (§2.6.3): no alternation, no `\d`, no counted
  quantifiers; validate with `isRegexSupported` before apply.
- **Header allowlist**: `modifyHeaders` accepts only the names in
  `reference/safari-dnr-header-allowlist.json` (lower-cased comparison);
  anything else fails rule validation with "The header … is not
  recognized". Custom names route to Tier B.
- **No response-header conditions, no response `modifyHeaders`, no
  `tabIds`, no `testMatchOutcome`, no `onRuleMatchedDebug`.**
- **`webRequest` listeners must be synchronous** and are observation only.
- **`storage.sync` does not sync.** Not used.
- **`allowAllRequests`** only for `main_frame`; the compiler only uses plain
  `allow`.
- **Service worker** background is supported (15.4+) and is what WXT emits
  for MV3. State lives in storage only.
- **iOS**: options page must work at phone width and with touch (up/down
  buttons as well as drag); the popup is a sheet. Badge colour is ignored.
- `browser_specific_settings.safari.strict_min_version: "26.0"` so older
  Safari refuses to load instead of mis-matching methods.

## 4.5 Numbers (from WebKit source, main branch, 2026-10-10)

| Constant | Value |
|----------|-------|
| `MAX_NUMBER_OF_DYNAMIC_AND_SESSION_RULES` | 30,000 (shared) |
| `MAX_NUMBER_OF_STATIC_RULESETS` / `MAX_NUMBER_OF_ENABLED_STATIC_RULESETS` | 100 / 50 |
| Content-blocker list maximum (expanded rules) | 150,000 |
| `storage.local` / `storage.session` quota | 10 MiB each (`unlimitedStorage` lifts local) |
| `storage.sync` quota (local-only behaviour) | 100 KiB, 8 KiB/item, 512 items |
| Request methods accepted in `requestMethods` | get, head, options, trace, put, delete, post, patch, connect |

## 4.6 Spike list (Phase 0, needs a live Safari 27)

| # | Question | Why it matters |
|---|----------|----------------|
| S1 | Does `regexFilter` with capture groups plus `redirect.regexSubstitution` work end to end, and which `\n` references are honoured? | regex-substitution redirect mode on Safari |
| S2 | Does `webRequest.onBeforeRequest.details.requestBody` carry `formData`/`raw` for POST navigations? | Tier A body conditions on Safari |
| S3 | Is cross-world `CustomEvent` dispatch synchronous (MAIN → isolated → reply within the same handler)? | Tier B design for `sendBeacon` and sync XHR |
| S4 | Does `getMatchedRules` return hits for dynamic rules, and does `setExtensionActionOptions` badge counting work? | match log and badge on Safari |
| S5 | What happens at the 30,000 shared limit and when expansion approaches 150,000 (error text, partial apply)? | budget error handling |
| S6 | Does `registerContentScripts({world: "MAIN"})` run at `document_start` before page scripts on iOS as well as macOS? | Tier B reliability |
| S7 | Confirm `||host^` conversion matches `https://host/` and `https://host:8443/` but not `https://host.evil.com/` | host leaf correctness |

Results are written into `core/src/capabilities/safari.json` and the
matrix in `01-overview.md` is updated to remove the **[spike]** marks.
