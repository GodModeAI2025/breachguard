# Gemeinsamer Workflow

## Schritt-fuer-Schritt

1. **Input einsammeln.** Code/Repo/Snippet vom User.

2. **Dreidimensional routen:**
   - **Modus** (audit/triage/deep-audit/variant/verify-fix/bugfix/feature/
     discover/deploy/custom/opensource/content) — siehe `modes.md`
   - **Domain** (security/performance/...) — siehe SKILL.md Tabelle
   - **Rolle** (Auditor/Developer/Architect/Reporter/Agent/Detector) —
     siehe `roles.md`

3. **Wenn Security-Modus** (`audit`/`triage`/`deep-audit`/`variant`/
   `verify-fix` gegen `security`-Domain): **5-Phasen-Workflow** aus
   `methodology.md` starten. Sonst: klassischer Lens-Anwendungs-Workflow
   (RepoLens-Stil).

4. **Lens-Auswahl innerhalb der Domain:**
   - Snippet <200 LOC → 2-4 passende Lenses
   - Repo 1-50 Files → alle Lenses der Domain
   - Repo >50 Files → **Eskalation** zu Original-CLI-Kommando

5. **Domain-Referenz laden:** `references/lenses/<domain-id>.md`.
   - Bei `security`-Domain zusaetzlich bei Bedarf:
     `lenses/insecure-defaults.md`, `lenses/sharp-edges.md`,
     `lenses/supply-chain.md` (die drei neuen breachguard-Lenses).
   - Bei `compliance`-Domain zusaetzlich: `references/compliance-mappings.md`.
   - Bei Developer-Rolle zusaetzlich: `references/code-examples.md`.
   - Bei Security-Modi zusaetzlich: `methodology.md`, `fp-verification.md`,
     `attacker-models.md`, `bug-class-verification.md`.
   - Bei `variant`-Modus zusaetzlich: `variant-analysis.md`.
   - Bei Detector-Rolle: `roles.md` §Detector-Sektion.

6. **Anwenden.** Pro Lens das Artefakt durchgehen. **Nur substantielle
   Findings.**

7. **FP-Verifikation** (nur Security-Modi ausser `triage`): Jedes Raw
   Finding durch die 6 Gates aus `fp-verification.md` fuehren.
   Bug-Class-spezifische Checks aus `bug-class-verification.md` hinzufuegen.
   Verdict setzen: TRUE_POSITIVE / FALSE_POSITIVE / UNCERTAIN.

8. **Findings strukturieren** nach Modus + Rolle.

9. **~1-Hour-Rule.** Issues >1h → Tracking-Issue.

10. **Output**:
    - Rolle Auditor/Developer/Architect/Reporter → Markdown
    - Rolle Agent → YAML nach `agent-output.md`-Schema
    - Rolle Detector → Markdown-Finding **plus** Rule-Skeleton (Semgrep/
      CodeQL/Rego)
    - Dual → Beides
    Am Ende optional `gh issue create`-Kommandos zum Pasten anbieten.
    **Niemals selbst Issues erstellen** oder Shell-Kommandos ausfuhren ohne
    explizite Bestatigung.

## Guardrails

- **Keine Halluzinationen.** "Keine Findings" ist valide.
- **Severity + Confidence + Verdict ehrlich.** `CRITICAL` = Prod-Breaker/
  Security, `LOW` = Convention. Bei UNCLEAR: `UNCERTAIN` setzen.
- **Scope-Limits explizit.** Report-Header listet Included/Excluded.
- **Bei grossem Repo** → CLI-Fallback.
- **Bei `deploy`-Modus** → niemals Shell gegen Live-Systeme.
- **Bei Agent-Rolle** → nur valide YAML, kein Markdown-Wrapping ausser
  ```yaml-Block.
- **Bei Detector-Rolle** → Test-Cases (positiv + negativ) pflicht; kein
  Auto-Commit, nur Vorschlag.
- **Bei Security-Modi** → keine Exploit-Regurgitation (siehe
  `methodology.md` §Kern-Prinzip 5).

## Negative Beispiele (NICHT reporten)

- "Variable `x` konnte besser heissen" — Style
- "Funktion ist 40 Zeilen" — Lange allein kein Issue
- "Konnte TypeScript sein" — Tech-Praferenz
- "Hier fehlt ein Kommentar" — ausser die `documentation`-Domain huntet es
- Findings, die nach FP-Gate `FALSE_POSITIVE` sind (bei Security-Modi).
  **Ausnahme:** in Dual/Agent-Rolle koennen FPs in separater Sektion
  dokumentiert werden, wenn User das explizit will.

## Few-Shot (Auditor-Rolle, audit-Modus, Security-Domain)

**Input:** 12 Zeilen Node.js:
`app.get('/users/:id', (req,res) => { db.query("SELECT * FROM users WHERE id=" + req.params.id) })`

**Phasen-Durchlauf:**

*Phase 1 (Intake):* Scope = Snippet. Attacker-Model = `unauthenticated-remote /
network-only / public-api` (Default bei Webapp).

*Phase 2 (Context):* Entrypoint = HTTP-Route. Trust-Boundary =
Request-Parameter → DB. Kein Validator sichtbar.

*Phase 3 (Lens):* `injection`-Lens zuerst. Raw-Finding: SQL-Concat.

*Phase 4 (FP-Gate):*
- Sink: PASS (`db.query` mit String-Arg)
- Source: PASS (`req.params.id`)
- Reachability: PASS (direkter Pfad)
- Validation: PASS (keine)
- Attacker-Control: PASS (URL-Param, full)
- Impact: PASS (UNION-Select moeglich)
- Bug-Class (Parameterization): FAIL (String-Konkat) → TRUE_POSITIVE.

*Phase 5 (Report):*

**Finding:**

```markdown
### [CRITICAL / HIGH / TRUE_POSITIVE] SQL Injection via unsanitized route parameter
**Lens:** `injection`
**File:** `routes/users.js:4`
**Attacker-Model:** unauthenticated-remote / network-only / public-api
**Evidence-Level:** DIRECT
**Bug-Class:** sql_injection

**Problem:**
`req.params.id` wird unescaped in SQL konkateniert. Request `/users/1 OR 1=1--`
dumpt die users-Tabelle.

**Fix:**
Parametrized Query: `db.query("SELECT * FROM users WHERE id = ?", [req.params.id])`.
Zusatzlich `req.params.id` als Integer validieren.
```

## Few-Shot (Developer-Rolle, gleicher Input)

```markdown
### [CRITICAL / HIGH / TRUE_POSITIVE] SQL Injection via unsanitized route parameter
**Lens:** `injection`
**File:** `routes/users.js:4`

**Problem:** `req.params.id` wird unescaped in SQL konkateniert.

**Before:**
```js
app.get('/users/:id', (req, res) => {
  db.query("SELECT * FROM users WHERE id=" + req.params.id);
});
```

**After:**
```js
app.get('/users/:id', (req, res) => {
  const id = parseInt(req.params.id, 10);
  if (isNaN(id)) return res.status(400).send('Invalid ID');
  db.query('SELECT * FROM users WHERE id = $1', [id]);
});
```

**Notes:** Integer-Validierung vor der Query als Defense-in-Depth.
Alternativ: ein Validation-Schema (Zod/Joi) auf `req.params`.
```

## Few-Shot (Agent-Rolle, gleicher Input)

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
fix: "Use parametrized query: db.query('SELECT ... WHERE id = $1', [id])"
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
```

## Few-Shot (Detector-Rolle, gleicher Input) — NEU

```markdown
### [CRITICAL] SQL Injection via req-derived concat — Detection Rule
**Source-Finding-ID:** F-001
**Detection-Goal:** Findet String-Konkatenation von req.*-Werten in
                    db.query-Aufrufen (JavaScript/TypeScript).

```yaml
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
```

**Test-Cases** (fuer `semgrep --test`):
```js
// ruleid: breachguard-injection-sql-concat-req
db.query("SELECT * FROM users WHERE id=" + req.params.id);

// ruleid: breachguard-injection-sql-concat-req
pool.query(`SELECT * FROM items WHERE name='${req.body.name}'`);

// ok
db.query("SELECT * FROM users WHERE id = ?", [req.params.id]);

// ok
db.query("SELECT VERSION()");
```

**Hinweis:** Regel vor CI-Commit auf Repo-wide Matches pruefen, ggf.
`pattern-not`-Klauseln ergaenzen wenn False-Positives auftreten (siehe
`variant-analysis.md` 50%-FP-Regel).
```

## Few-Shot (Variant-Modus) — NEU

**Input:** "F-001 ist eine SQL-Injection in users.js:4. Finde die Geschwister
im Repo."

**Ablauf:**
1. Root-Cause abstrahieren (siehe `variant-analysis.md` Schritt 1).
2. Exact-Match-Pattern bauen.
3. Schrittweise abstrahieren, Matches klassifizieren.
4. Bei >50% FP → revert.
5. Report mit Variants + optional Rule-Skeleton.

**Output:** siehe `variant-analysis.md` Beispiel-Durchlauf. YAML-Variante:
`breachguard.variant_report` in `agent-output.md`.

## Few-Shot (Verify-Fix-Modus) — NEU

**Input:** "F-001 wurde in Commit abc123..def456 gefixt. Ist das sauber?"

**Ablauf:**
1. Fix-Delta extrahieren.
2. Remediation-Check: Adressiert der Fix die Root-Cause?
3. Regression-Check: Neue Vulns im Diff?
4. Completeness-Check: Alle Variants gefixt?
5. Report: Verdict FULL/PARTIAL/FAILED/REGRESSION_INTRODUCED.

**Output:** siehe `modes.md` §verify-fix Beispiel, bzw.
`breachguard.fix_verification`-Schema in `agent-output.md`.
