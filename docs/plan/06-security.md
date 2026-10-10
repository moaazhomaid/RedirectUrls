# 6. Security and privacy

The extension can rewrite any request on hosts the user grants. The threat
model below is what every development thread must preserve.

## 6.1 Assets and trust boundaries

| Asset | Where | Trust |
|-------|-------|-------|
| Rule set (may contain internal hostnames, tokens in header values) | `storage.local` | extension-private; never leaves the device |
| Granted host list | browser permission store | user-controlled |
| Extension pages (options, popup, blocked page) | extension origin | trusted code, untrusted data inside |
| Isolated-world bridge | per page, extension-private world | trusted code, untrusted page around it |
| MAIN-world shim | page's own world | **untrusted environment**: the page can read, detect and replace it |

## 6.2 Threats and mitigations

| # | Threat | Mitigation |
|---|--------|------------|
| T1 | Malicious or malformed rule file import (JSON bomb, huge regex, prototype pollution keys) | Zod strict schema with size limits (file ≤ 2 MB, ≤ 500 rules, value ≤ 4 KB); parse with `JSON.parse` then validate, never `eval`; reject unknown keys; regex validated against the portable subset (linear-time engines on both sides, but WebKit DFA compilation cost still bounded by length ≤ 1 KB) |
| T2 | Rule data exposure to web pages via the MAIN-world shim | rules stay in the isolated world; the shim sends request facts and receives only a decision (§2.7); the bridge answers only events carrying the per-document nonce it generated |
| T3 | Page tampering with the shim to bypass or spoof decisions | documented as best effort; the shim is never used for security-relevant blocking; DNR remains the enforcement layer |
| T4 | Redirect to dangerous schemes (`javascript:`, `data:`, `file:`, `blob:`, other extensions' pages) | validated in the model, the compiler and the shim; DNR itself rejects `javascript:`; only `http`, `https` and the extension's own `extensionPath` are allowed |
| T5 | Open-redirect style abuse by a third party | not applicable: rules are authored only by the local user; no remote rule source, no shared rule URLs auto-imported (share strings are pasted by the user and go through T1 validation) |
| T6 | Stored XSS in extension pages from rule names, URLs, header values, match-log entries | Preact text rendering only (no `dangerouslySetInnerHTML`); extension CSP `script-src 'self'; object-src 'none'`; no inline scripts (also a WXT default) |
| T7 | Over-broad permissions | `optional_host_permissions` only; per-rule origin requests; popup shows and revokes granted sites; content scripts registered only while fallback rules exist and only for granted hosts; `declarativeNetRequestFeedback` optional and dev-oriented |
| T8 | Supply-chain compromise of a dependency | pnpm lockfile, `pnpm audit` in CI, minimal dependency set (Preact, Zod, re2js, WXT toolchain), no post-install scripts, Dependabot weekly |
| T9 | Remote code execution via "remote rules" or update channels | none exist; Web Store policy compliance (no remotely hosted code); rules are data, never code |
| T10 | Header value secrets (Authorization) leaking into logs or exports | match log stores rule id and URL only, never header values; export dialog warns when a rule contains an `Authorization`/`Cookie` set operation and offers to redact values |
| T11 | Service-worker termination losing an in-flight apply | apply is a single `updateDynamicRules` call with remove+add; the compiled snapshot is stored before apply and reconciled on startup (`getDynamicRules` vs snapshot) |
| T12 | Safari host app attack surface | the native app is a single SwiftUI screen with no network code, no URL handlers and no native messaging in v1 |
| T13 | Denial of service of the user's own browsing by a bad rule (redirect everything) | loop protection; the "block all" / "redirect all" patterns (`urlFilter` `*` or regex `.*` with main_frame) require an explicit confirmation; the popup's global kill switch disables all rules in one tap; the blocked page offers "allow once" |

## 6.3 Privacy

- No telemetry, no analytics, no network requests from the extension.
- Match log is session-only (`storage.session`) and capped at 200 entries.
- Store listings declare: no data collected (Chrome Web Store privacy
  practices form, Apple privacy nutrition labels, `PrivacyInfo.xcprivacy`
  with no tracking domains and the `UserDefaults` reason only if used).
- Privacy policy page is a static file in `docs/` published via GitHub
  Pages.

## 6.4 Review checklist for every PR

- No new permission without an ADR.
- No rule data crosses into the MAIN world.
- Every new input is validated by the Zod schema with a length limit.
- Every redirect target path goes through `validateTarget()`.
- New dependencies are justified in the PR and pinned.
