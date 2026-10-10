# Sources (all checked 2026-10-10)

## Browser versions

- Apple, Safari Release Notes index: https://developer.apple.com/documentation/safari-release-notes (27.0 released 2026-09-14; 26.6 2026-07-27; 26.4 2026-03-24)
- Apple, Safari 27 release notes (Web Extensions section): https://developer.apple.com/documentation/safari-release-notes/safari-27-release-notes
- The Apple Wiki, Safari (27.0.1 on 2026-09-28): https://theapplewiki.com/wiki/Safari
- Chrome Releases, Stable Channel Update for Desktop, 2026-10-06 (155.0.8059.39/.40): https://chromereleases.googleblog.com/2026/10/stable-channel-update-for-desktop.html
- Chrome Releases, Early Stable Update for Desktop, 2026-10-07 (156.0.8078.12/.13): https://chromereleases.googleblog.com/2026/10/early-stable-update-for-desktop.html
- Chrome, Manifest V2 deprecation timeline: https://developer.chrome.com/docs/extensions/develop/migrate/mv2-deprecation-timeline

## Chrome declarativeNetRequest

- API reference (urlFilter syntax, limits, precedence, regexSubstitution, permissions): https://developer.chrome.com/docs/extensions/reference/api/declarativeNetRequest

## Safari web extensions

- Apple, Blocking content with your Safari web extension (supported actions, permissions): https://developer.apple.com/documentation/safariservices/blocking-content-with-your-safari-web-extension
- Apple, Packaging a web extension for Safari (`xcrun safari-web-extension-packager`, formerly converter): https://developer.apple.com/documentation/safariservices/packaging-a-web-extension-for-safari
- Apple, Packaging and distributing Safari Web Extensions with App Store Connect (web packager, no Mac): https://developer.apple.com/documentation/safariservices/packaging-and-distributing-safari-web-extensions-with-app-store-connect
- Apple Developer Forums, custom headers rejected by modifyHeaders (Apple engineer confirms allowlist): https://developer.apple.com/forums/thread/781214
- WebKit bug on custom headers: https://bugs.webkit.org/show_bug.cgi?id=290922

## WebKit source (main branch)

- DNR → content-blocker conversion, header allowlist, expansion per method/domain, error strings: `Source/WebKit/UIProcess/Extensions/Cocoa/_WKWebExtensionDeclarativeNetRequestRule.mm`
- Regex subset accepted by URL filters: `Source/WebCore/contentextensions/URLFilterParser.cpp`
- Content-blocker action parsing (redirect keys, transform validation, modify-headers): `Source/WebCore/contentextensions/ContentExtensionActions.cpp`
- Content-blocker rule list parser (150,000 rule maximum, trigger keys): `Source/WebCore/contentextensions/ContentExtensionParser.cpp`
- Constants (30,000 dynamic+session, storage quotas): `Source/WebKit/Shared/Extensions/WebExtensionConstants.h`
- DNR API IDL: `Source/WebKit/WebProcess/Extensions/Interfaces/WebExtensionAPIDeclarativeNetRequest.idl`

## MDN browser-compat-data (webextensions/api and webextensions/manifest JSON)

- `declarativeNetRequest.json`, `webRequest.json`, `scripting.json`, `storage.json`, `tabs.json`, `permissions.json`, `webNavigation.json`, `action.json`, `runtime.json`, manifest `permissions.json`, `background.json`, `content_scripts.json`, `optional_host_permissions.json`, `browser_specific_settings.json`: https://github.com/mdn/browser-compat-data

## Tooling

- WXT, targeting browsers (Safari defaults to MV2): https://wxt.dev/guide/essentials/target-different-browsers
- WXT, publishing (Safari not automated; build with `wxt build -b safari`): https://wxt.dev/guide/essentials/publishing
