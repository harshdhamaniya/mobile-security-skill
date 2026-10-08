# Steps to Reproduce

This is not a narrated example. Every command, agent, and output on this page was actually run against a small fixture built specifically for this walkthrough, across two passes as the walkthrough was deepened over time. The full output of every step is committed in this repo at [`examples/fixtures/vulnbank-demo/`](fixtures/vulnbank-demo/) so you can open the real files yourself rather than take this page's word for it: `target-profile.md`, `findings/findings.jsonl`, both `exploitation/*.md` transcripts, the dynamic session log, and the final report are all the actual artifacts the pipeline produced, not reconstructions.

Everything below uses a hand-authored, synthetic `AndroidManifest.xml` (no APK, no decompiled app, nothing downloaded from anywhere) so the whole thing is reproducible by anyone without needing a real target app or any special authorization beyond "I wrote this file myself." See [`fixtures/vulnbank-demo/README.md`](fixtures/vulnbank-demo/README.md) for what the fixture is and why it exists.

**The one command you actually need**, if you just want to run this yourself against your own target: see [`prompts.md`](prompts.md). Everything that follows here is what happens underneath that one request, shown step by step for anyone who wants to verify or debug the pipeline directly.

---

## Step 1

**Tool used:** `git`, shell

**Command used:**
```bash
git clone https://github.com/harshdhamaniya/mobile-security-skill.git
cd mobile-security-skill
mkdir -p ~/.claude/skills && cp -r skills/mobile-security-skill ~/.claude/skills/
mkdir -p ~/.claude/agents && cp agents/*.md ~/.claude/agents/
```

**Output:** no output on success; `ls ~/.claude/agents/` lists 18 `.md` files, `ls ~/.claude/skills/mobile-security-skill/` lists `SKILL.md` and `references/`.

**What was observed:** this is the README's own "Option B: Manual copy" install path. A separate Claude Code session (not this walkthrough) ran exactly this and confirmed the skill and all 18 agents loaded and resolved their reference docs correctly, which is what led to the reference-doc path fix documented in this repo's commit history. From this point on, Claude Code recognizes `mobile-security-skill` as an available skill and all 18 specialists as available agents.

---

## Step 2: the fixture

**Tool used:** none, a text editor

**Command used:** authored `examples/fixtures/vulnbank-demo/AndroidManifest.xml` by hand: three components (`LoginActivity`, `TransferMoneyActivity`, `AccountProvider`), with `android:debuggable="true"`, `android:allowBackup="true"`, and `TransferMoneyActivity`/`AccountProvider` both exported with zero permission guard. `TransferMoneyActivity` also carries a `BROWSABLE` intent-filter for a custom scheme, `vulnbank://transfer`.

**Output:** a 41-line manifest file, committed at [`fixtures/vulnbank-demo/AndroidManifest.xml`](fixtures/vulnbank-demo/AndroidManifest.xml).

**What was observed:** this is the only artifact that exists for this target. No DEX, no source, no APK. That absence turns out to matter a lot from Step 6 onward.

---

## Step 3: authorization (Step 0 of the skill)

**Tool used:** none, this is a conversational gate the skill enforces before touching anything

**Command used:** stated plainly that the target is a synthetic fixture authored specifically for this repo's own walkthrough, not a third-party app, and wrote that down.

**Output:** [`fixtures/vulnbank-demo/engagement-vulnbank-demo/scope.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/scope.md):
```
# Scope
- Target: VulnBank-Demo (com.vulnbankdemo.app), a synthetic manifest fixture authored for this repo's reproduction walkthrough
- Authorization: own-app - authored by this repo's maintainer specifically as a test fixture, not a third-party app
- In scope: AndroidManifest.xml review (manifest-level findings only, this fixture has no actual source/DEX)
- Out of scope: everything else, since no real code exists for this fixture
- Date: 2026-10-08
```

**What was observed:** `SKILL.md`'s Step 0 is a hard gate in the instructions, not a suggestion. Nothing in the steps below ran until this record existed. If you run this against your own app, this is the point where the skill asks you directly whether you own it, have an authorized engagement, or are pointing it at a CTF/training target, and will not proceed on an assumption.

---

## Step 4: recon and triage

**Tool used:** Claude Code's Agent tool, `subagent_type: mobile-recon-triage`

**Command used:** spawned the agent with the manifest's path and the engagement directory, matching exactly what `SKILL.md`'s Phase 1 instructs the orchestrator to do.

**Output:** [`fixtures/vulnbank-demo/engagement-vulnbank-demo/target-profile.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/target-profile.md) (full file committed). Key excerpt:
```
Platform/framework: Android only. Framework assumed native Java/Kotlin from
class-naming conventions, but this is unverifiable: no APK/AAB, no DEX,
no source tree, no native libs exist.

Attack surface: 3 declared components, all exported.
- .ui.LoginActivity: exported MAIN/LAUNCHER (expected, not a finding)
- .ui.TransferMoneyActivity: exported, BROWSABLE, custom scheme
  vulnbank://transfer, no permission guard, sensitive-sounding name
- .data.AccountProvider: exported, no permission, no path-permissions

Recommended Phase 2 subset: android-manifest-auditor (primary),
android-ipc-ui-auditor, deeplink-clipboard-auditor (DLK sub-scope only).
Skip: 10 other specialists, each with a documented reason (Android-only
target, no code/DEX to analyze, no backend/dependency artifacts, etc.)
```

**What was observed:** the agent correctly identified it could not determine the actual framework, min/target SDK, or presence of a WebView from a bare manifest, and said so explicitly rather than guessing. It also noticed a real `adb` device was connected on the test machine, and correctly reasoned that this was irrelevant here since no installable APK exists for a manifest-only fixture, rather than assuming dynamic analysis was viable just because a device was present.

---

## Step 5: static specialist (manifest audit)

**Tool used:** Claude Code's Agent tool, `subagent_type: android-manifest-auditor`

**Command used:** spawned the agent with the manifest path, the recon output above, and the absolute path to `manifest-plist.md`, matching `SKILL.md`'s Phase 2 instructions.

**Output:** 6 findings appended to [`fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/findings.jsonl`](fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/findings.jsonl) (full file committed, one JSON object per line). Summary:

| Finding | Check | Severity | Attacker model |
|---|---|---|---|
| AND-MAN-001 | `android:debuggable="true"` (MAN-01) | high | physical |
| AND-MAN-002 | `allowBackup` with no exclusion rules (MAN-02) | medium | physical |
| AND-MAN-003 | `TransferMoneyActivity` exported, BROWSABLE, no permission (MAN-04/09) | high | **remote** |
| AND-MAN-004 | `AccountProvider` exported, zero access control (MAN-04/06) | high | other-app |
| AND-MAN-005 | No `<uses-sdk>` declared (MAN-12) | low | none |
| AND-MAN-006 | Full exported-component inventory, informational | info | other-app |

**What was observed:** the agent correctly walked MAN-01 through MAN-21 and, just as importantly, explicitly recorded *why* it found nothing for the checks that didn't apply (no `<receiver>`/`<service>` exist, no custom permissions are defined so permission-squatting can't apply, no `taskAffinity` is set so there's no StrandHogg signal) instead of silently skipping them. It also correctly declined to flag `LoginActivity`'s export as a finding, since an exported launcher activity is expected, not a weakness, which is exactly the judgment call its instructions call for.

---

## Step 6: the adversarial panel, on the highest-value finding

This is the part of the skill this repo is actually about. `AND-MAN-003` (the remote-reachable deep link) is the best candidate: highest attacker model, highest severity, most interesting attack path.

### 6a. Prosecutor

**Tool used:** Agent tool, `subagent_type: exploit-verifier-prosecutor`

**Command used:** spawned with the finding, the raw manifest, and `severity-rating.md`'s absolute path; told to build its case and write it to `exploitation/AND-MAN-003.md`.

**Output (excerpt from the real, committed file):**
```
Proposed severity: High, not critical... because no implementation code
exists anywhere in this record to confirm that the reached code actually
executes a transfer rather than merely displaying a form still gated
behind authentication, the impact side of impact x attacker-model x
asset-sensitivity cannot be rated at its ceiling today. This also caps
the confirmation status at Potential under severity-rating.md's ladder...
pending either (a) real implementation code for TransferMoneyActivity,
or (b) a dynamic test against a built/installable version of this app.
```

**What was observed:** the prosecutor built the strongest honest case it could (unambiguous reachability: exported, `BROWSABLE`, zero permission infrastructure anywhere in the 41-line file), and then capped its own proposed rating at Potential rather than Confirmed, naming the exact gap (no code exists to confirm what happens once the activity is reached) instead of papering over it. Nobody told it to be conservative here. That's the instructed behavior, demonstrated for real.

### 6b. Two independent skeptics

**Tool used:** Agent tool, `subagent_type: exploitability-skeptic`, run twice in parallel, neither instance shown the other's verdict

**Command used:** each spawned with the prosecutor's case, the raw manifest, and `severity-rating.md`; told to independently attempt all six refutation angles from its own instructions.

**Output (both real verdicts, full text in the committed file):**
- **Skeptic 1: stands.** Independently re-verified every manifest line, checked `finding-schema.md`'s own `attacker_model` definitions, cross-checked `manifest-plist.md`'s MAN-04/MAN-09 rows, and specifically tried the "maybe this is just a legitimate pre-fill-form deep link, still gated behind auth" counter-argument. Found it was already disclosed by the prosecutor as an open question, not a fact it had missed.
- **Skeptic 2: stands.** Same independent manifest re-read, same six angles, same conclusion: no SDK-version default rescues the exported component (it's explicit, not inferred), no sanitizer exists anywhere to find, and the one real open question (does a confirmation screen exist before the transfer executes) is unverifiable either way given zero implementation code, not a fact that defeats the case.

**What was observed:** both skeptics did real, independent, re-derived verification work rather than restating the prosecutor's reasoning back. Both landed on the exact same gap without seeing each other's output, which is itself a useful signal that the gap is real rather than an artifact of one skeptic's framing.

### 6c. Orchestrator decision

**Tool used:** none, this is the one step `SKILL.md` explicitly reserves for the orchestrating session itself, not a subagent

**Command used:** applied `severity-rating.md`'s decision rule by hand to the finished transcript above.

**Output:**
```
Status: potential. Final severity: high.

No skeptic returned a well-evidenced refuted. Reachability clears the
Confirmed bar on its own. But the Confirmed bar for static-only evidence
requires "no conditional the skeptic can plausibly argue closes the gap" -
and both skeptics independently named exactly such a conditional: an
in-app confirmation step that can't be ruled out because no implementation
code exists for this fixture. That keeps this at Potential, not Confirmed.
```
Written into both `exploitation/AND-MAN-003.md` (full reasoning) and `findings/findings.jsonl` (`status`, `final_severity`, `final_rating_reason` fields on the `AND-MAN-003` record; `severity_initial` left untouched, exactly as the schema specifies).

**What was observed:** this is the actual payoff of the whole design. Two skeptics saying "stands" did not auto-confirm the finding. The rule asks a sharper question than majority vote: does the evidence bar in the rating ladder actually hold, for *this specific claim*? Here it held for reachability and not for impact, so the finding landed at Potential, high severity, with the exact missing evidence named in `reproduction_dynamic` for next time: real implementation code, or a dynamic test against an installable build.

---

## Step 7: dynamic confirmation (Phase 3), for real

The first pass through this walkthrough (Steps 1-6) skipped Phase 3 entirely, since no device was connected at the time. A device became available later, so this step actually ran it, against all six findings, not just the one the panel had reviewed.

**Tool used:** Agent tool, `subagent_type: dynamic-runtime-verifier`

**Command used:** spawned with the full `findings.jsonl`, the engagement directory, and `advanced-attacks.md`'s absolute path; told to follow `advanced-attacks.md` §G and attempt every finding's own `reproduction_dynamic` suggestion against whatever device is actually connected.

**Output (excerpt from the real, committed file, [`fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/dynamic-session-log.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/dynamic-session-log.md)):**
```
$ adb devices -l
List of devices attached
84f7a024        device product:sweetin model:M2101K6P device:sweetin transport_id:2

$ adb shell run-as com.vulnbankdemo.app id
run-as: unknown package: com.vulnbankdemo.app

$ adb shell pm list packages | grep -i bank
package:com.app.damnvulnerablebank
```

**What was observed:** a real, authorized Android device (Xiaomi Redmi, Android 11) was genuinely connected, and every finding's own suggested dynamic command was actually run against it, not just the panel-reviewed one. All six came back the same honest way: the target package has never been installed on this device, so `run-as`, the deep-link launch, the content-provider query, and `adb backup` all failed in the expected, consistent way (package/provider/activity not found). That is itself real dynamic evidence (a confirmed negative about install state), not a skipped phase.

Two things are worth calling out specifically. First, `pm list packages` surfaced an unrelated app already on the shared test device, `com.app.damnvulnerablebank`, that has nothing to do with this engagement. The agent named it explicitly and did not touch it beyond that one grep line, which is `scope.md`'s authorization boundary working as intended on a real shared device, not a hypothetical. Second, the `adb backup` attempt for `AND-MAN-002` drove the device into an interactive on-device confirmation dialog that cannot be completed non-interactively. Rather than blind-tap through a dialog it could not see on a device it did not have exclusive custody of, the agent declined, explained why in the log, and cleanly dismissed the dialog with `KEYCODE_BACK`, verifying afterward that the device returned to its prior state with nothing read or written. That restraint, not forcing a result just to have one, is exactly the judgment call this skill's agents are instructed to make.

---

## Step 8: a second finding through the adversarial panel, and a real concurrency bug

Step 6 ran the panel on one finding (`AND-MAN-003`). To show the mechanism holds up on a second, independently-run case rather than being a one-off, `AND-MAN-001` (the `android:debuggable="true"` finding) went through the same prosecutor/skeptic/orchestrator sequence here.

### 8a. Prosecutor and two skeptics

**Tool used:** Agent tool, `exploit-verifier-prosecutor` then two `exploitability-skeptic` instances run in true parallel, same pattern as Step 6.

**Output:** full transcript committed at [`fixtures/vulnbank-demo/engagement-vulnbank-demo/exploitation/AND-MAN-001.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/exploitation/AND-MAN-001.md). The prosecutor built the case for `run-as`/JDWP exposure, named the same kind of honest gap as before (no installable build exists to dynamically exercise the sink, and no `build.gradle` exists anywhere to confirm or rule out a release-build override), and proposed Potential rather than Confirmed itself. Both skeptics independently re-derived the same conclusion: no well-evidenced refutation, the one real open question (the missing `build.gradle`) already disclosed by the prosecutor rather than hidden.

### 8b. What actually happened when the two skeptics wrote in parallel

Both `exploitability-skeptic` instances were, at the time, instructed to append their verdict to the same shared file, `exploitation/AND-MAN-001.md`. That is what the agent definitions and `SKILL.md` said to do before this walkthrough, and it is exactly what a true-parallel fan-out makes risky: both instances read the file, both built their append based on that same read, and one instance's write landed first. The second instance's own append was rejected because the file had changed underneath it since its last read. On re-reading, that instance found the first skeptic's section present but not its own, reconstructed the full file from its own earlier read plus the other skeptic's now-committed section plus its own new section, and explicitly flagged to the orchestrating session that the result should be spot-checked for anything lost in the collision.

It was. The orchestrating session (this one) independently re-read the entire committed file end to end: the prosecutor's case, both skeptics' verdicts, nothing missing, nothing duplicated. The self-reported recovery had worked. But "it happened to recover correctly" is not the same as "this can't happen," so the actual race condition was fixed at the source rather than left as a one-time near-miss: [`agents/exploitability-skeptic.md`](../agents/exploitability-skeptic.md) and [`SKILL.md`](../skills/mobile-security-skill/SKILL.md)'s Phase 4 section were both updated so each parallel skeptic instance writes its own file (`exploitation/<finding_id>.skeptic-<label>.md`) instead of appending to a file every other instance is also writing to, and the orchestrating session consolidates all of them into the single canonical transcript only after every skeptic has finished, which is the one point in this phase that was never running in parallel with anything else. That removes the collision structurally rather than relying on timing. `AND-MAN-001.md`'s own committed content was left exactly as the agents actually produced it, race condition and all, since it's genuine evidence of a real run, not a retroactively tidied-up version of events.

### 8c. Orchestrator decision

**Tool used:** none, same as Step 6c: this is the orchestrating session's own step, not a subagent's.

**Output:**
```
Status: potential. Final severity: high.

Both skeptics returned stands with no well-evidenced refuted. But the
Confirmed bar isn't met either: the real dynamic attempt from Step 7
proved run-as was never exercised against a live process (package not
installed), and a genuinely open conditional remains on the static side
too (whether a release-build override exists cannot be checked either
way, since no build.gradle exists anywhere in this fixture). That
combination, static-only reachability plus a device/build that was
actually sought out and found unavailable, is Potential's definition,
not Confirmed's.
```
Written into both `exploitation/AND-MAN-001.md` (full reasoning appended) and `findings/findings.jsonl` (`status`, `final_severity`, `final_rating_reason` on the `AND-MAN-001` record).

**What was observed:** the same mechanism from Step 6 held on an independent second finding, reaching the same kind of result (Potential/High) for a structurally different reason (a missing build artifact here, versus a missing in-app confirmation step for `AND-MAN-003`), which is itself a useful signal that the panel is actually evaluating each case's own evidence rather than defaulting to one template answer. And the race condition in Step 8b is arguably the most honest thing in this entire walkthrough: a real multi-agent concurrency bug, caught by one of the agents involved, verified by the orchestrator, and fixed in the skill's own source rather than quietly worked around.

---

## Step 9: report synthesis (Phase 5), for real

**Tool used:** Agent tool, `subagent_type: report-synthesizer`

**Command used:** spawned with the complete engagement directory (all six findings, both exploitation transcripts, the dynamic-session log, `target-profile.md`) and the absolute paths to `report-template.md` and `report-template.html`, matching `SKILL.md`'s Phase 5 instructions exactly.

**Output:** two real files, committed at [`fixtures/vulnbank-demo/engagement-vulnbank-demo/report/final-report.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/report/final-report.md) and [`fixtures/vulnbank-demo/engagement-vulnbank-demo/report/final-report.html`](fixtures/vulnbank-demo/engagement-vulnbank-demo/report/final-report.html). The HTML report is a self-contained, dependency-free document: no CDN, no external font, no `<script>` tag, embedded CSS driving color-coded severity and status badges, and collapsible finding cards built with plain `<details>`/`<summary>` so they work with zero JavaScript.

**What was observed:** the agent correctly deduped (found no cross-specialist duplicates, since one specialist authored all six findings here, and explained why `AND-MAN-006`'s shared reference ID with `AND-MAN-003`/`004` isn't a duplicate), rolled up both panel-reviewed findings' real verdicts instead of writing a generic "confirmed by panel" placeholder, and correctly separated the two panel-reviewed Potential findings from the four that were never escalated, labeling the latter `Candidate` rather than silently treating them as equivalent to a panel-reviewed result. It also flagged two real inconsistencies in the record on its own initiative rather than smoothing them over: that two specialists `target-profile.md` recommended (`android-ipc-ui-auditor`, `deeplink-clipboard-auditor`) never actually produced any output in this engagement, and that a skeptic verdict referenced a `findings.jsonl.bak` file that doesn't actually exist in the directory as supplied. Both are stated plainly in the report's own Appendix C rather than hidden.

---

## Where to verify this yourself

Every file referenced above is committed in this repo, not reconstructed for this page:
- [`fixtures/vulnbank-demo/AndroidManifest.xml`](fixtures/vulnbank-demo/AndroidManifest.xml): the input
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/scope.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/scope.md): the authorization record
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/target-profile.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/target-profile.md): the full recon output
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/findings.jsonl`](fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/findings.jsonl): all 6 findings, full schema, final ratings included
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/dynamic-session-log.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/dynamic-session-log.md): the real Phase 3 session against a connected device, including the explicit list of what was and wasn't attempted and why
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/exploitation/AND-MAN-003.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/exploitation/AND-MAN-003.md) and [`exploitation/AND-MAN-001.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/exploitation/AND-MAN-001.md): both panel-reviewed findings' full transcripts, prosecutor's case, both skeptics' verdicts, and the orchestrator's decision appended at the end of each
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/report/final-report.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/report/final-report.md) and [`report/final-report.html`](fixtures/vulnbank-demo/engagement-vulnbank-demo/report/final-report.html): the actual deliverable report-synthesizer produced from everything above

Four of those files, `target-profile.md`, `exploitation/AND-MAN-003.md`, `exploitation/AND-MAN-001.md`, and the dynamic session log, are left exactly as the agents wrote them, including their own punctuation style (they predate, and don't match, the em-dash-free house style the rest of this repo's hand-edited prose follows). That's deliberate: they're being shown as real evidence, not as documentation, so they weren't touched up after the fact. `findings.jsonl` and both report files, by contrast, were brought in line with that house style after the agents produced them, since a deliverable report is documentation meant to be read by someone else, not a raw evidentiary transcript.

What this walkthrough does **not** claim: findings AND-MAN-002, 004, 005, and 006 were produced by the real manifest auditor shown in Step 5, and received real dynamic-confirmation attempts in Step 7, but were not run through the adversarial panel. They remain at `status: "candidate"` in the committed `findings.jsonl`, honestly, and `final-report.md`/`.html` both label them `Candidate` rather than implying they were panel-reviewed. Running every finding through a full panel for a demo fixture wasn't necessary to show the mechanism; two independent findings (AND-MAN-001 and AND-MAN-003), run through the panel separately and landing on the same Potential/High outcome for two structurally different reasons, demonstrate it end to end without needing all six.
