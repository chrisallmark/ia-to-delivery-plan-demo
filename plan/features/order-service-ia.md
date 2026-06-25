# Feature Plan — Add Instalment Payment Support

**Source IA:** order-service-ia.md
**Services in scope:** order-service

## Summary

This epic adds instalment payment support to the order service, enabling customers to split a single order into a series of scheduled payments via Stripe Instalments. The work introduces a new data model (instalment plans and schedules), a domain-layer calculator, a Stripe SDK integration layer, a new checkout endpoint, and an inbound webhook handler to process payment lifecycle events. The feature is entirely within `order-service`; downstream consumers (fulfilment, comms) are notified via domain events.

## Phased delivery

### Phase 1 — Foundation

**Goal:** Establish the data layer that all subsequent instalment work depends on.

| SI    | Headline                        | Service       | Track      | T-shirt (IA) | Depends On |
| ----- | ------------------------------- | ------------- | ---------- | ------------ | ---------- |
| SI-01 | Add instalment plan data model  | order-service | full-stack | M            | —          |

**Phase rationale:** SI-01 is the sole root node in the dependency graph. The `instalment_plans` and `instalment_schedules` tables, domain models, and repository layer must exist before the gateway, calculator, or endpoint can be built.

### Phase 2 — Domain & Gateway

**Goal:** Implement the pure business logic and the Stripe SDK integration independently of each other.

| SI    | Headline                                    | Service       | Track      | T-shirt (IA) | Depends On |
| ----- | ------------------------------------------- | ------------- | ---------- | ------------ | ---------- |
| SI-03 | Add instalment calculation service          | order-service | full-stack | S            | SI-01      |
| SI-02 | Integrate payment gateway for instalments   | order-service | full-stack | M            | SI-01      |

**Phase rationale:** SI-03 and SI-02 are both unblocked after SI-01. SI-03 ships first (smaller; pure domain logic with no external calls) so the calculator can be unit-tested before the gateway integration depends on the same schedule types. Both must be merged before the checkout endpoint can be wired.

### Phase 3 — Checkout Endpoint

**Goal:** Expose the public instalment checkout API that combines the calculator and gateway.

| SI    | Headline                            | Service       | Track      | T-shirt (IA) | Depends On      |
| ----- | ----------------------------------- | ------------- | ---------- | ------------ | --------------- |
| SI-04 | Expose instalment checkout endpoint | order-service | full-stack | S            | SI-02, SI-03    |

**Phase rationale:** SI-04 is a pure integration point — it calls `InstalmentCalculator` and `InstalmentGatewayClient` and persists the result. Both dependencies must be stable before this is meaningful to build or test.

### Phase 4 — Webhooks

**Goal:** Close the async payment lifecycle loop by handling Stripe-initiated events.

| SI    | Headline                              | Service       | Track      | T-shirt (IA) | Depends On |
| ----- | ------------------------------------- | ------------- | ---------- | ------------ | ---------- |
| SI-05 | Handle instalment payment webhooks    | order-service | full-stack | M            | SI-04      |

**Phase rationale:** Webhook handling requires the instalment schedule to already exist in the database (created by SI-04). Domain events emitted here (`instalment.paid`, `instalment.failed`) must extend `events/schema.py` — this schema extension is a dependency of the webhook handler, not of the endpoint.

## Risks and dependencies

- **Stripe account prerequisite:** Stripe Instalments is a separate product feature. The account must be enabled before SI-02 development can be integration-tested. Confirm with the Payments team before starting Phase 2.
- **Webhook registration per environment:** The `POST /webhooks/stripe/instalment` endpoint must be registered in the Stripe dashboard for each deployment environment (dev, staging, prod). Missing registration will cause silent webhook loss.
- **Event schema extension:** `events/schema.py` must be extended with `instalment.paid` and `instalment.failed` types before SI-05 is implemented. If this is missed it will block Phase 4.
- **Penny-rounding match:** SI-03 requires the calculator output to match Stripe's figures to the nearest penny. A discrepancy would cause silent checkout rejections from the gateway.
