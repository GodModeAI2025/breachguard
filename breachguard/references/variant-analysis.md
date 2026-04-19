# Variant Analysis — eine Vuln gegeben, finde die Geschwister

**Modus:** `variant`
**Input:** Eine konkrete, bestaetigte Vulnerability.
**Output:** Liste aehnlicher Stellen im selben Repo (oder in benachbarten
Repos).

Inspiriert von Trail-of-Bits variant-analysis.

## Wann nutzt man das?

- Ein CVE wurde in einer Dependency gefixt → suche die gleiche Pattern
  in eigenem Code
- Ein Pentest-Finding ist validiert → wo gibt's das Pattern sonst noch?
- Ein Bug-Bounty-Report kam rein → variant-hunt vor Trennung
- Ein Code-Review zeigte SQL-Injection an Stelle X → gleiche Stelle in
  anderen Modulen?

**Nicht** fuer initiale Vuln-Discovery (dann `audit` oder `deep-audit`).

## Die 5 Schritte

### Schritt 1: Root-Cause statt Symptom

Bevor gesucht wird, **verstehe die Ursache**. Nicht das Symptom.

**Symptom:** "SQL Injection in `users.js:4`"
**Root-Cause:** "String-Konkatenation mit `req.params.id` ohne
Parameterization — jede gleichartige Route ist ebenfalls verwundbar"

Wenn du nur das Symptom kopierst ("suche nach `users.js:4`"), findest du
nichts. Wenn du die Root-Cause abstrahierst ("suche String-Konkat in
DB-Queries mit Request-Daten"), findest du die Geschwister.

**Checkliste Root-Cause:**
- Was ist der Sink-Typ?
- Was ist die Source?
- Welche Validation-Schicht fehlt / wird umgangen?
- Unter welchen Bedingungen ist der Code verwundbar?
- Welche Bedingungen machen Attacker-Control moeglich?

### Schritt 2: Exact-Match-Pattern bauen

Starte mit einem **extrem spezifischen** Pattern, das nur die eine bekannte
Instanz trifft. Das ist der Anker.

**Beispiel:**
Bekannte Vuln: `db.query("SELECT * FROM users WHERE id=" + req.params.id)`

Exact-Match-Pattern (ripgrep):
```
db.query\(".*" \+ req\.params\.
```

Das trifft **nur** diese eine Struktur. Validiere, dass das Pattern die
bekannte Vuln findet und nichts anderes.

### Schritt 3: Schrittweise Abstraktion

Jetzt genau **eine** Dimension gleichzeitig abstrahieren. Nach jeder
Abstraktion: alle neuen Matches manuell klassifizieren (TRUE/FALSE).

**Typische Abstraktions-Achsen** (pro Schritt nur eine!):

1. **Sink-Variant:** `db.query` → `connection.query` → `pool.query` → `*.query`
2. **Source-Variant:** `req.params.*` → `req.query.*` → `req.body.*` → `req.*`
3. **Concat-Variant:** `" + varname"` → Template-Literal ``...${var}...``
4. **Var-Name-Variant:** Spezifischer Varname → beliebiger
5. **Language-Variant:** JS → TypeScript → Andere Sprachen im Mono-Repo

**Reihenfolge:** Erst Sink, dann Source, dann Concat-Variante, dann
Var-Name — das gibt ueblicherweise die beste Signal-to-Noise-Ratio.

### Schritt 4: Stop-Condition — die 50%-FP-Regel

Nach jeder Abstraktion: **klassifiziere ALLE neuen Matches**.

Wenn in einem Abstraktions-Schritt mehr als **50% der neuen Matches False
Positives** sind → **revert die Abstraktion**.

Versuche stattdessen:
- Andere Abstraktions-Achse
- Tiefere Analyse mit Semgrep Taint-Mode oder CodeQL Dataflow-Query
- Kleinere Abstraktions-Schritte

**Dokumentiere die Grenze:** "Pattern X erzeugt 60% FPs, nicht
verallgemeinert."

### Schritt 5: Triage & Report

Fuer jeden gefundenen Match:

| Feld | Inhalt |
|------|--------|
| **Location** | file:line |
| **Confidence** | HIGH / MEDIUM / LOW |
| **Exploitability** | trivial / moderate / conditional / theoretical |
| **Priority** | Impact × Exploitability |
| **Same-Pattern-Fingerprint** | Hash der Abstraktions-Stufe, die es fand |

Priorisierung:
- `HIGH` + `trivial` → gleich behandeln wie die Original-Vuln
- `MEDIUM` + `conditional` → Tracking-Issue
- `LOW` + `theoretical` → nicht tracken, aber dokumentieren fuer
  Pattern-Lernen

---

## Tool-Wahl (optional)

breachguard ist toolagnostisch (arbeitet mit Lens-Referenzen und Code).
Aber wenn der User Tool-Support hat, kann das Output der Variant-Analyse
als Rule-Skeleton exportiert werden (siehe Detector-Rolle in `roles.md`):

| Szenario | Tool | Warum |
|----------|------|-------|
| Quick-Scan, grobes Pattern | `ripgrep` | Schnell, kein Setup |
| Einfaches Pattern-Match | `semgrep` | Einfach, keine Build-Config |
| Dataflow-Tracking | `semgrep` (taint) oder `codeql` | Verfolgt Werte ueber Funktionen |
| Interprocedural-Analyse | `codeql` | Cross-Function-Dataflow |
| Nicht-buildbarer Code | `semgrep` | Arbeitet auf Quellcode |

Die Detector-Rolle kann am Ende einer Variant-Analyse eine **Semgrep-Rule**
oder **CodeQL-Query** als Skeleton generieren, die das finale abstrahierte
Pattern encoded. Dann landet das Pattern in der CI.

---

## Beispiel-Durchlauf

**Bekannte Vuln:** SQL-Injection in `routes/users.js:4`
```js
db.query("SELECT * FROM users WHERE id=" + req.params.id)
```

**Schritt 1 — Root-Cause:**
- Sink: `db.query` mit String-Argument (nicht parameterized)
- Source: `req.params.*` (URL-Parameter, untrusted)
- Missing: Parameterization via `?`-Platzhalter + Params-Array

**Schritt 2 — Exact-Match:**
```
db\.query\(".*" \+ req\.params\.id
```
→ 1 Match (die bekannte Vuln). Anker gesetzt.

**Schritt 3 — Abstraktions-Iteration:**

**Iter 1** (Source-Abstraktion): `req.params.id` → `req.params.*`
```
db\.query\(".*" \+ req\.params\.
```
→ 3 neue Matches in `orders.js`, `items.js`. Klassifikation:
- `orders.js:12` → TRUE (gleiches Muster mit `req.params.orderId`)
- `orders.js:28` → TRUE
- `items.js:45` → FALSE (ist eine LIKE-Query mit Escaping in Zeile 43)
FP-Rate: 33% → OK, Abstraktion behalten.

**Iter 2** (Source-Erweiterung): `req.params.*` → `req.*`
```
db\.query\(".*" \+ req\.
```
→ 12 neue Matches. Klassifikation:
- 4 TRUE (Body-Parameter)
- 3 TRUE (Query-Parameter)
- 5 FALSE (`req.headers.*` geht durch Sanitizer `sanitizeHeader()`)
FP-Rate: 42% → OK, Abstraktion behalten.

**Iter 3** (Sink-Abstraktion): `db.query` → `*.query`
```
\w+\.query\(".*" \+ req\.
```
→ 40 neue Matches. Klassifikation:
- 8 TRUE
- 32 FALSE (die meisten sind jQuery-Selectors, String-Methoden, Logger)
FP-Rate: 80% → **REVERT**. Abstraktion zu breit.

**Stopp bei Iter 2.** Resultat: 9 TRUE Findings + 1 Original = 10 Stellen
zu fixen.

**Schritt 5 — Report:**
```markdown
# breachguard Variant Analysis Report

**Original Vuln:** SQL Injection in routes/users.js:4
**Root Cause:** String-Konkat in db.query mit req.*-Source ohne
Parameterization.

## Geschwister-Findings (9 Stellen)

### [CRITICAL] SQL Injection — orders.js:12
Confidence: HIGH | Exploitability: trivial | Same-Pattern
[...]

### [CRITICAL] SQL Injection — orders.js:28
...

## Abstraktions-Grenzen
Pattern `\w+\.query + req.` erzeugt 80% FPs (meist jQuery/String-
Methoden), nicht verallgemeinert.

## Detection-Rule (fuer CI)
[siehe Detector-Rolle → Semgrep-Skeleton angehaengt]
```

---

## Variant-Mode vs Deep-Audit-Mode

| Aspekt | `variant` | `deep-audit` |
|--------|-----------|--------------|
| **Input** | Eine konkrete Vuln | Code/Repo ohne bekannte Vuln |
| **Ziel** | Geschwister finden | Alle relevanten Vulns finden |
| **Methode** | Pattern-Abstraktion, 50%-Stop | 5-Phasen-Workflow |
| **Dauer** | 30min - 2h | 2h - 6h |
| **Output** | Varianten-Liste + Rule-Skeleton | Voller Audit-Report |

Wenn der User `variant`-Modus triggert ohne konkrete Vuln zu nennen:
Anfrage nach der Original-Finding. Ohne Anker ist Variant-Analysis nicht
anwendbar.

---

## Integration mit Detector-Rolle

Die **Detector-Rolle** (`roles.md`) ergaenzt Variant-Analysis perfekt:

Am Ende des Variant-Runs wird das finale, validierte Pattern als
wiederverwendbare Detection-Rule exportiert:

**Semgrep-Skeleton-Beispiel:**
```yaml
rules:
  - id: sql-injection-concat-req
    message: "Potential SQL Injection — String-Konkat mit req-Source"
    severity: ERROR
    languages: [javascript, typescript]
    pattern-either:
      - pattern: $DB.query("..." + req.$X)
      - pattern: $DB.query(`...${req.$X}...`)
    metadata:
      cwe: CWE-89
      owasp: A03:2021
      source: breachguard-variant-analysis
```

Damit geht der Audit nicht nur als einmaliger Report raus, sondern als
permanenter Detektor in die CI.
