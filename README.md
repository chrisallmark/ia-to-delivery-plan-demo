# ia-to-delivery-plan-demo

Demo application for Claude /ia-to-delivery-plan skill.

## What's in this repo

Two sample inputs:

- `notifications-service-ia.md` — a sample feature-plan output (illustrates Phase 3 output)
- `order-service-ia.md` — a complete Impact Assessment with a valid Service Impact table (used as skill input)

Stub service repos (`order-service/`, `notification-service/`) each carry a `CLAUDE.md` that the skill reads for implementation hints.

The skill itself is defined in `ia-to-delivery-plan/SKILL.md`.

## Prerequisites

[Claude Code](https://claude.ai/code) must be installed and running. See the Claude Code docs for install instructions.

## Running the skill

Run `/ia-to-delivery-plan` in Claude Code from the repo root. The skill auto-discovers IAs, reads the service `CLAUDE.md` files, and writes `plan/features/`, `plan/stories/`, `plan/tshirt-validation.md`, `plan/dependency-graph.md`, and `plan/delivery-plan.md`.
