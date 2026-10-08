# Local Storage, Keystores, and Local File Encryption/Decryption

Run dynamic checks only if a device is connected. Exercise the app first (login, use features, background it, log out) then inspect.

## A. Android local storage (MST-AND-STO)
| ID | Check | How | Vulnerable signal | Fix |
|---|---|---|---|---|
| STO-01 | SharedPreferences plaintext secrets | `adb shell run-as <pkg> ls shared_prefs` (debuggable) or root; grep tokens, passwords, PII | Tokens/PII/keys in XML | Store in Keystore-backed EncryptedSharedPreferences/DataStore with Tink; keep tokens short-lived |
| STO-02 | SQLite/Room unencrypted | pull `databases/`, open with sqlite3 | Readable sensitive tables, `-wal`/`-journal` leftovers | SQLCipher w/ Keystore-wrapped key; secure_delete |
| STO-03 | Files dir and cache dir | list `files/`, `cache/`, `code_cache/`, `no_backup/` | Plaintext docs, logs, tokens, cached API responses | Encrypt, minimize, clear on logout |
| STO-04 | External/shared storage | `/sdcard/Android/data/<pkg>`, `/sdcard/`, MediaStore usage | Sensitive data world/app-readable | Use app-private storage; scoped storage |
| STO-05 | World-readable/writable modes | grep `MODE_WORLD_READABLE/WRITEABLE`, `setReadable(true,false)`, chmod 777 | Present | Remove |
| STO-06 | Backup exposure | manifest `allowBackup`, `fullBackupContent`, `dataExtractionRules`; `adb backup`/Auto Backup test | Sensitive files included | allowBackup=false or exclude rules, `android:fullBackupOnly` review |
| STO-07 | Logs | `adb logcat` during flows; grep Log.d/Timber, `System.out` | Secrets/PII/tokens in logs | Strip logs in release (R8 assumenosideeffects), no sensitive logging |
| STO-08 | Screenshots/recents | `FLAG_SECURE` on sensitive screens; recents thumbnail | Missing | Set FLAG_SECURE, setRecentsScreenshotEnabled(false) |
| STO-09 | Keyboard/autofill caches | `inputType` flags, `importantForAutofill`, custom dictionary leakage | Password fields cached | textNoSuggestions, textPassword, autofill hints |
| STO-10 | WebView storage | `app_webview/` cookies, localStorage, IndexedDB, Cache | Sessions/tokens persisted | Clear on logout, no persistence for sensitive sites |
| STO-11 | Temp files from share/export/camera/PDF | `getExternalCacheDir`, FileProvider paths | Left after use | Delete after use; secure FileProvider paths |
| STO-12 | Crash reports/analytics payloads | Inspect SDK (Crashlytics, Sentry) payload | PII/tokens in breadcrumbs | Scrub |
| STO-13 | Residual data after uninstall/logout | Re-install, check Keystore alias persistence, `Auto Backup` restore | Data survives | Clear aliases/data on logout |
| STO-14 | Clipboard/notification leakage into storage | See ref-clipboard-webview-oauth.md | | |
| STO-15 | Firebase/Realm/MMKV/Hive/ObjectBox/DataStore files | Locate vendor files in `files/` | Unencrypted | Use vendor encryption options |
| STO-16 | Android Keystore usage | `KeyGenParameterSpec`: `setUserAuthenticationRequired`, `setInvalidatedByBiometricEnrollment`, StrongBox, `setUnlockedDeviceRequired`, purposes, block modes | Keys with no auth binding for high-value data; keys exportable (pre-API 23 KeyStore) | Hardware-backed, auth-bound keys |
| STO-17 | Accounts/AccountManager, ContentProviders storing secrets | `dumpsys account` | Tokens readable | Minimize |
| STO-18 | Multi-user / work profile / app cloning data scope | | | |

## B. iOS local storage (MST-IOS-STO)
| ID | Check | How | Vulnerable signal | Fix |
|---|---|---|---|---|
| STO-01 | NSUserDefaults / plist | `Library/Preferences/*.plist` | Secrets/PII | Keychain |
| STO-02 | Keychain item accessibility | Dump own items (objection `ios keychain dump`); grep `kSecAttrAccessible` | `kSecAttrAccessibleAlways*`, `AfterFirstUnlock` for high-value; no `ThisDeviceOnly`; no access control | `WhenPasscodeSetThisDeviceOnly`/`WhenUnlockedThisDeviceOnly` + `SecAccessControl` (biometry) |
| STO-03 | Keychain survives uninstall | Reinstall, read items | Old tokens persist | First-run wipe, track install flag |
| STO-04 | Keychain access groups/sharing | entitlements `keychain-access-groups` | Overly broad | Narrow |
| STO-05 | CoreData/SQLite/Realm/YapDatabase | `Documents/`, `Library/` | Plaintext | SQLCipher/Realm encryption key from Keychain |
| STO-06 | File Data Protection class | `NSFileProtectionKey`, `.completeFileProtection`; entitlement `com.apple.developer.default-data-protection` | None/`CompleteUntilFirstUserAuthentication` for sensitive | `.complete` |
| STO-07 | Backup (iTunes/iCloud) inclusion | `isExcludedFromBackup`; Documents vs Library/Caches | Sensitive files backed up | Exclude, encrypt |
| STO-08 | Snapshots/app switcher | `Library/SplashBoard/Snapshots` | Sensitive screen captured | Overlay on `applicationWillResignActive`/scene `sceneWillResignActive` |
| STO-09 | Keyboard cache/autocorrection | `autocorrectionType`, `isSecureTextEntry`, `textContentType` | Cached | Disable |
| STO-10 | Logs | Console.app / `idevicesyslog`, NSLog/print/os_log `%{public}` | Secrets in logs | Use private log levels, strip |
| STO-11 | WKWebView/NSURLCache/Cookies | `Library/Caches`, `Cookies.binarycookies`, `WebKit/` | Persisted | Ephemeral data store, clear on logout |
| STO-12 | tmp/Caches residue | `tmp/`, `Library/Caches` | Plaintext exports/PDF | Delete |
| STO-13 | Pasteboard | See clipboard file | | |
| STO-14 | Shared containers (App Groups, extensions, widgets) | `group.*` container | Unprotected shared data | Encrypt, minimize |
| STO-15 | iCloud/CloudKit/ubiquity container | | Sensitive in iCloud Drive | Encrypt/E2EE |
| STO-16 | Secure Enclave / biometric binding | `kSecAttrTokenIDSecureEnclave`, `LAContext` usage | Boolean `evaluatePolicy` only (bypassable via Frida) | Keychain item with `.biometryCurrentSet`; verify result server-side or by key use |

## C. Local file encryption/decryption logic review (MST-BOTH-CRY)
Applies to Java/Kotlin, JNI/native (C/C++/Rust), Swift/ObjC, and JS/Dart bundles.

### C1. Java/Kotlin (Android)
| ID | Check | Search/How | Vulnerable | Fix |
|---|---|---|---|---|
| CRY-01 | Algorithm/mode | grep `Cipher.getInstance`, `MessageDigest`, `Mac`, `SecretKeySpec` | DES/3DES/RC4/Blowfish, `AES` (defaults to ECB), `AES/ECB`, `NoPadding` w/o integrity, CBC w/o MAC (padding oracle) | AES-GCM/ChaCha20-Poly1305 via Tink |
| CRY-02 | Key hardcoded/derived from constants | `SecretKeySpec(` with literal/`getBytes`, strings in resources, BuildConfig, gradle props | Key/IV/password in code, assets, `strings.xml`, native | Keystore-generated keys, wrapped per-install |
| CRY-03 | IV/nonce handling | `IvParameterSpec`, `GCMParameterSpec` | Static/zero IV, nonce reuse in GCM, IV derived from key | Random 12-byte nonce per message via `SecureRandom` |
| CRY-04 | Randomness | `java.util.Random`, `Math.random`, `new SecureRandom(seed)`, time seeds | Predictable | `SecureRandom()` default |
| CRY-05 | KDF | PBKDF2 iterations, MD5/SHA1 passwords, no salt | <600k PBKDF2-SHA256 / weak | Argon2/scrypt/PBKDF2 high iter, unique salt |
| CRY-06 | Integrity/authentication | encrypt-without-MAC, file header tamper, downgrade of algorithm id | Malleable ciphertext | AEAD; bind metadata as AAD |
| CRY-07 | Key storage | Key bytes written to prefs/files/logs; Keystore alias reuse | Plaintext key beside ciphertext | Keystore non-exportable |
| CRY-08 | Key lifetime/rotation | Same key for all users/files; key not invalidated on logout/biometric change | Global key | Per-user/per-file DEK wrapped by KEK |
| CRY-09 | Plaintext residue | Decrypt-to-temp-file, memory not zeroed, cache of decrypted bitmap/PDF, `FileProvider` exposure | Decrypted file left on disk | Stream decrypt in memory; delete; `EncryptedFile` |
| CRY-10 | Exception handling | Catch-all returning plaintext/default; error oracle messages | Fail-open | Fail closed, uniform errors |
| CRY-11 | Custom/home-grown crypto, XOR, base64 "encryption", obfuscated custom loops | Review classes under `*crypto*`, `*util*` | Present | Replace with vetted libs |
| CRY-12 | Compression+encryption ordering, signature verification of downloaded encrypted payloads | | Missing signature | Sign-then-encrypt / verify |
| CRY-13 | R8/ProGuard hides keys? | Obfuscation not a control | Key recoverable by hooking `Cipher.doFinal` | Treat as client-recoverable |
| CRY-14 | Runtime verification | Frida hook `javax.crypto.Cipher.init/doFinal`, `SecretKeySpec.<init>`, `Mac.doFinal`, `MessageDigest.digest` to log alg, key length, iv | Confirms static findings | Evidence |

### C2. Native / JNI / C++ / Rust (Android and iOS)
| ID | Check | How | Vulnerable | Fix |
|---|---|---|---|---|
| CRY-20 | Locate crypto in native libs | `ls lib/*/`, `strings`, `rabin2 -zz`, Ghidra; constants (AES S-box `63 7c 77 7b`, SHA-256 K, ChaCha "expand 32-byte k") | Embedded keys/IV in .so `.rodata`, tables | Move key to Keystore/Secure Enclave |
| CRY-21 | JNI boundary | `RegisterNatives`, `Java_*` exports; Java passes key in plain | Key crosses boundary in byte[] | Minimize lifetime |
| CRY-22 | Native key derivation | Hardcoded salts, anti-debug-as-security, XOR-obfuscated keys | Recoverable via emulation/Frida (`Interceptor.attach` on exports; Unicorn) | Server-held secrets, hardware keys |
| CRY-23 | Crypto library versions | OpenSSL/BoringSSL/mbedTLS/libsodium symbols/version strings | Old CVE versions | Update |
| CRY-24 | Memory handling | `memset` removal by compiler, key buffers not wiped, `memcmp` for MAC compare | Timing leaks, residue | `explicit_bzero`, constant-time compare |
| CRY-25 | Memory safety around crypto parsers | Fuzz file-format parsers, unbounded copies, integer overflow on lengths | Crash/RCE | Safe langs/fuzzing/ASAN/HWASAN |
| CRY-26 | Native file I/O encryption | `fopen` plaintext paths, custom container formats | Plain temp | Same as CRY-09 |
| CRY-27 | Debug/test keys, symbols left | `nm -D`, `strip` state | Symbols reveal key funcs | Strip |
| CRY-28 | Dynamic extraction attempt (own app) | Frida hook exports/`EVP_*`, `AES_*`, `CCCrypt`; dump args | Key visible | Evidence; reassess threat model |

### C3. Swift/Objective-C (iOS)
| ID | Check | How | Vulnerable | Fix |
|---|---|---|---|---|
| CRY-40 | CommonCrypto misuse | grep `CCCrypt`, `kCCOptionECBMode`, `kCCAlgorithmDES/RC4`, `CC_MD5`, `CC_SHA1` | ECB/weak algs | CryptoKit AES.GCM / ChaChaPoly |
| CRY-41 | CryptoKit/Security.framework | `SymmetricKey(data:)` from literal, `SecKeyCreateRandomKey` flags | Key from string/const | Key from Keychain/Secure Enclave |
| CRY-42 | Random | `arc4random` ok; `rand()/srand(time)` | Predictable | `SecRandomCopyBytes` |
| CRY-43 | Keys in plist/Info.plist/Assets/Realm config | `strings`, plist grep | Present | Keychain |
| CRY-44 | Nonce reuse, AAD, tag verification | `AES.GCM.SealedBox`, manual combined | Fixed nonce | Random nonce |
| CRY-45 | Biometric gate only in app logic | Hook `LAContext evaluatePolicy:` to return YES | Gate bypass | Keychain ACL binding |
| CRY-46 | Decrypt-to-disk residue | `NSTemporaryDirectory`, `UIDocumentInteractionController` | Plaintext copies | Delete, protection class |
| CRY-47 | File protection with encrypted containers | Combine `.complete` + app-level AEAD | Single layer | Defense in depth |
| CRY-48 | Runtime hook of CCCrypt/CryptoKit | Frida/objection | Evidence | |

### C4. Cross-platform
- Flutter: strings in `libapp.so`/Dart snapshot (blutter), `flutter_secure_storage` options (`encryptedSharedPreferences`, iOS accessibility), `shared_preferences` plaintext, `hive` boxes unencrypted.
- React Native: AsyncStorage plaintext; JS bundle/Hermes bytecode keys; `react-native-keychain` options; MMKV encryption key.
- Xamarin/.NET: `assemblies/*.dll` (ILSpy), `Preferences` plaintext, SecureStorage.
- Cordova/Ionic: `www/` JS keys, localStorage, SQLite plugin.
- Unity: `Assembly-CSharp.dll`, `PlayerPrefs` plaintext, IL2CPP metadata.
