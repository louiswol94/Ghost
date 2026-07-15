# Agentic payments for `.md` URLs - implementation plan

> Review of PR [#28291](https://github.com/TryGhost/Ghost/pull/28291) and a phased plan for shipping this safely.

---

## What #28291 gets right (preserve these)

- **Dual-toggle gating** (`machinePayments` labs flag + `llms_enabled` + `machine_payments_enabled`). Publisher opt-in, not just an operator flag.
- **Reuses the existing `.md` route** from the llms.txt work instead of inventing a new endpoint. Agents get one canonical URL.
- **Symmetric discovery** - paid posts only appear in `/llms.txt` and only get `<link rel="alternate" type="text/markdown">` when payments are actually available.
- **Two adapters behind one service** (x402 + MPP). Pluggability is right; agents in the wild will negotiate whichever scheme they know.
- **`application/problem+json` for 402/503** responses - correct per RFC 7807 and the paymentauth.org problem types.
- **Deposit-address cache validation** - a replayed payment header for a different address is rejected.

---

## Problems to fix before shipping

### Correctness / production safety

**1. In-process deposit-address cache breaks on multi-instance deployments.**
`StripeDepositAddressProvider` uses `@isaacs/ttlcache` with a 5-minute TTL. Ghost(Pro) runs multiple instances - instance A mints the address, agent pays, verifies on instance B → `NoPermissionError`. Fix: use Ghost's shared cache adapter (Redis in prod). The cache value should include `{amount, currency, network}` so cross-term replays are also rejected.

**2. Concurrent challenges create duplicate PaymentIntents.**
Two agents (or one retrying) hitting the same URL milliseconds apart both miss the cache and call `stripe.paymentIntents.create`. Fix: store an in-flight `Promise<Address>` in the cache under `(stripeSecretHash, amount, currency, network)` so parallel callers share one PaymentIntent.

**3. Every 402 challenge makes a live Stripe API call.**
A crawler can exhaust Stripe's 100 req/s live-mode quota and take down every other Stripe-backed feature. Fix: add a `machinePaymentsChallenge` rate-limit bucket in `spam-prevention.js`, keyed by IP, applied to `/*.md` requests on paid entries. Same pattern as the existing members buckets.

**4. A parallel Stripe client at a different API version.**
`StripeDepositAddressProvider` instantiates `new Stripe(secretKey, { apiVersion: '2026-03-04.preview' })`, bypassing `stripe-api.js` (pinned at `2020-08-27`). Fix: add a `createMachinePaymentsDepositAddress()` method to the existing `StripeAPI` class, using Stripe SDK's per-request `{apiVersion}` override for the preview version. One Stripe instance per process; the preview version is call-scoped.

**5. No receipt is recorded.**
A payment succeeds, the content is served, and nothing is written anywhere. Publishers have no way to know which posts drove revenue, whether a request looked abusive, or how to issue a refund. Fix: emit a `MachinePaymentReceivedEvent` via `DomainEvents` on every 2xx from an adapter, carrying `{resource_id, resource_type, amount, currency, network, scheme, credential_hash, received_at}`. Persist to a `machine_payment_events` table. A basic "activity" widget in the Tiers panel can follow later.

**6. `machinePayments` is in `PUBLIC_BETA_FEATURES`.**
Almost no site can currently enroll in Stripe's machine-payments preview, so ~100% of users who flip the toggle will see 503s. Fix: move to `PRIVATE_FEATURES` (developer experiments) until Stripe's program is broadly available. Controlled rollout via allowlist on Ghost(Pro) until then.

### Architecture / maintainability

**7. Hono on the hot path.**
`X402Adapter#handle` creates a new `Hono` app and re-requires `@x402/hono`, `@x402/core/server`, `@x402/evm/exact/server` on every request. Fix: use `@x402/core` directly to produce challenge headers and verify `X-Payment`. Removes the Hono dependency entirely - one less framework in `ghost/core`.

**8. Duplicated "paid members only" predicate.**
The same `tiers.every(tier => tier.type === 'paid')` logic appears in `llms/service.js`, `ghost_head.js`, and `paid-content-provider.js`. This is the authorisation rule for the whole feature. Fix: one shared helper at `server/services/machine-payments/lib/is-paid-members-only.js`, imported everywhere.

**9. `machine_payments_amount` introduces a new numeric setting type.**
This is Ghost's first `type: 'number'` setting, requiring changes to `settings-key-type-mapper.js`, `settings.js`, snapshots, the exporter, and integrity tests - a platform-wide ripple for a feature setting. Fix: keep it as `type: 'string'` and coerce with `Number()` on read. That's the established pattern (`donations_suggested_amount`), and it avoids the settings-plumbing churn.

**10. No Stripe disconnect invalidation.**
`getStripe()` holds the client for the lifetime of the process, reactivating only on key change. A publisher disconnecting Stripe mid-flight leaves the service with a live client continuing to mint PaymentIntents on an account they thought was disconnected. Fix: subscribe to `StripeDisconnectedEvent` in the service and nullify the cached client.

### Product / policy

**11. Privacy of paid-content discovery.**
Some publishers pay-gate content to avoid indexing entirely. Turning payments on implicitly makes titles and excerpts discoverable in `/llms.txt`. The UI copy should be explicit that enabling agent payments exposes those fields to AI discovery, or expose it as a separate toggle.

**12. No graceful degradation for unenrolled Stripe accounts.**
The first `paymentIntents.create` on an unenrolled account returns a 400; the adapter returns 503 forever with no admin signal. Fix: on the first "not enrolled" error, auto-disable machine payments for the site, log a `logging.warn` with actionable text, and surface it in the admin UI (consistent with existing Stripe health surfacing).

---

## Agreed approach: deferred items

These are valid long-term concerns but deliberately out of scope for v0:

- **Pre-minted address pool.** Rate limiting + shared cache + in-flight dedupe should cover Stripe QPS at realistic traffic volumes. Add a background refill job only if Stripe rate-limit headers start showing real pressure.
- **Per-post or per-tier pricing.** Flat site-wide pricing (`machine_payments_amount`) is the correct v0 scope. Document the extension path; no new schema needed now.
- **Refund surface in admin.** Stripe supports refunds to the originating wallet address. Out of scope for v0 but must be planned before GA.
- **`optionalDependencies` for `@x402/*` / `mppx`.** Ship them as normal deps behind the labs + site toggles. Optional deps create "silent 503 because a package didn't install" failure modes that are harder to debug.

---

## Phased build

Each phase is safe to ship on its own even if the next never lands. The `machinePayments` flag stays in `PRIVATE_FEATURES` through phase 3.

### Phase 1 - Shared helpers + settings

~200 LOC, no user-visible change.

- New `server/services/machine-payments/lib/is-paid-members-only.js` - one authoritative paid-visibility predicate.
- Move `machinePayments` to `PRIVATE_FEATURES` in `shared/labs.js`.
- Migration adds `machine_payments_enabled` (`boolean`, `site`), `machine_payments_currency` (`string`, `site`), `machine_payments_price_cents` (`string`, `site`, coerced with `Number()` on read).
- Admin: Tiers panel toggle (mostly reusable from #28291, minus the numeric-setting handling).

### Phase 2 - Stripe integration + adapters

~500 LOC.

- `StripeAPI` gets `createMachinePaymentsDepositAddress({amount, currency, network})` with a per-request preview API version override.
- New `DepositAddressStore` interface with two implementations:
  - `InMemoryDepositAddressStore` - for dev/test.
  - `SharedCacheDepositAddressStore` - backed by Ghost's cache adapter (Redis in prod).
- In-flight dedupe: store returns a shared `Promise<Address>` keyed on `(stripeSecretHash, amount, currency, network)`.
- Subscribe to `StripeDisconnectedEvent`; nullify cached Stripe client on disconnect.
- Drop Hono. Rewrite `X402Adapter` using `@x402/core` directly (~40 LOC, no framework allocation per request).
- MPP adapter unchanged in shape, wired to new deposit store.
- Full unit tests against mocked deposit store and mocked scheme libs.

### Phase 3 - Route wiring + rate limiting + observability

~300 LOC.

- Wire `machinePaymentsService` through `frontend/services/proxy.js` alongside `llmsService`.
- Update `entry/markdown.ts` to call `proxy.machinePayments.handlePaidMarkdownRequest(...)`.
- New `machinePaymentsChallenge` rate-limit bucket in `spam-prevention.js`, keyed by IP, applied at the `/*.md` entry point.
- On 2xx from any adapter: emit `MachinePaymentReceivedEvent` + structured `logging.info`.
- `machine_payment_events` table + model. Small subscriber writes the row.
- Graceful degradation: on first Stripe "not enrolled" error, auto-disable and surface in admin.

### Phase 4 - Discovery + admin UI + labs promotion

~300 LOC.

- `/llms.txt` inclusion of paid entries (the `isDiscoverable` change from #28291, using the shared predicate).
- `<link rel="alternate" type="text/markdown">` in `ghost_head.js`, gated on shared predicate.
- Basic "Machine payment activity" widget in the Tiers panel (7-day count + revenue by scheme). Uses Shade Card/Table primitives.
- Update UI copy to be explicit that enabling agent payments makes paid-post titles and excerpts discoverable in `/llms.txt`.
- Move `machinePayments` from `PRIVATE_FEATURES` → `PUBLIC_BETA_FEATURES` - **only when Stripe's program is broadly enrollable**.

---

## Summary

PR #28291 has the right shape: `.md` route reuse, adapter abstraction, gated discovery, and RFC 7807 responses are all correct calls. The blocker is that it lands as a single 2,800-LOC changeset that simultaneously introduces a multi-instance cache bug, a parallel Stripe client at a different API version, no rate limiting, and zero receipt/observability.

Split across four phases behind a private-features flag - with the deposit store on shared cache, one Stripe client, rate limiting from day one, and a `MachinePaymentReceivedEvent` - this becomes something you can turn on for one hosted site, watch for a week, and expand. That's the bar for safe to ship and build upon.
