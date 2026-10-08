# Target Profile — VulnBank Demo (com.vulnbankdemo.app)

## Platform & framework
- **Platform**: Android only. No iOS artifacts of any kind were provided (no `.ipa`, no `Info.plist`, no `embedded.mobileprovision`), so everything iOS-specific in this skill is out of scope for this target.
- **Framework**: Assumed **native Android (Java/Kotlin)** based on the class naming conventions visible in the manifest itself (`.ui.LoginActivity`, `.ui.TransferMoneyActivity`, `.data.AccountProvider`, `.VulnBankApp` — ordinary `<package>.<layer>.<ClassName>` layout with `Activity`/`Provider`/`App` suffixes, no framework-glue class names). This **cannot be verified** — there is no DEX, no `lib/` native libraries, no `assets/flutter_assets`, no `assemblies/*.dll`, and no `Assembly-CSharp.dll`/`global-metadata.dat` to check for Flutter, React Native, Xamarin/MAUI, Cordova/Ionic, or Unity signatures, because **no APK/AAB and no source tree exist for this target** — only a single standalone, already-decoded `AndroidManifest.xml` file.
- **Artifact type**: This is a hand-authored fixture manifest (confirmed via `engagement-vulnbank-demo/scope.md` and the fixture's own `README.md`), not extracted from a real APK. Treat every observation below as manifest-level only; there is no app behavior behind these declarations to confirm or refute statically.

## Attack surface inventory

### Exported components (Android)
All three declared components are the entirety of the manifest's `<application>` block — no `<service>` or `<receiver>` elements are present at all.

| Component | Type | exported | Intent filter | Guard (`android:permission`) |
|---|---|---|---|---|
| `.ui.LoginActivity` | Activity | `true` | `MAIN`/`LAUNCHER` | none |
| `.ui.TransferMoneyActivity` | Activity | `true` | `VIEW`/`DEFAULT`/`BROWSABLE`, data `scheme="vulnbank" host="transfer"` | none |
| `.data.AccountProvider` | ContentProvider | `true`, authority `com.vulnbankdemo.app.provider` | n/a | none, no `<path-permission>`, no `android:grantUriPermissions` restriction |

- `LoginActivity` being exported is expected/required (it's the `LAUNCHER` activity) — not itself a finding.
- `TransferMoneyActivity` exported with a `BROWSABLE` custom-scheme deep link and **no permission guard** on an activity whose name strongly implies a sensitive money-movement action is the standout item here.
- `AccountProvider` exported with **no permission, no path-permissions, no URI-permission restriction** on a provider named for account data is the second standout item. Whether it's actually exploitable depends on its `query()`/`insert()`/`update()`/`delete()` implementation and any `path-permission` scoping — none of which exists in this fixture to inspect.

### Custom URL schemes / App Links / Universal Links
- Custom scheme: `vulnbank://transfer` (declared on `TransferMoneyActivity`, category `BROWSABLE`, so also reachable from a mobile browser/webpage, not just `am start`).
- No Android App Links (no `android:autoVerify="true"`, no `https`-scheme intent filter, no `assetlinks.json` reference).
- iOS Universal Links / `LSApplicationQueriesSchemes`: not applicable (Android-only artifact).

### WebViews found
None observed — but this is **not** a meaningful negative. There is no DEX/source to grep for `WebView`/`WKWebView` instantiation sites, and the manifest itself would not show WebView usage even if present (WebViews are typically embedded inside an Activity's code, not declared in the manifest). Treat WebView presence/absence as **unknown**, not "none."

### Network endpoints observed in strings
None observed. There is no `res/`, `strings.xml`, `resources.arsc`, or any other string table in this fixture to grep `https?://` against. The only network-relevant manifest fact is the `INTERNET` permission below. Treat as **unknown**, not "no endpoints."

### Permissions / entitlements
- `android.permission.INTERNET` — the only permission declared. No dangerous/runtime permissions (camera, contacts, SMS, location, storage, biometrics, etc.) are requested anywhere in this manifest.
- No custom permissions (`<permission>`) are defined anywhere in the file, consistent with none of the three exported components having an `android:permission` guard — there's no custom permission even available to apply.
- iOS entitlements: not applicable.

### Third-party SDKs
Undeterminable. There is no `build.gradle`/`build.gradle.kts`, no dependency lockfile, no embedded `.jar`/`.aar`, and no `.so` native libraries to inspect for analytics/ads/crash-reporter/payment-SDK fingerprints. Nothing in the manifest itself (no SDK-characteristic `<meta-data>` keys, no SDK-owned `<activity>`/`<service>`/`<provider>` entries such as Firebase's `FirebaseMessagingService`, Facebook's `FacebookActivity`, Google Pay's services, etc.) hints at any embedded SDK. Assume **zero evidence either way** rather than "no third-party SDKs."

### Versions
No `<uses-sdk>` element is present in this manifest at all, so **`minSdkVersion`/`targetSdkVersion` are not declared** and cannot be read from this artifact. They would normally come from `build.gradle`/`build.gradle.kts` (source tree) or from `aapt2 dump badging`/`aapt dump badging` against a built APK — neither exists for this fixture. This matters downstream: several reference-doc checks are version-gated (e.g. MAN-12-style cleartext-traffic defaults, which flip based on `targetSdkVersion >= 28`; default `exported` values for components with intent-filters on `targetSdkVersion < 17`). Here, however, all three components set `android:exported` **explicitly**, so the API-17 default-exported ambiguity doesn't actually bite for them specifically — but any later specialist reasoning about cleartext-traffic defaults, backup defaults, or other SDK-version-gated behavior should treat the SDK level as **unknown**, not assume a modern target.

## Notable at-a-glance observations
(Flagged here per instructions so the relevant Phase 2 specialist doesn't miss them; these are observations, not formal findings — no `findings.jsonl` entry has been written.)

1. `android:debuggable="true"` on `<application>` — set explicitly in this manifest. If this ships in a release build, it allows debugger attachment, code injection via JDWP, and generally undermines any other control checked by `resilience-anti-tamper-auditor`.
2. `android:allowBackup="true"` with no `android:fullBackupContent` restriction — permits `adb backup`-style extraction of app data (subject to device/OS backup policy).
3. `TransferMoneyActivity` — exported, `BROWSABLE`, custom-scheme deep link, **zero permission guard**, and a name that implies a sensitive financial action. This is the single most interesting manifest-level fact in this fixture; `android-ipc-ui-auditor` and `deeplink-clipboard-auditor` should both look at it from their respective angles.
4. `AccountProvider` — exported, **zero permission guard**, no path-permissions, name implies account/financial data. Second most interesting fact; `android-ipc-ui-auditor`'s primary target.
5. No custom `<permission>` is defined anywhere, so there isn't even a same-app signature-permission option being used and skipped — the app has no permission infrastructure at all for its own components.
6. No `<uses-sdk>` at all — flag downstream specialists against assuming a specific SDK-version-driven default; treat as unknown (see Versions above).

## Device/emulator availability for dynamic phase
- `adb devices` reports **one connected, authorized device**: `84f7a024	device`.
- However, this is **moot for this specific target**: the fixture is a standalone `AndroidManifest.xml` with no corresponding APK/AAB and no source/DEX to build one. There is nothing installable to put on that device, so Phase 3 (`dynamic-runtime-verifier`) has **no app to exercise** regardless of device availability. Do not let the connected device's presence imply dynamic confirmation is viable here — it is not, for this target, absent a built/installable package. If the orchestrator wants dynamic confirmation of the exported-component exposure (e.g., `adb shell am start -a android.intent.action.VIEW -d "vulnbank://transfer"` or querying the provider via `adb shell content query --uri content://com.vulnbankdemo.app.provider`), that would require first building a minimal stub APK from this manifest — out of scope for recon and not attempted here.

## Recommended Phase 2 specialist subset
Of the 13 static specialists in SKILL.md's Phase 2 table:

**Run:**
- `android-manifest-auditor` (`manifest-plist.md` §A, MAN-01..21) — the primary specialist for this target. Everything available to analyze (debuggable, allowBackup, exported components, missing permission guards, missing `<uses-sdk>`, custom scheme intent-filter) is manifest-level and squarely its remit.
- `android-ipc-ui-auditor` (`advanced-attacks.md` §A, ADV-01..25) — relevant specifically for the exported `TransferMoneyActivity` and exported `AccountProvider`. Scope its work to manifest-level exposure analysis (what an unguarded exported component/provider permits at the IPC surface) since no implementation code exists to trace actual data handled.
- `deeplink-clipboard-auditor` — relevant but **only its DLK sub-scope** (`clipboard-webview-oauth.md` §D, DLK-01..08) for the `vulnbank://transfer` custom-scheme deep link on `TransferMoneyActivity`. Its CLP sub-scope (§A, clipboard read/write) should be skipped — there's no code to show clipboard usage.

**Skip:**
- `ios-plist-entitlements-auditor` — Android-only manifest, no `Info.plist`/entitlements exist.
- `ios-platform-attack-auditor` — Android-only manifest.
- `storage-crypto-auditor` — STO/CRY checks need actual storage/crypto implementation code (SharedPreferences, SQLite, Keystore/Keychain usage, crypto API calls); none exists. `allowBackup="true"` is already captured as a manifest-level fact for `android-manifest-auditor`; don't duplicate it as a storage finding with no implementation evidence behind it.
- `taint-flow-analyst` — requires source/DEX to trace source->sink flows; none exists for this fixture.
- `webview-security-auditor` — no WebView evidence either way (see WebViews section above), and no code to find instantiation sites even if present.
- `oauth-oidc-auditor` — no OAuth/OIDC flow code or configuration present anywhere in this artifact.
- `resilience-anti-tamper-auditor` — ADV-60..72 anti-tamper/root-detection/pinning checks need either code (root-detection logic, pinning implementation) or a dynamic device pass against a running app; neither is available. `debuggable="true"` is already captured for `android-manifest-auditor`.
- `network-backend-auditor` — no backend API code, no network security config, no observable endpoints (see Network endpoints section); only fact available is the bare `INTERNET` permission, already covered by the manifest auditor's permission review.
- `supply-chain-framework-auditor` — no dependency manifest, no embedded SDK/framework artifacts of any kind to review (see Third-party SDKs section).
- `business-logic-auditor` — no implementation exists for the money-transfer flow the activity name implies; nothing to audit for logic abuse.

## Tooling gaps
- `apktool` (jar + bat found at `C:\Program Files (x86)\apktool\` and `C:\apktool\`), `jadx` (`C:\Program Files\jadx\jadx`), and `aapt2` (`C:\Program Files (x86)\apktool\Resources\aapt2`) are all **present on this machine's PATH** — this is not a missing-tool situation in the general sense.
- They are nonetheless **inapplicable to this specific target**: all three operate on an APK/AAB container (binary AXML, DEX, `resources.arsc`, compiled resources) or a source tree. This fixture supplies none of that — only a single plaintext, already human-readable `AndroidManifest.xml`. There is no binary AXML to decode, no DEX to decompile, no `resources.arsc`/`strings.xml` to dump. Running any of these tools against the lone manifest file would do nothing useful, so none were invoked.
- `xcrun`/`simctl` (iOS) were not probed meaningfully since this bash environment is not macOS and the target has no iOS artifacts anyway; irrelevant either way.
- Net effect: every "undeterminable" item above (WebView presence, network endpoints, third-party SDKs, actual min/target SDK, provider implementation behavior) is undeterminable **because the input artifact itself lacks the data**, not because a decompiler was missing or failed. Any later specialist or the final report should state these as "not assessable from available artifacts" rather than "absent."
