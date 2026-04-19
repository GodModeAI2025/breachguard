# Updater / Self-Sync

Diese Skill ist aus RepoLens-Commit `7c630ea` erzeugt. Upstream kann
sich weiterentwickeln (neue Lenses, geaenderte Mode-Templates, zusaetzliche
Domains). Dieses Dokument erklaert, wie der Skill sich selbst aktualisieren
kann.

## Quick Check (kein Rebuild)

**Trigger:** User fragt "gibt es Updates?", "check for RepoLens updates",
"ist der Skill noch aktuell?"

**Vorgehen:**

1. Fetch der RepoLens HEAD-Commit-SHA:
   ```bash
   git ls-remote https://github.com/TheMorpheus407/RepoLens HEAD
   ```
   oder via GitHub-API:
   ```
   GET https://api.github.com/repos/TheMorpheus407/RepoLens/commits/main
   ```

2. Vergleich mit eingebauter Version: `7c630ea` (siehe SKILL.md
   Metadata-Block)

3. Falls Delta existiert:
   - Fetch der geaenderten Files via GitHub API:
     ```
     GET /repos/TheMorpheus407/RepoLens/compare/7c630ea...HEAD
     ```
   - Report generieren: wie viele Lens-Files neu/geaendert/geloescht, ob
     `config/domains.json` sich geaendert hat, ob Mode-Templates angepasst
     wurden

4. **Kein Rebuild** in diesem Modus — nur Info: "Upstream hat X neue Lenses,
   Y geaendert. Willst du einen Rebuild?"

## Full Rebuild

**Trigger:** User sagt "update the skill", "rebuild", "neue Version bauen",
"pull upstream"

**Vorgehen:**

1. Klone Upstream:
   ```bash
   git clone --depth=1 https://github.com/TheMorpheus407/RepoLens /tmp/repolens-upstream
   ```

2. Lies neue Struktur:
   - `config/domains.json` -> Liste der Domains/Lenses
   - `prompts/lenses/<domain>/<lens>.md` -> Lens-Content

3. Regeneriere pro Domain `references/lenses/<domain-id>.md` aus den neuen
   Lens-Files. Struktur beibehalten: Titel, TOC, pro Lens `## \`<id>\` — <name>`
   + Specialist Role + Body.

4. **Mode-Templates NICHT automatisch uebernehmen**. Diese sind im Skill
   bewusst adaptiert (autonomous-agent-Direktiven entfernt). Wenn Upstream die
   Templates aendert: Delta anzeigen, User entscheidet ob relevante Aenderungen
   uebernommen werden (manuelle Adaption noetig).

5. Update Metadata in SKILL.md:
   - `Version:` hochzaehlen (semver: Minor bei neuen Domains/Lenses, Patch bei
     Content-Updates)
   - `Source:` auf neuen Commit-SHA

6. Regeneriere `references/compliance-mappings.md` wenn neue compliance-Lenses
   dazukommen (nicht automatisch — User muss Mapping zu Regulierungen ergaenzen).

7. **Validierung nach Rebuild:**
   ```
   python3 /mnt/skills/examples/skill-creator/scripts/quick_validate.py /path/to/repolens
   ```

8. Neue `.skill`-ZIP paketieren:
   ```bash
   cd repolens/.. && zip -r repolens.skill repolens/
   ```

## Partial Rebuild

**Trigger:** User sagt "nur Security-Lenses aktualisieren", "refresh compliance
domain"

**Vorgehen:**
1. Fetch nur die spezifische `prompts/lenses/<domain>/`-Files aus Upstream
2. Regeneriere nur `references/lenses/<domain>.md`
3. Metadata-Update auf neuen Commit-SHA fuer diese Domain (optional separat
   tracken)

## Cascading Updates

Wenn eine Domain sich aendert, koennen abhaengige Teile mit-aktualisiert werden:

| Geaenderte Quelle | Cascade zu |
|-------------------|------------|
| `compliance`-Lenses | `references/compliance-mappings.md` pruefen — neue Lenses brauchen Mapping |
| `security`-Lenses | `references/code-examples.md` pruefen — ggf. neue Sprach-Beispiele |
| Mode-Templates | SKILL.md Routing-Regeln pruefen |
| `domains.json` | SKILL.md Domain-Tabelle regenerieren |

## Nicht-automatisierbare Schritte

- **Description in SKILL.md** — wenn neue Domains dazukommen, muss User neue
  Trigger-Keywords in die 1024-char-Description einpassen
- **Compliance-Mappings** — neue Lenses zu Regulierungen zu mappen ist
  fachliche Entscheidung, nicht mechanisch
- **Code-Beispiele** — fuer neue Finding-Typen Bad/Good-Snippets zu schreiben
  ist kuratierte Arbeit

## Historie tracken

Optional: `CHANGELOG.md` im Skill-Root mit jedem Rebuild fortschreiben, Format
nach [Keep a Changelog](https://keepachangelog.com/). Eintrag pro Rebuild mit:
- neuer Version
- Datum
- Source-Commit (alt -> neu)
- neu/geaendert/entfernt (Lenses, Domains, Modes)
