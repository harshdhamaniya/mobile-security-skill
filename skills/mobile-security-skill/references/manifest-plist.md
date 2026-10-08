# Manifest (Android) and Info.plist/Entitlements (iOS)

## A. AndroidManifest.xml (MST-AND-MAN)
Extract with `apktool d`, `aapt2 dump xmltree`, `jadx`. Also check merged manifest (library manifests add components).

| ID | Check | Vulnerable | Fix |
|---|---|---|---|
| MAN-01 | `android:debuggable="true"` | run-as, jdb attach | false in release |
| MAN-02 | `android:allowBackup` / `dataExtractionRules` / `fullBackupContent` | true w/ sensitive data | false or excludes |
| MAN-03 | `usesCleartextTraffic`, `networkSecurityConfig` (cleartextTrafficPermitted, user CAs in `<base-config>`, pin-set expiry, debug-overrides left in release) | Cleartext/user CA trust | HTTPS only, pins |
| MAN-04 | Exported components: activity, service, receiver, provider with `exported=true` or intent-filter w/o `exported` (API<31 default true) | Reachable by any app | exported=false; permission `signature` |
| MAN-05 | Custom permissions: `protectionLevel` normal/dangerous for sensitive; permission squatting (defined in other app first); `permission-tree` | Weak level | `signature` |
| MAN-06 | Content providers: `grantUriPermissions`, `readPermission/writePermission`, `path-permission`, FileProvider `paths.xml` (`root-path`, `.` too broad) | SQLi/path traversal/data leak | Narrow paths, enforce perms |
| MAN-07 | Broadcast receivers: implicit exported, sticky, `registerReceiver` w/o `RECEIVER_NOT_EXPORTED`, ordered broadcast leaks | Spoof/intercept | Permissions, package-scoped |
| MAN-08 | Services: bound services w/o permission, AIDL/Messenger exposure, foreground service types | Unauth IPC | Check caller UID |
| MAN-09 | Intent filters: `autoVerify` for App Links, BROWSABLE + custom scheme, wildcard hosts/pathPattern, `data` w/o host | Link hijack | Verified App Links, assetlinks.json |
| MAN-10 | `taskAffinity`, `launchMode`, `allowTaskReparenting` | StrandHogg-style task hijack | `taskAffinity=""`, singleInstance review |
| MAN-11 | Permissions requested: `READ_SMS`, `READ_CONTACTS`, `MANAGE_EXTERNAL_STORAGE`, `QUERY_ALL_PACKAGES`, `SYSTEM_ALERT_WINDOW`, `REQUEST_INSTALL_PACKAGES`, accessibility, `READ_LOGS`, location background | Over-privilege | Least privilege |
| MAN-12 | `targetSdkVersion`/`minSdkVersion` | Old target disables protections (<28 cleartext, <31 export defaults, <33 notification, <34 FGS) | Raise |
| MAN-13 | `sharedUserId`, `android:process`, `isolatedProcess`, `usesNativeLibrary` | Shared UID data sharing | Remove |
| MAN-14 | `android:testOnly`, `largeHeap`, `extractNativeLibs`, `hasCode` | Test builds in prod | Remove |
| MAN-15 | `android:enabled`/`directBootAware` components handling secrets | Data before unlock | Review |
| MAN-16 | `<queries>` and package visibility | Fingerprinting | Minimal |
| MAN-17 | Signing: v1 only (Janus on <7), v2/v3/v4, debug cert, key reuse across apps, `apksigner verify --print-certs -v` | Weak/ debug sign | v2+/v3, Play App Signing |
| MAN-18 | Dynamic verification: `adb shell dumpsys package <pkg>`, `drozer run app.package.attacksurface`, `am start/broadcast/startservice` fuzz | Confirms exposure | |
| MAN-19 | `android:autoRemoveFromRecents`, `excludeFromRecents`, `windowSoftInputMode`, `showWhenLocked`, `turnScreenOn` | Leak/lockscreen | Review |
| MAN-20 | Credential/Autofill/Passkey (`CredentialProviderService`), `<profileable>` | Perf/profiling exposure | Review |
| MAN-21 | Embedded firebase/google-services values | See advanced file | |

## B. iOS Info.plist and entitlements (MST-IOS-PLS)
Extract: `plutil -p Info.plist`, `codesign -d --entitlements :- App.app`, `otool -l`/`jtool2` for embedded provisioning (`embedded.mobileprovision`).

| ID | Check | Vulnerable | Fix |
|---|---|---|---|
| PLS-01 | `NSAppTransportSecurity`: `NSAllowsArbitraryLoads`, `NSAllowsArbitraryLoadsInWebContent`, `NSAllowsLocalNetworking`, per-domain `NSExceptionAllowsInsecureHTTPLoads`, `NSExceptionMinimumTLSVersion`, `NSRequiresCertificateTransparency` | HTTP/old TLS | ATS strict, pinning (`NSPinnedDomains`) |
| PLS-02 | `CFBundleURLTypes` custom schemes | Hijackable, sensitive actions | Universal Links, validate params |
| PLS-03 | `LSApplicationQueriesSchemes` (scheme probing), `UIApplicationShortcutItems` | Fingerprinting | Minimal |
| PLS-04 | Associated Domains `applinks:`/`webcredentials:`/`activitycontinuation:` + AASA file (`/.well-known/apple-app-site-association`) wildcards, `components` exclusions | Overbroad | Narrow |
| PLS-05 | Usage description strings and permissions (camera, mic, contacts, photos, location Always, Bluetooth, Face ID `NSFaceIDUsageDescription`, tracking) | Over-privilege | Minimal |
| PLS-06 | `UIBackgroundModes` | Unneeded background | Remove |
| PLS-07 | `UIFileSharingEnabled`, `LSSupportsOpeningDocumentsInPlace` | Documents dir exposed in Files app/iTunes | Disable |
| PLS-08 | `CFBundleDocumentTypes`/`UTExportedTypeDeclarations`/`UTImportedTypeDeclarations` | File-open handler abuse | Validate input |
| PLS-09 | Entitlements: `get-task-allow` (debuggable), `com.apple.security.application-groups`, `keychain-access-groups`, `aps-environment`, `com.apple.developer.associated-domains`, `com.apple.private.*`, `dynamic-codesigning`, `cs.allow-jit`, `disable-library-validation` | get-task-allow true in release; excessive groups | Remove |
| PLS-10 | `embedded.mobileprovision`: type (dev/adhoc/enterprise), ProvisionsAllDevices, team ID, expiry | Dev profile shipped | App Store distribution |
| PLS-11 | Binary protections: PIE, stack canaries, ARC, bitcode n/a, `otool -hv`, `otool -Iv \| grep stack_chk` | Missing | Enable |
| PLS-12 | Encrypted binary check (cryptid) and FairPlay | Info | |
| PLS-13 | Embedded frameworks/dylibs, privacy manifest `PrivacyInfo.xcprivacy`, required-reason APIs | Missing/misdeclared | Add |
| PLS-14 | `NSUserActivityTypes` (Handoff), `NSExtension` types (share, widget, keyboard `RequestsOpenAccess`, Intents), `INIntent` | Data exposed to extensions | Review |
| PLS-15 | `UIApplicationSceneManifest`, `UISupportsDocumentBrowser`, `UIRequiresPersistentWiFi` | Info | |
| PLS-16 | Hardcoded API keys/secrets in plist, `GoogleService-Info.plist` values restrictions | Unrestricted keys | Restrict by bundle ID/API |
| PLS-17 | `WKAppBoundDomains`, `NSAppleEventsUsageDescription`, `NSLocalNetworkUsageDescription`/Bonjour services | Review | |
| PLS-18 | Dynamic: `objection ios plist cat`, `ios bundle`, `ios url`, `lsof`, `fsmon` | Evidence | |
