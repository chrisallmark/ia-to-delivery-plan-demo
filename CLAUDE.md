# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a demo repository for the `/ia-to-delivery-plan` Claude Code skill. It contains sample Impact Assessment (IA) documents and stub service repositories that illustrate the inputs and expected behaviour of that skill.

## Repository structure

```
/                           # Root — contains IA documents and this file
  order-service-ia.md       # IA for the instalment payment epic (order-service)
  notifications-service-ia.md  # Feature plan output for the push notification epic
  order-service/            # Stub Python service (FastAPI + PostgreSQL)
  notification-service/     # Stub Node.js service (Fastify + PostgreSQL)
  ia-to-delivery-plan/      # Skill definition (SKILL.md)
```

## The `/ia-to-delivery-plan` skill

Defined in `ia-to-delivery-plan/SKILL.md`. Running `/ia-to-delivery-plan` triggers a seven-phase method:

| Phase | Output |
|-------|--------|
| 0 | Discover IA documents (root folder) and repo `CLAUDE.md` files |
| 1 | Extract Service Impact table rows into a flat list |
| 2 | Group SIs into sequential delivery phases via topological sort |
| 3 | Write `features/<ia-slug>.md` per IA |
| 4 | Write `stories/<ia-slug>/<si-no>-<component>.md` per SI with independent T-shirt estimates |
| 5 | Write `tshirt-validation.md` comparing IA vs story estimates |
| 6 | Write `dependency-graph.md` (prose + Mermaid) |
| 7 | Write `delivery-plan.md` (Mermaid Gantt + critical path + key risks) |

**Key rules for the skill:**
- Only files with a valid Service Impact table (`SI No | Component | Headline Req | Change Description | Dependent On | Assumed Team | Req Devs | T-Shirt Size | Notes`) are processed as IAs.
- Story T-shirt estimates must be independent — do not copy IA values without reasoning.
- Delivery plan durations use story estimates, not IA estimates (XS=2d, S=4d, M=6d, L=10d, XL=16d).
- Missing repos do not block the run; flag them and offer to continue.

## Service stubs

Each service subfolder contains a `CLAUDE.md` describing its tech stack, layout, and conventions. Read these before writing implementation hints in story files.

- **`order-service/`** — Python 3.12, FastAPI, asyncpg, Stripe SDK. Tests via `pytest`. Lint via `ruff`.
- **`notification-service/`** — Node.js 20, TypeScript, Fastify, `pg`. Tests via `pnpm test` (Vitest). Lint via `pnpm lint`.
