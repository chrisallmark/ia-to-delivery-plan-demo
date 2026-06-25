# notification-service

## Overview

REST service that dispatches transactional notifications across email, SMS, and
(soon) push channels. Owned by the Platform team.

## Tech stack

- **Runtime:** Node.js 20, TypeScript 5
- **Framework:** Fastify 4
- **Database:** PostgreSQL 15 via `pg` (no ORM — raw SQL with typed query
  helpers in `src/db/`)
- **Build:** `pnpm build` (tsc → `dist/`)
- **Test:** `pnpm test` (Vitest, co-located `*.test.ts` files)
- **Lint:** `pnpm lint` (ESLint + Prettier)

## Project layout

```
src/
  channels/         # One subfolder per notification channel
    email/
    sms/
  db/               # Query helpers and migration runner
  routes/           # Fastify route registrations
  preferences/      # User opt-in/opt-out logic
  index.ts          # Entry point
migrations/         # Numbered SQL migration files (e.g. 003_add_push_prefs.sql)
```

## Key conventions

- **Channel pattern:** each channel lives in `src/channels/<name>/` and must
  export a class implementing `src/channels/BaseChannel.ts`. The channel
  factory in `src/channels/index.ts` resolves by name at runtime.
- **Database migrations:** add a new numbered `.sql` file to `migrations/` and
  register it in `src/db/migrate.ts`. Migrations run on service startup in
  non-production environments; production requires a manual apply step.
- **Secrets:** all credentials are read from environment variables injected by
  Vault at deploy time. Do not hard-code or commit secrets.
- **Domain events:** after a notification is dispatched or its status updated,
  emit an event via `src/events/emit.ts`. The event schema is defined in
  `src/events/schema.ts` — all channels must use the same schema.

## External dependencies

- **SendGrid** — email dispatch (`src/channels/email/`)
- **Twilio** — SMS dispatch (`src/channels/sms/`)
- **Firebase Admin SDK** — not yet integrated; will be used for push dispatch
- **Vault** — secrets at runtime (`SENDGRID_API_KEY`, `TWILIO_AUTH_TOKEN`, etc.)

## Programme touchpoints (Customer Comms Upgrade — Push epic)

This service is the sole target for the push notification epic. The work
involves:

1. Adding `src/channels/push/` following the existing channel pattern
2. Adding a `push_preferences` table via a new migration
3. Adding `POST /notifications/push` and `POST /notifications/push/status`
   route registrations in `src/routes/`
4. Emitting delivery events via the existing `src/events/emit.ts` mechanism

The existing `BaseChannel` interface is the correct abstraction point — do not
bypass it.
