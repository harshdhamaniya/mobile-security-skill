# Final Report Template

`report-synthesizer` writes two renderings of the same report: `engagement-<slug>/report/final-report.md` using this exact skeleton, and `engagement-<slug>/report/final-report.html` using the companion file `report-template.html` in this same directory. The HTML version is the primary deliverable, a well-formed, self-contained, styled document meant to actually be opened and read, not a markdown export. See `report-template.html` for its own structure and fill-in rules; the content requirements below apply equally to both.

Fill every section; write "None identified" rather than deleting a heading.

```markdown
# Mobile Application Security Assessment: <App Name> <version/build>

**Platform(s):** Android / iOS / cross-platform (<framework>)
**Engagement scope & authorization:** <one line, pointer to scope.md>
**Assessment window:** <dates>
**Methodology:** OWASP MASTG-aligned static + dynamic analysis with adversarial exploitation confirmation (see Methodology section)

## Executive summary
2-4 sentences, plain language, for a non-technical stakeholder: overall posture, the single worst Confirmed finding if any, and the headline recommendation.

## Findings summary

| # | Finding | Severity | Status | Attacker model | Ref IDs |
|---|---|---|---|---|---|
| 1 | <title> | Critical/High/Medium/Low | Confirmed/Potential/Informational | remote/other-app/physical/local-root/none | WEB-02, TNT-06 |

Sort Confirmed-Critical first, then by severity within status tier (Confirmed > Potential > Informational). Omit False Positives from this table; list them in an appendix instead so reviewers can see what was checked and cleared.

## Confirmed findings (detailed)
One subsection per Confirmed finding:
### <finding_id>: <title>
- **Severity:** ... **Attacker model:** ... **MASVS:** ...
- **Description:** what the issue is, in the target app's actual code/component names.
- **Attack path:** source -> propagation -> sink, referencing the taint methodology.
- **Evidence:** link each `evidence/` artifact; summarize dynamic PoC output if present.
- **Exploitation panel verdict:** 2-3 sentence summary of why the skeptic's refutation attempts failed; point at `exploitation/<finding_id>.md` for the full transcript.
- **Remediation:** specific, actionable fix referencing the applicable reference-doc fix column.

## Potential findings (detailed)
Same structure as Confirmed, but close with **why it's not Confirmed** (what evidence would upgrade it, e.g. "needs a jailbroken test device" or "sanitizer adequacy unresolved, recommend code-owner review of X").

## Informational / hardening observations
Lighter-weight list format is fine here (title, ref ID, one-line rationale, suggested fix). This is where resilience/anti-tamper gaps land for apps without high-value assets, and pure best-practice deviations.

## Appendix A: False positives / ruled out
Table: finding, what was suspected, why the skeptic's refutation held. Demonstrates coverage without inflating the findings count.

## Appendix B: Attack surface inventory
From `target-profile.md` (exported components, URL schemes/App Links, WebViews, network endpoints, permissions/entitlements, third-party SDKs) so a reader can see what was in scope and judge completeness.

## Appendix C: Methodology & limitations
- Which specialist agents ran, which reference-doc sections they covered (link back to this repo's `agents/` roster).
- Static-only vs. device-connected dynamic coverage: call out explicitly if no device/emulator was available, since several checklist items (STO, CRY runtime, ADV-60..72) require it.
- Tools actually used (decompilers, Frida scripts, proxy) and tool versions if known.
- Anything explicitly out of scope per `scope.md`.
```

## Dedup rule before writing the summary table
Multiple specialists can legitimately flag the same root cause from different angles (e.g. `taint-flow-analyst` flags a deep-link-to-WebView path that `webview-security-auditor` also flagged as a bridge-exposure issue). Merge into one finding entry, union the `ref_ids`, keep the higher of the two `severity_initial` values as a starting point for the panel, and note the merge in that finding's `notes`. Never report the same underlying bug twice under different IDs. A reader counting findings should get a true count of distinct issues.
