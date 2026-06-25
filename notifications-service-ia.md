# Impact Assessment — Notifications Epic

**Programme:** Customer Comms Upgrade
**Epic:** Add Push Notification Channel

---

## Service Impact Table

| SI No | Component            | Headline Req                     | Change Description                                                                                                                                | Dependent On | Assumed Team | Req Devs | T-Shirt Size | Notes                                                    |
| ----- | -------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------ | -------- | ------------ | -------------------------------------------------------- |
| SI-01 | notification-service | Add push provider abstraction    | Introduce a `PushProvider` interface and a factory that resolves the correct provider at runtime. Current code hard-codes email and SMS channels. | —            | Platform     | 1        | S            | Foundational — all other SIs depend on this              |
| SI-02 | notification-service | Implement FCM push provider      | Implement `FirebasePushProvider` satisfying the `PushProvider` interface. Requires Firebase Admin SDK credentials from Vault.                     | SI-01        | Platform     | 1        | M            | Credentials must be rotated per environment              |
| SI-03 | notification-service | Persist notification preferences | Add a `push_preferences` table to store device tokens and per-user opt-in state. Wire into the existing preferences API.                          | SI-01        | Platform     | 1        | M            | Schema migration required                                |
| SI-04 | notification-service | Expose push send endpoint        | Add `POST /notifications/push` endpoint. Validates payload, resolves provider via factory, dispatches. Returns 202 Accepted.                      | SI-02, SI-03 | Platform     | 1        | S            | Must honour opt-out state from SI-03                     |
| SI-05 | notification-service | Add delivery status webhook      | Receive FCM delivery receipts via `POST /notifications/push/status`. Update delivery record and emit a domain event.                              | SI-04        | Platform     | 1        | S            | Event schema must match existing email/SMS domain events |
