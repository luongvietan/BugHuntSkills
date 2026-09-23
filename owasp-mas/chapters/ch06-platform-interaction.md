# Ch06: MASVS-PLATFORM — Interaction with the Mobile Platform

Source: MASTG platform-API chapters, MASWE-0018/0029..0040, platform tests, best-practices MASTG-BEST-0008/0011..0019/0026/0027/0032..0035/0039/0040/0044/0045.

## Weakness types (MASWE)

| ID | Weakness | Core idea |
|---|---|---|
| MASWE-0018 | Exported components lack auth | Any app can invoke privileged activities/services/providers/receivers |
| MASWE-0029 | Insecure deep links | Custom schemes/unvalidated universal links → forced navigation, param injection |
| MASWE-0030 | Improper clipboard use | Secrets copied readable by any app / pasteboard persistence |
| MASWE-0031 | Untrusted app extensions | Third-party keyboards/extensions granted full access to sensitive input |
| MASWE-0032 | Insecure intents | Implicit intents carrying sensitive extras, spoofed broadcast/intent injection |
| MASWE-0033 | Sensitive native functionality exposed in WebViews | JS bridges calling native APIs from web content |
| MASWE-0034 | WebViews access local resources with untrusted content | `allowFileAccess`/`loadURL file://`/content-provider access from remote pages |
| MASWE-0035 | WebViews loading untrusted content | No origin validation → remote XSS-grade content in app context |
| MASWE-0036 | UI exposure of sensitive data | Keyboard caching, unmasked fields, accessibility leakage |
| MASWE-0037 | Sensitive data via notifications | Tokens/OTPs rendered to lock-screen/system UI |
| MASWE-0038 | Insufficient screenshot/screen-record protection | App-switcher snapshots + screen capture expose sensitive views |
| MASWE-0039 | Overlay attacks | Malicious overlay captures taps/credentials (tapjacking) |
| MASWE-0040 | Accessibility-service leakage | Sensitive text readable via accessibility APIs |

## Android surface

- **Exported components** (manifest first read): `android:exported="true"` or implicit export via intent-filter on activity/service/receiver/provider. Test invocation: `adb shell am start -n pkg/.Activity`, `am broadcast`, `content query --uri content://...`. Missing permission/signature protection + sensitive function = finding (MASWE-0018).
- **Content providers**: URI grants (`grantUriPermissions`), path-permission scoping, FileProvider with over-broad roots; provider-backed SQL = SQLi surface via projection/selection args (MASTG-BEST-0039).
- **Intents**: implicit intents for internal comms leak extras to any registered receiver; sensitive data in intent extras = readable by other apps; unvalidated incoming intents inject actions.
- **Deep links/App Links**: custom `scheme://` declared in manifest — any app/web page can fire it; missing `autoVerify` on App Links = hijackable; unvalidated link params → open redirect/XSS/navigation to privileged screens.

## iOS surface

- **Custom URL schemes**: `CFBundleURLTypes` — any app can open them; validate source app and sanitize params (MASWE-0029).
- **Universal links**: domain-verified via `apple-app-site-association` file; missing/incorrect association or unvalidated path params = takeover/phishing.
- **Pasteboard**: general pasteboard readable device-wide; must clear after use, set expiration, restrict to local device (MASWE-0030).
- **App extensions/keyboards**: custom keyboards requesting Full Access can transmit keystrokes; apps should block third-party keyboards on sensitive fields (`application(_:shouldAllowExtensionPointIdentifier:)`) (MASWE-0031/0066).
- **App Groups**: shared containers between app + extensions — same leak class as external storage.

## WebViews — the cross-platform hotspot

- **Bridges**: Android `addJavascriptInterface` = native methods callable from any loaded page (RCE-adjacent on old APIs); iOS `WKScriptMessageHandler` + `evaluateJavaScript` reply — check isolation (content worlds) and never write secrets into the DOM via the bridge (MASWE-0033/0034).
- **File/content access**: `setAllowFileAccess`, `setAllowContentAccess`, `loadUrl("file://…")`, iOS `loadFileURL` with broad read scope — untrusted content reading local files (MASWE-0034).
- **Origin handling**: no allowlist on `shouldOverrideUrlLoading`/`decidePolicyFor` → arbitrary remote content in privileged WebView; deprecated `UIWebView` on iOS = findings by itself (MASWE-0035).
- **Misc**: SafeBrowsing disabled, WebView debugging enabled, WebView caches retaining sensitive data (MASTG-BEST-0028), password fields inside WebView HTML.

## UI leakage (both platforms)

- **Background snapshots**: Android recents screenshot + iOS `Snapshots` cache — verify app masks/blurs sensitive screens on backgrounding (runtime test: open sensitive screen → background → pull snapshot file).
- **Screen capture**: `FLAG_SECURE`/`setRecentsScreenshotEnabled`/`setSecure`/`SecureOn` (Compose) vs iOS screen-recording detection — absence = MASWE-0038 for L2 apps.
- **Keyboard caching**: sensitive fields need `textNoSuggestions`/input-type flags or `isSecureTextEntry`/disable autocorrect; type a secret → check learned-words cache.
- **Notifications**: OTPs/tokens in notification bodies visible on lock screen (MASWE-0037).
- **Overlay/tapjacking**: check `filterTouchesWhenObscured`/overlay protections on sensitive buttons (MASWE-0039/0040).

## Test procedures

1. Manifest/Info.plist review → inventory all entry points: exported components, intent filters, schemes, universal-link domains, WebView usage, extension points.
2. Exercise each entry point: invoke exported components directly, fire crafted deep links (`am start -a android.intent.action.VIEW -d "scheme://payload"`, `simctl openurl`), send spoofed broadcasts, query providers for injection/traversal.
3. Runtime WebView audit: load remote test page → probe bridge objects (`window.<bridge>`), file access, origin checks; monitor `evaluateJavaScript` traffic with Frida.
4. Runtime UI checks: background snapshots, screen-record attempt, pasteboard contents, notification text, keyboard dictionary files.
5. Static grep for the dangerous APIs listed above + runtime Frida hooks confirming reachability.

## Interpretation

- Exported + sensitive-function + no permission check = the classic mobile finding; harmless exported launcher activities are not.
- Deep-link findings need a demonstrated effect: navigation to privileged state, param-driven action, or token-in-link leakage.
- WebView bridge + remote content = critical chain; bridge without remote content is lower.
- UI leaks are moderate individually but chain (snapshot + unencrypted storage + backup = full secret recovery).
