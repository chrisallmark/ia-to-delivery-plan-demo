# T-shirt Validation

| IA | SI | Service | IA Size | Story Estimate | Δ |
|----|----|---------|---------|---------------|---|
| order-service-ia | SI-01 | order-service | M | M | = |
| order-service-ia | SI-02 | order-service | M | M | = |
| order-service-ia | SI-03 | order-service | S | S | = |
| order-service-ia | SI-04 | order-service | S | S | = |
| order-service-ia | SI-05 | order-service | M | M | = |
| notifications-service-ia | SI-01 | notification-service | S | S | = |
| notifications-service-ia | SI-02 | notification-service | M | M | = |
| notifications-service-ia | SI-03 | notification-service | M | M | = |
| notifications-service-ia | SI-04 | notification-service | S | S | = |
| notifications-service-ia | SI-05 | notification-service | S | S | = |

## Pattern analysis

**All match** — IA and story estimates are aligned across both epics. The IA sizing is well-calibrated relative to the implementation complexity visible in the service `CLAUDE.md` files.

A few observations worth noting even though the sizes agree:

- **SI-02 (order-service — gateway integration):** The M estimate is correct but sits at the upper end of M. Stripe Instalments being a separate product feature (with separate account enablement) is a risk that could push this to L if the team encounters unexpected API behaviour or account provisioning delays. Worth flagging before sprint planning.
- **SI-05 (order-service — webhooks):** M is appropriate. The idempotency and signature verification requirements are correctness constraints that are easy to underestimate. The IA flagged these explicitly, which validates the sizing.
- **SI-03 (notifications-service — preferences):** M is right. The unresolved multi-device design question (one device token vs many per user) could scope-creep this if not clarified before development starts.

## Outliers

No outliers — all story estimates match their IA sizes exactly. There are no cases where |Δ| > 1 size band.
