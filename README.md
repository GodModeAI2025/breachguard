# breachguard

**A methodology-driven security-audit skill for Claude.** Inspired by the practice of experienced audit firms: structure first, findings second, false-positive verification before every conclusion.

breachguard is not a scanner. It is a framework that guides Claude through the same phases a senior auditor walks: intake, context, lens application, FP-verification, reporting. Output is structured, calibrated by confidence, and survives peer review.

---

## What this is

- **A Claude skill** (single `.skill` archive) you load in Claude.ai, Claude Desktop, or Claude Code
- **283 lenses** across 27 domains — security is 14 of them; the rest cover performance, architecture, code quality, testing, documentation, API design, database, concurrency, error handling, observability, DevOps, frontend, UX, i18n, compliance, and OSS-readiness
- **12 operating modes** — `audit`, `triage`, `deep-audit`, `variant`, `verify-fix`, `bugfix`, `feature`, `discover`, `deploy`, `custom`, `opensource`, `content`
- **6 role personas** — `Auditor`, `Developer`, `Architect`, `Reporter`, `Agent`, `Detector`
- **5-phase workflow** — Intake → Context → Lens → FP-Verification → Report
- **6 FP-Gates** — every finding passes through Sink, Source, Reachability, Validation, Attacker-Control, Impact before getting a verdict
- **20 bug-class verification rubrics** — SQL injection, XSS, CSRF, auth bypass, secret exposure, crypto misuse, etc. — each with concrete checks
- **Attacker-Model framework** — WHO / ACCESS / INTERFACE triples that force explicit threat-modeling instead of vague "an attacker"

## What this isn't

- Not a replacement for a professional security firm or pentest
- Not a CVE-scanner — it complements SAST/SCA tools, doesn't replace them
- Not for mobile-app-specific audits (use OWASP MAS — there's an owasp-mas skill for that)
- Not for prompt-injection scanning of LLM prompts/documents (use prompt-injection-scanner)
- Not for pure UX/design review (use enbw-impeccable or similar)

---

## Installation

### Claude.ai / Claude Desktop

1. Download `breachguard.skill` from [Releases](#)
2. Settings → Skills → Upload → select the file
3. Enable the skill in your workspace

### Claude Code

```bash
# Copy the skill into your user skills directory
unzip breachguard.skill -d ~/.claude/skills/
```

The skill auto-activates on trigger phrases.

---

## Usage

breachguard triggers on security-audit intent in a wide range of phrasings:

```
"Audit this repo for security issues"
"Find SQL injection risks in src/"
"Check my auth flow for vulnerabilities"
"Review this PR for secret leakage"
"Run a deep-audit on core/sandbox/"
"Variant-hunt: same pattern as CVE-XXXX-NNNNN"
"Verify fix for finding F-003"
```

Other domain-keywords (performance, architecture, compliance, KRITIS, NIS2, WCAG) also trigger — security is the primary but not sole domain.

### Mode selection

Most runs auto-select a mode from context. If you want control:

```
"Run in triage mode — I just need the scope assessment, no deep dive"
"Deep-audit please, full source access, don't stop at first finding"
"Variant mode — look for this pattern elsewhere in the code"
"Detector mode — generate Semgrep rules for the true-positive findings"
```

### Role selection

```
"Act as Auditor" — structured report with CVEs, severities, fixes (default)
"Act as Developer" — fix-oriented, minimal explanation
"Act as Architect" — big-picture coupling and design concerns
"Act as Agent" — YAML/JSON machine-readable output for CI pipelines
"Act as Detector" — output Semgrep/CodeQL rule skeletons
"Act as Reporter" — executive-summary output for non-technical stakeholders
```

---

## Anatomy of a finding

Every true-positive finding breachguard emits looks like this:

```
[SEVERITY / CONFIDENCE / VERDICT] F-NNN — Title
Lens:            supply-chain, insecure-defaults
File:            path/to/file.ts:line
Attacker-Model:  WHO / ACCESS / INTERFACE
Exploitability:  trivial | moderate | conditional | theoretical
Evidence-Level:  DIRECT | INFERENCE | HEURISTIC

Gates:
- Sink:             PASS/FAIL/UNCERTAIN
- Source:           PASS/FAIL/UNCERTAIN
- Reachability:     PASS/FAIL/UNCERTAIN
- Validation:       PASS/FAIL/UNCERTAIN
- Attacker-Control: PASS/FAIL/UNCERTAIN
- Impact:           PASS/FAIL/UNCERTAIN

Bug-Class: <class name from the 20-class taxonomy>

<Problem description — what, why, how exploitable>

Fix: <concrete remediation, not hand-waving>
```

Findings that don't pass FP-verification are marked `UNCERTAIN` with the specific gate(s) that failed, so you know exactly what additional context would resolve them.

---

## Scope limits, honestly stated

A breachguard run is bounded by what Claude can see. For repos larger than ~50 files, expect the output to mark several findings `UNCERTAIN` because complete cross-file data-flow tracing is not feasible in a single session. The skill makes these limits **explicit in the report** rather than pretending to comprehensive coverage.

For real assurance on production-critical systems, use breachguard as a *first pass* and hand the structured output to a human auditor or a professional firm. It surfaces the hot-spots and the attacker-models; it does not certify absence of issues.

---

## Structure

```
breachguard/
├── SKILL.md                              # Entrypoint
├── NOTICE                                # Attribution
├── LICENSE                               # Apache-2.0
└── references/
    ├── methodology.md                    # 5-phase workflow + 6 principles
    ├── fp-verification.md                # 6 gates with TP/FP/UNCERTAIN verdicts
    ├── attacker-models.md                # WHO/ACCESS/INTERFACE framework
    ├── bug-class-verification.md         # 20 bug-classes with rubrics
    ├── variant-analysis.md               # 5-step variant hunt
    ├── modes.md                          # 12 modes with examples
    ├── roles.md                          # 6 roles with output formats
    ├── agent-output.md                   # Machine-readable schemas
    ├── workflow.md                       # Phase integration + few-shots
    ├── domains.json                      # 27 domains × 283 lenses
    ├── compliance-mappings.md            # GDPR, NIS2, KRITIS, DORA, WCAG
    ├── code-examples.md                  # Canonical patterns
    ├── updater-sync.md                   # Maintenance
    └── lenses/
        ├── insecure-defaults.md          # Fallback-secrets, fail-open, footguns
        ├── sharp-edges.md                # Dangerous APIs, pit-of-success test
        ├── supply-chain.md               # CDN, transitive deps, pinning
        └── <23 more lens files>
```

---

## Attribution

breachguard's false-positive verification model — the six-gate FP-check (Sink / Source / Reachability / Validation / Attacker-Control / Impact), the WHO / ACCESS / INTERFACE attacker-triad, and the DIRECT / INFERENCE / HEURISTIC evidence classification — draws from [Trail of Bits' published audit practice](https://appsec.guide). Concepts are reimplemented in fresh prose; no text or code is copied.

## License

Apache-2.0. See `LICENSE`.

## Contributing

Issues and pull requests welcome. New lenses should include:

- A clear lens definition (what it catches, what it doesn't)
- At least 3 concrete code examples (vulnerable → detected → fixed)
- Mapping to existing bug-class if applicable
- At least one attacker-model that exercises the pattern

If your contribution is compliance-focused (GDPR, NIS2, etc.), extend `compliance-mappings.md` with the regulatory article or control reference.

## Warranty

None. breachguard is provided as-is. Security audits are judgment-laden work; a skill that guides Claude through structured analysis will still produce errors, miss classes of issues, and occasionally generate findings that sound plausible but aren't. Verify every finding against the code yourself before acting on it.
