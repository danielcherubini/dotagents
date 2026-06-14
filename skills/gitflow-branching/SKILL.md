---
name: gitflow-branching
description: Use when starting any new work that requires creating a git branch
---

# Git Branching

Trunk-based development: `main → feature branch → main`

## Branch types

| Branch | Purpose | From | Merges to |
|--------|---------|------|-----------|
| `main` | Production-ready | - | - |
| `feature/*` | New features | `main` | `main` |
| `bugfix/*` | Bug fixes | `main` | `main` |

## Naming

- `feature/<ticket-id>-<description>`
- `bugfix/<ticket-id>-<description>`

## Starting a feature

```bash
git checkout main && git pull origin main
git checkout -b feature/<ticket-id>-<description>
```

## Starting a bugfix

```bash
git checkout main && git pull origin main
git checkout -b bugfix/<ticket-id>-<description>
```

## Finishing a feature or bugfix

```bash
git push -u origin feature/<ticket-id>-<description>
# Create PR to main
```

## Rules

- **Always branch from `main`** — never from another feature branch
- **Always PR back to `main`** — never to `develop` or other branches
- **Never create a `develop` branch** — it does not exist in this workflow
- Keep branches short-lived — merge back to main frequently