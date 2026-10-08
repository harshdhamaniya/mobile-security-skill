# Dynamic Confirmation Session Log — VulnBank Demo (com.vulnbankdemo.app)

**Role**: dynamic-runtime-verifier (Phase 3)
**Date**: 2026-10-09
**Operator context**: harshdhamaniyainfosec@gmail.com (local session)
**Scope authority**: `scope.md` — "Target: VulnBank-Demo (com.vulnbankdemo.app)... In scope: AndroidManifest.xml review... Out of scope: everything else, since no real code exists for this fixture."

## 0. Why this session is unusually short, and why that's the honest answer

`target-profile.md` already flagged, before I started, that this engagement's artifact is a single hand-authored `AndroidManifest.xml` with **no APK/AAB, no DEX, no source tree** behind it. `exploitation/AND-MAN-003.md` (the prosecutor/skeptic transcript for the one finding that went through the panel) independently reconfirms this and explicitly names the dynamic gap as the reason that finding is capped at `potential`/`high` rather than `confirmed`/`critical`.

My job per the agent brief is to produce *evidence*, not to assume the recon note is still accurate without checking. So rather than skip Phase 3 outright on the strength of `target-profile.md`'s say-so, I independently re-verified today, against the device that is actually connected right now, that there is genuinely nothing installable to exercise. That re-verification — and its negative result — is itself the dynamic evidence this session produces. There is no app behavior to hook, no traffic to MITM, no storage to diff, no biometric gate to bypass, and no exported component to fuzz at runtime, because there is no running app. Sections 3–9 of my own method (hooking sinks, fuzzing deep links with real traffic, MITM, resilience-bypass attempts, biometric-gate bypass, background/foreground residue diffing) are **not attempted** below, and this section states why for each, rather than silently omitting them.

## 1. Baseline: device and tooling check

Commands run, in order:

```
$ adb version
Android Debug Bridge version 1.0.41
Version 35.0.2-12147458
Installed as C:\Program Files (x86)\platform-tools\adb.exe

$ adb devices -l
List of devices attached
84f7a024        device product:sweetin model:M2101K6P device:sweetin transport_id:2
```

One authorized device is genuinely connected, confirming `target-profile.md`'s claim rather than just trusting it.

```
$ adb shell getprop ro.build.version.release        -> 11
$ adb shell getprop ro.build.version.sdk             -> 30
$ adb shell getprop ro.product.model                 -> M2101K6P
$ adb shell getprop ro.product.manufacturer          -> Xiaomi
$ adb shell getprop ro.build.fingerprint             -> Redmi/sweetin/sweetin:11/RKQ1.200826.002/V12.5.10.0.RKFINXM:user/release-keys
$ adb shell getprop ro.debuggable                    -> 0
$ adb shell getprop ro.secure                        -> 1
$ adb shell which su                                 -> /system/bin/su
```

Device is a Xiaomi/Redmi phone (codename `sweetin`), Android 11 (SDK 30), a `user`/`release-keys` (production) build. `ro.debuggable=0` / `ro.secure=1` describe the *device's own OS build*, not the target app — recorded for baseline completeness only; they are not a substitute for, and must not be conflated with, the app's own `android:debuggable="true"` manifest attribute (`AND-MAN-001`), which is a separate per-APK property that can only be confirmed by installing the actual app. `/system/bin/su` being present on a MIUI stock image is not uncommon and was not pursued further (device rooting status is not part of this engagement's target).

## 2. Confirming the target package's install state (the actual Phase 3 question for this engagement)

```
$ adb shell pm list packages | grep -i vulnbank
(no output, exit code 1)

$ adb shell pm list packages | wc -l
389
```

`com.vulnbankdemo.app` is not among the 389 installed packages on this device.

```
$ adb shell pm list packages | grep -i bank
package:com.app.damnvulnerablebank
```

The only "bank"-named package present is `com.app.damnvulnerablebank` — an unrelated, pre-existing app on this shared test device (not the target of this engagement; not authorized by `scope.md`; **not touched or queried beyond this one grep line**). Flagging this explicitly so no reader mistakes this device's general contents for evidence about `com.vulnbankdemo.app`.

```
$ adb shell dumpsys package com.vulnbankdemo.app
...
  Unable to find package: com.vulnbankdemo.app
  Unable to find package: com.vulnbankdemo.app
...
```

```
$ adb shell run-as com.vulnbankdemo.app id
run-as: unknown package: com.vulnbankdemo.app
```

Three independent, mechanically different lookups (`pm list packages` string match, `dumpsys package` by name, `run-as` by name) all agree: **`com.vulnbankdemo.app` has never been installed on this device.** This matches `target-profile.md`'s prediction, but it is now a freshly, independently dynamically confirmed fact as of today rather than an inherited assumption.

## 3. Attempting each finding's own `reproduction_dynamic` command, verbatim

Rather than declare the whole phase moot on the strength of section 2 alone, I ran every concrete command the existing findings themselves proposed, so the evidence trail shows an actual attempt and actual tool output per finding, not a blanket skip.

### AND-MAN-003 (TransferMoneyActivity exported deep link)

```
$ adb shell am start -a android.intent.action.VIEW -d "vulnbank://transfer" com.vulnbankdemo.app
Starting: Intent { act=android.intent.action.VIEW dat=vulnbank://transfer pkg=com.vulnbankdemo.app }
Error: Activity not started, unable to resolve Intent { act=android.intent.action.VIEW dat=vulnbank://transfer flg=0x10000000 pkg=com.vulnbankdemo.app }

$ adb shell am start -a android.intent.action.VIEW -d "vulnbank://transfer"
Starting: Intent { act=android.intent.action.VIEW dat=vulnbank://transfer }
Error: Activity not started, unable to resolve Intent { act=android.intent.action.VIEW dat=vulnbank://transfer flg=0x10000000 }
```

The second, package-less form rules out a specific alternative explanation: it's not merely that `com.vulnbankdemo.app` itself is missing — *no app at all* on this device currently claims the `vulnbank://` scheme. Confirms the gating fact; does **not** confirm or refute whether `TransferMoneyActivity` would execute a transfer without re-authentication if it existed — that remains the open question `exploitation/AND-MAN-003.md` already identified, and nothing here resolves it either way.

### AND-MAN-004 (AccountProvider exported, unguarded)

```
$ adb shell content query --uri content://com.vulnbankdemo.app.provider
Error while accessing provider:com.vulnbankdemo.app.provider
java.lang.IllegalStateException: Could not find provider: com.vulnbankdemo.app.provider
    at com.android.commands.content.Content$Command.execute(Content.java:518)
    at com.android.commands.content.Content.main(Content.java:727)
    ...
```

Same conclusion for the provider: nothing to query, insert, update, or delete against. The SQL-injection-via-selection/sortOrder question this finding raises remains completely unverified — it requires implementation code that does not exist.

### AND-MAN-001 (debuggable=true)

```
$ adb shell run-as com.vulnbankdemo.app id
run-as: unknown package: com.vulnbankdemo.app
```

Already shown above; recorded again here against its specific finding. The real-world confirmation this finding calls for (successful `run-as` shell / JDWP attach against a genuinely shipped debuggable build) remains untested.

### AND-MAN-002 (allowBackup=true)

```
$ adb backup -noapk com.vulnbankdemo.app
WARNING: adb backup is deprecated and may be removed in a future release
Now unlock your device and confirm the backup operation...
[command blocked waiting for on-device interaction; killed after 15s via shell `timeout`]
```

Unlike the other three commands, `adb backup` does **not** fail fast on an unknown package — it drives the device into its interactive `BackupRestoreConfirmation` activity and waits indefinitely for a physical unlock + on-screen tap (and potentially a backup-password prompt) before it would even get far enough to discover the package doesn't exist. I did not attempt to blindly script through that confirmation dialog, for two concrete reasons:

1. `com.vulnbankdemo.app` is already confirmed absent (section 2), so a completed backup would contain nothing relevant to this finding regardless of outcome.
2. This is evidently a shared physical device with other apps installed (`com.app.damnvulnerablebank`, and `com.catalent.onehub.dev.debug` — the app that was in the foreground before my session touched anything, observed only incidentally during cleanup below). Blind-tapping through an on-device dialog I cannot see, on a device I don't have exclusive custody of, risks interacting with the wrong element or the wrong app's data. That is outside both this engagement's scope (`scope.md` authorizes only `com.vulnbankdemo.app`) and reasonable operator caution.

**Cleanup performed** (recorded because it changed device state and I want that fully auditable, not hidden):

```
$ adb shell input keyevent KEYCODE_BACK
$ adb shell dumpsys activity activities | grep mResumedActivity
    mResumedActivity: ActivityRecord{... com.catalent.onehub.dev.debug/com.catalent.onehub.MainActivity ...}
```

One `KEYCODE_BACK` dismissed the stuck backup-confirmation dialog (`com.android.backupconfirm`) and the device returned to the foreground app it was already on before my `adb backup` attempt. Verified via `dumpsys activity activities` that the device is back to a normal resumed state. No backup file was produced; no app data was read, written, or exfiltrated; `com.catalent.onehub.dev.debug` was not interacted with beyond observing it was the pre-existing foreground app on a shared device — it is unrelated to this engagement and was not otherwise touched.

### AND-MAN-005 / AND-MAN-006 (no `<uses-sdk>`; exported-component inventory)

```
$ adb shell dumpsys package com.vulnbankdemo.app
...
  Unable to find package: com.vulnbankdemo.app
  Unable to find package: com.vulnbankdemo.app
```

No SDK metadata and no live component inventory obtainable — confirms absence of an installed build rather than revealing the real `min`/`targetSdkVersion` or a live-confirmed version of the exported-component surface map. The manifest-derived inventory remains the only surface map available for this engagement; `drozer`'s `app.package.attacksurface` was not attempted for the identical reason (it, too, requires an installed package).

## 4. What was explicitly not attempted, and why (method §§3–9 mapped to this engagement)

- **Hooking crypto/storage/network/WebView/IPC sinks with Frida, pushing taint markers through the app** (method §3): no process to attach to — the app has never run on any device, because it has never been built. Nothing to hook.
- **Fuzzing exported components/deep links with varied extras/URIs beyond the two forms already tried** (method §4): attempted the two most relevant forms (package-qualified, bare implicit); further variation would only change the URI string, not the outcome, since no component on the device resolves the scheme at all. Not pursued further as low-value repetition.
- **MITM with/without certificate-pinning bypass** (method §5): no network traffic exists to capture — the app has never run.
- **Root/jailbreak-detection, debugger-attach, Frida-detection bypass attempts (ADV-60..63)** (method §6): no app process exists in which to test resilience checks. `resilience-anti-tamper-auditor` was itself explicitly skipped for this engagement per `target-profile.md`'s "Recommended Phase 2 specialist subset" (no code/running app to examine), so there is no corresponding static hypothesis to confirm or refute here either. No new ADV-60..72-style finding is being fabricated from nothing: this is a documented absence of opportunity, not a tested-and-passed control.
- **Biometric-gate bypass (BiometricPrompt/CryptoObject)** (method §7): no `android-ipc-ui-auditor` finding exists flagging this (that specialist was scoped to manifest-level IPC exposure only, per `target-profile.md`), and there is no running app with a biometric gate to attack.
- **Background/foreground storage diffing, logout, uninstall/reinstall residue checks** (method §8): no app to install, background, or uninstall.
- **Re-testing a prior fix** (method §9): not applicable — this is an initial audit, not a retest engagement.

## 5. Findings updated

Appended one `type: "dynamic"` evidence entry and prefixed `reproduction_dynamic` with the actual attempted-command-and-result (keeping the specialist's original suggested text below it, not deleting it) for:

- `AND-MAN-001` — debuggable=true (run-as attempt; confirms package absent, debuggable behavior itself unconfirmed)
- `AND-MAN-002` — allowBackup=true (adb backup attempt; blocked on interactive confirmation, cleanly aborted; package confirmed absent regardless)
- `AND-MAN-003` — TransferMoneyActivity exported deep link (am start attempt, two forms; confirms no app claims the scheme; the central "does it execute a transfer without re-auth" question is unchanged/still open)
- `AND-MAN-004` — AccountProvider exported (content query attempt; confirms provider absent; injection/data-exposure question unchanged/still open)
- `AND-MAN-005` — no `<uses-sdk>` (dumpsys package attempt; confirms no SDK metadata obtainable)
- `AND-MAN-006` — exported-component inventory (dumpsys package + pm list attempt; confirms no live inventory obtainable; also surfaced the unrelated `com.app.damnvulnerablebank` package, explicitly flagged as out of scope and not investigated)

Per `finding-schema.md`'s explicit rule ("`severity_initial`/`confidence_initial`... set once by the specialist that created the finding and never overwritten"), I did **not** modify `confidence_initial`, `severity_initial`, `status`, or `final_severity`/`final_rating_reason` on any record — those remain exactly as the specialists/orchestrator set them. `status` for `AND-MAN-003` remains `potential`/`high` as the orchestrator decided; nothing observed in this session provides grounds to revisit that call, since the specific gap the skeptics identified (does the activity execute a transfer without re-auth) requires implementation code or an installed build, neither of which exists or was created by this session.

No new `finding_id` was created. Nothing discovered in this session constitutes a new, independently-evidenced vulnerability outside the existing six manifest-level findings — the only thing this session newly establishes is that the "no installable build exists" premise, which every existing finding's `reproduction_dynamic` field already stated prospectively, is now also true as independently, dynamically re-verified today against a real connected device, rather than merely carried forward from recon.

## 6. Bottom line for the exploitation panel / report synthesizer

This is a manifest-only fixture with no corresponding APK/DEX/source. A real, authorized, connected Android device (`84f7a024`, Android 11) was available and was genuinely used — not merely assumed present — but there is nothing installable to put on it for this specific target. Every dynamic command any existing finding's `reproduction_dynamic` field proposed was actually run today and failed in the single, consistent, expected way (package/provider/activity not found), which is itself the correct and honest dynamic result for this engagement: it is a confirmed negative about *install state*, not a confirmed or refuted statement about the underlying manifest-level exposure claims (export + no permission guard), which remain exactly as strong or weak as the static evidence already made them. Nothing in this session should be read as either strengthening or weakening `AND-MAN-003`'s `potential`/`high` rating or any other finding's severity — it only adds an honest, dated dynamic-attempt record to each.

If this engagement is ever supplied a real built APK/AAB matching this manifest, every command in section 3 above should be re-run against it verbatim — they are already the correct commands, now proven to execute cleanly against this tooling/device combination, and would very likely produce materially different (confirming or refuting) output against an actual installed build.
