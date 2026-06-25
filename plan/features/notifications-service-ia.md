# Feature Plan — Add Push Notification Channel

**Source IA:** notifications-service-ia.md
**Services in scope:** notification-service

## Summary

This epic adds a push notification channel to the notification service, allowing transactional messages to be dispatched to mobile devices via Firebase Cloud Messaging (FCM). The work follows the existing channel pattern established for email and SMS: a `PushProvider` interface backed by a `FirebasePushProvider` implementation, a `push_preferences` table for device tokens and opt-in state, new send and status webhook endpoints, and domain events emitted on dispatch and delivery. The feature is entirely within `notification-service`.

## Phased delivery

### Phase 1 — Abstraction

**Goal:** Establish the `PushProvider` interface and factory that all push-specific work builds on.

| SI    | Headline                       | Service              | Track      | T-shirt (IA) | Depends On |
| ----- | ------------------------------ | -------------------- | ---------- | ------------ | ---------- |
| SI-01 | Add push provider abstraction  | notification-service | full-stack | S            | —          |

**Phase rationale:** SI-01 is the root node for the entire push epic. The `PushProvider` interface and factory update must land first; without it, no concrete provider or route can be wired correctly.

### Phase 2 — Provider & Preferences

**Goal:** Implement the Firebase provider and the user preference storage in parallel (sequential for solo).

| SI    | Headline                             | Service              | Track      | T-shirt (IA) | Depends On |
| ----- | ------------------------------------ | -------------------- | ---------- | ------------ | ---------- |
| SI-02 | Implement FCM push provider          | notification-service | full-stack | M            | SI-01      |
| SI-03 | Persist notification preferences     | notification-service | full-stack | M            | SI-01      |

**Phase rationale:** SI-02 and SI-03 are both unblocked after SI-01 and are logically independent of each other. Ordered by SI number (equal size). Both must be merged before the send endpoint can honour opt-out state and dispatch via the correct provider.

### Phase 3 — Send Endpoint

**Goal:** Expose the `POST /notifications/push` endpoint that ties together the provider and preferences.

| SI    | Headline                     | Service              | Track      | T-shirt (IA) | Depends On      |
| ----- | ---------------------------- | -------------------- | ---------- | ------------ | --------------- |
| SI-04 | Expose push send endpoint    | notification-service | full-stack | S            | SI-02, SI-03    |

**Phase rationale:** SI-04 is purely integration work — it resolves the provider via factory (SI-02) and validates opt-in state (SI-03). Neither dependency is meaningful to mock at this stage; both should be merged and deployed first.

### Phase 4 — Status Webhook

**Goal:** Close the delivery loop by receiving and persisting FCM delivery receipts.

| SI    | Headline                    | Service              | Track      | T-shirt (IA) | Depends On |
| ----- | --------------------------- | -------------------- | ---------- | ------------ | ---------- |
| SI-05 | Add delivery status webhook | notification-service | full-stack | S            | SI-04      |

**Phase rationale:** SI-05 requires delivery records to exist (created by SI-04 dispatches). The domain event schema emitted here must match the existing email/SMS event contract defined in `src/events/schema.ts`.

## Risks and dependencies

- **Firebase Admin SDK credentials:** FCM credentials must be available in Vault for each environment before SI-02 can be integration-tested. Credential rotation per environment (per IA notes) must be coordinated with Platform Ops before Phase 2 starts.
- **Event schema conformance:** SI-05 domain events must match the email/SMS schema in `src/events/schema.ts`. A schema mismatch would break any consumer that unifies notification events (e.g. analytics, customer comms).
- **Opt-out enforcement:** SI-04 must check preferences from SI-03 before dispatching. A missed check could result in notifications sent to users who have opted out — a compliance risk.
- **BaseChannel contract:** All push work must implement `BaseChannel.ts`. Bypassing the interface would break the factory pattern and require future rework.
