# Steps to Reproduce

This is not a narrated example. Every command, agent, and output on this page was actually run, once, against a small fixture built specifically for this walkthrough. The full output of every step is committed in this repo at [`examples/fixtures/vulnbank-demo/`](fixtures/vulnbank-demo/) so you can open the real files yourself rather than take this page's word for it: `target-profile.md`, `findings/findings.jsonl`, and `exploitation/AND-MAN-003.md` are the actual artifacts the pipeline produced.

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

**What was observed:** this is the only artifact that exists for this target. No DEX, no source, no APK. That absence turns out to matter a lot by Step 6.

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

## Where to verify this yourself

Every file referenced above is committed in this repo, not reconstructed for this page:
- [`fixtures/vulnbank-demo/AndroidManifest.xml`](fixtures/vulnbank-demo/AndroidManifest.xml): the input
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/scope.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/scope.md): the authorization record
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/target-profile.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/target-profile.md): the full recon output
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/findings.jsonl`](fixtures/vulnbank-demo/engagement-vulnbank-demo/findings/findings.jsonl): all 6 findings, full schema, final ratings included
- [`fixtures/vulnbank-demo/engagement-vulnbank-demo/exploitation/AND-MAN-003.md`](fixtures/vulnbank-demo/engagement-vulnbank-demo/exploitation/AND-MAN-003.md): the prosecutor's case and both skeptics' verdicts, plus the orchestrator's decision appended at the end

Two of those files, `target-profile.md` and `exploitation/AND-MAN-003.md`, are left exactly as the agents wrote them, including their own punctuation style (they predate, and don't match, the em-dash-free house style the rest of this repo's hand-edited prose follows). That's deliberate: they're being shown as real evidence, not as documentation, so they weren't touched up after the fact.

What this walkthrough does **not** claim: findings AND-MAN-001, 002, 004, 005, and 006 were produced by the real manifest auditor shown in Step 5, but were not run through the adversarial panel for this walkthrough. They remain at `status: "candidate"` in the committed `findings.jsonl`, honestly. Running every finding through a full panel for a demo fixture wasn't necessary to show the mechanism; AND-MAN-003 alone demonstrates it end to end.
