# order-service

## Overview

REST service that manages the full order lifecycle — basket, checkout, payment
and fulfilment handoff. Owned by the Payments team.

## Tech stack

- **Runtime:** Python 3.12
- **Framework:** FastAPI 0.111
- **Database:** PostgreSQL 15 via `asyncpg` with hand-written SQL (no ORM)
- **Payment gateway:** Stripe Python SDK (`stripe` 9.x)
- **Build:** `pip install -e ".[dev]"` (standard `pyproject.toml`)
- **Test:** `pytest` with `pytest-asyncio`; co-located `test_*.py` files
- **Lint:** `ruff check .` + `ruff format .`

## Project layout

```
src/
  orders/
    router.py          # FastAPI route registrations for /orders/*
    models.py          # SQLAlchemy-free domain models (dataclasses)
    repository.py      # Async DB query helpers
  payments/
    router.py          # Route registrations for /webhooks/stripe/*
    gateway.py         # Stripe SDK wrapper
    calculator.py      # Pure repayment schedule logic
  db/
    migrations/        # Numbered .sql files applied at startup in non-prod
    connection.py      # asyncpg pool setup
  events/
    emit.py            # Domain event publisher (wraps internal message bus)
    schema.py          # Pydantic event schemas — all emitters must use these
  main.py              # FastAPI app factory
```

## Key conventions

- **Repository pattern:** all DB access goes through `*Repository` classes in
  `repository.py`. Raw SQL only — no query builder.
- **Domain events:** after any state change (payment succeeded, payment failed,
  order fulfilled) call `events.emit.publish(event)`. The event schema is
  versioned in `events/schema.py` — do not invent a new schema.
- **Database migrations:** add a new numbered `.sql` file to `db/migrations/`
  and register it in `db/migrations/__init__.py`. Production migrations are
  applied manually via the `migrate` CLI command.
- **Secrets:** all credentials (`STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`,
  `DATABASE_URL`) are injected as environment variables. Never commit secrets.
- **Stripe webhooks:** signature verification is mandatory — use
  `stripe.Webhook.construct_event(payload, sig_header, secret)` before
  processing any event. Return `400` on verification failure.

## External dependencies

- **Stripe** — payment intents and webhooks (`payments/gateway.py`)
- **Internal message bus** — domain event publishing (`events/emit.py`)
- **Fulfilment service** — consumes `order.paid` domain events (downstream,
  not called directly)

## Programme touchpoints (Customer Comms Upgrade — Instalment epic)

This service requires four areas of change:

1. New migration adding `instalment_plans` and `instalment_schedules` tables
   (`db/migrations/`)
2. New `InstalmentCalculator` in `payments/calculator.py`
3. Extended `payments/gateway.py` with a `create_instalment_intent` method
4. New routes in `orders/router.py` (`POST /orders/{id}/checkout/instalment`)
   and `payments/router.py` (`POST /webhooks/stripe/instalment`)

The existing `events/schema.py` must be extended with `instalment.paid` and
`instalment.failed` event types before the webhook handler is implemented.
