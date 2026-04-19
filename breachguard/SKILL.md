---
name: breachguard
description: "Security-first Multi-Lens-Code-Audit, 283 Lenses/27 Domaenen, Phasen-Workflow, FP-Gates. IMMER verwenden bei: Security-Review, Vuln-Scan, Secure-Code-Audit, CVE-Analyse, Supply-Chain-Risiko, Insecure Defaults, Fail-Open, Footgun-APIs, Haertung, Fix-Verifikation, Variant-Hunt, SQL-Injection, XSS, CSRF, Auth/Session, Authorization, Secrets, Crypto, Security-Headers, Data Exposure, Rate-Limit, Dependency-CVEs, 'Sicherheitsluecken pruefen'. Auch fuer breite Audits: Performance, Architecture, Code Quality, Testing, Documentation, API Design, Database, Concurrency, Error Handling, Observability, DevOps, Deployment, Frontend, UX, i18n, Compliance (GDPR/NIS2/KRITIS/DORA/WCAG), OSS-Readiness. 12 Modi (audit/triage/deep-audit/variant/verify-fix/bugfix/feature/discover/deploy/custom/opensource/content), 6 Rollen (Auditor/Developer/Architect/Reporter/Agent/Detector). NICHT fuer: Prompt-Injection (prompt-injection-scanner), Mobile-MASVS (owasp-mas), reine UX/Design (enbw-impeccable)."
license: Apache-2.0
---

# breachguard — Security-first Multi-Lens-Code-Audit

Code-Audit-Skill mit Security-Kern. **280+ Specialist-Lenses** ueber **27 Domaenen**
und **12 Modi**, mit **5-Phasen-Methodologie** und **FP-Gate-Reviews**.

Abstammung:

- Basis: [RepoLens](https://github.com/TheMorpheus407/RepoLens) (Apache-2.0,
  Mark Zimmermann / TheMorpheus407) — 280 Lenses, 27 Domaenen, 5 Rollen, YAML-
  Schemas, Compliance-Mappings.
- Security-Methodologie: strukturell inspiriert von [Trail of Bits Skills](https://github.com/trailofbits/skills)
  (CC BY-SA 4.0) — Phasen-Workflow, FP-Gate-Reviews, Attacker-Modelling,
  Bug-Class-Verifikation, Variant-Analysis.
- Neue Lenses (insecure-defaults, sharp-edges, supply-chain) sind originaere
  breachguard-Inhalte, von ToB-Konzepten inspiriert.

**Version:** 0.3.0-breachguard — **Schema:** `breachguard.v0`

## Was dieser Skill macht

Appliziert kuratierte Experten-Lenses auf Code / Repos / Snippets. Produziert
strukturierte Findings-Reports — wahlweise als Markdown fuer Menschen, als
YAML/JSON fuer Agent-Pipelines, oder beides im Dual-Modus.

**Neu gegenueber RepoLens:** Jedes Security-Finding durchlaeuft eine
**FP-Verifikations-Phase** mit expliziten Gate Reviews und erhaelt ein
Verdikt: `TRUE_POSITIVE`, `FALSE_POSITIVE` oder `UNCERTAIN`.

## Was dieser Skill NICHT macht

- **Keine Shell-Kommandos gegen Live-Systeme** (auch nicht im `deploy`-Modus)
- **Kein automatisches `gh issue create`** — nur Vorschlaege zum Pasten
- Kein Spawning paralleler Subprocesses, keine DONE-Streak-Loops
- Keine Rekonstruktion von Secrets, Exploits oder Malware-Payloads
- Keine Prompt-Injection-Scans von Skills/Prompts (→ siehe
  `prompt-injection-scanner`-Skill)

## Kern-Prinzipien (fuer jedes Audit gueltig)

Diese Prinzipien gelten durchgaengig, egal in welchem Modus/Rolle:

1. **Untrusted bis Beweis des Gegenteils.** Jede Eingabe ausserhalb der
   Trust-Boundary ist adversarial bis bewiesen ist, dass sie validiert wird.
2. **Fail Closed bei Unklarheit.** Wenn nicht entscheidbar ist, ob eine
   Validierung greift → Finding als `UNCERTAIN` markieren, nicht raten.
3. **Coverage-Limits explizit machen.** "Nur `auth/` geprueft, nicht
   `legacy/`" am Anfang des Reports. Keine impliziten Abdeckungs-Claims.
4. **Confidence pro Finding.** Jedes Finding hat `confidence: HIGH|MEDIUM|LOW`
   plus `evidence_level: DIRECT|INFERENCE|HEURISTIC`.
5. **Keine Exploit-Regurgitation.** Probleme beschreiben, nicht step-by-step
   ausnutzen.
6. **Pit of Success.** Ein sicheres API-Design ist der Pfad des geringsten
   Widerstands. APIs, bei denen der falsche Weg einfacher ist als der
   richtige, sind selbst der Defekt.

Details: **`references/methodology.md`**.

## Dreidimensionales Routing

Jede Anfrage wird auf drei Achsen gemappt:

| Achse | Werte | Default | Referenz |
|-------|-------|---------|----------|
| **Modus** | audit, triage, deep-audit, variant, verify-fix, bugfix, feature, discover, deploy, custom, opensource, content | `audit` | `references/modes.md` |
| **Domain** | 27 Domaenen (siehe Tabelle unten) | je nach Anfrage | `references/lenses/<domain>.md` |
| **Rolle** | Auditor, Developer, Architect, Reporter, Agent, Detector | `Auditor` | `references/roles.md` |

### Rollen-Trigger (Output-Persona)

- **Auditor** (Default) — Standard-Findings mit Severity/Location/Fix
- **Developer** — Trigger: "wie fixe ich", "show the fix", "gib Code" — Before/After-Snippets pro Finding
- **Architect** — Trigger: "root cause", "systemic", "architectural review" — Querschnitts-Patterns ueber Findings hinweg
- **Reporter** — Trigger: "executive summary", "Management-Zusammenfassung" — Metriken, Trends, Risiko-Rating
- **Agent** — Trigger: "YAML", "JSON", "for my pipeline", "machine-readable" — strukturiert nach `agent-output.md`
- **Detector** (NEU) — Trigger: "Semgrep-Regel", "CodeQL-Query", "detection rule", "in CI einbauen" — exportiert Rule-Skeleton (YAML/QL) pro Finding fuer wiederverwendbare Detection

### Modi-Trigger (Workflow-Dimension)

Security-orientierte Modi (neu in breachguard):

- `audit` (Standard) — Standard-Review mit vollem Phasen-Workflow
- `triage` — **Quick Scan** (Phasen 1+2+3 only, kein FP-Gate, <15min)
- `deep-audit` — **Voller Phasen-Workflow** inkl. FP-Gate-Review + Attacker-Model + Bug-Class-Verifikation
- `variant` — **Variant-Hunt** (eine bekannte Vuln gegeben → Geschwister im Repo finden; siehe `variant-analysis.md`)
- `verify-fix` — **Fix-Verifikation** (Finding + Fix-Commit → Ist der Root-Cause geschlossen? Regressionen?)

Weitere Modi (aus RepoLens):

- `bugfix` — nur echte Bugs
- `feature` — fehlende Capabilities
- `discover` — Brainstorming
- `deploy` — Infrastruktur-Config im Repo (read-only, **keine Live-Shell**)
- `custom` — Change-Impact-Analyse (braucht Change-Statement)
- `opensource` — Pre-Publication-Check
- `content` — Content-Audit oder -Erstellung

## 5-Phasen-Methodologie (fuer Security-Modi)

Bei `audit` / `deep-audit` / `variant` laeuft jeder Security-Check durch:

| Phase | Aufgabe | Output | Ref |
|-------|---------|--------|-----|
| 1. **Intake & Triage** | Scope, Attacker-Model, Critical Paths | Scope-Liste, Attacker-Definition | `attacker-models.md` |
| 2. **Context Building** | Entrypoints, Trust-Boundaries, Actors, Storage | System-Modell | `methodology.md` §2 |
| 3. **Lens Application** | Security-Lenses auf Critical Paths | Raw Findings | `lenses/security.md` + Zusatzlenses |
| 4. **FP Verification** | Gate Reviews, Bug-Class-Rubrik | TRUE/FALSE/UNCERTAIN-Verdikt | `fp-verification.md` + `bug-class-verification.md` |
| 5. **Report** | Strukturierter Output (Rolle-abhaengig) | Markdown/YAML | `roles.md` + `agent-output.md` |

Bei `triage`: nur Phasen 1+2+3, ohne FP-Gate (schneller, mehr Noise).
Bei `audit` (Default): alle 5 Phasen.
Bei `deep-audit`: alle 5 Phasen mit vollem Attacker-Modelling pro Finding.

Details: **`references/methodology.md`**.

## Domaenen (27)

Die Security-Domain ist auf **14 Lenses erweitert** (11 klassische + 3 neue).
Alle anderen Domaenen bleiben unveraendert aus RepoLens.

| Domain-ID | Name | Lenses |
|-----------|------|--------|
| `security` | Security | **14** (11 + insecure-defaults, sharp-edges, supply-chain) |
| `compliance` | Compliance | 56 |
| `deployment` | Deployment | 26 |
| `toolgate` | Tool Gate | 18 |
| `content-quality` | Content Quality | 17 |
| `code-quality` | Code Quality | 14 |
| `discovery` | Product Discovery | 14 |
| `open-source-readiness` | Open Source Readiness | 13 |
| `architecture` | Architecture | 9 |
| `testing` | Testing | 9 |
| `performance` | Performance | 9 |
| `interaction-design` | Interaction Design | 8 |
| `error-handling` | Error Handling | 6 |
| `api-design` | API Design | 6 |
| `database` | Database | 6 |
| `information-architecture` | Information Architecture | 6 |
| `ux-antipatterns` | UX Anti-Patterns | 6 |
| `devops` | DevOps | 6 |
| `maintainability` | Maintainability | 6 |
| `frontend` | Frontend | 5 |
| `visual-design` | Visual Design | 5 |
| `adaptive-ux` | Adaptive UX | 5 |
| `observability` | Observability | 5 |
| `design-system` | Design System | 4 |
| `documentation` | Documentation | 4 |
| `concurrency` | Concurrency | 4 |
| `i18n` | Internationalization | 2 |

**Total: 283 Lenses.** Struktur-Map: `references/domains.json`.

### Security-Lenses im Detail (14)

**Klassisch (in `lenses/security.md`):**
`injection`, `xss-csrf`, `auth-session`, `authorization`, `secrets`,
`dependency-cves`, `security-headers`, `cryptography`, `input-sanitization`,
`data-exposure`, `rate-abuse`.

**Neu (eigene Dateien):**
- `insecure-defaults` (`lenses/insecure-defaults.md`) — Fallback-Secrets,
  Default-Credentials, Fail-Open-Patterns.
- `sharp-edges` (`lenses/sharp-edges.md`) — Footgun-APIs, Primitive-vs-
  Semantic, Pit-of-Failure-Designs.
- `supply-chain` (`lenses/supply-chain.md`) — Maintainer-Health,
  Typosquatting, Protestware-Risiko, Build-Integritaet, Transitive-
  Abhaengigkeiten, Lockfile/Signing.

## Compliance-spezifisch

Bei `compliance`-Domain zusaetzlich **`references/compliance-mappings.md`**
laden — Mapping der 56 Lenses auf konkrete Regulierungen (GDPR, KRITIS, NIS2,
EAA/BFSG, WCAG, DORA, PSD2, AI Act, ...).

## Agent-Output fuer Pipelines

Wenn der User YAML/JSON anfragt (Rolle = Agent): nach den Schemas aus
`references/agent-output.md` ausgeben:

- `breachguard.finding` — einzelner Befund (mit `attacker_model`, `confidence`, `verdict`, `evidence_level`)
- `breachguard.audit_summary` — Lauf-Metriken + `release_gate`-Decision + `next_agent_actions`
- `breachguard.compliance_matrix` — fuer Compliance-Audits
- `breachguard.change_impact` — fuer custom-Mode
- `breachguard.fix_verification` — fuer verify-fix-Mode
- `breachguard.variant_report` — fuer variant-Mode
- `breachguard.detection_rule` — fuer Detector-Rolle (Semgrep/CodeQL-Skeleton)

Jede Ausgabe enthaelt `next_agent_actions` mit Downstream-Agent-Katalog
(`ticket_creator`, `ci_gate`, `remediation_agent`, `grc_agent`, ...).

## Workflow-Kurzfassung

1. **Input einsammeln** (Code/Repo/Snippet)
2. **Routen** auf Modus × Domain × Rolle
3. **Bei Security-Modus:** 5-Phasen-Workflow aus `methodology.md` starten
4. **Lens-Referenzen laden:** `references/lenses/<domain-id>.md` (+ bei
   Security: `insecure-defaults.md`, `sharp-edges.md`, `supply-chain.md` bei
   Bedarf)
5. **Lenses anwenden, Findings strukturieren** (Format nach Rolle)
6. **FP-Gate-Review** bei `audit`/`deep-audit`/`variant` →
   `fp-verification.md` + `bug-class-verification.md`
7. **Output ausgeben** (Markdown fuer Human-Rollen, YAML fuer Agent-Rolle,
   Rule-Skeleton fuer Detector)
8. **Optional:** `gh issue create`-Kommandos zum Pasten anbieten

Detail-Workflow + Few-Shot pro Rolle: **`references/workflow.md`**.

## Self-Sync (Updates vom Upstream)

Wenn der User fragt "gibt es Updates?" / "update the skill": **`references/updater-sync.md`**
laden.

## CLI-Fallback (bei grossen Repos)

Bei Repos >50 Files ODER Wunsch nach Voll-Audit: **nicht selbst versuchen**,
stattdessen Original-CLI-Kommando vorschlagen (die RepoLens-CLI deckt auch
breachguard-Security-Lenses ab):

```bash
# Single Lens
./repolens.sh --project ~/repo --agent claude --focus <lens-id>

# Single Domain parallel
./repolens.sh --project ~/repo --agent claude --domain security --parallel

# Voll-Audit (teuer!)
./repolens.sh --project ~/repo --agent claude --parallel --max-parallel 8 --max-cost 200

# Local Mode (kein gh)
./repolens.sh --project ~/repo --agent claude --local --output ./audit-results
```

**Pflicht-Disclaimer dazu:**

> RepoLens spawnt AI-Agents mit Shell-Zugriff. Vollaudit kostet leicht
> dreistellig. Nur gegen eigene Repos. Read-Only-Sandbox empfohlen.
> Details: https://github.com/TheMorpheus407/RepoLens#warnings--limits

## Severity + Confidence + Verdikt (alle Modi, alle Rollen)

**Severity** (Impact bei Ausnutzung):
- `[CRITICAL]` — Production-Breaker, Security-Holes, Datenverlust-Risiko
- `[HIGH]` — Echtes Bug-/Sicherheits-Risiko mit Impact
- `[MEDIUM]` — Wartbarkeit, Code-Quality mit Business-Impact
- `[LOW]` — Style, Convention, Nice-to-have

**Confidence** (wie sicher ist das Finding?):
- `HIGH` — Direkter Code-Beleg, Datenfluss rekonstruiert, Exploit plausibel
- `MEDIUM` — Pattern erkannt, Kontext wahrscheinlich anfaellig, nicht
  final verifiziert
- `LOW` — Heuristik/Indiz, kann Finding sein, kann FP sein

**Verdict** (FP-Gate-Ergebnis, nur Security-Modi mit Gate):
- `TRUE_POSITIVE` — Alle Gates durchlaufen, Finding steht
- `FALSE_POSITIVE` — An einem Gate gescheitert (z.B. Validation greift,
  Memory-Safe, Attacker hat keinen Zugriff)
- `UNCERTAIN` — Nicht entscheidbar mit verfuegbarem Kontext

**~1-Hour-Rule:** Jedes Issue ~1h fixbar; groesser = Tracking-Issue.

## Reference-Files

**RepoLens-Erbe:**
- `references/domains.json` — Lens/Domain-Map (283 Lenses, aktualisiert)
- `references/workflow.md` — Workflow + Few-Shot (Auditor/Developer/Agent/Detector)
- `references/modes.md` — Alle 12 Mode-Templates
- `references/roles.md` — 6 Rollen (inkl. neue Detector-Rolle)
- `references/agent-output.md` — YAML-Schemas + Downstream-Agent-Katalog
- `references/compliance-mappings.md` — 56 Compliance-Lenses → Regulierungen
- `references/code-examples.md` — Bad/Good-Snippets fuer Developer-Rolle
- `references/updater-sync.md` — Self-Sync mit Upstream
- `references/lenses/<domain-id>.md` — 27 Dateien, je eine pro Domain

**breachguard-Erweiterungen (NEU, inspiriert von Trail of Bits):**
- `references/methodology.md` — 5-Phasen-Workflow + Kern-Prinzipien
- `references/fp-verification.md` — 6 Gate Reviews, TRUE/FALSE/UNCERTAIN
- `references/attacker-models.md` — WHO/ACCESS/INTERFACE + Exploitability-Rating
- `references/bug-class-verification.md` — Bug-class-spezifische Rubriken (SQLi, XSS, Auth, Crypto, Race, Path-Traversal, Memory, Logic)
- `references/variant-analysis.md` — 5-Schritt-Variant-Hunt, <=50% FP-Stop
- `references/lenses/insecure-defaults.md` — NEU, Fail-Open/Fallback-Secret
- `references/lenses/sharp-edges.md` — NEU, Footgun-APIs
- `references/lenses/supply-chain.md` — NEU, Threat-Landscape

## Security-Hinweis

Die Original-RepoLens-Templates enthielten Direktiven fuer autonome Claude-
Code-Agents mit `--dangerously-skip-permissions`. **Diese wurden fuer den
Skill-Kontext entfernt.** Der Skill fuhrt **keine Shell-Kommandos aus** und
**erstellt keine Issues** ohne explizite User-Bestatigung.

## Lizenz

Apache-2.0. Siehe `LICENSE` und `NOTICE`.
