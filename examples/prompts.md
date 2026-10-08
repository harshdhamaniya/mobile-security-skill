# Prompt Examples

Copy-paste starting points, organized by what you actually want to happen. Each one is a real prompt you could type to Claude Code once the skill is installed (see the [README](../README.md#installation)). It's plain language, not command syntax. The skill's own description is written to trigger on this kind of request automatically; you don't need to name any agent by hand.

Every example below still has to clear **Step 0** first: the skill will ask you to confirm the target is your own app, a documented engagement, or a CTF/training target before it touches anything. The prompts that already state this explicitly will sail through that gate; the ones that don't will just get asked once.

**Jump to:** [Full audits](#full-audits) · [Quick targeted checks](#quick-targeted-checks) · [Confirm one finding](#confirm-one-already-found-finding) · [Framework-specific](#framework-specific-apps) · [Dynamic / device-connected](#dynamic-device-connected) · [CTF & training](#ctf--training) · [Re-test after a fix](#re-test-after-a-fix) · [Tips for better results](#tips-for-better-results)

## Full audits

Runs the whole five-phase pipeline: recon, the relevant subset of the 13 static specialists, dynamic confirmation if a device is connected, the adversarial panel on everything that matters, and a final report.

> Do a full mobile security audit of this APK: `/builds/app-release.apk`. I own this app, it's the production build of my own product. Give me a complete MASVS-style report.

> Pentest this iOS app source tree at `~/code/ourapp-ios`. This is an authorized internal engagement, scope doc is in `./SECURITY-SCOPE.md`. I want every category covered, not just the obvious stuff.

> Here's our React Native app's bundle and source. I'm the security lead, this is our own app. Run the full assessment. I especially care about anything in the Hermes bundle or AsyncStorage.

> Full OWASP MASTG assessment of this Flutter app (`app-release.aab`). My own app, pre-launch review before we submit to the Play Store.

## Quick targeted checks

Skips recon's full attack-surface inventory and the report step, and runs just the one relevant specialist plus a focused panel pass on anything it finds. Good for "I have one worry, settle it" questions.

> Does this Android app actually pin its certificates, or is the pinning trivially bypassable? It's my own app. Here's the APK.

> Check this app's local storage and Keystore usage for hardcoded keys or plaintext secrets. I own this codebase.

> Is there an SSRF/host-check bypass in how this WebView validates URLs before loading them? My own app, source attached.

> Review our OAuth login flow for PKCE and redirect-URI issues. We use a custom URL scheme for the callback. Authorized engagement, scope covers auth only.

> Can a deep link in this app trigger a payment or account change with no confirmation screen? My own app.

> Check whether this app's root/jailbreak detection is just a client-side boolean, or if it's actually paired with server-side verification.

> Grep this decompiled APK for hardcoded API keys, Firebase URLs, or AWS credentials. My own app, just want to know what's actually exposed if someone decompiles it.

> Is there a race condition in our promo-code redemption flow that would let someone redeem it more than once? Own app, this is the specific flow I'm worried about: `PromoRedeemActivity.kt`.

## Confirm one already-found finding

Skips straight to the adversarial panel (Phase 4) on a single finding you already have. Useful when a scanner, a colleague, or a bug bounty report already flagged something and you want a real exploitability verdict, not another tool re-finding the same pattern match.

> MobSF flagged this as "Insecure WebView implementation: addJavascriptInterface exposed." Here's the method and the class it's in. Is this actually exploitable, or is there a sanitizer I'm not seeing? My own app.

> A bug bounty report claims our deep link handler lets an attacker bypass login. Here's the report and the relevant activity's code. Confirm or refute this before we pay out. Authorized by our own program.

> Our static analyzer says this hardcoded AES key is a critical finding. Build the strongest case that it's actually exploitable, then try to break that case. Own app.

> I think this `PendingIntent` is mutable and exploitable for intent redirection. Here's the code. Prove it one way or the other.

## Framework-specific apps

The skill covers native Android/iOS and five cross-platform frameworks explicitly. Say which one it is if recon can't tell from the artifact alone, so the right specialist and the right section of `supply-chain-framework-auditor` engage.

> This is a Flutter app (`libapp.so` + Dart snapshot, no readable Java/Kotlin business logic). Audit it, I especially want to know if there's anything recoverable from the Dart snapshot and whether pinning survives Flutter's own BoringSSL stack.

> React Native app, Hermes bytecode. Debug menu might still be reachable in the release build, check that specifically, plus whatever else stands out.

> Unity game with IL2CPP. I'm mainly worried about client-trusted in-app-purchase receipts being forgeable. Check that, and anything else that looks exploitable in `global-metadata.dat`.

> Xamarin/.NET MAUI app, decompile the assemblies and check for embedded secrets or logic that should be server-side.

> Cordova/Ionic hybrid app, check `config.xml` for wildcard `allow-navigation`/`access origin` and whether the JS bridge is scoped correctly.

## Dynamic / device-connected

Mention the device explicitly: recon's attack-surface summary decides whether Phase 3 runs, so say so if you have one, and say what's already confirmed vs. still static-only if you're mid-engagement.

> I have a rooted Pixel 8 connected over `adb` with Frida already running. Confirm the findings from the static pass with live hooks, especially the crypto and biometric-bypass ones.

> Device's connected. Specifically: hook `BiometricPrompt` and tell me if the success callback can be force-triggered without a real CryptoObject-bound authentication.

> MITM this app's traffic with and without a pinning bypass script. I want to know if pinning is real or decorative. Proxy's already set up on this machine.

> We did a static-only pass last week (attached findings). I've got a jailbroken device now, run Phase 3 on top of that and update anything that changes.

## CTF & training

Named practice targets clear Step 0 on their own.

> Audit DVIA-v2 (Damn Vulnerable iOS App), this is the public training target, full sweep, I want to see how many of the known-intentional bugs the pipeline actually confirms vs. just flags as potential.

> Run this against InsecureBankv2. CTF/training target, go for a full MASVS pass.

> This is one of the OWASP MSTG crackme challenges. Find and confirm the intended vulnerability.

## Re-test after a fix

> We patched the WebView origin check you flagged as `AND-WEB-002` last time. Here's the engagement folder from the previous run. Re-check just that finding and tell me if it's actually closed now.

> Re-run the full audit on the new build. Diff it against last month's report if you still have it. I want to know what's newly confirmed and what got fixed.

## Tips for better results

- **State your authorization up front** ("my own app," "authorized engagement, scope attached," "CTF target"); it clears Step 0 in one pass instead of a back-and-forth.
- **Say what you already know.** If you have a device connected, a prior report, a specific file/class you're worried about, or a scanner's raw finding, hand it over. The pipeline uses whatever's already true rather than re-deriving it from scratch.
- **Name the framework if recon can't guess it from the artifact** (minified/obfuscated builds sometimes hide the tell). Saves a round-trip and gets you the right framework-specific checks in `supply-chain-framework-auditor`.
- **Ask narrow when you mean narrow.** "Does this app pin certs?" runs in minutes. "Full audit" is thorough but takes longer. Both are legitimate asks; the skill scales to which one you actually made (see `SKILL.md`'s "Scaling to the ask").
- **A plain scanner finding is a fine starting point.** You don't need to pre-verify anything yourself before asking. Handing over a raw MobSF/Semgrep/static-analyzer hit and asking "is this real?" is exactly what the prosecutor/skeptic panel exists for.
