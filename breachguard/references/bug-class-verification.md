# Bug-Class-Verifikation

Zusaetzlich zu den 6 generischen Gates aus `fp-verification.md` hat jede
Bug-Klasse spezifische Verifikations-Kriterien. Diese laufen parallel, nicht
statt der Gates.

Inspiriert von fp-check/bug-class-verification, originaerer breachguard-Text.

## Klassen im Ueberblick

| Bug-Class | Wesen | Typische Lens |
|-----------|-------|---------------|
| SQL Injection | Untrusted Input in SQL-Query ohne Parametrisierung | `injection` |
| NoSQL Injection | Untrusted Input in Mongo-/etc.-Query-Objekten | `injection` |
| Command Injection | Untrusted Input in Shell/Subprocess | `injection` |
| XSS (Reflected/Stored/DOM) | Untrusted Input in HTML/JS-Kontext | `xss-csrf` |
| CSRF | State-Change ohne Origin-Verifikation | `xss-csrf` |
| Auth Bypass | Authentifikation umgehbar | `auth-session` |
| Session Fixation/Hijack | Session-Token angreifbar | `auth-session` |
| IDOR / Authorization Failure | Zugriff ohne Ownership/Role-Check | `authorization` |
| Path Traversal | Untrusted Input in Filesystem-Pfad | `input-sanitization` |
| SSRF | Untrusted URL in Server-seitigem HTTP-Call | `input-sanitization` |
| Deserialization | Untrusted Serialized Data deserialisiert | `input-sanitization` |
| XXE | XML mit externen Entities | `input-sanitization` |
| Crypto Misuse | Falsche Primitive, Key-Handling, IV-Reuse | `cryptography` |
| Memory Corruption | Buffer-Overflow, UAF, Double-Free | (nicht Haupt-Web-Fokus) |
| Race Condition | TOCTOU, Shared-State ohne Lock | `concurrency` (RepoLens) |
| Logic Bug | Business-Regel verletzt | (applikations-spezifisch) |
| Secret Exposure | Hardcoded/Logged/Leaked Secret | `secrets` |
| Insecure Defaults | Fallback-Secret, Default-Cred, Fail-Open | `insecure-defaults` (NEU) |
| Sharp Edge | Misuse-prone API, Pit-of-Failure | `sharp-edges` (NEU) |
| Supply-Chain | Dep-Risiko ueber CVE hinaus | `supply-chain` (NEU) |

---

## SQL Injection

**Zusatz-Checks:**

1. **Parameterization-Check:** Ist die Query via Prepared-Statement /
   Parameterized-Query / ORM-Safe-API gebaut?
   - `db.query(sql, [params])` mit echten Parametern → OK
   - `db.query(sql + userInput)` → NICHT OK, TRUE
   - `db.query(${sql})` mit Template-String + konkatenierten User-Vars →
     NICHT OK, TRUE

2. **ORM-Escape-Hatch-Check:** Wird eine "Raw"-API des ORMs mit User-Input
   genutzt?
   - `Model.findBySql(rawSql)`, `prisma.$queryRawUnsafe`, `sequelize.query`
     mit Replacement-Mode `raw` → suspekt
   - `prisma.$queryRaw\`SELECT ... WHERE id = ${id}\`` (Template-Tag) → OK,
     parametrisiert automatisch

3. **Second-Order-Check:** Wird ein Wert aus der DB gelesen, irgendwo
   konkateniert, und wieder als SQL ausgefuehrt?
   - Beispiel: User setzt `username = "admin'--"`, spaeter wird
     `SELECT * FROM logs WHERE user = '${username}'` aus DB-Wert gebaut.
   - Trigger: Second-Order-Injection, oft uebersehen.

4. **Stored-Procedure-Check:** Wird eine Stored Procedure mit
   konkateniertem SQL aufgerufen?

**Bug-Class FAIL → FALSE_POSITIVE** bei:
- Framework macht Parametrisierung automatisch (Django-ORM-`.filter()`,
  ActiveRecord-Basics, Prisma-Non-Raw)

---

## NoSQL Injection

**Zusatz-Checks:**

1. **Query-Operator-Injection:** Akzeptiert der Endpoint JSON-Body, der
   direkt in die Query geht?
   - `db.users.find(req.body.filter)` → Attacker kann `{$ne: null}` senden,
     Auth umgehen
2. **Raw-Where-Clause-Check:** MongoDB `$where` mit User-String =
   JavaScript-Execution.

---

## Command Injection

**Zusatz-Checks:**

1. **Shell-Option-Check:** `child_process.exec` (nutzt Shell) vs
   `execFile`/`spawn` ohne Shell.
   - `exec(cmd)` + User-Input → TRUE
   - `spawn("ls", [userDir])` → OK (keine Shell-Interpretation)
2. **Unsafe-shell-Flag:** `subprocess.Popen(cmd, shell=True)` → immer
   suspekt.
3. **Backtick-Execution:** Ruby/Perl/Shell-Backticks mit User-String.

---

## XSS

**Context-Check (kritisch):**

1. **HTML-Context:** `<div>{{userInput}}</div>` — HTML-Escape noetig
   (`<` → `&lt;`).
2. **Attribute-Context:** `<div class="{{userInput}}">` — zusaetzlich
   Quote-Escape noetig.
3. **JavaScript-Context:** `<script>let x = "{{userInput}}";</script>` —
   JS-String-Escape, nicht HTML-Escape.
4. **URL-Context:** `<a href="{{userInput}}">` — URL-Validation + scheme-
   check (`javascript:`-Scheme blocken).
5. **CSS-Context:** `<style>body { bg: {{userInput}} }</style>` — CSS-
   spezifisches Escaping.

**Framework-Check:**
- React/Vue/Angular bei normalem Binding: auto-escape. TRUE wird nur bei
  `dangerouslySetInnerHTML` / `v-html` / `[innerHTML]`.
- Jinja/Handlebars: Context-sensitive Escape muss konfiguriert sein.

---

## CSRF

**Zusatz-Checks:**

1. **State-Change-Check:** Ist der Endpoint GET oder POST/PUT/DELETE?
   CSRF-Risiko nur bei State-Change (POST/PUT/DELETE/PATCH).
2. **Token-Check:** Wird ein CSRF-Token im Form/Header verifiziert?
3. **SameSite-Check:** Cookies mit `SameSite=Lax` / `Strict` →
   CSRF-Risiko reduziert.
4. **Origin/Referer-Check:** Wird `Origin`/`Referer` gegen Allowlist
   gecheckt?

**FALSE_POSITIVE wenn:** Methode ist GET und Endpoint ist wirklich
idempotent und nicht-state-aendernd.

---

## Auth Bypass

**Zusatz-Checks:**

1. **Check-Reihenfolge:** Auth-Middleware VOR dem Handler registriert?
   Reihenfolge von `app.use()` oder Decorator kann falsch sein.
2. **Fail-Open-Check:** Bei Exception in Auth-Logik — wird Request
   abgelehnt oder durchgelassen?
   - `try { verify(token) } catch (e) { /* log and continue */ }` → TRUE,
     Fail-Open.
3. **Token-Validation-Completeness:**
   - JWT: wird Signature verifiziert? Algorithm geprueft (kein `none`)?
     Expiry gecheckt?
   - Session-Token: wird Expiry + Widerruf geprueft?
4. **Default-Deny-vs-Default-Allow:** Neue Endpoints — was ist der
   Default? Allow ohne explizite Policy (oft Fail-Open) oder Deny ohne
   explizite Policy?

---

## IDOR / Authorization Failure

**Zusatz-Checks:**

1. **Ownership-Check:** `WHERE user_id = req.user.id` oder aequivalent?
2. **Role-Check:** `if (req.user.role !== 'admin')` vor der Aktion?
3. **Resource-Permission-Matrix:** Wird pro Aktion geprueft welche Rolle
   sie darf?
4. **Indirect-Object-Check:** Nicht nur IDs sondern auch Slugs/Tokens als
   Referenzen — werden die ebenfalls Ownership-geprueft?

**Typisch TRUE:** `GET /api/invoices/:id` lookup per `invoiceId` ohne
Owner-Filter.

---

## Path Traversal

**Zusatz-Checks:**

1. **Canonicalization:** Wird der Pfad vor Nutzung canonicalized und gegen
   den Base-Dir geprueft?
   - Python: `os.path.realpath(os.path.join(base, user)).startswith(base)`
   - Node: `path.resolve(base, user).startsWith(path.resolve(base))`
2. **Null-Byte-Check** (nur C/aeltere Langs): `\0` in Input?
3. **URL-Decode-Double-Decode:** Wird zweimal decoded oder Rohinput
   verwendet?

---

## SSRF

**Zusatz-Checks:**

1. **Allow-List-Check:** Ist das Ziel gegen eine Allow-List validiert?
2. **Internal-IP-Block:** Wird `169.254.169.254` (Cloud-Metadata),
   `localhost`, RFC1918-IPs blockiert?
3. **DNS-Rebinding-Check:** Wird IP direkt resolved und gecheckt, oder
   nur Hostname?
4. **Redirect-Follow-Check:** Folgt der HTTP-Client Redirects? Wenn ja,
   wird bei Redirect erneut validiert?

---

## Deserialization

**Zusatz-Checks:**

1. **Type-Whitelist:** Sind nur bestimmte Klassen deserialisierbar?
   - Python `pickle.loads(userInput)` ohne Restriction → TRUE
   - Java `ObjectInputStream` ohne LookAheadObjectInputStream → TRUE
   - PHP `unserialize` mit `allowed_classes=false` → OK
2. **Format-Check:** Ist das Format ueberhaupt deserialisiert-anfaellig?
   - JSON-Parse mit strict Mode → meistens OK
   - YAML mit `yaml.load` (unsafe) vs `yaml.safe_load` → unsafe ist TRUE

---

## Crypto Misuse

**Zusatz-Checks:**

1. **Primitive-Passend:** Ist die Primitive dem Zweck angemessen?
   - MD5/SHA1 fuer Passwoerter → TRUE (Brute-Force-anfaellig)
   - Argon2/bcrypt/scrypt fuer Passwoerter → OK
   - HMAC-SHA256 fuer Integritaet → OK
2. **Key-Management:** Woher kommt der Key?
   - Hardcoded im Source → TRUE (Secret-Exposure + Crypto-Misuse)
   - Env-Var ohne Rotation → MEDIUM
   - Secret-Manager (KMS/Vault) → OK
3. **IV/Nonce-Check:**
   - AES-CBC mit konstantem IV → TRUE
   - AES-GCM mit Nonce-Reuse → TRUE (Key-Recovery moeglich!)
   - ChaCha20-Poly1305 mit unique Nonce → OK
4. **Compare-Check:** String-Compare fuer MACs/Tokens → Timing-Attack
   moeglich.
   - `if (a === b)` fuer HMAC-Compare → TRUE
   - `crypto.timingSafeEqual(a, b)` → OK
5. **Random-Source-Check:**
   - `Math.random()` fuer Security-Token → TRUE
   - `crypto.randomBytes()` → OK

---

## Memory Corruption

**Memory-Safe-Subset-Check (kritischer FP-Filter):**

Wenn der Code in einer Memory-Safe-Subset laeuft, ist Memory Corruption
fast sicher FALSE_POSITIVE:

- **Safe Rust** (kein `unsafe`-Block) → FALSE
- **Go ohne `unsafe.Pointer` und ohne cgo** → FALSE
- **Managed Languages ohne FFI/JNI/P/Invoke** (Java, C#, Python, JS,
  Ruby, etc.) → FALSE

Ausnahmen: Compiler-Bug, Soundness-Hole in der Language-Spec (selten, aber
real — z.B. CVE-2022-21658 in Rust std).

---

## Race Condition

**Feasibility-Check:**

1. **Window-Size:** Wie gross ist das TOCTOU-Fenster in Milli-/Mikro-
   sekunden?
   - `<1ms` mit lokalem Check → theoretisch, aber oft nicht
     ausnutzbar ohne Privileg
   - `>100ms` oder netzwerk-basiert → trivial ausnutzbar
2. **Repeatability:** Kann Attacker die Race beliebig wiederholen
   (Scripted)?
3. **Impact-Verhaeltnis:** Steht der Impact im Verhaeltnis zur
   Ausnutzungs-Komplexitaet?

---

## Logic Bug

**Spec-Existenz-Check:**

1. **Gibt es eine dokumentierte Spezifikation / Invariant?**
   - Requirements-Doc, API-Spec, Unit-Test-Suite als implizite Spec, etc.
2. **Verletzt der Code die Spec?**
3. **Ist die Spec selbst sicher?**

**Wenn keine Spec existiert:** Finding kann dennoch valid sein, aber
muss mit UNCERTAIN markiert werden und der User muss die Spec bestaetigen.

**Typische Logic-Bugs:**
- Off-by-one in Quota-Berechnung
- Vorzeichen-Fehler in Guthaben-Logik
- State-Machine-Transitionen, die nicht in Spec enthalten sind
- Concurrency-Logic-Fehler (was bei gleichzeitigen Updates passiert)

---

## Secret Exposure

**Zusatz-Checks:**

1. **Hardcoded-Check:** String matches Secret-Pattern
   (JWT-Secret, API-Key-Regex, Base64-URL-Safe mit Length-Range)?
2. **Logging-Check:** Wird Secret in Log/Error/Console ausgegeben?
3. **Response-Leak:** Wird Secret in API-Response oder Error-Response
   zurueckgegeben?
4. **Git-History:** Wenn Finding ein Hardcoded-Secret ist, ist es auch in
   Git-History? (erzeugt `BLOCKER` + Hinweis auf Git-History-Bereinigung)

---

## Insecure Defaults (NEU)

Siehe ausfuehrlich `lenses/insecure-defaults.md`.

**Zusatz-Checks:**

1. **Fallback-Secret-Check:** `os.getenv("SECRET", "default")` mit
   statischem Default?
2. **Default-Credentials-Check:** `admin` / `admin` im Repo oder Docs?
3. **Fail-Open-Check:** Bei Verify-Failure wird durchgelassen?
4. **Default-Deny-vs-Default-Allow:** Was ist die Haltung bei fehlender
   Policy?

---

## Sharp Edges (NEU)

Siehe `lenses/sharp-edges.md`.

**Zusatz-Checks:**

1. **Pit-of-Failure-Test:** Schreibt man den sicheren oder unsicheren Pfad
   leichter?
2. **Primitive-vs-Semantic:** Ist die API so primitiv, dass jeder Call
   Security-Entscheidungen wiederholen muss?
3. **Default-Unsicher:** Muss der Entwickler aktiv Parameter setzen, um
   sicher zu sein?

---

## Supply-Chain (NEU)

Siehe `lenses/supply-chain.md`.

**Zusatz-Checks:**

1. **CVE-Scan** (klassisch, wie `dependency-cves`)
2. **Maintainer-Health:** Wann letzte Release, wie viele Committer, wie
   schnell werden Issues gefixt?
3. **Typosquatting-Check:** Gibt es sehr aehnliche Paket-Namen?
4. **Lockfile-Check:** package-lock.json / poetry.lock / Gemfile.lock
   vorhanden und committed?
5. **Pinning-Check:** Sind Versionen gepinnt oder Ranges?
6. **Subresource-Integrity:** Bei CDN-geladenen Scripts `integrity`-Attr?

---

## Verwendung im Workflow

In Phase 4 (FP-Verification) laueft pro Finding:

1. Generische 6 Gates aus `fp-verification.md`
2. **Zusaetzlich** die Bug-Class-spezifischen Checks aus dieser Datei

Beides zusammen ergibt das Final-Verdict.

Im Finding-YAML:
```yaml
bug_class: sql_injection
bug_class_checks:
  parameterization: FAIL    # → TRUE_POSITIVE-Indikator
  orm_escape_hatch: NA
  second_order: NA
  stored_procedure: NA
```
