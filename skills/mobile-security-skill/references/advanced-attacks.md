# Advanced / Complex Attack Types and Detections

For each: what the attack is, how to TEST your own app for exposure, how to DETECT it in code/runtime, and the fix. IDs: MST-<platform>-ADV-NN.

## A. Android IPC and UI-layer attacks
| ID | Attack | Test (own app) | Detection signal | Fix |
|---|---|---|---|---|
| ADV-01 | Intent redirection (nested intent, `Intent` in extras launched by exported component) | Send crafted extra intent to exported component targeting private component | `startActivity((Intent) getIntent().getParcelableExtra(...))` | Don't launch intents from input; verify target; `FLAG_IMMUTABLE` |
| ADV-02 | Confused-deputy via mutable/implicit `PendingIntent` | grep `PendingIntent.get*`; flags | Missing `FLAG_IMMUTABLE`, empty base intent | Immutable, explicit |
| ADV-03 | Tapjacking / overlay / clickjacking (`SYSTEM_ALERT_WINDOW`, toast overlays) | Overlay test app over sensitive buttons | Missing `filterTouchesWhenObscured`, `setHideOverlayWindows` (API 31) | Set filter flag, confirm dialogs, `FLAG_SECURE` |
| ADV-04 | Task hijacking / StrandHogg / StrandHogg 2 | Attacker app with matching `taskAffinity`, `allowTaskReparenting` | Manifest affinity config, launchMode | `taskAffinity=""`, `singleInstance`, target >= 30 |
| ADV-05 | Accessibility service abuse (overlay credential theft, auto-click) | Enable test a11y service, read nodes | Sensitive views lack `importantForAccessibility=no`, `setAccessibilityDataSensitive` (API 34) | Mark sensitive, `isAccessibilityTool` checks, detect enabled a11y services |
| ADV-06 | Fragment injection (`PreferenceActivity` with `:android:show_fragment`) | Exported PreferenceActivity | Extends PreferenceActivity w/o `isValidFragment` | Validate |
| ADV-07 | Content provider SQLi / path traversal / `openFile` abuse | `content query --uri ... --where "1=1"`, `../` in segments | See TNT-03 | Parameterize, canonicalize, perms |
| ADV-08 | Broadcast spoof/sniff/ordered hijack, sticky broadcasts | `am broadcast` | Unprotected receivers | Permission, explicit |
| ADV-09 | Bound service/AIDL caller impersonation | Bind from other app | No `getCallingUid` check | Signature perms |
| ADV-10 | Bundle mismatch / Parcel mismatch (LaunchAnywhere-class) | Review custom Parcelable `writeToParcel/createFromParcel` symmetry | Asymmetric read/write | Symmetry, fixed SDK |
| ADV-11 | Deserialization gadgets (Java `Serializable`, Gson polymorphism, Jackson default typing, Parcelable) | Inspect classpath/libs | `ObjectInputStream` on external data | Avoid, allow-list |
| ADV-12 | Dynamic code loading from external/world-writable path or network (`DexClassLoader`, `PackageManager` install, Play Core split install, WebView JS bundles OTA) | Trace loader paths | Loads from SD card/HTTP, no signature check | Signed, app-private, verify |
| ADV-13 | APK/DEX signature bypass classes (Janus <7, Master Key legacy) | `apksigner verify`; minSdk <24 with v1 only | v1-only | v2+ |
| ADV-14 | Zip Slip / path traversal on extract | Malicious archive in own test | See TNT-05 | Canonical check |
| ADV-15 | Native memory corruption (parsers, JNI) | Fuzz with libFuzzer/AFL++, HWASAN/ASAN builds | Crashes | Fix, safe langs, MTE (Pixel 8+) |
| ADV-16 | Screen recording/capture/MediaProjection, Presentation API on secondary display, cast mirroring | Test sensitive screens | FLAG_SECURE missing | Set |
| ADV-17 | Device/ID tracking, fingerprinting, `ANDROID_ID`, `Build.SERIAL` misuse | Review | Persistent identifiers | Scoped IDs |
| ADV-18 | Notification/lock-screen leak, `setVisibility(VISIBILITY_PUBLIC)` | Check lock screen | Sensitive text | Private |
| ADV-19 | Autofill phishing / `autofill` service spoof, Credential Manager misuse | Review | | Domain verification |
| ADV-20 | Instant app / App Clips equivalents, Slices, App Widgets, QS tiles exported providers | Manifest | Exposed | Restrict |
| ADV-21 | Work profile/ multi-user / secondary user data bleed; `Direct Boot` storage | Test | | Use CE storage |
| ADV-22 | Biometric prompt misuse: `BiometricPrompt` w/o `CryptoObject` (bypass via Frida to call success callback) | Hook `onAuthenticationSucceeded` | No crypto-bound result | CryptoObject + server check |
| ADV-23 | Install-time/permission attacks: permission squatting, `sharedUserId` privilege inheritance | | | Signature |
| ADV-24 | Misuse of `ContentResolver.takePersistableUriPermission`, SAF URIs | | Overbroad grants | Scope |
| ADV-25 | Intent scheme and Chrome Custom Tabs-to-app handoffs | | | Validate |

## B. iOS-specific attack classes
| ID | Attack | Test | Detection | Fix |
|---|---|---|---|---|
| ADV-40 | URL scheme hijack / universal link downgrade | Another app registers scheme | Custom scheme for auth | Universal links |
| ADV-41 | Pasteboard sniffing/universal clipboard | See CLP | | |
| ADV-42 | XPC/extension/Handoff/Spotlight/Siri donations leaking data | Inspect `NSUserActivity` `userInfo`, `CSSearchableItem` | Sensitive in index | `isEligibleForSearch=false` |
| ADV-43 | Keychain extraction on jailbroken/backups (`ThisDeviceOnly` absent) | Own device | Accessible classes | See STO-02 |
| ADV-44 | Method swizzling/ObjC runtime tampering of security checks (pinning, jailbreak, LocalAuth) | Frida/Cycript hooks | Checks implemented as single boolean | Layered, server-side verification |
| ADV-45 | Insecure deserialization (`NSKeyedUnarchiver` w/o secure coding, plist, `NSCoding`) | Review | `unarchiveObject(with:)` | `requiringSecureCoding` |
| ADV-46 | Format-string / memory bugs in ObjC/C | Fuzz | | Fix |
| ADV-47 | Binary patching/repack (re-sign with own cert) | Re-sign IPA, check integrity | No integrity check/attestation | App Attest + server checks |
| ADV-48 | dylib injection (`DYLD_INSERT_LIBRARIES` on jailbreak, `LC_LOAD_DYLIB` patch), hooking frameworks (Substrate/Substitute/Frida gadget) | Inspect loaded images `_dyld_image_count` | Unknown dylibs | Detect, treat as untrusted client |
| ADV-49 | Screenshot/screen recording capture detection (`UIScreen.isCaptured`, `userDidTakeScreenshotNotification`) | Test | | Obscure content |
| ADV-50 | Sandbox/App Group data sharing with extensions (keyboard, share) | Review | | Minimize |
| ADV-51 | Local network/Bonjour/Multipeer exposure | | Unauth services | Auth/ATS |
| ADV-52 | Background snapshot, Today widget/Live Activities/Lock screen data exposure | Review | | Redact |
| ADV-53 | Jailbreak/instrumentation detection quality (file checks, `fork`, `dyld`, syscalls) bypass resilience | Try objection bypass | Trivially bypassed | Risk signals + server attestation |
| ADV-54 | Enterprise/TestFlight/dev-signed builds in wild | `embedded.mobileprovision` | Non-App Store | |

## C. Runtime tampering, anti-reverse (MASVS-RESILIENCE), BOTH
Resilience is defense-in-depth, not a replacement for server-side controls. Report absence as Low/Info unless the app protects high-value assets (banking, DRM, payments).
| ID | Check | Test | Detection | Fix |
|---|---|---|---|---|
| ADV-60 | Root/jailbreak detection | Run on rooted/JB device, Magisk DenyList/Shamiko, objection bypass | Single-point checks | Play Integrity (`MEETS_STRONG_INTEGRITY`) / App Attest, server verdict |
| ADV-61 | Debugger detection | Attach jdb/lldb | `isDebuggerConnected`, `ptrace`, `sysctl P_TRACED` | Multiple signals |
| ADV-62 | Frida/hooking detection | Run frida-server/gadget; port 27042, `frida-agent` maps, named threads, `/proc/self/maps` | Absent | Native checks, obfuscation |
| ADV-63 | Emulator detection | Run on emulator | `Build.*` heuristics | Attestation |
| ADV-64 | Repackaging/integrity | Re-sign APK/IPA, patch smali, change strings | App runs normally | Signature check via server-verified attestation, native checksum |
| ADV-65 | Obfuscation/hardening | Read decompiled output | Unobfuscated, class/method names, string constants | R8 full mode, string encryption, native hardening |
| ADV-66 | Anti-tamper of critical logic | Patch license/premium checks | Local-only entitlement | Server entitlements |
| ADV-67 | SSL pinning present and robust | Burp + Frida pinning bypass; `network_security_config` pin-set; OkHttp `CertificatePinner`; iOS `NSPinnedDomains`/TrustKit; check backup pins/expiry | No pinning or trivially bypassed | Pin SPKI, rotation plan |
| ADV-68 | Pinning bypass resilience incl. native/Flutter (BoringSSL in libflutter), `TrustManager` custom, `HostnameVerifier` always-true | grep `X509TrustManager`, `ALLOW_ALL_HOSTNAME_VERIFIER`, `setHostnameVerifier` | Accepts all | Proper validation |
| ADV-69 | Time/clock tampering for licenses/OTP/expiry | Change device time | Client-only time | Server time |
| ADV-70 | Secure screen lock / app lock bypass (task kill, link into screen, biometric result patch) | Test | | Server session + keys bound |
| ADV-71 | Memory inspection (dump process memory for secrets: tokens, keys, PINs) | `fridump`/objection memory search after login | Secrets linger | Zero buffers, short life |
| ADV-72 | Instrumentation via `LD_PRELOAD`/Xposed/LSPosed/Zygisk | Hook detection | | Detect/attest |

## D. Network and backend interface
| ID | Check |
|---|---|
| ADV-80 | TLS: cleartext, weak versions/ciphers, TrustManager overrides, user-CA trust, HSTS, cert validity; test MITM with Burp |
| ADV-81 | API authN/authZ: BOLA/IDOR, mass assignment, JWT flaws (alg none, weak secret, no expiry), missing server-side checks for premium/role flags |
| ADV-82 | Replay/tampering: request signing, nonce, timestamp; HMAC key in app |
| ADV-83 | Rate limiting, OTP brute-force, account enumeration, password reset flow |
| ADV-84 | Hardcoded secrets: API keys, Firebase DB URLs (`/.json` open), Firestore/Storage rules, AWS keys, Google Maps keys restrictions, Stripe secret, Twilio, Algolia admin keys; `trufflehog`/`gitleaks` over decompiled output |
| ADV-85 | GraphQL introspection/over-fetching, debug/staging endpoints in config, hidden feature flags |
| ADV-86 | Certificate/Token binding to device (DPoP/mTLS), attestation validation (Play Integrity token nonce, App Attest assertion counters) |
| ADV-87 | WebSocket/MQTT/gRPC auth; push (FCM/APNs) token leakage and silent push abuse |
| ADV-88 | Third-party SDK network calls (analytics, ads) transmitting PII; SDK versions with CVEs |
| ADV-89 | SMS/OTP auto-read (SMS Retriever vs `READ_SMS`), SIM swap resilience, OTP in notifications |
| ADV-90 | Proxy-unaware traffic (non-HTTP, QUIC/HTTP3, Flutter ignoring proxy) interception methods |

## E. Supply chain and platform frameworks
| ID | Check |
|---|---|
| ADV-100 | Dependency CVEs (OSV-scanner, Dependency-Check, `gradle dependencies`, SPM/CocoaPods lockfiles); abandoned libs |
| ADV-101 | Malicious/over-permissioned SDKs; SDK data collection; obfuscated native libs of unknown origin |
| ADV-102 | Build pipeline: debug flags, test endpoints, unsigned artifacts, secrets in CI logs, source maps shipped (RN), `BuildConfig` leakage |
| ADV-103 | OTA code updates (CodePush/Expo Updates/Shorebird): signature verification, rollback |
| ADV-104 | Flutter: `libapp.so` analysis, platform channels exposure, `--obfuscate --split-debug-info`, pinning via `SecurityContext` |
| ADV-105 | React Native: Hermes bytecode decompile, `NativeModules` exposure, `Linking` handlers, debug menu in release, AsyncStorage |
| ADV-106 | Xamarin/MAUI: DLL decompile, `Mono` runtime flags, embedded secrets |
| ADV-107 | Unity/IL2CPP: global-metadata.dat, cheat/tamper resistance, IAP receipt validation server side |
| ADV-108 | Cordova/Ionic: plugin whitelist, `config.xml`, CSP `<meta>`, `cordova.exec` bridge |
| ADV-109 | Kotlin Multiplatform/Compose: shared-module secrets, expect/actual platform differences |
| ADV-110 | Privacy and compliance: data inventory vs declared (Play Data Safety, Apple Privacy Labels/PrivacyInfo.xcprivacy), consent before SDK init, ATT, children's data |

## F. Logic and business-flow attacks
Client-side trust (price, role, feature flags), race conditions (double-spend/redeem), deep-link-triggered actions, in-app purchase receipt forgery/replay, jailbreak-store bypass (Lucky Patcher-style), license checks, promo abuse, session fixation, concurrent sessions, offline-mode data tampering and sync conflicts, QR/NFC payload trust, biometrics enrollment changes (new fingerprint invalidating keys), device-binding/migration flows, account recovery.

## G. Dynamic checklist per device-connected session
1. Baseline: install, record `adb shell pm list/dumpsys`, file tree snapshot (before/after).
2. Exercise flows, snapshot storage again (diff).
3. Hook crypto, storage, network, WebView, clipboard, IPC sinks; log with stack traces.
4. Fuzz exported components/deep links.
5. MITM traffic with and without pinning bypass.
6. Test background/foreground, logout, uninstall/reinstall residue.
7. Re-test fixes; keep evidence.

## H. Detection engineering (for defenders)
Server-side: attestation verdicts, velocity/anomaly rules, token reuse detection, device-binding mismatch, geo/ASN changes. Client-side signals (send, do not rely on local decision): root/JB indicators, hooking frameworks, overlay/a11y active, debugger, repackaged signature hash, emulator, VPN/proxy, clipboard-change patterns near payment screens. Log security events without sensitive data.
