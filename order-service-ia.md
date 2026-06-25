# Impact Assessment — Order Epic
**Programme:** Customer Comms Upgrade
**Epic:** Add Instalment Payment Support

---

## Service Impact Table

| SI No | Component | Headline Req | Change Description | Dependent On | Assumed Team | Req Devs | T-Shirt Size | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| SI-01 | order-service | Add instalment plan data model | Introduce `instalment_plans` and `instalment_schedules` tables. Add corresponding domain models and repository layer. | — | Payments | 1 | M | Schema migration required. Used by all downstream SIs |
| SI-02 | order-service | Integrate payment gateway for instalments | Add a `InstalmentGatewayClient` that wraps the existing Stripe SDK to create payment intents with instalment schedules. Read gateway credentials from environment. | SI-01 | Payments | 1 | M | Stripe Instalments is a separate product feature — confirm account is enabled before dev starts |
| SI-03 | order-service | Add instalment calculation service | Implement `InstalmentCalculator` that derives a repayment schedule from order total, term length and APR. Must match the figures the gateway returns to the nearest penny. | SI-01 | Payments | 1 | S | Pure domain logic — no external calls. Add unit tests with rounding edge cases |
| SI-04 | order-service | Expose instalment checkout endpoint | Add `POST /orders/{id}/checkout/instalment` that validates the request, runs the calculator, creates a gateway payment intent and persists the schedule. Returns 201 with the schedule and gateway reference. | SI-02, SI-03 | Payments | 1 | S | Must reject if the order is already paid or cancelled |
| SI-05 | order-service | Handle instalment payment webhooks | Add `POST /webhooks/stripe/instalment` to receive `payment_intent.succeeded` and `payment_intent.payment_failed` events. Update instalment schedule status and emit a domain event per instalment. | SI-04 | Payments | 1 | M | Webhook endpoint must be registered in the Stripe dashboard per environment. Verify signature using Stripe-Signature header |
