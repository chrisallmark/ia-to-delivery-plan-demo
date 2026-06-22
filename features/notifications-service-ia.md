# Feature Plan — Add Push Notification Channel

**Source IA:** notifications-service-ia.md
**Services in scope:** notification-service

## Summary

This feature adds a push notification channel to the notification service, which currently supports only email and SMS. It introduces a `PushProvider` interface and factory, implements a Firebase Cloud Messaging (FCM) provider, persists per-user device tokens and opt-in preferences, exposes send and webhook endpoints, and emits delivery domain events. All work is contained within `notification-service` and follows the existing `BaseChannel` channel pattern.

## Phased delivery

### Phase 1 — Foundation

**Goal:** Establish the provider abstraction that all push-specific work depends on without touching the live email/SMS paths.

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-01 | Add push provider abstraction | notification-service | S | — |

**Phase rationale:** The `PushProvider` interface and factory are the integration point for every downstream SI. Merging this first keeps Phase 2 work isolated and avoids merge conflicts on `src/channels/index.ts`.

### Phase 2 — Provider & Preferences

**Goal:** Deliver the two independent building blocks — FCM implementation and user preference storage — that the send endpoint will require.

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-02 | Implement FCM push provider | notification-service | M | SI-01 |
| SI-03 | Persist notification preferences | notification-service | M | SI-01 |

**Phase rationale:** SI-02 and SI-03 both depend on SI-01 but do not depend on each other — one extends the channel abstraction, the other is a database and preferences API change. They can be developed and reviewed in parallel.

### Phase 3 — Send endpoint

**Goal:** Wire the FCM provider and preference store together behind a single public API endpoint.

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-04 | Expose push send endpoint | notification-service | S | SI-02, SI-03 |

**Phase rationale:** SI-04 must honour the opt-out state from SI-03 and dispatch via the FCM provider from SI-02. Both must be merged before this endpoint is testable end-to-end.

### Phase 4 — Delivery webhook

**Goal:** Close the notification loop by receiving FCM delivery receipts and emitting domain events with a schema consistent with the existing email/SMS events.

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-05 | Add delivery status webhook | notification-service | S | SI-04 |

**Phase rationale:** The webhook updates delivery records written by SI-04 and emits events. The event schema must match existing email/SMS patterns, which are only confirmed stable once SI-04 is merged.

## Risks and dependencies

- **Firebase Admin SDK credentials** — credentials must be provisioned in Vault for each environment before SI-02 can be tested. Confirm with the Platform team and Vault admin early.
- **Schema migration timing** — the `push_preferences` migration (SI-03) must be applied to production before SI-04 or SI-05 are deployed.
- **Event schema compatibility** — SI-05 must emit events that match the existing email/SMS domain event schema in `src/events/schema.ts`. Validate this against the schema file before implementing the webhook handler.
- **Opt-out enforcement** — SI-04 must check opt-in state from SI-03 at dispatch time. This is a correctness invariant, not just a feature — missing it would send notifications to users who have opted out.
- **BaseChannel interface contract** — do not bypass `BaseChannel`; the existing channel factory resolution depends on it. Any deviation will break the factory resolution at runtime for all channels.
