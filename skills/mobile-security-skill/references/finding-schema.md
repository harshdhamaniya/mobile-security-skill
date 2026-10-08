# Finding Schema

Every specialist agent emits findings in this exact shape so the exploitation panel and report synthesizer can process them mechanically. Write findings as a JSON array (or JSON objects appended to a shared `findings.jsonl`) under the engagement's working directory (see `SKILL.md` for where that is). One object per candidate issue; do not pre-merge duplicates, the synthesizer dedupes later.

```json
{
  "finding_id": "AND-WEB-002",
  "title": "addJavascriptInterface bridge reachable from any loaded origin",
  "agent": "webview-security-auditor",
  "platform": "android",
  "ref_ids": ["WEB-02", "TNT-06"],
  "masvs": ["MASVS-PLATFORM-2"],
  "category": "webview",
  "component": "com.example.app.WebActivity",
  "attacker_model": "remote",
  "source": "URL loaded into WebView without origin pinning",
  "sink": "@JavascriptInterface method `getAuthToken()` exposed to JS",
  "sanitizer_present": false,
  "sanitizer_notes": "No origin allow-list on addJavascriptInterface; shouldOverrideUrlLoading does substring match only",
  "evidence": [
    {"type": "static", "detail": "jadx decompile shows WebActivity.java:142 addJavascriptInterface(bridge, \"Android\")", "artifact": "evidence/AND-WEB-002/jadx_snippet.txt"}
  ],
  "reproduction_static": "1. Decompile APK with jadx. 2. Inspect WebActivity.onCreate for addJavascriptInterface. 3. Inspect shouldOverrideUrlLoading for host validation.",
  "reproduction_dynamic": null,
  "severity_initial": "high",
  "confidence_initial": "static-only",
  "status": "candidate",
  "final_severity": null,
  "final_rating_reason": null,
  "notes": ""
}
```

## Field notes
- `finding_id`: `<PLATFORM>-<DOMAIN>-<NNN>` (e.g. `AND-STO-014`, `IOS-CRY-003`, `BOTH-OAU-007`). Keep stable once assigned: downstream agents reference it.
- `ref_ids`: cite every matching ID from `taint-analysis.md`, `advanced-attacks.md`, `clipboard-webview-oauth.md`, `manifest-plist.md`, `storage-crypto.md`. Never invent a finding with no ref ID. If nothing in the reference set matches, say so explicitly and tag `ref_ids: ["CUSTOM"]` with justification in `notes`.
- `masvs`: best-effort mapping to OWASP MASVS v2 control IDs (MASVS-STORAGE, MASVS-CRYPTO, MASVS-AUTH, MASVS-NETWORK, MASVS-PLATFORM, MASVS-CODE, MASVS-RESILIENCE, MASVS-PRIVACY). Leave `[]` if genuinely unmapped, don't force it.
- `attacker_model`: one of `remote` (any server/network attacker), `other-app` (malicious app on same device, no root), `physical` (attacker has the unlocked/locked device briefly), `local-root` (rooted/jailbroken device or physical+root), `none` (not exploitable by any realistic attacker, informational only). This is the single most important field for the exploitation panel: it is what "Confirmed" is actually confirming reachability *by*.
- `sanitizer_present` / `sanitizer_notes`: record what mitigation exists even if you believe it's inadequate; the skeptic agent checks this first.
- `evidence`: append-only list. Static agents add `type: "static"` entries (file:line, decompiled snippet, grep hit). The dynamic-runtime-verifier adds `type: "dynamic"` entries (Frida hook output, logcat/idevicesyslog capture, pcap). The exploitation panel adds `type: "poc"` entries. Never delete evidence; corrections are new entries with `notes` explaining what superseded what.
- `severity_initial` / `confidence_initial`: the specialist's own best guess (`critical|high|medium|low|info`, `static-only|dynamic-confirmed|theoretical`), set once by the specialist that created the finding and **never overwritten**: it's the audit trail of what the original call was, kept even after the panel rules differently.
- `status`: `candidate` (fresh from a specialist) -> `under-review` (the orchestrating session sets this the moment it hands a finding to `exploit-verifier-prosecutor`) -> `confirmed` | `potential` | `informational` | `false-positive` (the panel's final call; see `severity-rating.md` for the gate each requires).
- `final_severity` / `final_rating_reason`: **who writes these and when**: the prosecutor and skeptic agents never edit `findings.jsonl` directly (they're scoped to `Read`/`Grep`/`Glob`/`Write`-to-`exploitation/` only, by design, so they can't quietly short-circuit each other). Once every skeptic pass for a finding is in, the **orchestrating session** (the one running `SKILL.md`, i.e. you) reads the finished `exploitation/<finding_id>.md` transcript, applies the decision rule in `severity-rating.md` itself, and writes the resulting `status`, `final_severity` (`critical|high|medium|low|info`, independent of `severity_initial`), and a one-line `final_rating_reason` into this record. This is the only field update step in the whole pipeline the orchestrator does by hand rather than delegating. `SKILL.md`'s Phase 4 says so explicitly. `report-synthesizer` reads `final_severity`/`status` as the authoritative rating; it falls back to `severity_initial` only for findings that skipped the full panel (routed straight to `informational` per `SKILL.md`'s low-severity shortcut).

## Where findings live
Use a per-engagement working directory (the skill sets this up, typically `./engagement-<slug>/`) with:
```
engagement-<slug>/
├── scope.md                 # authorization record (see SKILL.md Step 0)
├── target-profile.md        # recon-triage output
├── findings/
│   ├── findings.jsonl       # one finding object per line, append-only
│   └── evidence/<finding_id>/...
├── exploitation/
│   └── <finding_id>.md      # prosecutor case + skeptic rebuttal + verdict
└── report/
    └── final-report.md
```
Agents append to `findings.jsonl` rather than rewriting it, so parallel specialist runs never clobber each other. The report-synthesizer is the only agent that reads the whole file to dedupe and roll up.
