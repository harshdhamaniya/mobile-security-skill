# Taint Analysis (Android + iOS)

Goal: prove whether attacker-controlled data (source) reaches a dangerous operation (sink) without validation (sanitizer). Also reverse direction: sensitive data (source) reaching leak points (sink).

## 1. Method
1. Define two taint classes:
   - A. Untrusted input to dangerous sink (injection, file, exec, WebView, IPC).
   - B. Sensitive data to leak sink (log, network, clipboard, storage, IPC, analytics).
2. Static: build call graph, inter-procedural data flow, include lifecycle/callback modeling, reflection, JNI, Kotlin coroutines/lambdas, Swift closures, ObjC runtime selectors, Flutter/RN bridges.
3. Dynamic confirm: Frida/objection hooks on sinks, log arguments with call stack, feed marker strings (e.g. `TAINT_7f3a`) through each source and grep.
4. Triage: reachable by whom (remote, other app, physical, none)? Sanitizer adequate? Report path (source -> propagators -> sink).

## 2. Tools
- Android: FlowDroid + Soot, Amandroid, Jadx + Semgrep rules, CodeQL (Java/Kotlin), MobSF, Frida `Stalker`/hooks, Taintdroid-style markers, `jadx --show-bad-code` for obfuscated code, mapping.txt for deobfuscation.
- iOS: CodeQL (Swift), Semgrep (Swift/ObjC), Ghidra/Hopper + `class-dump` for ObjC selector flow, Frida `ObjC.choose`, Cycript-like inspection, MobSF.
- Cross-platform: JS bundle analysis (Semgrep JS, CodeQL JS) for RN/Cordova; Dart snapshot analysis; .NET IL taint (ILSpy + manual).

## 3. Android sources
- Intent extras, data URI, ClipData, `getIntent().getData()`, `getStringExtra`, `Bundle`, deep link params, App Links
- ContentProvider `query/insert/update/delete/openFile` args, `Uri` segments, `selection`, `sortOrder`
- BroadcastReceiver intents, bound Service `Messenger`/AIDL args
- WebView: `@JavascriptInterface` args, `shouldOverrideUrlLoading` URLs, `onJsPrompt/Alert`, postMessage, `WebMessagePort`
- Network responses, WebSocket frames, push/FCM payload, SMS, NFC NDEF, Bluetooth/BLE GATT, USB, QR scan
- Clipboard (`ClipboardManager.getPrimaryClip`), `SharedPreferences`/files/DB read-back, external storage, MediaStore, DocumentsProvider
- `Build`, `Settings.Secure`, user-controlled system data, `Accessibility` events, keyboard IME text, notifications, Share targets (`ACTION_SEND`), drag and drop, Autofill
- Sensitive-data sources (class B): `AccountManager`, Keystore secrets, passwords in EditText, tokens, contacts, SMS, location, device IDs

## 4. Android sinks
- Exec/injection: `Runtime.exec`, `ProcessBuilder`, `System.load/loadLibrary`, `DexClassLoader/PathClassLoader/InMemoryDexClassLoader`, reflection `Class.forName/Method.invoke`
- SQL: `rawQuery`, `execSQL`, `SQLiteQueryBuilder` w/o projection map, Room `@RawQuery`
- File: `new File(path)`, `FileInputStream/OutputStream`, `openFile`, `ZipEntry.getName` (Zip Slip), `Uri.getLastPathSegment`, `FileProvider`, `MediaStore`
- WebView: `loadUrl`, `loadData(WithBaseURL)`, `evaluateJavascript`, `addJavascriptInterface`, `setAllowFileAccess*`
- IPC re-dispatch: `startActivity/Service/sendBroadcast` with intent from input (intent redirection), `PendingIntent` mutable/implicit, `setResult`
- Crypto: `SecretKeySpec`, `Cipher.init`, `Mac` w/ attacker keys/IV
- Deserialization: `ObjectInputStream.readObject`, `Parcel.readSerializable`, Gson/Jackson polymorphic, `Parcelable` mismatch (bundle mismatch), `XmlPullParser` XXE, `Yaml`
- Leak sinks (class B): `Log.*`, `System.out`, `Toast`, network `HttpURLConnection/OkHttp/Retrofit`, `SharedPreferences.edit()`, external files, `ClipboardManager.setPrimaryClip`, `Intent.putExtra` to implicit, `Notification`, analytics SDK calls (`FirebaseAnalytics.logEvent`), `SmsManager`, `Bundle` in `onSaveInstanceState`, `WebView.loadUrl` with query tokens, screenshots
- Auth/logic: `PackageManager` checks, `Binder.getCallingUid` bypass, URL parse (`Uri.parse(...).getHost()` vs actual host; `startsWith` validations), `String.contains` host checks
- Crash/DoS: unchecked casts, `Integer.parseInt`, unbounded allocations

## 5. Android sanitizers to verify
- Canonical path checks (`getCanonicalPath().startsWith(base)`), allow-lists, `Uri` scheme/host exact match, `PendingIntent.FLAG_IMMUTABLE`, `intent.setPackage/setComponent`, `ParcelableCompat`, parameterized queries, `HtmlCompat`/JS encoding, signature permission checks, `isExported` + caller verification (`getCallingActivity`, `getReferrer`).

## 6. iOS sources
- URL handlers: `application(_:open:options:)`, `scene(_:openURLContexts:)`, `application(_:continue:)` Universal Links, `NSUserActivity`, URL scheme params
- Pasteboard `UIPasteboard.general`, drag/drop `UIDropInteraction`, share extensions, Document picker/`UIDocumentInteractionController`, AirDrop files
- Network: `URLSession`, WebSocket, push payload `userInfo`, `didReceiveRemoteNotification`
- WKWebView: `WKScriptMessageHandler` messages, `decidePolicyFor navigationAction`, JS bridge args
- XPC/Mach services, `NSXPCConnection`, custom `CFMessagePort`, `MultipeerConnectivity`, Bluetooth, NFC (CoreNFC), QR (AVCapture)
- Keychain/UserDefaults/files read-back, Siri intents, Shortcuts parameters, widget/extension `AppGroup` data, `NSItemProvider`
- Sensitive sources (B): Keychain items, `UITextField` (secure), Contacts, Photos, location, `identifierForVendor`, health data

## 7. iOS sinks
- `NSTask`/`posix_spawn`/`system` (rare, jailbreak), `dlopen`, `NSClassFromString` + `performSelector`, `NSInvocation`
- SQL: `sqlite3_exec`, `sqlite3_prepare` with string concat, FMDB `executeQuery:` w/ format strings, CoreData `NSPredicate(format:)` with user input
- Format strings: `NSLog(userInput)`, `String(format:)`, `stringWithFormat:`
- File: `FileManager` paths from input (traversal), `NSKeyedUnarchiver` (use `requiringSecureCoding`), `NSCoding` object injection, `plist` deserialization, `ZIPFoundation/SSZipArchive` Zip Slip
- WKWebView: `load`, `loadHTMLString(baseURL:)`, `evaluateJavaScript`, `add(scriptMessageHandler)`, `UIWebView` (deprecated)
- `UIApplication.shared.open` w/ attacker URL, `canOpenURL`
- Leak sinks (B): `NSLog/print/os_log(.public)`, `URLSession` to third parties, `UserDefaults`, `UIPasteboard.general.string`, files in backup, analytics SDKs, `UIActivityViewController` items, `NSUserActivity` userInfo (Handoff/Spotlight indexing `CSSearchableItem`), crash reporters
- Memory: `strcpy/sprintf/memcpy` in C/ObjC with attacker length

## 8. Test cases (MST-BOTH-TNT)
| ID | Scenario | Pass criteria |
|---|---|---|
| TNT-01 | Deep link param -> WebView `loadUrl` | Scheme/host allow-list enforced |
| TNT-02 | Intent extra -> `startActivity` (intent redirection) / URL -> `UIApplication.open` | Target component fixed |
| TNT-03 | Provider `selection`/URI segment -> SQL / file path | Parameterized/canonicalized |
| TNT-04 | Network/push payload -> reflection/class loading/deserialization | No dynamic type from input |
| TNT-05 | Archive entry name -> file write (Zip Slip) | Canonical path check |
| TNT-06 | JS bridge args -> exec/file/SQL/intents | Validated, origin-gated |
| TNT-07 | Clipboard content -> processing (parse URL/command) | Treated untrusted |
| TNT-08 | Password/token -> Log/clipboard/analytics/crash | No flow |
| TNT-09 | PII (contacts/location/IDs) -> third-party SDK/network | Consent + minimization |
| TNT-10 | Input -> `format string` (NSLog/String.format) | Literal format only |
| TNT-11 | Input -> crypto params (key/IV/alg id) | Not controllable |
| TNT-12 | Input -> XML/JSON/YAML parser (XXE, polymorphic types) | Secure parser config |
| TNT-13 | Input -> native (JNI/C) buffer lengths | Bounds-checked |
| TNT-14 | Server redirect/`Location` -> auth token attached to other host | Host re-validated |
| TNT-15 | Intent/URL -> `PendingIntent`/Handoff data carrying secrets | Immutable, minimal |
| TNT-16 | Autofill/IME/accessibility text -> sensitive-field logic | Not trusted |
| TNT-17 | Inter-app results (`onActivityResult`, `openURL` callbacks, x-callback-url) -> trust decisions | Verified caller |
| TNT-18 | Taint through storage round trip (write attacker data to prefs/DB, later used in sink) | Re-validated on read |
| TNT-19 | Dynamic confirmation with markers for each static path rated High | Evidence logged |
| TNT-20 | Cross-language flows (Dart/JS -> platform channel/native bridge -> sink) | Channel args validated |

## 9. Semgrep/CodeQL starting rules (conceptual)
- Java: source `Intent.get*Extra` -> sink `WebView.loadUrl`, `Runtime.exec`, `rawQuery`, `new File`.
- Kotlin: same plus `Uri.parse` host `startsWith` anti-pattern, `apply {}` chains.
- Swift: `URLComponents` queryItems -> `WKWebView.load`, `FileManager`, `NSPredicate(format:)`.
- ObjC: `NSURL` from `openURL` options -> `performSelector`, `sqlite3_exec`, `stringWithFormat`.
- Treat missing sanitizers as findings only if the path is reachable by the stated attacker model.
