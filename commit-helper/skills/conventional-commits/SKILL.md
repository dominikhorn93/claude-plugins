---
name: conventional-commits
description: Erstellt Commit-Messages im Conventional-Commits-Format. Wird verwendet, wenn der User einen Commit erstellen möchte oder nach einer Commit-Message fragt.
---

# Conventional Commits

Wenn dieser Skill aktiv ist, schreibe Commit-Messages nach dem Conventional-Commits-Standard.

## Format

```
<type>(<scope>): <subject>

<body>
```

## Erlaubte Types

- `feat`: neues Feature
- `fix`: Bugfix
- `docs`: nur Dokumentation
- `refactor`: Code-Umstrukturierung ohne Verhaltensänderung
- `test`: Tests hinzugefügt oder angepasst
- `chore`: Build, Tooling, Dependencies

## Regeln

1. `subject` im Imperativ ("add" statt "added"), kleingeschrieben, ohne Punkt am Ende, max. 72 Zeichen.
2. `scope` ist optional und beschreibt den betroffenen Bereich (z. B. `auth`, `api`).
3. `body` erklärt das *Warum*, nicht das *Was* — der Diff zeigt schon das Was.
4. Bei Breaking Changes: `!` nach Type/Scope **und** `BREAKING CHANGE:` im Body.

## Beispiele

```
feat(auth): add OAuth2 login flow

Replaces the legacy session cookie flow because the mobile client
cannot store cookies reliably.
```

```
fix!: drop support for Node 16

BREAKING CHANGE: Node 18 is now the minimum supported version.
```
