---
name: test-bimp
description: Bestimmt die nächste SemVer-Version anhand der Unreleased-Einträge im CHANGELOG. Wird verwendet, wenn der User ein Release vorbereitet oder fragt, welche Versionsnummer als Nächstes kommt.
---

# SemVer Bump

Wenn dieser Skill aktiv ist, leite die nächste Versionsnummer nach [Semantic Versioning](https://semver.org) aus dem `[Unreleased]`-Abschnitt von `CHANGELOG.md` ab.

## Vorgehen

1. Lies den `[Unreleased]`-Abschnitt aus `CHANGELOG.md`.
2. Bestimme den höchsten erforderlichen Bump nach folgenden Regeln (höchster gewinnt):
   - **MAJOR** (`X.0.0`): Es gibt einen Eintrag mit `**BREAKING:**` oder etwas in `### Removed` bzw. eine inkompatible Änderung in `### Changed`.
   - **MINOR** (`x.Y.0`): Es gibt Einträge in `### Added` oder `### Deprecated`, ohne Breaking Changes.
   - **PATCH** (`x.y.Z`): Nur `### Fixed` und/oder `### Security`.
3. Suche die letzte Version im Changelog (oberster `## [X.Y.Z]`-Header) und erhöhe entsprechend.
4. Bei `0.x.y`-Versionen gilt: Breaking Changes erhöhen MINOR statt MAJOR (Pre-1.0-Konvention).

## Output-Format

Gib genau diese Struktur zurück:

```
Aktuelle Version: X.Y.Z
Nächste Version:  X.Y.Z
Bump-Typ:         major | minor | patch
Begründung:       <ein Satz, der den entscheidenden Eintrag nennt>
```

Wenn `[Unreleased]` leer ist, sage das explizit und schlage keinen Bump vor.

## Beispiele

**Beispiel 1 — Patch:**
```
Aktuelle Version: 1.4.2
Nächste Version:  1.4.3
Bump-Typ:         patch
Begründung:       Nur Fixed-Einträge im Unreleased (Race Condition beim Speichern).
```

**Beispiel 2 — Minor:**
```
Aktuelle Version: 1.4.2
Nächste Version:  1.5.0
Bump-Typ:         minor
Begründung:       Added: OAuth2-Login für mobile Clients.
```

**Beispiel 3 — Major:**
```
Aktuelle Version: 1.4.2
Nächste Version:  2.0.0
Bump-Typ:         major
Begründung:       BREAKING: Mindest-Node-Version ist jetzt 18.
```
