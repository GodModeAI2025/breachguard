# Rollen (Persona/Output-Dimension)

Zusaetzlich zum **Modus** (audit/bugfix/feature/...) und der **Domain**
(security/performance/...) gibt es eine dritte Routing-Dimension: die **Rolle**,
die steuert, **wie der Output formatiert** ist und **welche Aspekte betont**
werden.

## Sechs Rollen

| Rolle | Trigger | Output-Schwerpunkt |
|-------|---------|-------------------|
| **Auditor** (Default) | "audit", "review", "findings" | Severity + Fundstelle + knapper Fix + Confidence + Verdict |
| **Developer** | "wie fixe ich", "show the fix", "gib mir den Code", "fix-first" | Arbeitsfaehiger Code pro Fix, Before/After-Snippets |
| **Architect** | "systemic issues", "root cause", "architectural review", "patterns" | Querschnitts-Analyse, Root-Causes, Refactoring-Vorschlaege uber Einzel-Findings hinweg |
| **Reporter** | "executive summary", "Management-Zusammenfassung", "report fuer den Vorstand" | Metriken, Trends, Risiko-Einschaetzung, kein Code |
| **Agent** | "YAML", "JSON", "for my pipeline", "agent output", "machine-readable" | Nur strukturierte YAML/JSON-Artifacts (siehe `agent-output.md`) |
| **Detector** (NEU) | "Semgrep-Regel", "CodeQL-Query", "detection rule", "in CI einbauen", "als Regel exportieren", "rule skeleton" | Semgrep-YAML oder CodeQL-QL-Skeleton pro Finding |

## Default-Regel

Wenn der User keine explizite Rolle triggert: **Auditor-Rolle**. Das ist die
Basis-Sicht, die Findings strukturiert auflistet ohne besondere Schwerpunkte.

## Kombinierte Routing-Beispiele

**"Review mein Security-Code als Developer"**
-> Mode: audit, Domain: security, Rolle: Developer
-> Pro Finding: konkreter Code-Fix im Fix-Block, ggf. vor/nach-Snippets

**"Executive Summary fuer compliance audit"**
-> Mode: audit, Domain: compliance, Rolle: Reporter
-> Keine Einzel-Findings, stattdessen: Regulations-Matrix-Status,
   Blocking-Regulations, High-Level-Aktionen, Risk-Rating

**"GDPR-Check als YAML fuer unsere Pipeline"**
-> Mode: audit, Domain: compliance, Rolle: Agent
-> Nur YAML mit compliance_matrix-Schema, keine Prosa

**"Architekturreview des Repo"**
-> Mode: audit, Domain: architecture + code-quality + maintainability,
   Rolle: Architect
-> Querschnitts-Findings: "Circular Dependency zwischen A und B", "Concern X
   ist auf 5 Module verteilt", Refactoring-Plan

**"Security-Audit und dann Semgrep-Regeln fuer CI"** (NEU)
-> Mode: deep-audit, Domain: security, Rolle: Detector (oder Dual
   Auditor+Detector)
-> Erst klassischer Audit, dann fuer jedes TRUE_POSITIVE-Finding ein
   Semgrep-Rule-Skeleton, das das Muster fuer CI-Detection encoded

**"Variant-Analysis und Rule-Export"** (NEU)
-> Mode: variant, Rolle: Detector
-> Variant-Hunt, finales Pattern als Rule-Skeleton exportiert

## Output-Schema pro Rolle

### Auditor (Default)

```markdown
### [SEVERITY / CONFIDENCE / VERDICT] Titel
**Lens:** `<lens-id>`
**File:** `path:line`
**Attacker-Model:** <WHO + ACCESS + INTERFACE>
**Evidence-Level:** DIRECT | INFERENCE | HEURISTIC
**Bug-Class:** <bug-class-id> (bei Security)

**Gates** (bei Security-Modi): (Sink:PASS, Source:PASS, Reachability:PASS,
                                Validation:PASS, Attacker-Control:PASS,
                                Impact:PASS)

**Problem:** <1-3 Saetze>

**Fix:** <Kern-Idee, ~1h Aufwand>
```

### Developer

```markdown
### [SEVERITY / CONFIDENCE / VERDICT] Titel
**Lens:** `<lens-id>`
**File:** `path:line`

**Problem:** <1-3 Saetze>

**Before:**
```<sprache>
<aktueller Code>
```

**After:**
```<sprache>
<gefixter Code, arbeitsfaehig>
```

**Notes:** <Edge-Cases, zusaetzliche Tests, Docs-Updates>
```

### Architect

```markdown
## Querschnitts-Befund: <Title>
**Pattern:** <Beschreibung des Anti-Patterns>
**Betroffene Dateien:** <file list>
**Lenses:** <lens-ids>

**Root Cause:** <systemische Ursache>

**Refactoring-Vorschlag:**
1. Schritt 1
2. Schritt 2
...

**Einzelne Fundstellen-Indizes:** F-001, F-003, F-007 (siehe Auditor-Report fuer Details)
```

### Reporter

```markdown
# <Domain> Audit — Executive Summary
**Audit-Umfang:** <Files/Domains/Modi>
**Attacker-Model:** <WHO/ACCESS/INTERFACE>
**Datum:** <ISO>

## Key Metrics
- Findings Gesamt: X (CRITICAL: a, HIGH: b, MEDIUM: c, LOW: d)
- Verdict-Verteilung: TRUE_POSITIVE: a, UNCERTAIN: b
- Blocking Release: ja/nein
- Geschaetzter Fix-Aufwand: X.Y Stunden

## Top-3 Risiken
1. ...
2. ...
3. ...

## Empfohlene Massnahmen
- Sofort: ...
- Kurzfristig: ...
- Strategisch: ...

## Coverage-Limits
- ...
```

### Agent

Siehe `agent-output.md` — nur YAML/JSON, kein Markdown.

### Detector (NEU)

Die Detector-Rolle exportiert **wiederverwendbare Detection-Rules** —
Semgrep, CodeQL, oder (bei Infrastruktur-Findings) OPA/Rego-Skeletons.
Das macht aus einem einmaligen Audit eine permanente CI-Guard.

**Auswahl der Ziel-Technologie:**
- User sagt "Semgrep" → Semgrep-YAML
- User sagt "CodeQL" → CodeQL-QL-Skeleton
- User sagt "OPA"/"policy" → Rego
- User sagt nichts → Semgrep (einfachster Einstieg, breiteste Language-
  Coverage)

**Output-Schema Semgrep:**
````markdown
### [SEVERITY] <Finding-Titel>
**Source-Finding-ID:** F-001
**Detection-Goal:** Findet <Muster-Beschreibung> in <Sprache/Framework>

```yaml
rules:
  - id: breachguard-<lens-id>-<kurz-titel>
    message: "<1-Satz-Beschreibung der Vuln>"
    severity: ERROR  # ERROR|WARNING|INFO, mapt SEVERITY
    languages: [javascript, typescript]  # oder je nach Finding
    metadata:
      cwe: CWE-89
      owasp: A03:2021
      lens: injection
      source: breachguard
      confidence: HIGH
    pattern-either:
      - pattern: $DB.query("..." + $REQ.$X)
      - pattern: $DB.query(`...${$REQ.$X}...`)
    pattern-not-either:
      - pattern: $DB.query($SAFE_CONST)
```

**Test-Cases** (fuer Semgrep `--test`):
```js
// ruleid: breachguard-injection-sql-concat-req
db.query("SELECT * FROM users WHERE id=" + req.params.id)

// ruleid: breachguard-injection-sql-concat-req
db.query(`SELECT * FROM items WHERE name='${req.body.name}'`)

// ok
db.query("SELECT * FROM users WHERE id = ?", [req.params.id])
```

**Validation-Hinweis:** Pattern auf Original-Finding + manuell
klassifizierten Geschwister-Stellen testen (wenn aus `variant`-Mode),
auf 50%-FP-Rate pruefen, ggf. `pattern-not`-Klauseln verengen.
````

**Output-Schema CodeQL:**
````markdown
### [SEVERITY] <Finding-Titel>
**Source-Finding-ID:** F-001

```ql
/**
 * @name SQL Injection via req-derived concat
 * @description Detects string concatenation of req.* values into SQL.
 * @kind path-problem
 * @severity error
 * @precision high
 * @id breachguard/injection/sql-concat-req
 * @tags security
 *       external/cwe/cwe-89
 *       external/owasp/owasp-a03
 */

import javascript
import DataFlow::PathGraph

class SqlConcatConfig extends TaintTracking::Configuration {
  SqlConcatConfig() { this = "SqlConcatConfig" }

  override predicate isSource(DataFlow::Node source) {
    exists(PropAccess pa | pa = source.asExpr() |
      pa.getBase().(PropAccess).getPropertyName() = "params" or
      pa.getBase().(PropAccess).getPropertyName() = "body" or
      pa.getBase().(PropAccess).getPropertyName() = "query"
    )
  }

  override predicate isSink(DataFlow::Node sink) {
    exists(MethodCallExpr call |
      call.getMethodName() = "query" and
      sink.asExpr() = call.getArgument(0)
    )
  }
}

from SqlConcatConfig cfg, DataFlow::PathNode source, DataFlow::PathNode sink
where cfg.hasFlowPath(source, sink)
select sink.getNode(), source, sink,
  "Untrusted $@ flows to SQL query.", source.getNode(), "user input"
```
````

**Output-Schema OPA/Rego** (fuer Infrastruktur-Findings wie Terraform,
K8s, Cloud-Configs):

````markdown
### [SEVERITY] <Finding-Titel>

```rego
package breachguard.kubernetes.pod_security

deny[msg] {
  input.kind == "Pod"
  input.spec.containers[_].securityContext.privileged == true
  msg := sprintf("Privileged container: %s", [input.metadata.name])
}
```
````

**Detector-Regeln:**

1. **Ein Rule-Skeleton pro TRUE_POSITIVE-Finding.** Keine Rule fuer
   FALSE/UNCERTAIN-Findings (ausser User fordert es explizit).
2. **Immer mit Test-Cases** (positiv + negativ).
3. **Metadata-Block pflicht:** CWE, OWASP-Kat, Confidence, Source-Tool-Name.
4. **Default-Severity-Mapping:**
   - `CRITICAL`/`HIGH` → Semgrep `ERROR`, CodeQL `error`
   - `MEDIUM` → Semgrep `WARNING`, CodeQL `warning`
   - `LOW` → Semgrep `INFO`, CodeQL `note`
5. **Kein Auto-Commit.** Die Regel wird **vorgeschlagen**, User entscheidet
   ueber Aufnahme in CI.
6. **Rule-ID-Naming:** `breachguard-<lens-id>-<kebab-titel>` fuer Semgrep,
   `breachguard/<lens-id>/<kebab-titel>` fuer CodeQL.

**Wann Detector-Rolle ideal:**
- Nach `variant`-Mode → finales Pattern wird Rule.
- Nach `deep-audit` mit systemischen Findings → Rule verhindert Regression.
- Bei Lib-Sharp-Edges → Rule verbietet Misuse projekt-weit.

## Rolle explizit ansagen

Der User kann die Rolle jederzeit in der Anfrage benennen:

- "... als Developer"
- "... im Architect-Modus"
- "... als YAML fuer Agent"
- "... Executive Summary"
- "... als Semgrep-Regel" (Detector)
- "... und dann Rule fuer CI exportieren" (Dual Auditor+Detector)

Wenn nicht benannt: Claude triagiert aus den sonstigen Triggern (s.o.).
