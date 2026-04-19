# Attacker Models — WHO / ACCESS / INTERFACE

Jedes Security-Finding wird im Kontext eines **expliziten Attacker-Models**
bewertet. Ohne Attacker-Model ist Severity-Rating Kaffeesatzleserei.

Inspiriert von Trail-of-Bits differential-review (Phase 5 Adversarial).

## Das WHO/ACCESS/INTERFACE-Framework

Ein Attacker-Model besteht aus drei Dimensionen:

### WHO — Wer greift an?

| WHO | Beschreibung | Beispiel-Kontexte |
|-----|--------------|-------------------|
| `unauthenticated-remote` | Jeder im Internet ohne Account | Public-Web-App, Public-API |
| `authenticated-user` | Gewoehnlicher Nutzer mit gueltigem Account | SaaS-App, Member-Portal |
| `privileged-user` | User mit besonderen Rechten (Admin, Power-User) | Backoffice, Admin-Console |
| `colocated-tenant` | Anderer Kunde auf derselben Multi-Tenant-Plattform | SaaS mit Mandanten-Trennung |
| `compromised-dependency` | Supply-Chain-Angreifer via kompromittierte Library | npm/PyPI-Supply-Chain |
| `compromised-account` | Attacker hat valide Credentials eines realen Users | Credential-Stuffing-Opfer |
| `operator-insider` | Legitimer Operator/Employee mit boesen Absichten | Interne Threats |
| `physical-access` | Attacker hat physischen Geraete-Zugriff | Diebstahl, Evil-Maid |
| `network-peer` | Attacker sitzt im selben Netzwerk (MITM) | Cafe-WLAN, Corp-LAN |
| `observer-only` | Passive Beobachtung (Logs, Timing, Side-Channels) | Log-Leser, Shared Infra |

### ACCESS — Welchen Zugriff hat der Attacker?

| ACCESS | Beschreibung |
|--------|--------------|
| `network-only` | Nur Netzwerk-Erreichbarkeit, kein Account |
| `valid-account` | Login mit gewoehnlichem Account |
| `admin-account` | Login mit Admin-/Operator-Account |
| `api-key` | Besitzt gueltigen API-Key (moeglich gestohlen) |
| `session-token` | Besitzt gueltiges Session-Cookie/JWT |
| `supply-chain-write` | Kann Code in eine Dependency pushen |
| `subdomain-takeover` | Kontrolle ueber benachbarte Subdomain |
| `physical-device` | Direkter Hardware-Zugriff |
| `filesystem-read` | Kann Files auf dem Server lesen (z.B. ueber LFI) |
| `filesystem-write` | Kann Files schreiben (z.B. ueber Upload oder Path-Traversal) |

### INTERFACE — Welche Schnittstelle nutzt der Attacker?

| INTERFACE | Beschreibung |
|-----------|--------------|
| `public-api` | Oeffentlich erreichbare HTTP-API |
| `internal-api` | Nur-intern-erreichbare API (VPN, SG-Regeln) |
| `admin-api` | Admin-Only-API (hinter Auth + Role-Check) |
| `websocket` | Persistente WS-Verbindung |
| `webhook-receiver` | Endpoint der externe Webhooks akzeptiert |
| `message-queue` | Consumer auf Queue mit externen Producern |
| `cli` | Command-Line-Tool |
| `file-upload` | Multipart-Upload-Endpoint |
| `oauth-flow` | OAuth/OIDC-Callback |
| `frontend-js` | Browser-ausgelieferter JS-Code |
| `native-app` | Mobile/Desktop-App |
| `smart-contract-call` | On-Chain-Interaction |

---

## Typische Attacker-Model-Kombinationen

Fuer gaengige Szenarien die passenden Kombinationen:

| Szenario | WHO | ACCESS | INTERFACE |
|----------|-----|--------|-----------|
| **Public SaaS-App** | `unauthenticated-remote` + `authenticated-user` | `network-only` + `valid-account` | `public-api` |
| **Internal Tool** | `operator-insider` + `colocated-tenant` | `valid-account` | `internal-api` |
| **Admin-Panel** | `authenticated-user` (Priv-Esc-Ziel) | `valid-account` | `admin-api` |
| **Webhook-Handler** | `unauthenticated-remote` | `network-only` | `webhook-receiver` |
| **File-Processor** | `unauthenticated-remote` | `network-only` | `file-upload` |
| **npm-Package** | `compromised-dependency` | `supply-chain-write` | N/A |
| **Mobile-Backend** | `unauthenticated-remote` + `compromised-account` | `network-only` + `session-token` | `public-api` |
| **Multi-Tenant-SaaS** | `colocated-tenant` | `valid-account` | `public-api` |

## Exploitability-Rating

Pro Finding zusaetzlich zum Severity-Rating: **Wie trivial ist die
Ausnutzung?**

| Rating | Bedeutung |
|--------|-----------|
| `trivial` | Ein-Request-Exploit, kein besonderer Tool-Einsatz, Attacker-Control offensichtlich |
| `moderate` | Mehrere Requests noetig, aber Standard-Tooling (Burp, curl) reicht |
| `conditional` | Braucht bestimmte Vorbedingungen (Race-Window, specific Timing, bestimmtes Input-Format) |
| `theoretical` | Exploit-Pfad existiert, aber Real-World-Ausnutzung braucht viele Annahmen |

**Priorisierungs-Regel:** Bei gleicher Severity-Stufe sticht `trivial` >
`moderate` > `conditional` > `theoretical`.

**Report-Regel:** Bei Exploitability `theoretical` → Severity maximal
`MEDIUM` (ausser dokumentierte Gegenbeispiele, z.B. Crypto-Schwachstellen
mit Langzeit-Impact).

---

## Attacker-Model im Finding-Template

Jedes Finding enthaelt den Attacker-Kontext:

**Auditor-Rolle:**
```markdown
### [CRITICAL / HIGH confidence / TRUE_POSITIVE] IDOR on /api/orders/:id
**Lens:** `authorization`
**File:** `routes/orders.js:14`
**Attacker-Model:** authenticated-user + valid-account + public-api
**Exploitability:** trivial

**Problem:** Route loads order by ID without checking ownership.
Attacker with own account calls `GET /api/orders/42` and reads order
that belongs to user 99.

**Fix:** Add `WHERE user_id = req.user.id` to query.
```

**Agent-Rolle (YAML-Feld):**
```yaml
attacker_model:
  who: authenticated-user
  access: valid-account
  interface: public-api
exploitability: trivial
```

---

## Wie das Attacker-Model den Audit-Scope formt

Das gewaehlte Attacker-Model entscheidet, welche Findings ueberhaupt
Findings sind:

- **Attacker-Model `authenticated-user`** → Missing-Rate-Limit auf Admin-
  Endpoint ist relevant (User koennte Admin-Endpoint brute-forcen).
- **Attacker-Model `unauthenticated-remote`** → Authorization-Logik-Fehler
  HINTER Auth-Check sind nicht direkt erreichbar; Priorisierung auf
  Unauth-reachable-Code.
- **Attacker-Model `operator-insider`** → Audit-Logging-Luecken werden
  relevant (Operator loescht Spuren).

**Praxis:** Phase 1 (Intake) setzt 1 primaeres + 0-2 sekundaere Attacker-
Models. Beispiel:
- Primaer: `unauthenticated-remote / network-only / public-api`
- Sekundaer: `authenticated-user / valid-account / public-api`
  (fuer Priv-Esc-Findings)
- Sekundaer: `colocated-tenant / valid-account / public-api`
  (bei Multi-Tenancy)

Out-of-Scope explizit markieren:
- Out-of-Scope: `operator-insider` (interner Threat nicht im Audit-Scope),
  `physical-access`.

---

## Framework-Bezuege

Das WHO/ACCESS/INTERFACE-Modell laesst sich auf STRIDE und DREAD abbilden,
falls der User das bevorzugt:

- **STRIDE** (Spoofing/Tampering/Repudiation/Info-Disclosure/DoS/Priv-Esc)
  → Spoofing → WHO-Shift; Tampering → INTERFACE-write; Info-Disclosure →
  ACCESS-read; Priv-Esc → ACCESS-upgrade.
- **DREAD** (Damage/Reproducibility/Exploitability/Affected-Users/
  Discoverability) → Damage ≈ Severity; Exploitability ≈ Exploitability-
  Rating oben.

breachguard bleibt intern bei WHO/ACCESS/INTERFACE (einfacher, direkter),
exportiert aber bei explizitem User-Wunsch nach STRIDE/DREAD.

---

## Anti-Pattern: Attacker-Model nachtraeglich anpassen

**Nicht** das Attacker-Model waehrend der Audit-Phase umformulieren, um
ein Finding zu rechtfertigen oder zu entschaerfen. Das ist Cherry-Picking.

Beispiel fuer das Anti-Pattern: Ein Finding sieht nach `FALSE_POSITIVE`
aus, weil der Endpoint nur authentifizierte User erreichen. Dann wird das
Attacker-Model implizit auf "compromised-account" aufgebohrt, damit das
Finding "TRUE_POSITIVE" bleibt.

Stattdessen: Primaer-Model festlegen (Phase 1), Finding gegen Primaer-
Model bewerten. Wenn das Finding nur bei einem erweiterten Model gilt:
Sekundaer-Model explizit hinzufuegen, Finding dort verorten.

---

## Wenn kein Attacker-Model genannt wird

Wenn der User kein Attacker-Model spezifiziert und auch keins impliziert
(z.B. "audit mein Repo") → Phase 1 nimmt den safest-default:

**Default Attacker-Model (bei Webapp/Backend):**
- Primaer: `unauthenticated-remote / network-only / public-api`
- Sekundaer: `authenticated-user / valid-account / public-api`

**Default bei CLI-Tools / Libraries:**
- Primaer: `compromised-dependency / supply-chain-write / cli`
- Sekundaer: `physical-access / physical-device / cli`

**Default bei internen Tools:**
- Primaer: `operator-insider / valid-account / internal-api`
- Sekundaer: `compromised-account / valid-account / internal-api`

Default wird im Report-Header explizit gemacht, damit der User widersprechen
kann.
