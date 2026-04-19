# FP-Verifikation — Gate Reviews

Die groesste Lernkurve aus Trail-of-Bits-Skills: **Jedes Security-Finding
durchlaeuft sechs Gate Reviews**, bevor es im finalen Report erscheint.
Verdict am Ende: `TRUE_POSITIVE`, `FALSE_POSITIVE` oder `UNCERTAIN`.

Diese Datei ist originaere breachguard-Inhalt, inspiriert von fp-check.

## Warum Gates?

LLM-basierte Security-Audits haben notorisch hohe FP-Raten. Ohne Gate-
Verifikation landen viele Patterns als Finding, die in Wirklichkeit durch
Validierung, Framework-Schutz, Memory-Safety oder fehlende Attacker-Control
gemildert sind. Ohne Gate-Filter ist der Audit-Output Muell.

## Die Regel "Fail Closed bei Unklarheit"

Wenn ein Gate nicht entscheidbar ist mit dem verfuegbaren Kontext →
Verdict `UNCERTAIN`. Nicht raten. Nicht zu TRUE hochstufen "um sicher zu
gehen". Nicht zu FALSE runterstufen um Noise zu vermeiden.

UNCERTAIN ist ein vollwertiges Verdikt — **es wird im Report separat
ausgewiesen** und triggert eine explizite User-Entscheidung ("soll ich
tiefer pruefen?").

---

## Die 6 Gate Reviews

Jedes Raw Finding aus Phase 3 durchlaeuft diese Gates in Reihenfolge.
Fehler bei einem Gate → `FALSE_POSITIVE`. Unklarheit bei einem Gate →
`UNCERTAIN`. Alle 6 Gates bestanden → `TRUE_POSITIVE`.

### Gate 1: Sink Gate

**Frage:** Ist die als "vulnerable" identifizierte Code-Stelle wirklich
der relevante Sink fuer diese Bug-Class?

Der **Sink** ist die Operation, die die Schwachstelle konkret ausloest —
der `memcpy`, die SQL-Query, der Template-Render, der Shell-Exec.

**Gate bestanden, wenn:**
- Code-Stelle ist nachweislich der Sink-Typ (z.B. wirklich ein
  `child_process.exec` mit Shell-Option, nicht `execFile` ohne Shell)
- Die Bug-Class passt zur Sink-Semantik (SQL-Injection-Verdacht und der
  Sink ist wirklich eine SQL-Execution)

**Gate gescheitert (→ FALSE_POSITIVE), wenn:**
- Der vermutete Sink ist eigentlich eine sichere API-Variante (z.B.
  `prisma.$queryRaw` mit Template-Tag → parametrisiert automatisch)
- Code-Stelle ist gar kein Sink sondern eine Log-Ausgabe mit String-Format

**Gate UNCERTAIN, wenn:**
- API ist framework-spezifisch und Konfiguration/Version ist nicht
  sichtbar

### Gate 2: Source Gate

**Frage:** Ist die Datenquelle wirklich untrusted?

**Gate bestanden, wenn:**
- Quelle ist: Request-Parameter, Header, Cookie, Body, File-Upload,
  WebSocket-Message, Umgebungsvariable in Multi-Tenant-Setup,
  Webhook-Payload, DB-Feld das von Nutzern gesetzt wird, S3-Object mit
  Dritter-Upload-Recht, Message-Queue mit externem Producer

**Gate gescheitert (→ FALSE_POSITIVE), wenn:**
- Quelle ist ein konstanter Wert im Code
- Quelle ist Umgebungsvariable in Single-Tenant-Setup ohne User-Einfluss
- Quelle ist ein Operator-kontrollierter Admin-Feed

**Gate UNCERTAIN, wenn:**
- Quelle ist DB-Feld unklarer Herkunft (Second-Order-Injection-Verdacht)
- Header-Wert in Reverse-Proxy-Setup, Middleware-Filterung unklar

### Gate 3: Reachability Gate

**Frage:** Kann die Source den Sink tatsaechlich erreichen?

Source → Transformer → Sink. Der Pfad muss real existieren.

**Gate bestanden, wenn:**
- Datenfluss ist nachvollziehbar
- Transformationen unterwegs sind neutral (einfache String-Konkatenation,
  Parsing ohne Sanitization)

**Gate gescheitert (→ FALSE_POSITIVE), wenn:**
- Die Source wird gar nicht an den Sink weitergereicht (Sink nutzt einen
  anderen konstanten/internen Wert)
- Dead Code — der Code-Pfad ist nicht erreichbar

**Gate UNCERTAIN, wenn:**
- Weiterreichung laeuft durch Framework-Magic (Middleware, Interceptor,
  Decorator) dessen Verhalten nicht im Code sichtbar ist
- Async/Event-Pfade — nicht klar ob Handler wirklich aktiv ist

### Gate 4: Validation Gate

**Frage:** Greift vor dem Sink eine Validierung/Escaping/Neutralisation,
die den Angriff entschaerft?

**Gate bestanden (Finding bleibt), wenn:**
- Keine Validierung auf dem Pfad, oder
- Validierung ist nachweislich unvollstaendig (Blocklist statt Allowlist,
  nur `<script>` gefiltert aber nicht `<img onerror=...>`, etc.)

**Gate gescheitert (→ FALSE_POSITIVE), wenn:**
- Middleware wendet wirksame Schema-Validierung an (Zod/Joi/Pydantic/
  class-validator) mit passendem Schema
- Escaping-Funktion ist korrekt fuer die Sink-Semantik (HTML-Escaping fuer
  HTML-Sinks, URL-Encoding fuer URL-Sinks, Shell-Escaping fuer Shell-
  Sinks)
- Prepared-Statement/Parameterized-Query wird genutzt
- Framework macht das automatisch (z.B. Django ORM)

**Gate UNCERTAIN, wenn:**
- Validator ist in Datei ausserhalb des Scopes
- Validator-Regex ist lang/komplex und nicht klar bewertbar
- Middleware-Reihenfolge unklar (z.B. ist der Sanitize-Middleware wirklich
  VOR dem Handler registriert?)

### Gate 5: Attacker-Control Gate

**Frage:** Kann der Attacker den Input so setzen, dass der Exploit funktioniert?

**Gate bestanden, wenn:**
- Full-Control: Attacker kann den Wert beliebig setzen
- Partial-Control: Attacker kann innerhalb von Constraints setzen, und
  der Exploit passt in die Constraints (z.B. Username-Feld mit Max-Length
  50 — SQL-Injection-Payload in 50 Chars moeglich)

**Gate gescheitert (→ FALSE_POSITIVE), wenn:**
- Field-Type beschraenkt (boolean, enum, numeric, UUID) und Exploit passt
  nicht rein
- Length-Constraint ist so strikt, dass keine Payload funktioniert
- Attacker hat keinen Zugriff auf die Quelle (z.B. nur-Admin-Endpoint
  und Attacker-Model ist "unauthenticated remote")

**Gate UNCERTAIN, wenn:**
- Feld ist String-Type aber Length-Constraint wird im DB-Schema und nicht
  im App-Code gesetzt, Schema nicht sichtbar
- Attacker-Zugang unklar (z.B. Endpoint hinter OAuth, aber OAuth-Config
  nicht im Scope)

### Gate 6: Impact Gate

**Frage:** Ist der Impact real oder nur theoretisch/kosmetisch?

**Gate bestanden (Finding bleibt), wenn:**
- Ausnutzung fuehrt zu: Unauthorized Access, Data Exfiltration, RCE,
  Privilege Escalation, Denial of Service, Data Corruption, Session
  Hijack, Financial Loss, PII-Leak

**Gate gescheitert (→ FALSE_POSITIVE), wenn:**
- Bug existiert nur in Dev-Code-Pfad der nie in Prod laeuft
- Exploit erfordert Preconditions, die unrealistisch sind
- "Defense in Depth"-Bypass ohne Primaerschutz-Bruch (z.B. missing
  Security-Header auf Endpoint der zusaetzlich Auth-Check hat und
  Auth-Check ist funktional — Header ist Defense-in-Depth, nicht
  Primaerschutz)

**Gate UNCERTAIN, wenn:**
- Unklar ob Code in Prod aktiv ist (Feature-Flag, Dead-Code-Vermutung)
- Impact-Klasse nicht klar ohne Laufzeit-Test

---

## Integration mit Bug-Class-Verifikation

Pro Bug-Klasse gibt es zusaetzlich spezifische Checks. Siehe
`bug-class-verification.md`:

- **Memory Corruption** → Memory-Safe-Subset-Check (safe Rust, Go ohne
  unsafe.Pointer, managed Langs ohne FFI)
- **Logic Bug** → Spec-Existenz-Check
- **Race Condition** → Feasibility-Check (wie eng ist das Window?)
- **SQL Injection** → Parameterization-Check + ORM-Escape-Hatch-Check
- **XSS** → Context-Check (HTML/Attribute/JS/URL)
- **Auth Bypass** → Check-Reihenfolge-Check
- **Crypto Misuse** → Primitive-Passend-Check + Key-Management-Check
- **Path Traversal** → Canonicalization-Check
- **SSRF** → Allow-List-Check fuer Ziel-Hosts
- **Deserialization** → Type-Whitelist-Check

Diese Bug-Class-Checks laufen **zusaetzlich** zu den 6 Gates, nicht statt.

---

## Verdict-Zusammenstellung

Aus den 6 Gates (und Bug-Class-Checks) ergibt sich das Final-Verdict:

| Gate-Ergebnisse | Bug-Class-Check | Verdict |
|-----------------|-----------------|---------|
| Alle 6 bestanden | bestanden | `TRUE_POSITIVE` |
| Mindestens eines gescheitert | egal | `FALSE_POSITIVE` |
| Eines oder mehrere UNCERTAIN (Rest bestanden) | bestanden | `UNCERTAIN` |
| Eines oder mehrere UNCERTAIN (Rest bestanden) | UNCERTAIN | `UNCERTAIN` |
| Alle 6 bestanden | UNCERTAIN | `UNCERTAIN` |

### UNCERTAIN darf nicht zu 100% werden

Wenn am Ende eines Audits >40% der Findings `UNCERTAIN` sind → das ist ein
Scope-Problem, kein Audit-Ergebnis. Explizit an den User zurueckmelden:

> "Scope zu eng: Von 18 Findings konnten 9 nicht verifiziert werden weil
> Validierungs-Code ausserhalb des bereitgestellten Scopes liegt. Empfehle
> Scope-Erweiterung um: /src/validators/, /src/middleware/, Framework-
> Config."

---

## Dokumentations-Pflicht pro Finding

Im Report muss pro Finding der Gate-Trace dokumentiert sein:

**Auditor-Rolle (Markdown):**
```markdown
### [CRITICAL / HIGH confidence / TRUE_POSITIVE] SQL Injection via route param
**Lens:** `injection`
**File:** `routes/users.js:4`
**Bug-Class:** SQL Injection
**Evidence-Level:** DIRECT

**Gates:**
- Sink: PASS (`db.query` mit String-Konkat)
- Source: PASS (`req.params.id`)
- Reachability: PASS (direkter Fluss Source → Sink)
- Validation: PASS (keine)
- Attacker-Control: PASS (URL-Param, full control)
- Impact: PASS (SELECT Dump ueber UNION moeglich)
- Bug-Class (Parameterization): PASS (String-Konkat statt Parameter)
**Verdict: TRUE_POSITIVE**

**Problem:** [...]
**Fix:** [...]
```

**Agent-Rolle (YAML):** siehe `agent-output.md`, Felder `gates`,
`bug_class`, `verdict`, `confidence`, `evidence_level`.

---

## Triage-Modus: Gates skippen

Im Modus `triage` wird die FP-Verifikations-Phase **uebersprungen**. Output
sind dann Raw Findings ohne Verdict, mit dem Warning aus
`methodology.md` §3.3.

Das ist erlaubt und gewollt fuer Quick-Scans — aber **im Report explizit
markieren**, damit User nicht unverifizierte Findings fuer verifizierte
haelt.

---

## Kollisionen mit Auto-Ticket-Creation

Wenn am Ende `gh issue create`-Kommandos zum Pasten angeboten werden:

- `TRUE_POSITIVE`-Findings → vorgeschlagen als Issue
- `UNCERTAIN`-Findings → vorgeschlagen als Issue **mit Label `uncertain`**
  und Hinweis im Body dass manuelle Verifikation noetig ist
- `FALSE_POSITIVE`-Findings → NICHT als Issue vorgeschlagen

Damit endet die FP-Verifikations-Disziplin nicht mit dem Report — sie
verhindert auch FP-verseuchte Tracker.
