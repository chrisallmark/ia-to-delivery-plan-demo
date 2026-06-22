# Feature Plan — Add Instalment Payment Support

**Source IA:** order-service-ia.md
**Services in scope:** order-service

## Summary

This feature adds end-to-end instalment payment support to the order service. It introduces a new data model for instalment plans and schedules, wires in a Stripe Instalments gateway integration, exposes a new checkout endpoint that returns a payment schedule, and handles incoming Stripe webhook events to update instalment status and emit domain events. All work is contained within `order-service` and delivered in four sequential phases.

## Phased delivery

### Phase 1 — Foundation

**Goal:** Introduce the instalment data model so all downstream work has a stable schema and repository layer to build on.

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-01 | Add instalment plan data model | order-service | M | — |

**Phase rationale:** The `instalment_plans` and `instalment_schedules` tables plus their domain models and repository layer are prerequisites for every other SI. Nothing else can be coded or tested without this foundation.

### Phase 2 — Gateway & Calculator

**Goal:** Build the two independent domain components — payment gateway wrapper and repayment calculator — that the checkout endpoint will compose.

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-02 | Integrate payment gateway for instalments | order-service | M | SI-01 |
| SI-03 | Add instalment calculation service | order-service | S | SI-01 |

**Phase rationale:** SI-02 and SI-03 both depend on SI-01 but are independent of each other — one is a Stripe SDK extension, the other is pure domain logic. They can be developed and reviewed in parallel within this phase.

### Phase 3 — Checkout endpoint

**Goal:** Expose the public checkout API that composes the calculator and gateway into a single transactional operation.

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-04 | Expose instalment checkout endpoint | order-service | S | SI-02, SI-03 |

**Phase rationale:** SI-04 requires both SI-02 (gateway client) and SI-03 (calculator) to be merged before it can be implemented and tested end-to-end.

### Phase 4 — Webhooks

**Goal:** Close the payment loop by handling Stripe delivery receipts and emitting domain events for downstream consumers.

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-05 | Handle instalment payment webhooks | order-service | M | SI-04 |

**Phase rationale:** The webhook handler updates instalment schedule records created by SI-04 and emits events using the `instalment.paid` / `instalment.failed` schemas that must be defined once SI-04's data model is finalised.

## Risks and dependencies

- **Stripe Instalments product enablement** — the Stripe account must have Instalments enabled before SI-02 can be tested in any environment. Confirm with the Payments team before starting Phase 2.
- **Schema migration timing** — production migrations are applied manually; SI-01's migration must be applied to production before any Phase 3 or 4 code is deployed.
- **Event schema extension** — `events/schema.py` must be extended with `instalment.paid` and `instalment.failed` types (noted in CLAUDE.md). This should be completed during Phase 4 before the webhook handler is merged.
- **Webhook registration** — the `POST /webhooks/stripe/instalment` endpoint must be registered in the Stripe dashboard for each environment (dev, staging, prod). This is an operational step outside the code change and must not be forgotten before SI-05 goes live.
- **Rounding correctness** — SI-03 requires the calculator output to match Stripe figures to the nearest penny. Failure here will cause SI-04 and SI-05 discrepancies. Prioritise rounding edge-case unit tests.
