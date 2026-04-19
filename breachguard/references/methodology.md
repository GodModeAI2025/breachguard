# Audit-Methodologie — 5-Phasen-Workflow

Der Kern-Unterschied zwischen breachguard und einem reinen Lens-Katalog:
**strukturierter Phasen-Workflow** fuer Security-Audits, nicht nur
"Lens anwenden → Finding".

Inspiriert von Trail-of-Bits-Patterns (differential-review,
audit-context-building, fp-check), Text originaer breachguard.

## Wann welcher Workflow?

| Modus | Phasen | Dauer | Anwendung |
|-------|--------|-------|-----------|
| `triage` | 1 → 2 → 3 | <15 min | Quick-Scan, eventuell mehr FPs |
| `audit` (Default) | 1 → 2 → 3 → 4 → 5 | 30-90 min | Standard-Review |
| `deep-audit` | 1 → 2 → 3 → 4 → 5 (mit Attacker-Modell pro Finding) | 2-6h | Fuer Prod-Releases |
| `variant` | Eigener Workflow | variabel | Siehe `variant-analysis.md` |
| `verify-fix` | 1 → Fix-Delta → 4 → 5 | 15-45 min | Nach Finding-Remediation |

## Kern-Prinzipien (gelten in allen Phasen)

Diese 6 Regeln sind nicht verhandelbar:

### 1. Untrusted bis Beweis des Gegenteils
Jede Eingabe ausserhalb der Trust-Boundary ist adversarial, bis bewiesen ist,
dass sie validiert wird. Das gilt fuer: Request-Parameter, Headers, Cookies,
File-Uploads, WebSocket-Messages, Datenbankwerte die von Usern kommen,
Umgebungsvariablen in Container-Setups mit Mandanten-Mix, Webhook-Payloads,
Nachrichten aus Message-Queues, Inhalte aus S3/Object-Storage mit Upload-
Rechten fuer Dritte.

**Praxis:** Wenn Daten aus einer dieser Quellen an einen Sink (Query,
Shell, Template, Filesystem, Deserializer) gelangen — Finding produzieren,
bis Validierung nachgewiesen ist.

### 2. Fail Closed bei Unklarheit
Wenn nicht entscheidbar ist, ob eine Validierung greift → `UNCERTAIN`
markieren. **Nicht** raten. Nicht zu `TRUE_POSITIVE` hochstufen um
"sicher zu gehen", nicht zu `FALSE_POSITIVE` runterstufen um Noise zu
reduzieren. `UNCERTAIN` ist ein vollwertiges Verdikt.

**Typische Unklarheits-Quellen:**
- Validator-Funktion in Datei, die nicht im Scope ist
- Framework-Konfiguration nicht sichtbar (Middleware-Order unklar)
- Laufzeit-Feature-Flag entscheidet ueber Pfad
- Schema-Validierung deklariert aber nicht sichtbar angewendet

### 3. Coverage-Limits explizit machen
Am Anfang jedes Reports: **Was wurde geprueft, was nicht.**

Beispiel:
```
Scope: /src/auth/ (14 Files), /src/api/v2/ (28 Files)
Out of scope: /src/legacy/, /src/vendor/, Frontend-Teil
Nicht-triviale Limitations: Pipeline-Config (.github/workflows/) nicht
geprueft, DB-Migrations nicht analysiert.
```

Keine impliziten "wir haben alles gecheckt"-Claims.

### 4. Confidence pro Finding
Jedes Finding traegt zwei Dimensionen:

**Confidence** (wie sicher ist die Finding-Behauptung?):
- `HIGH` — Direkter Code-Beleg sichtbar, Datenfluss rekonstruiert,
  Attacker-Control plausibel.
- `MEDIUM` — Pattern erkannt, Umfeld wahrscheinlich anfaellig, aber
  einzelne Annahmen nicht final verifiziert.
- `LOW` — Heuristik/Indiz, kann Finding sein, kann FP sein.

**Evidence Level** (worauf basiert die Confidence?):
- `DIRECT` — Code-Zeile beobachtet, Datenfluss gelesen.
- `INFERENCE` — Aus Konventionen/Architektur geschlossen.
- `HEURISTIC` — Muster-Match ohne Tiefen-Analyse.

Beide Felder sind pflicht im `breachguard.finding`-Schema (siehe
`agent-output.md`).

### 5. Keine Exploit-Regurgitation
Probleme **beschreiben**, nicht step-by-step ausnutzen. Ein SQL-Injection-
Finding enthaelt:
- Ort, Sink, Source, Fehlende Validierung, Impact-Klasse.

Es enthaelt **nicht**:
- Ausgearbeitetes Payload-Script, Exfiltrations-Kette, Tool-Anleitung.

Ausnahmen: Kurze PoC-Strings, die den Umfang der Kontrolle illustrieren
(`/users/1 OR 1=1--` → "Zeilen-Exfiltration moeglich"). Keine fertige
Exploit-Chain.

### 6. Pit of Success
Ein API-Design, bei dem die sichere Nutzung schwerer ist als die unsichere,
ist selbst der Defekt — nicht "der User hat falsch benutzt". Wenn Entwickler
Dokumentation lesen oder Spezialregeln erinnern muessen um Vulns zu
vermeiden, hat die API versagt.

**Diagnostisch:**
- Default-Pfad fuehrt zu sicherem Verhalten? → gut
- Default-Pfad fuehrt zu unsicherem Verhalten? → Sharp Edge (`sharp-edges`-Lens)
- Sicher und unsicher sehen identisch aus? → Sharp Edge

Siehe Lens `sharp-edges.md` fuer Detail-Patterns.

---

## Phase 1: Intake & Triage

**Ziel:** Scope klaeren, Attacker-Model setzen, Critical Paths identifizieren.

### 1.1 Scope einsammeln

Fuer jeden Audit pflicht:
- **Artefakt-Typ:** Snippet, Datei, Ordner, ganzes Repo?
- **Zweck des Codes:** Webapp-Handler? Backend-Job? Library? CLI-Tool?
- **Umgebung:** Prod-geplant? Intern-only? Oeffentlich erreichbar?
- **Known-Constraints:** Compliance-Anforderung (GDPR/NIS2/...)? Regulierter
  Kontext (Finanz, Gesundheit, Energie)?

Wenn User nichts sagt: einmal fragen. Nie raten.

### 1.2 Attacker-Model setzen

Aus `attacker-models.md` das passende WHO/ACCESS/INTERFACE waehlen:

- **WHO** — unauthenticated remote / authenticated user / privileged user /
  privileged insider / compromised-dependency / colocated-tenant
- **ACCESS** — network-only / valid-account / admin-account / physical /
  supply-chain
- **INTERFACE** — public API / internal API / CLI / file / queue / direct DB

Beispiel: Webhook-Handler → `unauthenticated remote / network-only / public API`.

### 1.3 Critical Paths identifizieren

Nicht jeder Code ist gleich kritisch. Fuer Security-Audits diese Hot-Zones
priorisieren:

| Hot-Zone | Indikatoren im Code |
|----------|---------------------|
| **Auth-Pfade** | `login`, `signin`, `session`, `token`, `jwt`, `password`, `oauth` |
| **Authorization-Checks** | `is_admin`, `role`, `permission`, `can_`, `authorize_` |
| **Crypto-Operationen** | `encrypt`, `decrypt`, `sign`, `verify`, `hash`, `hmac`, `rsa`, `aes` |
| **Value-Transfer** | Zahlungen, Token-Mints, Credit-Anpassungen, Quotas |
| **Externe Calls** | HTTP-Client, Subprocess, eval, Deserialization, Shell |
| **Datenbank-Writes** | ORM-Writes, Raw-SQL, Batch-Imports |
| **Datei-I/O mit User-Input** | Upload-Handler, Template-Loader, Path-Construction |
| **Admin-Endpoints** | `/admin`, `/internal`, Debug-Routen |

**Output Phase 1:** Scope-Liste, Attacker-Model, Hot-Zones-Liste.

---

## Phase 2: Context Building

**Ziel:** Das System verstehen, bevor nach Vulns gesucht wird.

Ohne Kontext produziert man Musterfehler und verfehlt echte Schwachstellen.
Inspiriert von audit-context-building: Gist-Level-Verstaendnis ist nicht
genug, und externer Kontext ist adversarial bis bewiesen.

### 2.1 Entrypoints kartieren

Wo beginnt untrusted data ihre Reise?

- **HTTP-Routes** (Express, Flask, FastAPI, Spring, Rails-Routes, ...)
- **WebSocket-Handler**
- **Message-Queue-Consumer** (Kafka, RabbitMQ, SQS, Redis Pub/Sub)
- **Cron/Scheduled-Jobs** (die von User-Daten getriggert werden)
- **Webhook-Receiver**
- **CLI-Argumente** bei Tools
- **File-Watchers** (Ordner mit Upload-Rechten Dritter)
- **gRPC/GraphQL-Resolver**

### 2.2 Trust-Boundaries identifizieren

Wo kreuzt Data eine Vertrauensgrenze?

- Network → Application
- Application → Database
- Application → Subprocess/Shell
- Container → Host
- Tenant A → Tenant B (Multi-Tenancy)
- Public-Net → Admin-Net

Jede Trust-Boundary ist ein Validation-Anchor. Fehlende Validierung dort
erzeugt Findings.

### 2.3 Actors & Storage

- **Actors:** Wer kann agieren? (anonymous users, auth users, admins,
  service-accounts, colocated-tenants, operators)
- **Storage:** Wo landen sensitive Daten? (DBs mit PII, Caches, Logs,
  externe Services wie S3, File-Uploads)

### 2.4 Workflow-Mapping fuer Critical Paths

Fuer die Top-3-Hot-Zones aus Phase 1: kurzen Datenfluss skizzieren.

Beispiel (Auth-Login):
```
POST /login
  → parseBody(req)
  → rateLimiter.check(req.ip)      ← ist das da?
  → validateCredentials(email, pwd)
  → loadUser(email) → DB
  → bcrypt.compare(pwd, user.hash)  ← timing-safe?
  → session.create(user.id)         ← cookie-attrs?
  → res.cookie('sid', session.id)
```

**Output Phase 2:** Entrypoint-Liste, Trust-Boundary-Map, Actor-Liste,
Storage-Liste, 3-5 Critical-Path-Skizzen.

---

## Phase 3: Lens Application

**Ziel:** Security-Lenses (und ggf. benachbarte Domaenen) auf die Critical
Paths anwenden, Raw Findings produzieren.

### 3.1 Lens-Auswahl

Basis: Alle 11 klassischen Security-Lenses aus `lenses/security.md` plus
die 3 neuen Lenses:
- `insecure-defaults` (Fail-Open, Fallback-Secret, Default-Creds)
- `sharp-edges` (Footgun-APIs, Pit-of-Failure)
- `supply-chain` (Dep-Threat-Landscape)

Je nach Hot-Zone sind zusaetzlich nicht-Security-Lenses wertvoll:

| Hot-Zone | Zusatz-Lenses |
|----------|---------------|
| Auth-Pfade | `timeout-retry` (Error-Handling), `rate-abuse` (Security) |
| Crypto | `error-swallowing` (Error-Handling) |
| Externe Calls | `timeout-retry`, `error-boundaries` (Error-Handling) |
| Concurrency-heavy | `concurrency`-Domain komplett |
| API-Endpoints | `request-validation`, `rest-conventions` (API-Design) |
| DB-Writes | `transaction-safety`, `query-safety` (Database) |

### 3.2 Raw Findings

Fuer jede Lens: das Artefakt durchgehen, Beobachtungen notieren.
**Noch keine FP-Filterung** — die kommt in Phase 4.

Ein Raw Finding enthaelt minimal:
- Titel
- Lens-ID
- Ort (file:line)
- Beobachtung (was genau ist der Verdacht?)
- Confidence-Estimate (HIGH/MEDIUM/LOW)

### 3.3 Stop-Conditions fuer triage-Modus

Im `triage`-Modus endet die Phase hier. Output sind Raw Findings mit
Confidence, aber ohne Gate-Review. Entsprechend explizit markieren:

> **WARNUNG:** Triage-Modus — Findings sind nicht FP-verifiziert. Erwarte
> 20-40% False Positives. Fuer Prod-Review `deep-audit`-Modus nutzen.

**Output Phase 3:** Raw-Findings-Liste (vor FP-Gate).

---

## Phase 4: FP Verification (Gate Reviews)

**Ziel:** Jedes Raw Finding durch 6 Gate Reviews fuehren. Verdict: TRUE /
FALSE / UNCERTAIN.

Siehe **`fp-verification.md`** fuer die kompletten Gates und
**`bug-class-verification.md`** fuer bug-class-spezifische Rubriken.

**Kurzfassung der 6 Gates:**

1. **Sink Gate** — Ist der alleged-vulnerable-Call wirklich der Sink?
2. **Source Gate** — Ist die Source wirklich untrusted?
3. **Reachability Gate** — Erreicht Source den Sink?
4. **Validation Gate** — Greift eine Validierung vorher?
5. **Attacker-Control Gate** — Kann Attacker den Wert tatsaechlich setzen?
6. **Impact Gate** — Ist der Impact real oder nur kosmetisch (Defense-in-
   Depth-Bypass ohne Primaerschutz-Bruch → oft FALSE_POSITIVE)?

**Output Phase 4:** Findings-Liste mit Verdict (TRUE/FALSE/UNCERTAIN).
FALSE werden nicht gereportet, UNCERTAIN mit expliziter Markierung, TRUE
gehen in Phase 5.

---

## Phase 5: Report

**Ziel:** Strukturierter Output passend zur Rolle.

Siehe `roles.md` fuer Output-Templates pro Rolle und `agent-output.md` fuer
YAML-Schemas.

**Pflichtbestandteile fuer Security-Modi:**

- **Scope-Block** am Anfang (was geprueft, was nicht)
- **Attacker-Model-Block** am Anfang (aus Phase 1)
- **Finding-Liste** mit Severity + Confidence + Verdict + Evidence-Level
- **UNCERTAIN-Sektion** separat (nicht mit TRUE-Findings mischen)
- **Coverage-Limits-Block** am Ende (was nicht getestet wurde)

Beispiel-Kopfzeile eines Reports:

```markdown
# breachguard Security-Audit

**Scope:** /src/auth/ + /src/api/payments/
**Attacker-Model:** unauthenticated remote + network-only + public API
**Out-of-scope:** /src/legacy/, Frontend-Code, Pipeline-Config
**Modus:** deep-audit
**Datum:** 2026-04-19

## Zusammenfassung
- 3 CRITICAL (TRUE), 2 HIGH (TRUE), 1 HIGH (UNCERTAIN), 2 MEDIUM
- Release-Gate: BLOCK

## Findings
[...]

## UNCERTAIN
- F-006 (bcrypt-Compare ohne sichtbarer Timing-Safe-Flag)

## Coverage-Limits
- Middleware-Order nicht verifiziert
- Rate-Limit-Konfig nicht im Repo gefunden
```

---

## Phasen-Eskalation

Wenn waehrend des Workflows klar wird, dass der gewaehlte Modus zu flach
ist, Eskalation anbieten:

- Raw-Findings in Phase 3 zeigen komplexe Flows → Eskalation von `audit`
  auf `deep-audit`
- Ein Finding erfordert DB-Zugriff zum Verifizieren → Eskalation zu
  CLI-Fallback
- Repo >50 Files → sofort CLI-Fallback (siehe SKILL.md)

**Nie heimlich eskalieren** — immer sichtbar anbieten, User entscheidet.
