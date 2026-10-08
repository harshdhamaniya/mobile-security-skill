# Worked example: one finding, start to finish

This is a condensed, illustrative walkthrough of a single finding moving through the whole pipeline, so the mechanics in `SKILL.md` and `references/severity-rating.md` are concrete rather than abstract. The app, package name, and specifics below are invented for illustration; this is not a real audit.

## Setup

> **User:** Audit `com.example.walletapp` (APK attached) for security issues. I own this app, it's my team's production wallet app, full MASVS sweep please.

The skill's **Step 0** records this in `engagement-walletapp-2026-10-08/scope.md`: authorization = own-app, in scope = full sweep, out of scope = nothing stated beyond "static APK only, no production backend testing" (no device mentioned, so Phase 3 will likely be skipped unless the user connects one).

## Phase 1: Recon

`mobile-recon-triage` unpacks the APK, identifies native Kotlin, `targetSdkVersion 33`, finds three WebViews, a custom URL scheme `walletapp://`, and 40+ exported-false activities plus one exported Activity with a BROWSABLE intent-filter for `walletapp://pay`. No device connected -> Phase 3 noted as will-be-skipped unless one appears.

## Phase 2: Static specialists (parallel, relevant subset)

`android-manifest-auditor` flags the exported `PayActivity` (MAN-04/MAN-09) as a surface note, not yet a finding on its own. `taint-flow-analyst`, tracing from that same component, finds that `PayActivity.onCreate` reads an `amount` and `recipient` query parameter straight off the `walletapp://pay?...` deep link and calls `PaymentProcessor.send(amount, recipient)` with **no confirmation screen** before funds move. It writes:

```json
{
  "finding_id": "AND-TNT-004",
  "title": "walletapp://pay deep link triggers fund transfer with no user confirmation",
  "agent": "taint-flow-analyst",
  "platform": "android",
  "ref_ids": ["TNT-02", "DLK-03"],
  "masvs": ["MASVS-PLATFORM-1"],
  "attacker_model": "remote",
  "source": "walletapp://pay?amount=&recipient= query params (BROWSABLE intent-filter)",
  "sink": "PaymentProcessor.send(amount, recipient) called directly from onCreate",
  "sanitizer_present": false,
  "sanitizer_notes": "No confirmation dialog, no re-auth, no amount/recipient re-display before send() is called",
  "evidence": [{"type": "static", "detail": "PayActivity.kt:34-41, jadx decompile", "artifact": "evidence/AND-TNT-004/jadx_payactivity.txt"}],
  "reproduction_static": "1. Decompile APK. 2. PayActivity.onCreate (line 34) reads intent.data query params. 3. Line 41 calls PaymentProcessor.send() directly, no dialog/activity in between.",
  "severity_initial": "critical",
  "confidence_initial": "static-only",
  "status": "candidate"
}
```

`deeplink-clipboard-auditor`, auditing the same scheme independently, confirms the scheme has no collision protection either (DLK-01) but (correctly, per `SKILL.md`'s dedup guidance) doesn't re-file the confirmation-bypass issue since `taint-flow-analyst` already caught it; it cross-references `AND-TNT-004` in its own notes instead of duplicating.

## Phase 4: Exploitation panel

**Prosecutor**, given `AND-TNT-004`: states the attacker model precisely as `remote`: *"any webpage, SMS, or QR code can carry a `walletapp://pay?...` link; Android resolves BROWSABLE intent-filters from any source without user gesture beyond a tap."* Walks the path source→sink citing the exact decompiled lines. Addresses the "sanitizer": *"none exists. There is no confirmation step at all between parsing the link and calling send(), confirmed by reading lines 34-41 directly, not inferred."* Notes a gap honestly: *"static-only: have not dynamically fired the link against a running instance to observe the actual transfer API call hit the network."* Proposes severity: critical (direct unauthorized fund movement, remote attacker model, no precondition beyond the victim tapping a link).

**Skeptic 1**: re-reads `PayActivity.kt` in full (not just the cited lines) looking for a guard the prosecutor might have missed earlier in `onCreate` or in `PaymentProcessor.send()` itself. Finds none. Checks whether `targetSdkVersion 33` changes exported-intent-filter defaults in a way that would block this. It doesn't; BROWSABLE intent-filters are a different mechanism from the exported-default change. Checks whether `PaymentProcessor.send()` itself does server-side confirmation before actually moving funds (would downgrade this from "funds move" to "a request is queued that still needs approval"). Reads `PaymentProcessor.kt` and finds `send()` calls the `/api/v1/transfer/execute` endpoint directly, not a `/propose` endpoint. **Verdict: stands.** "Checked for an in-code guard, a version-based mitigation, and a server-side confirmation step; none exist; the path is exactly as the prosecutor describes."

**Skeptic 2**: focuses on attacker-model precision instead: is this really zero-click/remote, or does it need the app already in the foreground, or the user to have an active session? Reads the authentication flow and finds `PaymentProcessor` reuses a long-lived session token from `EncryptedSharedPreferences` with no re-auth requirement for `/transfer/execute`. **Verdict: stands.** "Attempted to downgrade the attacker model to require an active logged-in session as a meaningful precondition; it does, but that precondition is normal product usage (the wallet is only useful logged in), not a mitigating control, and it doesn't change the remote classification in any way that matters."

The prosecutor rated this `critical`, so per `severity-rating.md` a third independent pass runs rather than stopping at two:

**Skeptic 3**: checks whether Android itself interposes anything on a BROWSABLE intent-filter match that would count as a systemic mitigation. A disambiguation dialog only appears when multiple apps can handle the link, and `walletapp://` is this app's own unique scheme, so no such dialog fires here. Also checks whether `PaymentProcessor.send()` has any amount cap or velocity limit that would at least bound the damage. Finds none in `PaymentProcessor.kt`. **Verdict: stands.** "Checked for an OS-level disambiguation gate and for a damage-bounding control (amount cap/velocity limit) as two mitigations the other two skeptics hadn't tried; neither exists."

**Decision**: 3/3 skeptics `stands`; none of the three found a defeating fact, so there is no well-evidenced `refuted` to weigh against them. Static-only evidence is unambiguous (no conditional anywhere that could close the gap) per the Confirmed bar in `severity-rating.md`. Per the evidence-primacy rule, that settles it without needing to count votes: **Confirmed, Critical.** The orchestrating session now writes `status: "confirmed"` and `final_severity: "critical"` into `AND-TNT-004`'s record in `findings.jsonl`. Note that `severity_initial` (also `"critical"` here, set earlier by `taint-flow-analyst`) is left untouched; `final_severity` is the separate field the panel's own verdict owns.

## Phase 5: Report

`report-synthesizer` places this as finding #1 in the summary table (Confirmed, Critical, remote), writes the detailed section quoting both skeptics' specific checks under "Exploitation panel verdict," and remediation pointing at DLK-03's fix column: *"require confirmation + nonce before a deep link triggers a sensitive action."*

---

Contrast this with a finding that reaches **Potential** instead: if `PaymentProcessor.send()` had called a `/transfer/propose` endpoint that still required a separate server-side approval step the client couldn't see into, Skeptic 1's check would have found that and returned `refuted` with the specific defeating fact, and the finding would land as a lower-severity, possibly-informational note about deep-link hygiene rather than a critical confirmed fund-theft path. That's the mechanism doing its job either way: the label follows the evidence, not the other way around.
