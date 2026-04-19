# breachguard Mode-Templates

**Zwoelf Modi.** Adaptierung aus RepoLens: Die autonomous-agent-Direktiven
aus dem Original (z.B. "Use `gh issue create` directly via Bash. Do NOT ask
the caller to run commands.") wurden entfernt. breachguard erstellt
**niemals selbst** GitHub-Issues und fuehrt **keine Shell-Kommandos gegen
Live-Systeme** aus.

## Modi-Uebersicht

| Modus | Phasen | Anwendung |
|-------|--------|-----------|
| `audit` (Default) | 1-5 | Standard-Review |
| `triage` (NEU) | 1-3 | Quick Scan ohne FP-Gate |
| `deep-audit` (NEU) | 1-5 + Attacker-Modell pro Finding | Prod-Release-Gate |
| `variant` (NEU) | 5-Schritt-Variant-Hunt | Eine bekannte Vuln → Geschwister |
| `verify-fix` (NEU) | Fix-Delta + FP-Gate + Regression-Check | Nach Remediation |
| `bugfix` | 1-5 (nur Bug-Klasse) | Nur Defekte, kein Code-Smell |
| `feature` | Brainstorm | Fehlende Capabilities |
| `discover` | Brainstorm | Produkt-Strategie |
| `deploy` | 1-3 | Infra-Config (READ-ONLY, kein Live-Shell) |
| `custom` | Change-Impact | User kuendigt Change an |
| `opensource` | 1-5 (pre-publish) | Pre-Publication-Check |
| `content` | separat | Content-Audit/Erstellung |

---

## `audit` (Standard)

Echte, umsetzbare Issues im existierenden Code finden. Voller
5-Phasen-Workflow aus `methodology.md`.

**Output-Schema:**
```markdown
### [SEVERITY / CONFIDENCE / VERDICT] Titel
**Lens:** `<lens-id>`
**File:** `path/to/file.ext:line`
**Attacker-Model:** <WHO/ACCESS/INTERFACE>
**Evidence-Level:** DIRECT/INFERENCE/HEURISTIC

**Problem:** <1-3 Satze>

**Fix:** <~1h Aufwand>
```

**Severity:**
- `[CRITICAL]` — Production-Breaker, Security-Holes, Datenverlust-Risiko
- `[HIGH]` — Echtes Bug-Risiko mit Impact
- `[MEDIUM]` — Wartbarkeit, Code-Quality mit Business-Impact
- `[LOW]` — Style, Convention, Nice-to-have

**Confidence:** HIGH / MEDIUM / LOW
**Verdict** (bei Security-Modi): TRUE_POSITIVE / FALSE_POSITIVE / UNCERTAIN

**~1-Hour-Rule:** Jedes Issue muss in ~1h fixbar sein.

---

## `triage` (NEU)

**Quick-Scan.** Phasen 1-3 aus `methodology.md`. **Kein FP-Gate.** Dauer:
<15 min.

**Wann:** Sehr schneller erster Eindruck, z.B. vor tieferer Analyse; bei
sehr grossen Repos als Priorisierungs-Hilfe.

**Output enthält Pflicht-Warning:**
```markdown
> **WARNUNG:** Triage-Modus aktiv — Findings sind **NICHT FP-verifiziert**.
> Erwarte 20-40% False Positives. Fuer Prod-Review `deep-audit` nutzen.
```

**Format:** Wie `audit`, aber ohne `Gates:`-Block. `Verdict` ist immer
`RAW` (ersatzweise fuer TRUE/FALSE/UNCERTAIN).

---

## `deep-audit` (NEU)

**Voller Phasen-Workflow mit Attacker-Modell-Detail pro Finding.** Dauer:
2-6h.

**Wann:** Pre-Prod-Release, regulated Environments (Finanz, Health,
KRITIS), nach Security-Incident.

**Unterschied zu `audit`:**
- **Attacker-Model-Sektion am Report-Anfang** (Primaer + Sekundaer).
- Pro Finding voller Gate-Trace (nicht nur `PASS`/`FAIL`, sondern
  kurze Begruendung pro Gate).
- **Coverage-Limits-Sektion** am Ende ist ausfuehrlich.
- Bug-Class-Verifikation pflicht (siehe `bug-class-verification.md`).

---

## `variant` (NEU)

**Eine bestaetigte Vuln gegeben. Finde die Geschwister im Repo.**

**Input:** Original-Finding (Location + Pattern-Beschreibung).
**Output:** Variants-Liste + optional Detection-Rule-Skeleton (bei
Rolle Detector).

**Methodologie:** 5-Schritt-Prozess aus `variant-analysis.md`.

**Stop-Condition:** Abstraktions-Schritt mit >50% FP-Rate → Revert.

**Wann:**
- Pentest-Finding → suche gleiche Pattern in Repo.
- CVE in Dep gefixt → suche eigenen Code mit gleichem Pattern.
- Bug-Bounty-Report validiert → Variant-Hunt vor Release.

---

## `verify-fix` (NEU)

**Finding + Fix-Commit gegeben. Ist der Root-Cause geschlossen?
Regressionen?**

**Input:** Original-Finding + Commit-Range oder Diff.
**Output:** 3-Punkt-Verifikation:
1. **Remediation:** Adressiert der Fix den Root-Cause? (nicht nur das
   Symptom)
2. **Regression:** Führt der Fix neue Vulns ein?
3. **Completeness:** Sind alle Variants gefixt, nicht nur die
   ursprüngliche Stelle?

**Ablauf:**
- Phase 1 (Intake): Finding + Fix-Context einsammeln.
- Fix-Delta-Analyse: Was hat sich geaendert?
- Phase 4 (FP-Gate): Neuer Fix-Code durch die 6 Gates.
- Phase 5 (Report): Verifikations-Ergebnis.

**Output-Schema:**
```markdown
# Fix-Verifikation: F-001 (SQL Injection in users.js:4)

**Fix-Commit(s):** <hash-range>
**Fix-Autor:** <name>
**Fix-Datum:** <ISO>

## Remediation-Check
- Root-Cause (String-Konkat) addressed: JA (wechselt auf Parameterized)
- Symptom-only fix: NEIN

## Regression-Check
- Neue Findings im Fix-Diff: keine
- Fix-Code selbst lensbar gegen injection/auth-session/secrets: clean

## Completeness-Check
- Variants (aus F-001-Analyse 9 Stellen) alle gefixt:
  - 8 von 9 gefixt
  - **Offen:** `orders.js:12` im Fix-Commit-Range nicht adressiert
    → Tracking empfohlen

## Verdict
- Fix-Status: PARTIAL_REMEDIATION
- Neue Release-Gate-Decision: WARN (nicht BLOCK, aber Tracking noetig)
```

---

## `bugfix`

Nur **echte Bugs und Defekte** — kein Code-Smell-Hunting. Zusatzfelder pro
Finding: Repro-Szenario + Erwartetes vs. Tatsachliches Verhalten.

---

## `feature`

Fehlende Capabilities identifizieren.

**Struktur:** `[P0]/[P1]/[P2]/[P3]` Titel, Lens, Was fehlt, Warum relevant,
Implementierungs-Vorschlag.

---

## `discover`

Produktstrategie-Brainstorming, explorativ.

**Struktur:** `[IDEA]/[OPPORTUNITY]/[QUICK-WIN]` Titel, Lens, Was, Warum jetzt,
Aufwand S/M/L. Keine Issue-Creations.

---

## `deploy` (READ-ONLY im Skill-Kontext)

**Skill-Kontext-Hinweis:** Dieser Modus im Original-CLI lauft gegen
**Live-Server mit Shell-Zugriff**. Im Skill-Kontext **erzeugt Claude niemals
Shell-Kommandos gegen Server**.

Stattdessen: **Infrastruktur-Konfiguration im Repo** analysieren (Dockerfiles,
docker-compose, systemd-Units, nginx.conf, Terraform, Ansible, k8s-Manifests,
CI/CD-Pipelines).

Bei echtem Live-Server-Audit-Bedarf: Original-CLI verweisen:
```bash
./repolens.sh --project /srv/app --agent claude --mode deploy
```

---

## `custom` (Change Impact)

User kuendigt Change an. Nur Findings die **direkte Konsequenz** des Changes
sind — kein allgemeines Code-Feedback.

**Struktur:** `[BREAKING]/[REQUIRED]/[RECOMMENDED]/[OPTIONAL]` Titel, Lens,
Change-Bezug (direkt/indirekt/Downstream), Fundstelle, Anpassung.

---

## `opensource`

Pre-Publication-Check — voller 5-Phasen-Workflow, aber fokussiert auf:
- Secrets-Leak-Hunt (Git-History!)
- Dependency-Supply-Chain-Risk
- Lizenz-Kompatibilitaet
- README-/LICENSE-/SECURITY.md-Vorhandensein
- Default-Config-Safety

**Struktur:** `[BLOCKER]/[RECOMMENDED]/[POLISH]` Titel, Lens, Problem, Fix.

`[BLOCKER]` = darf nicht veroffentlicht werden ohne Fix. Bei Secrets sofort
als `[BLOCKER]` + Hinweis auf Git-History-Bereinigung (`git filter-repo`, BFG).

---

## `content`

Zwei Sub-Modi: **Audit** (bestehender Content) oder **Erstellung** (neuer
Content aus Source-Material).

Bei Erstellung **nie ungeprueft committen** — immer als Vorschlag, User reviewt.

---

## Ausgabe-Abschluss (alle Modi)

Am Ende optional anbieten:

> Soll ich `gh issue create`-Kommandos zum Pasten generieren?

**Regeln dafuer bei Security-Modi:**
- `TRUE_POSITIVE`-Findings → Issue-Vorschlag
- `UNCERTAIN`-Findings → Issue-Vorschlag **mit Label `uncertain`** und
  Body-Hinweis auf manuelle Verifikation
- `FALSE_POSITIVE`-Findings → **nicht** als Issue vorschlagen

**Niemals selbst Issues erstellen** ohne explizite Bestatigung.

## Zusammenspiel mit Rollen und Agent-Output

Jeder Modus laeuft in einer **Rolle** (siehe `roles.md`: Auditor/Developer/
Architect/Reporter/Agent/**Detector**). Die Rolle entscheidet ueber die
Output-Form; der Modus entscheidet ueber die inhaltliche Ausrichtung.

Bei Rolle **Agent**: YAML-Schemas aus `agent-output.md` verwenden statt
Markdown-Templates oben.

Bei Rolle **Detector**: zusaetzlich zum normalen Finding-Output Rule-
Skeletons (Semgrep/CodeQL/Rego) generieren — siehe `roles.md` §Detector.

## Security-Modi vs Generische Modi

| Modus | Security-Workflow | Phasen |
|-------|-------------------|--------|
| `audit` | ja (Default-Breite) | 1-5 |
| `triage` | ja (ohne Gate) | 1-3 |
| `deep-audit` | ja (voll) | 1-5 + Detail |
| `variant` | ja (spezial) | 5-Schritt |
| `verify-fix` | ja (spezial) | Intake-Fix-Gate-Report |
| `bugfix` | partiell (wenn Bug Security-relevant) | 1-5 |
| `feature` | nein | Brainstorm |
| `discover` | nein | Brainstorm |
| `deploy` | ja (Infra-Config) | 1-3 |
| `custom` | ja (wenn Change Security-relevant) | Change-Impact |
| `opensource` | ja (Pre-Publish-Variante) | 1-5 |
| `content` | nein | separat |

Bei Security-Workflow-Modi: **FP-Gate ist pflicht**, ausser `triage`.
