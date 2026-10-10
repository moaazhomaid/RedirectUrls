# 4. Safari specifics

## 4.1 Packaging

1. `pnpm -F extension build:safari` → `packages/extension/.output/safari-mv3/`.
2. First time only: `xcrun safari-web-extension-converter .output/safari-mv3 --project-location apps/safari --app-name RedirectUrls --bundle-identifier <reverse-dns, owner decides> --macos-only=false --swift` generates a macOS + iOS Xcode project. **The generated project is committed**; later builds only copy fresh extension resources into it (`scripts/build-safari.sh` uses `rsync` into `apps/safari/Shared (Extension)/Resources`).
3. Host app: a one-screen SwiftUI window that explains how to enable the extension in Safari Settings → Extensions and shows status via `SFSafariExtensionManager.getStateOfSafariExtension`.
4. Development: Safari → Develop → Allow Unsigned Extensions, then run the macOS target from Xcode.

These steps need macOS with Xcode; cloud threads cannot run them. The
Safari build/test tasks in `phases.md` are therefore written for a **Remote
Control session on the owner's Mac** or for the owner to run manually.

## 4.2 Distribution

| Channel | Requirement |
|---------|-------------|
| Mac App Store / iOS App Store | Apple Developer Program membership, App Store review, privacy nutrition labels (extension collects no data) |
| Outside the App Store (macOS only) | Developer ID signing + notarization; users download the .app, open it, enable the extension in Safari |
| Internal testing | TestFlight (iOS/macOS) or ad-hoc signed builds |

Decision needed from the owner: App Store, notarized direct download, or both
(see `decisions.md`).

## 4.3 Safari-specific behaviour the code must handle

- **Per-site permissions are mandatory.** `host_permissions` do not grant
  access at install. Use `optional_host_permissions` + `permissions.request`
  and handle `permissions.onAdded/onRemoved` to re-apply DNR rules (Safari
  drops rules for hosts without access).
- **Redirect requires `declarativeNetRequestWithHostAccess`** and host access
  to the request URL.
- **Shared dynamic+session rule pool** (`MAX_NUMBER_OF_DYNAMIC_AND_SESSION_RULES`);
  measure in Phase 0 and store the number in `Capabilities`.
- **No response-header conditions, no response `modifyHeaders`, no
  `tabIds`, no `testMatchOutcome`.** The compiler routes these to fallback
  or marks the rule "unsupported in Safari" (response modifyHeaders has no
  fallback at all).
- **`webRequest` listeners must be synchronous** and are observation only.
- **`storage.sync` does not sync.** Not used.
- **Service worker** background works (15.4+) but Safari may also run a
  persistent-less background page; WXT's `safari` target handles the manifest.
  Keep the background stateless: every piece of state lives in `storage`.
- **iOS**: the options page must work at phone width (Tailwind responsive),
  and the designer must be usable with touch (no drag-only interactions;
  provide up/down buttons as well as drag).
- `browser_specific_settings.safari.strict_min_version: "26.0"` so older
  Safari (no `requestMethods`) refuses to load instead of silently
  mis-matching.

## 4.4 Known unknowns to verify in Phase 0 (spike on the owner's Mac)

1. Exact value of `MAX_NUMBER_OF_DYNAMIC_AND_SESSION_RULES` on Safari 27
   macOS and iOS.
2. Whether `regexFilter` with `regexSubstitution` works for all capture
   groups `\1..\9` in Safari 27.
3. Whether `content_scripts[].world: "MAIN"` in the manifest is honoured or
   whether `registerContentScripts` is required.
4. Whether `webRequest.onBeforeRequest.details.requestBody` is populated for
   POST navigations in Safari 27 (BCD says the field exists since 14, but
   content may be empty).
5. Whether `declarativeNetRequestFeedback` / `getMatchedRules` returns data
   for dynamic rules in Safari 27 (needed for the match log; else the popup
   falls back to `webRequest.onCompleted` correlation).
