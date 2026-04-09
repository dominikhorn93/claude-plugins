---
name: keep-a-changelog
description: Erstellt Changelog-Einträge im Keep-a-Changelog-Format. Wird verwendet, wenn der User ein Changelog aktualisieren möchte oder nach einem Release-Eintrag fragt.
---

# Keep a Changelog

Wenn dieser Skill aktiv ist, schreibe oder ergänze `CHANGELOG.md` nach dem Format von [keepachangelog.com](https://keepachangelog.com).

## Struktur

```
## [Unreleased]

### Added
- Neue Features

### Changed
- Änderungen an bestehender Funktionalität

### Deprecated
- Bald entfernte Features

### Removed
- Entfernte Features

### Fixed
- Bugfixes

### Security
- Sicherheits-relevante Fixes
```

## Regeln

1. Neueste Version steht **oben**.
2. Jede Version hat einen Header `## [X.Y.Z] - YYYY-MM-DD`. `[Unreleased]` darüber für noch nicht veröffentlichte Änderungen.
3. Nur die Sektionen aufnehmen, die tatsächlich Einträge haben — keine leeren Headings.
4. Einträge sind benutzerorientiert und im Past Tense ("Added dark mode toggle"), nicht commit-orientiert.
5. Bei Breaking Changes Sektion `### Changed` mit explizitem Hinweis "**BREAKING:**" am Zeilenanfang.

## Beispiel

```markdown
## [Unreleased]

### Added
- Dark mode toggle in der Sidebar.

## [1.2.0] - 2026-04-09

### Added
- OAuth2-Login für mobile Clients.

### Fixed
- Race Condition beim parallelen Speichern von Drafts.

### Changed
- **BREAKING:** Mindest-Node-Version ist jetzt 18.
```
