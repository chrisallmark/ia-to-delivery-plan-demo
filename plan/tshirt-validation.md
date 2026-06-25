# T-shirt Validation

| IA                        | SI    | Service              | IA Size | Story Estimate | Δ |
| ------------------------- | ----- | -------------------- | ------- | -------------- | - |
| order-service-ia          | SI-01 | order-service        | M       | M              | = |
| order-service-ia          | SI-02 | order-service        | M       | S              | ↓ |
| order-service-ia          | SI-03 | order-service        | S       | S              | = |
| order-service-ia          | SI-04 | order-service        | S       | S              | = |
| order-service-ia          | SI-05 | order-service        | M       | M              | = |
| notifications-service-ia  | SI-01 | notification-service | S       | S              | = |
| notifications-service-ia  | SI-02 | notification-service | M       | M              | = |
| notifications-service-ia  | SI-03 | notification-service | M       | M              | = |
| notifications-service-ia  | SI-04 | notification-service | S       | S              | = |
| notifications-service-ia  | SI-05 | notification-service | S       | S              | = |

## Pattern analysis

**Predominantly matched — one story down-size.** Nine of ten stories align with the IA estimates, which indicates consistent sizing methods across both IAs. The IA estimates are broadly reliable for planning purposes.

The single down-size (SI-02) is isolated and reasoned — it does not indicate a systematic bias. The IA estimates are neither universally conservative nor aggressive.

## Outliers

**order-service-ia SI-02 (M → S, ↓ 1 band):** The IA estimated M for `InstalmentGatewayClient`. Story analysis found that the Stripe SDK abstracts the majority of the complexity — the implementation is adding a class to an existing `gateway.py` file using a credential pattern already in place. The main uncertainty (Stripe Instalments parameter shape) is a documentation lookup, not engineering work. Sized to S. The IA estimate may have factored in integration testing time against a live Stripe account; if so, that time is better tracked as a prerequisite/coordination task rather than story scope.
