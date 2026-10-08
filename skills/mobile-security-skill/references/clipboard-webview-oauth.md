# Clipboard, WebView, OAuth/OIDC, Deep Links

## A. Clipboard (MST-BOTH-CLP)
| ID | Platform | Check | How | Vulnerable | Fix |
|---|---|---|---|---|---|
| CLP-01 | Both | Sensitive fields allow copy | Long-press password/OTP/card/PIN fields | Copy enabled | Disable copy/paste menu on sensitive fields; `isSecureTextEntry`, `textIsSelectable=false`, custom `ActionMode.Callback` |
| CLP-02 | Android | App writes secrets to clipboard | grep `setPrimaryClip`, `ClipData.newPlainText`; hook ClipboardManager | Tokens/passwords/OTP copied | Avoid; if needed set `ClipDescription.EXTRA_IS_SENSITIVE` (API 33+), short expiry `clearPrimaryClip` |
| CLP-03 | Android | Clipboard monitoring by other apps | `addPrimaryClipChangedListener` in test app; API 29+ restricts background reads, but foreground/IME/accessibility apps still read | Leak | Treat clipboard as public |
| CLP-04 | Android 12+ | Clipboard-read toast shown | Observe | Indicates reads | |
| CLP-05 | iOS | `UIPasteboard.general` writes | grep `UIPasteboard.general`, hook | Secrets on general pasteboard | Use `setItems(_:options:[.localOnly:true,.expirationDate:...])`; named private pasteboards deprecated |
| CLP-06 | iOS | Universal Clipboard/Handoff exposure | Copy on device A, paste on device B | Syncs to other devices | `.localOnly` |
| CLP-07 | iOS 14+ | Paste banner reveals silent reads | Observe banners; `UIPasteboard.detectPatterns` use | Unneeded reads (privacy) | Use `UIPasteControl` |
| CLP-08 | Both | App reads clipboard on launch/foreground | Hook `getPrimaryClip` / `UIPasteboard.string` | Silent harvest, SDK reads | Remove, review SDKs |
| CLP-09 | Both | Clipboard content used as input (URLs, deep links, commands, addresses) | Taint (TNT-07) | Hijack: attacker-swapped crypto address/URL | Verify with user, validate |
| CLP-10 | Both | Clipboard hijack/swap resilience (payment/crypto address copy) | Replace clipboard between copy and paste in test | App pastes unverified address | Re-display and confirm last chars; checksum validation |
| CLP-11 | Android | Clipboard in `ClipData` URIs (content://) granting permissions | Review `FLAG_GRANT_READ_URI_PERMISSION` | Unintended grants | Scope |
| CLP-12 | Both | Keyboard/IME and autofill leakage, third-party keyboards `RequestsOpenAccess` (iOS) | Review fields | Sensitive typed through custom keyboard | Force system keyboard for secure fields (`UIApplicationDelegate shouldAllowExtensionPointIdentifier` returns false for keyboards) |
| CLP-13 | Both | Drag and drop, share sheet, text-selection "Share/Translate/Look Up" leak | Select text in sensitive view | Leaves app | Disable |
| CLP-14 | Both | Cleanup on background/logout | Check clipboard after logout | Secret remains | Clear if app wrote it |

## B. WebView (MST-AND-WEB, MST-IOS-WEB)
### B1. Android WebView
| ID | Check | Search/How | Vulnerable | Fix |
|---|---|---|---|---|
| WEB-01 | JavaScript enabled | `setJavaScriptEnabled(true)` | On for untrusted content | Disable unless needed |
| WEB-02 | `addJavascriptInterface` exposure | grep; list `@JavascriptInterface` methods; test from any loaded origin | Exposed to remote/untrusted origins, methods returning secrets/doing file/exec | Origin allow-list (`WebViewCompat.addWebMessageListener` with allowed origins), minimal methods |
| WEB-03 | File access | `setAllowFileAccess`, `setAllowFileAccessFromFileURLs`, `setAllowUniversalAccessFromFileURLs`, `setAllowContentAccess` | true (default allowFileAccess true <API 30) | Set false; use `WebViewAssetLoader` |
| WEB-04 | Loading untrusted/user URLs | taint: deep link/intent -> `loadUrl` | Arbitrary URL, `javascript:`, `file://`, `content://`, `intent://` | Host allow-list, HTTPS only |
| WEB-05 | `shouldOverrideUrlLoading` logic | Review parsing | Prefix checks, `contains`, userinfo bypass (`https://trusted.com@evil.com`), unicode/ports | Strict `Uri` host equals + scheme |
| WEB-06 | TLS errors | `onReceivedSslError` calling `proceed()` | Ignored cert errors | `cancel()` |
| WEB-07 | Mixed content | `setMixedContentMode(MIXED_CONTENT_ALWAYS_ALLOW)` | Allowed | NEVER_ALLOW |
| WEB-08 | Safe Browsing, `onRenderProcessGone`, `setSafeBrowsingEnabled` | | Disabled | Enable |
| WEB-09 | Remote debugging | `WebView.setWebContentsDebuggingEnabled(true)` in release | chrome://inspect exposure | Debug-only |
| WEB-10 | Storage/cookies | `CookieManager` third-party cookies, persistent session, `setDomStorageEnabled`, cache | Tokens persisted | Clear on logout, `setAcceptThirdPartyCookies(false)` |
| WEB-11 | Intent scheme handling | `intent://` parsed via `Intent.parseUri` and launched | Intent redirection to non-exported components | `setComponent(null)`, `addCategory(BROWSABLE)`, `setSelector(null)`, match allow-list |
| WEB-12 | Custom Tabs/TWA vs WebView for auth | Auth in embedded WebView | Credential phishing/keystroke capture by host app | Custom Tabs/ASWebAuthenticationSession |
| WEB-13 | Content injection: `loadData`/`loadDataWithBaseURL` with untrusted HTML, null/`file://` baseURL | | XSS with app privileges | Sanitize, safe base URL |
| WEB-14 | `evaluateJavascript` string concat with untrusted data | taint | JS injection | JSON-encode |
| WEB-15 | Download listener / file chooser / `onShowFileChooser` / `onPermissionRequest` / geolocation grants | Review | Auto-grant camera/mic/location to any origin | Check origin |
| WEB-16 | `postMessage`/`WebMessagePort` origin check | | `*` target origins | Verify origin |
| WEB-17 | Webview in other contexts: `ChromeClient.onCreateWindow` popups, `setSupportMultipleWindows`, `window.open` into privileged context | | Popup gets bridge | Block |
| WEB-18 | Fragment/hybrid frameworks: Cordova (`config.xml` `<access origin="*">`, `allow-navigation`, `allow-intent`), Capacitor (`server.url`, `allowNavigation`), Ionic live reload left on | | Wildcards | Restrict |
| WEB-19 | WebView cache/history in backups, `clearCache`, form data/autofill saved (`setSaveFormData`) | | Persisted | Disable |
| WEB-20 | Data URI / blob / `about:` / `view-source:` handling, JavaScript `javascript:` in links | | Script exec | Block |
| WEB-21 | Custom `WebViewClient.shouldInterceptRequest` serving local files from user path | | Path traversal | Fixed asset loader |
| WEB-22 | Trusted Web Activity / Custom Tabs `CustomTabsService` warmup validation, `Digital Asset Links` | | | Verify relations |

### B2. iOS WKWebView / SFSafariViewController
| ID | Check | Vulnerable | Fix |
|---|---|---|---|
| WEB-30 | Use of deprecated `UIWebView` | Present | Migrate to WKWebView |
| WEB-31 | `WKPreferences.javaScriptEnabled` / `WKWebpagePreferences.allowsContentJavaScript` | Enabled for untrusted | Disable |
| WEB-32 | `WKScriptMessageHandler` bridge exposed to all frames/origins | Check `message.frameInfo.securityOrigin`, `isMainFrame` | Verify origin, `WKContentWorld` isolation |
| WEB-33 | `allowFileAccessFromFileURLs`, `allowUniversalAccessFromFileURLs` (private KVC keys), `loadFileURL(_:allowingReadAccessTo:)` too broad | File read | Narrow dir |
| WEB-34 | `decidePolicyFor navigationAction` missing or weak host checks; `UIApplication.open` with unvalidated URLs; custom scheme launches | Open redirect, scheme abuse | Allow-list |
| WEB-35 | `didReceive challenge` accepting any cert (`.useCredential` with `serverTrust` unvalidated) | MITM | Default handling + pinning |
| WEB-36 | `evaluateJavaScript` with concatenated data | Injection | Encode |
| WEB-37 | Data store: `WKWebsiteDataStore.default()` vs `.nonPersistent()`; cookies in `HTTPCookieStorage`; cache | Persisted sessions | Ephemeral |
| WEB-38 | `isInspectable` (iOS 16.4+) true in release | Safari Web Inspector | Debug-only |
| WEB-39 | Auth in WKWebView | Credential capture | `ASWebAuthenticationSession` |
| WEB-40 | `SFSafariViewController` cookies shared (iOS 11+ not shared), delegate URL tracking | Review | |
| WEB-41 | `WKWebView` content from HTTP, mixed content, `NSAllowsArbitraryLoadsInWebContent` | Insecure | HTTPS |
| WEB-42 | JS to native deep-link `window.location = "myapp://"` triggers sensitive action w/o confirmation | CSRF-style | Confirm, nonce |
| WEB-43 | Universal links opened inside WKWebView bypassing AASA validation | | Review |
| WEB-44 | Pasteboard/`document.execCommand('copy')`, `navigator.clipboard` permissions | Review | |

### B3. Web-layer tests applied inside the WebView (BOTH)
XSS/DOM XSS, open redirects, CSRF on in-app flows, `postMessage`, CSP/headers, JS bridge fuzzing, cache poisoning, token in URL fragments/query, `Referer` leakage, clickjacking (X-Frame-Options when embedding), WebView-to-native privilege escalation chain tests, UXSS via old Android System WebView versions (check WebView provider version).

## C. OAuth 2.0 / OIDC (MST-BOTH-OAU)
Review the authorization flow end to end with Burp/mitmproxy and code review. Key references: RFC 6749, 7636 (PKCE), 8252 (native apps), 9700 (BCP), OAuth 2.1 draft, OpenID Connect Core.

| ID | Check | How | Vulnerable | Fix |
|---|---|---|---|---|
| OAU-01 | Flow type | Inspect authorize request `response_type` | Implicit (`token`) or ROPC (password) in mobile | Authorization Code + PKCE |
| OAU-02 | PKCE | `code_challenge`, `code_challenge_method` | Missing or `plain`; verifier predictable/reused; server doesn't enforce | S256, random 43-128 chars, server enforces |
| OAU-03 | User agent | Embedded WebView vs system browser | WebView for IdP | `ASWebAuthenticationSession`, Custom Tabs, AppAuth |
| OAU-04 | Redirect URI type | Custom scheme vs claimed https | Custom scheme (other app can register same scheme -> code interception) | App Links/Universal Links redirect (verified), PKCE mandatory |
| OAU-05 | Redirect URI validation (server) | Mutate `redirect_uri`: subdomain, path suffix, `..`, case, userinfo, extra params, open redirect on client domain | Loose matching | Exact match |
| OAU-06 | `state` | Present, unpredictable, bound to session, verified on return | Missing/static/unverified (login CSRF) | Verify |
| OAU-07 | `nonce` (OIDC) | In authorize request and `id_token` | Missing/not checked (replay) | Validate |
| OAU-08 | ID token validation | `iss`, `aud`, `exp`, `iat`, `nbf`, signature, `alg` (reject `none`, HS/RS confusion), `azp`, `at_hash`, JWKS fetched over TLS & `kid` handling | Skipped client-side or server-side | Validate fully |
| OAU-09 | Token storage | Keystore/Keychain vs prefs/files/logs | Plaintext | See storage file |
| OAU-10 | Token lifetime & refresh | Access TTL, refresh rotation, reuse detection, binding (DPoP/mTLS), revocation on logout, refresh in background | Long-lived non-rotating refresh tokens | Rotation + revocation |
| OAU-11 | Scopes | Requested vs needed; scope upgrade by tampering | Over-broad | Least privilege |
| OAU-12 | Client secret in app | grep `client_secret`, strings in binary/JS | Embedded secret | Public client; no secret |
| OAU-13 | Token in URL / logs / Referer / history | Observe redirect URI fragments/queries, analytics | Leak | Code flow, POST |
| OAU-14 | Authorization code interception & replay | Intercept code via second app registering same scheme; reuse code | Succeeds | PKCE, single-use short code |
| OAU-15 | Mix-up / IdP confusion (multi-IdP) | `iss` response param (RFC 9207) | No issuer check | Validate `iss` |
| OAU-16 | Account linking / social login: email trust (unverified email), account takeover via pre-registered email | Test w/ attacker IdP identity | Takeover | Use `sub`, `email_verified` |
| OAU-17 | Deep link handler for callback | Callback handler accepts arbitrary `code`/`state`/tokens from any app/URL | Login CSRF/session fixation | Bind to pending request state |
| OAU-18 | Token exchange at server: audience restriction, token substitution (id_token from other client), JWT `aud` | Server accepts tokens for other clients | Confused deputy | Validate `aud` |
| OAU-19 | Logout/session: end_session, token revocation (RFC 7009), IdP session persistence, `prompt=login`, `max_age` | Tokens valid after logout | Revoke |
| OAU-20 | Biometric/step-up gating tokens | Client-only gate | Bypass via hooking | Server-enforced step-up (acr) |
| OAU-21 | Device authorization/QR login flows | Phishing/device code | Verify | Number matching |
| OAU-22 | Dynamic client registration/CIMD, public client impersonation, app attestation (Play Integrity / App Attest) | Client spoofing | Attest |
| OAU-23 | Android account chooser, `AccountManager` tokens shared between apps via `getAuthToken` | Unauthorized app gets token | Review authenticator |
| OAU-24 | iOS: `ASWebAuthenticationSession` `prefersEphemeralWebBrowserSession`, callbackURLScheme collisions; Sign in with Apple `identityToken`/`authorizationCode` server verification, `credentialState` | Review |
| OAU-25 | Google/Facebook SDK use: `serverClientId`, token audience at backend, `FacebookSdk` client token embedded | Token substitution | Backend verify |
| OAU-26 | Tampering matrix with Burp: remove state, swap code, swap redirect, replay, downgrade PKCE, change `response_mode` | Document each | |

## D. Deep links / Universal links (MST-BOTH-DLK)
| ID | Check | Fix |
|---|---|---|
| DLK-01 | Custom scheme collisions / hijack (second app registers scheme) | Verified App/Universal Links |
| DLK-02 | `assetlinks.json` / AASA fetch, wildcards, fingerprints, subdomain takeover of linked hosts | Narrow, monitor |
| DLK-03 | Link parameters trigger sensitive actions (payments, auth, settings) without confirmation | Confirm, nonce |
| DLK-04 | Parameter injection into WebView/intent/SQL/file | See taint |
| DLK-05 | Deferred deep links, install referrer (Android `InstallReferrerClient`), attribution SDK (Branch/AppsFlyer/Adjust) trust | Validate |
| DLK-06 | Auth state bypass: link opens authenticated screen without session/biometric lock | Gate in destination |
| DLK-07 | Link fuzzing: `adb shell am start -a android.intent.action.VIEW -d "<uri>"`; `xcrun simctl openurl booted "<uri>"` / `uiopen` | Crash/exception handling |
| DLK-08 | iOS `x-callback-url`, `UIApplication.open` return trust | Verify source app (`options[.sourceApplication]` is spoofable) |
