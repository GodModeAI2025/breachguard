# Agent Output (Machine-Readable)

breachguard kann Findings als **strukturierte YAML/JSON-Artifacts** ausgeben,
die downstream-Agents (Ticket-Creator, CI-Gate, Remediation-Bot, Compliance-
Dashboard) direkt konsumieren koennen.

Alle Schemas fuehren den Prefix `breachguard.*` (vormals `repolens.*` im
Upstream-Skill).

## Drei Output-Modi

| Modus | Wann | Output |
|-------|------|--------|
| **Human** (Default) | User liest selbst | Markdown-Report mit Prosa |
| **Agent** | User sagt "als YAML", "for my pipeline", "machine-readable", "agent output" | NUR YAML-Block, keine Prosa |
| **Dual** | User sagt "beides", "mit Agent-Output", "also give me YAML" | Erst Markdown-Report, dann YAML-Block |

**Trigger fuer Agent-Modus:** "YAML", "JSON", "for my pipeline", "machine-readable",
"agent output", "structured", "fuer meine CI/CD", "als Artifact".

## Schema: `finding` (einzelner Befund)

```yaml
schema: breachguard.finding
version: 1.0
id: F-001                          # Sequenziell ab F-001
severity: CRITICAL                 # CRITICAL | HIGH | MEDIUM | LOW
confidence: HIGH                   # HIGH | MEDIUM | LOW  (NEU)
verdict: TRUE_POSITIVE             # TRUE_POSITIVE | FALSE_POSITIVE | UNCERTAIN | RAW (NEU)
evidence_level: DIRECT             # DIRECT | INFERENCE | HEURISTIC  (NEU)
lens: injection                    # Lens-ID aus domains.json
domain: security
bug_class: sql_injection           # siehe bug-class-verification.md  (NEU)
file: routes/users.js
line: 4
snippet: |
  db.query("SELECT * FROM users WHERE id=" + req.params.id)
problem: "SQL Injection via unsanitized route parameter"
fix: |
  Use parametrized query:
  db.query("SELECT * FROM users WHERE id = ?", [req.params.id])
effort_minutes: 30                 # ~1-Hour-Rule: <=60 default, groessere als tracking
cwe: CWE-89                        # Wenn bekannt; sonst weglassen
owasp_top10: A03:2021              # Wenn zutreffend
attacker_model:                    # NEU
  who: unauthenticated-remote
  access: network-only
  interface: public-api
exploitability: trivial            # trivial | moderate | conditional | theoretical  (NEU)
gates:                             # FP-Verifikations-Trace (NEU)
  sink: PASS                       # PASS | FAIL | UNCERTAIN
  source: PASS
  reachability: PASS
  validation: PASS
  attacker_control: PASS
  impact: PASS
bug_class_checks:                  # NEU
  parameterization: FAIL           # FAIL = vulnerable-pattern bestaetigt
  orm_escape_hatch: NA
  second_order: NA
references:                        # Regulierungs-/Standard-Referenzen
  - "OWASP Top 10 A03:2021 - Injection"
  - "CWE-89"
next_agent_actions:
  - agent: ticket_creator
    command: create_ticket
    payload: {title: "[CRITICAL] SQL Injection in routes/users.js:4", priority: high, labels: [security, verdict-true-positive]}
  - agent: ci_gate
    command: block_release
    reason: critical_security_finding
```

**Pflicht-Felder bei Security-Modi:** `confidence`, `verdict`,
`evidence_level`, `bug_class`, `attacker_model`, `exploitability`, `gates`.
Bei nicht-Security-Modi koennen sie weggelassen werden.

**UNCERTAIN-Findings:** Mit Label `verdict-uncertain` ins Ticket; Body
enthaelt welche Gates uncertain waren.

**FALSE_POSITIVE-Findings:** Werden **nicht** im `next_agent_actions`-Block
fuer `ticket_creator` vorgeschlagen.

## Schema: `audit_summary` (Top-Level-Block am Ende)

```yaml
schema: breachguard.audit_summary
version: 1.0
run:
  mode: audit                      # audit | triage | deep-audit | variant | verify-fix | bugfix | feature | discover | deploy | custom | opensource | content
  domain: security                 # einzelne Domain ODER "multi"
  lens_ids_applied: [injection, auth-session, secrets, insecure-defaults, sharp-edges]
  target: "path/to/repo or snippet description"
  agent: breachguard-skill
  version: 0.3.0
  started_at: "2026-04-19T10:00:00Z"
  finished_at: "2026-04-19T10:05:00Z"
scope:                             # NEU
  included: ["/src/auth/", "/src/api/v2/"]
  excluded: ["/src/legacy/", "/src/vendor/"]
  coverage_limits:
    - "Middleware-Order nicht verifiziert"
    - "Rate-Limit-Config nicht im Repo gefunden"
attacker_model_primary:            # NEU
  who: unauthenticated-remote
  access: network-only
  interface: public-api
attacker_models_secondary:         # NEU
  - {who: authenticated-user, access: valid-account, interface: public-api}
metrics:
  total_findings: 8
  by_severity: {CRITICAL: 2, HIGH: 3, MEDIUM: 2, LOW: 1}
  by_verdict: {TRUE_POSITIVE: 6, UNCERTAIN: 2, FALSE_POSITIVE: 0}   # NEU
  by_confidence: {HIGH: 5, MEDIUM: 3, LOW: 0}                       # NEU
  by_lens:
    injection: 2
    auth-session: 2
    secrets: 2
    insecure-defaults: 2
  effort_total_hours: 5.5
release_gate:
  decision: BLOCK                  # BLOCK (has CRITICAL-TP) | WARN (has HIGH-TP or multiple UNCERTAIN) | PASS
  blocking_findings: [F-001, F-003]
  rationale: "2 CRITICAL TRUE_POSITIVE findings in security.injection and security.auth-session"
next_agent_actions:
  - agent: ticket_creator
    command: bulk_create
    payload: {finding_ids: [F-001, F-002, F-003, F-004, F-005, F-006]}
  - agent: ticket_creator
    command: bulk_create
    payload: {finding_ids: [F-007, F-008], labels: [verdict-uncertain, needs-manual-review]}
  - agent: remediation_agent
    command: generate_pr
    payload: {finding_ids: [F-001], strategy: quick-fix}
  - agent: notification_agent
    command: alert_security_team
    reason: critical_finding
```

## Schema: `compliance_matrix` (fuer Compliance-Domain)

```yaml
schema: breachguard.compliance_matrix
version: 1.0
run: {mode: audit, domain: compliance, ...}
matrix:
  - regulation: "GDPR Art. 32"
    requirement: "Technical and organizational measures"
    lens: secure-sdlc
    status: FAIL                   # PASS | FAIL | PARTIAL | NOT_TESTED
    findings: [F-002]
    evidence: "No SAST in CI/CD pipeline"
  - regulation: "NIS2 Art. 21(2)(h)"
    requirement: "Cybersecurity training"
    lens: secure-sdlc
    status: NOT_TESTED
    findings: []
    evidence: null
release_gate:
  decision: BLOCK
  blocking_regulations: ["GDPR Art. 32", "NIS2 Art. 21"]
  non_compliance_severity: high
next_agent_actions:
  - agent: grc_agent
    command: update_compliance_dashboard
    payload: {regulations: [GDPR, NIS2], status_changes: [...]}
  - agent: audit_trail_agent
    command: log_finding
    reason: regulatory_non_compliance
```

## Schema: `change_impact` (fuer Custom-Mode)

```yaml
schema: breachguard.change_impact
version: 1.0
change_statement: "Migrating from REST to GraphQL"
run: {mode: custom, ...}
impacts:
  - id: I-001
    impact_level: BREAKING           # BREAKING | REQUIRED | RECOMMENDED | OPTIONAL
    lens: api-versioning
    file: api/routes/v1.ts
    line: 12
    impact_type: direct              # direct | indirect | downstream
    description: "REST route definitions need replacement"
    adaptation: "Generate GraphQL schema + resolvers for existing endpoints"
    effort_minutes: 120
next_agent_actions:
  - agent: migration_planner
    command: create_migration_plan
    payload: {change: "REST->GraphQL", impact_count: 15}
```

## Schema: `fix_verification` (NEU, fuer verify-fix-Mode)

```yaml
schema: breachguard.fix_verification
version: 1.0
source_finding_id: F-001
source_finding_summary: "SQL Injection in users.js:4"
fix_commits: ["abc123", "def456"]
fix_author: "jane@example.com"
fix_date: "2026-04-18T14:30:00Z"
remediation:
  addresses_root_cause: true
  symptom_only: false
  root_cause_diagnosis: "String-Konkat -> Parameterized Query, korrekt"
regression:
  new_findings_in_diff: []           # Leere Liste = keine neuen Vulns
  regressions_detected: false
completeness:
  variants_known_count: 9
  variants_fixed_count: 8
  variants_unfixed: [
    {file: "orders.js", line: 12, reason: "not in fix commit range"}
  ]
verdict: PARTIAL_REMEDIATION         # FULL_REMEDIATION | PARTIAL_REMEDIATION | FAILED_REMEDIATION | REGRESSION_INTRODUCED
new_release_gate:
  decision: WARN                     # BLOCK/WARN/PASS
  rationale: "1 variant unfixed (orders.js:12)"
next_agent_actions:
  - agent: ticket_creator
    command: create_ticket
    payload: {title: "Unfixed variant of F-001 at orders.js:12", priority: medium}
```

## Schema: `variant_report` (NEU, fuer variant-Mode)

```yaml
schema: breachguard.variant_report
version: 1.0
original_finding:
  id: F-001
  file: "routes/users.js"
  line: 4
  lens: injection
  bug_class: sql_injection
root_cause_abstraction: |
  String-Konkatenation in db.query() mit req.*-Source, ohne Parameterization.
search_iterations:
  - iter: 1
    pattern: 'db\.query\(".*" \+ req\.params\.id'
    matches_total: 1
    matches_true: 1
    matches_false: 0
    fp_rate: 0.00
    decision: KEEP_AND_EXPAND
  - iter: 2
    pattern: 'db\.query\(".*" \+ req\.params\.'
    matches_total: 4
    matches_true: 3
    matches_false: 1
    fp_rate: 0.33
    decision: KEEP_AND_EXPAND
  - iter: 3
    pattern: '\w+\.query\(".*" \+ req\.'
    matches_total: 40
    matches_true: 8
    matches_false: 32
    fp_rate: 0.80
    decision: REVERT_EXCEEDS_50_PERCENT
final_pattern: 'db\.query\(".*" \+ req\.'
variants_found:
  - {id: V-001, file: "orders.js", line: 12, confidence: HIGH, exploitability: trivial}
  - {id: V-002, file: "orders.js", line: 28, confidence: HIGH, exploitability: trivial}
  - {id: V-003, file: "reports.js", line: 8, confidence: MEDIUM, exploitability: moderate}
  # ...
metrics:
  variants_total: 9
  by_confidence: {HIGH: 6, MEDIUM: 3, LOW: 0}
  by_exploitability: {trivial: 5, moderate: 3, conditional: 1, theoretical: 0}
next_agent_actions:
  - agent: ticket_creator
    command: bulk_create
    payload: {finding_ids: [V-001, V-002, ...], parent: F-001}
  - agent: detector_agent                       # NEU (Detector-Rolle)
    command: export_semgrep_rule
    payload: {final_pattern: "db.query + req concat", rule_id: "breachguard-injection-sql-concat-req"}
```

## Schema: `detection_rule` (NEU, fuer Detector-Rolle)

```yaml
schema: breachguard.detection_rule
version: 1.0
source_finding_id: F-001
rule_format: semgrep               # semgrep | codeql | rego
rule_id: "breachguard-injection-sql-concat-req"
severity: ERROR                    # Gemapped aus Finding-Severity
languages: [javascript, typescript]
rule_body: |
  rules:
    - id: breachguard-injection-sql-concat-req
      message: "Potential SQL Injection - string concat with req-source"
      severity: ERROR
      languages: [javascript, typescript]
      metadata:
        cwe: CWE-89
        owasp: A03:2021
        lens: injection
        confidence: HIGH
        source: breachguard
      pattern-either:
        - pattern: $DB.query("..." + req.$X)
        - pattern: $DB.query(`...${req.$X}...`)
      pattern-not:
        - pattern: $DB.query($CONST_STRING)
test_cases:
  positive:
    - 'db.query("SELECT * FROM users WHERE id=" + req.params.id)'
    - 'pool.query(`SELECT * FROM items WHERE n=${req.body.n}`)'
  negative:
    - 'db.query("SELECT * FROM users WHERE id = ?", [req.params.id])'
    - 'db.query("SELECT VERSION()")'
validation:
  tested_on_source_finding: true
  fp_rate_on_repo: 0.11             # 1 FP in 9 Matches
  recommended_for_ci: true
next_agent_actions:
  - agent: ci_configurator
    command: add_semgrep_rule
    payload: {rule_file: "breachguard-rules.yml", rule_id: "breachguard-injection-sql-concat-req"}
```

## Downstream-Agent-Katalog

Moegliche `agent:`-Werte in `next_agent_actions`:

| Agent | Zweck | Typische Commands |
|-------|-------|-------------------|
| `ticket_creator` | GitHub/Jira/Linear Issues anlegen | `create_ticket`, `bulk_create`, `link_to_epic` |
| `ci_gate` | CI/CD-Pipeline blocken | `block_release`, `require_review`, `post_status` |
| `remediation_agent` | Auto-Fix PR generieren | `generate_pr`, `suggest_patch`, `apply_codemod` |
| `code_reviewer` | Second-Opinion-Review | `review_finding`, `validate_fix` |
| `notification_agent` | Stakeholder benachrichtigen | `alert_security_team`, `email_owner`, `slack_channel` |
| `dashboard_agent` | Metrics-Dashboard aktualisieren | `update_dashboard`, `push_metric` |
| `audit_trail_agent` | GRC-Log / Audit-Trail | `log_finding`, `archive_evidence` |
| `grc_agent` | Compliance-System syncen | `update_compliance_dashboard`, `trigger_risk_assessment` |
| `sbom_agent` | SBOM regenerieren | `regenerate_sbom`, `diff_sbom` |
| `security_agent` | Security-Review eskalieren | `escalate`, `assign_to_security_team` |
| `detector_agent` (NEU) | Detection-Rule exportieren / in CI einfuegen | `export_semgrep_rule`, `export_codeql_query`, `add_to_ci` |
| `migration_planner` | Migrations-Plan erstellen | `create_migration_plan`, `estimate_effort` |
| `ci_configurator` (NEU) | CI-Config anpassen | `add_semgrep_rule`, `add_codeql_query`, `update_workflow` |

Dies ist der **kanonische Katalog**. Weitere Agents koennen User-seitig
definiert werden — dann im `agent:`-Feld freie Zeichenkette, muss User kennen.

## Release-Gate-Regel (aktualisiert)

```
IF any finding has (verdict = TRUE_POSITIVE AND severity = CRITICAL):
  decision = BLOCK
ELIF any finding has (verdict = TRUE_POSITIVE AND severity = HIGH):
  decision = WARN
ELIF count(verdict = UNCERTAIN) >= 3:
  decision = WARN  (Scope-Problem, braucht menschliche Review)
ELIF any finding has severity = CRITICAL AND verdict != FALSE_POSITIVE:
  decision = WARN
ELSE:
  decision = PASS
```

FALSE_POSITIVES zaehlen in der Gate-Decision nicht.

## Regeln fuer Agent-Modus

1. **Nur valide YAML** ausgeben — kein Markdown-Wrapping ausser dem ```yaml-Block
2. **Keine Halluzinationen** in strukturierten Feldern — wenn `cwe` nicht bekannt,
   weglassen statt raten
3. **`effort_minutes` ehrlich** — wenn groesser als 60, Issue splitten oder als
   Tracking markieren
4. **`next_agent_actions` konservativ** — nur Actions vorschlagen, die sinnvoll
   sind; keine leere Aktion fuer jedes Finding
5. **`release_gate`-Decision regelbasiert** (siehe oben)
6. **Bei Security-Modi: Gate- und Confidence-Felder pflicht.**
7. **UNCERTAIN-Findings:** immer mit `labels: [verdict-uncertain]` im
   Ticket-Vorschlag.

## Minimal-Example (Dual Mode)

**User:** "Audit diesen Code auf Security, gib mir beides."

**Output:**

```markdown
### [CRITICAL / HIGH / TRUE_POSITIVE] SQL Injection via unsanitized route parameter
**Lens:** `injection`
**File:** `routes/users.js:4`
**Attacker-Model:** unauthenticated-remote / network-only / public-api
**Evidence-Level:** DIRECT

**Problem:** ...

**Fix:** ...
```

```yaml
schema: breachguard.finding
version: 1.0
id: F-001
severity: CRITICAL
confidence: HIGH
verdict: TRUE_POSITIVE
evidence_level: DIRECT
lens: injection
domain: security
bug_class: sql_injection
file: routes/users.js
line: 4
problem: "SQL Injection via unsanitized route parameter"
fix: "Parametrized Query via db.query(sql, [params])"
effort_minutes: 30
cwe: CWE-89
owasp_top10: A03:2021
attacker_model: {who: unauthenticated-remote, access: network-only, interface: public-api}
exploitability: trivial
gates: {sink: PASS, source: PASS, reachability: PASS, validation: PASS, attacker_control: PASS, impact: PASS}
bug_class_checks: {parameterization: FAIL}
next_agent_actions:
  - {agent: ticket_creator, command: create_ticket, payload: {priority: high, labels: [security, verdict-true-positive]}}
  - {agent: ci_gate, command: block_release, reason: critical_security}
---
schema: breachguard.audit_summary
version: 1.0
run: {mode: audit, domain: security, lens_ids_applied: [injection]}
metrics:
  total_findings: 1
  by_severity: {CRITICAL: 1, HIGH: 0, MEDIUM: 0, LOW: 0}
  by_verdict: {TRUE_POSITIVE: 1, UNCERTAIN: 0, FALSE_POSITIVE: 0}
release_gate: {decision: BLOCK, blocking_findings: [F-001]}
```
