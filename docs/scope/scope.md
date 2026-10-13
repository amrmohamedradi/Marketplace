# Scope: Multi-vendor Marketplace

A full stack marketplace where every signed-up person is a buyer by default and can switch on a seller store instantly, with no approval. The product's real value is the engineering underneath it: no overselling the last item under concurrent buyers, Stripe webhooks that never pay or confirm twice, a per-seller sub-order state machine, and live SignalR updates. One owner runs the whole site.

**Build approach:** Tracer Bullet (vertical slices; each feature built end to end through DB, API, and UI, working at every step).
**Workflow:** GA (after `/develop`, run `/check verify`, then `/test`, then a fresh-model `/check review`, then `/document`; most features need a spec). Chosen because this is a real production launch touching money, auth, and data integrity.

_These are recommendations to keep your build orderly, not requirements. Skip anything that does not fit: if you already know how to build a feature, use `/develop` and skip `/architect`. You decide when a feature is `done`._

## At a glance

| # | Feature | Phase | Status |
|---|---------|-------|--------|
| 1 | Stack & architecture | Foundation | planned |
| 2 | Coding standards & tooling | Foundation | planned |
| 3 | Data model | Foundation | planned |
| 4 | Auth & identity | Foundation | planned |
| 5 | Design system & UI foundation | Foundation | planned |
| 6 | Product catalog: listing | Slice 1 | planned |
| 7 | Product detail & reviews | Slice 1 | planned |
| 8 | Seller products & commission preview | Slice 1 | planned |
| 9 | Cart (optimistic) | Slice 2 | planned |
| 10 | Checkout & Stripe payment | Slice 2 | planned |
| 11 | Stock reservation & optimistic locking | Slice 3 | planned |
| 12 | Webhook, outbox & payment idempotency | Slice 4 | planned |
| 13 | Order lifecycle & fulfillment | Slice 5 | planned |
| 14 | Refunds | Slice 5 | planned |
| 15 | SignalR real-time | Slice 6 | planned |
| 16 | Seller dashboard & stats | Slice 6 | planned |
| 17 | Product search & filtering | Slice 7 (stretch) | planned |
| 18 | Email notifications | Slice 7 (stretch) | planned |
| 19 | Cloud deploy & CI/CD | Slice 7 (stretch) | planned |

## Foundations

### 1. Stack & architecture · needs a decision
Decide the stack (ASP.NET Core Clean Architecture, CQRS/MediatR, EF Core, SQL Server, Redis, Hangfire, SignalR; React + TypeScript; Docker) and scaffold a runnable solution so every later slice builds on real structure, including a thin walking-skeleton record that flows DB → API → UI.
**Done when:** the stack is recorded in a spec, the empty solution boots on docker-compose with SQL + Redis, the React app loads, and one trivial record is created and rendered end to end.
- [ ] Decide the stack (spec): `/architect stack & architecture`
- [ ] Scaffold from the decision: `/develop stack & architecture`

### 2. Coding standards & tooling
Capture conventions, then install lint, format, type strictness, pre-commit, and CI from the real scaffolded project.
**Done when:** root `AGENTS.md` reflects the real stack, and lint/format/CI run clean on the skeleton.
- [ ] Capture conventions + tooling choices: `/audit`
- [ ] Install the tooling: `/develop tooling`

### 3. Data model · needs a decision
The core entities every feature builds on: User, Store, Product (with a concurrency token), Cart/CartItem, Order, SubOrder, OrderItem, Payment, StockReservation, OutboxMessage, ProcessedWebhookEvent, LedgerEntry, Review.
**Done when:** entities, relationships, and concurrency tokens support reservations, the per-seller sub-order split, the outbox/inbox, and the ledger without a breaking migration later.
- [ ] Design it (spec): `/architect data model`

### 4. Auth & identity · needs a decision
Register and sign in; buyer by default; instant seller-store activation (no approval); the dual buyer/seller identity; and own-store authorization policies (a seller sees only their store, the owner sees all).
**Done when:** a person can register, sign in, activate a store instantly, and every protected action and own-store scope is enforced server-side.
- [ ] Design it (spec): `/architect auth & identity`

### 5. Design system & UI foundation · needs a decision
Visual language, layout primitives, and accessible base components so every page feels cohesive and meets the GA accessibility bar.
**Done when:** `design.md` covers type, color, spacing, and components, and base components handle focus and keyboard.
- [ ] Design it (spec): `/architect design system & UI foundation`

## Slice 1: Catalog & selling

### 6. Product catalog: listing · needs a decision
The buyer entry point: a paged, sorted listing of live products from all sellers.
**Done when:** buyers see live products with paging and sorting; empty and error states render; unlisted/draft products never appear.
- [ ] Design it (spec): `/architect product catalog listing`

### 7. Product detail & reviews · needs a decision
A product page with details and reviews; the review form appears only for a buyer who purchased the product (proof of purchase, assumed gated on Delivered).
**Done when:** the detail page renders live stock; a review can be written only by a verified purchaser; non-purchasers see reviews but no form.
- [ ] Design it (spec): `/architect product detail & reviews`

### 8. Seller products & commission preview · needs a decision
A seller creates and edits their own products, with a live 15% commission preview as they set a price ("you set $100 → platform takes $15 → you receive $85").
**Done when:** a seller can create, edit, and unlist their own products; the commission split shows live at price-set; products go live to buyers immediately.
- [ ] Design it (spec): `/architect seller products & commission preview`

## Slice 2: Cart & checkout

### 9. Cart (optimistic) · needs a decision
A per-buyer cart with optimistic add/remove/quantity backed by React Query, reconciled against the server; a seller cannot add their own product (enforced server-side, not only in the UI).
**Done when:** cart updates feel instant and reconcile with the server; the "cannot buy own product" rule is enforced in the backend and surfaced cleanly.
- [ ] Design it (spec): `/architect cart`

### 10. Checkout & Stripe payment · needs a decision
Checkout against Stripe test mode with one PaymentIntent per order (money to the platform, commission split in the ledger), and a visible 15-minute reservation hold with a countdown.
**Done when:** a buyer completes a test-mode payment; the reservation hold and countdown are visible; payment failure and timeout are handled gracefully.
- [ ] Design it (spec): `/architect checkout & payment`

## Slice 3: Concurrency

### 11. Stock reservation & optimistic locking · needs a decision
Hold stock at checkout start with a 15-minute TTL, guarded by the product's concurrency token, so two buyers can never both take the last unit. Includes the TTL sweeper and the headline load test.
**Done when:** a 100-concurrent last-item load test asserts exactly one buyer paid, zero oversold, final stock zero, and it gates CI; expired holds are swept and stock restored.
- [ ] Design it (spec): `/architect stock reservation & locking`

## Slice 4: Reliability

### 12. Webhook, outbox & payment idempotency · needs a decision
A fast Stripe webhook endpoint that records and returns, a Hangfire processor, the outbox + inbox dedupe, and the payment state machine, so an order is never paid or confirmed twice.
**Done when:** the same Stripe event delivered many times confirms the order exactly once; the webhook endpoint stays fast under duplicate bursts; events are emitted from the outbox and match committed state.
- [ ] Design it (spec): `/architect webhook, outbox & payment idempotency`

## Slice 5: Lifecycle & fulfillment

### 13. Order lifecycle & fulfillment · needs a decision
The order, sub-order, and payment state machines wired end to end; a seller order-management UI offering only legal transitions (Paid → Shipped → Delivered); buyer self-cancel before shipping.
**Done when:** every forbidden transition is blocked by the domain (not just the UI); the order status is derived from its sub-orders; a buyer can self-cancel only before ship.
- [ ] Design it (spec): `/architect order lifecycle & fulfillment`

### 14. Refunds · needs a decision
Seller- or admin-initiated refunds through Stripe (idempotent), driven by the refund webhook, moving the sub-order to Refunded.
**Done when:** a refund issued twice refunds once; the sub-order reaches Refunded only on the confirmed refund webhook; the order rolls up to PartiallyRefunded or Refunded.
- [ ] Design it (spec): `/architect refunds`

## Slice 6: Real-time & surfaces

### 15. SignalR real-time · needs a decision
Live seller notifications on new paid orders and live stock to buyers, pushed from the outbox as lightweight "refetch" signals; the client re-joins groups and refetches authoritative state on reconnect.
**Done when:** a seller sees a new-order event live; buyers see live stock changes on a product page; a dropped connection recovers by refetching, with no missed-event corruption.
- [ ] Design it (spec): `/architect signalr real-time`

### 16. Seller dashboard & stats · needs a decision
A seller dashboard with sales, revenue, commission, and units, computed on read from orders and the ledger, scoped strictly to the seller's own store (the owner sees all).
**Done when:** the dashboard shows correct, current numbers for the signed-in seller's store only; the owner sees platform-wide revenue and commission.
- [ ] Design it (spec): `/architect seller dashboard & stats`

## Slice 7: Stretch (MVP-late, the trim points)

### 17. Product search & filtering · needs a decision
Buyer search and filtering over the catalog. First trim point if the 8-week window tightens.
**Done when:** buyers can search and filter; the URL reflects the active query; empty and error states render.
- [ ] Design it (spec): `/architect product search & filtering`

### 18. Email notifications · needs a decision
Transactional email (order confirmation, shipped) sent from the outbox and retried on failure. Second trim point.
**Done when:** a buyer receives a confirmation email after payment and a shipped email on dispatch; sends are retried and never duplicated.
- [ ] Design it (spec): `/architect email notifications`

### 19. Cloud deploy & CI/CD · needs a decision
A real cloud deployment with a CI/CD pipeline, managed SQL + Redis, secrets, and (once multi-instance) a SignalR backplane.
**Done when:** the app is reachable in a cloud environment via an automated pipeline, with migrations and secrets handled, and real-time works across more than one instance.
- [ ] Design it (spec): `/architect cloud deploy & ci-cd`

## Deferred
Out of scope for the current build pass, kept so the plan stays honest.
- **Seller payouts & balance ledger**: actually paying sellers their net out of the platform balance (commission is tracked from Slice 1; payout movement is separate) · Next · needs a decision
- **Buyer order history & tracking**: a rich past-orders view · Next · needs a decision
- **Admin moderation console**: a dedicated owner console for users, products, and orders (owner acts via endpoints in MVP) · Next · needs a decision
- **Advanced seller analytics**: trends and conversion on top of the on-read basics · Next · needs a decision
- **Multi-currency / internationalization**: more than one currency and locale (MVP assumes a single currency) · Later · needs a decision
- **Discount codes & promotions**: pricing adjustments that would complicate commission and the state machine · Later · needs a decision
- **Wishlists & recommendations**: engagement polish, not core value · Later · needs a decision
- **Public seller API**: no stated consumer yet · Later · needs a decision
- **Auctions**: explicitly dropped per the brief; recorded here only to keep the decision visible · dropped

## Open decisions
Ranked by how soon they block work; each `/architect` spec resolves one.
1. **Stack specifics** (version, React tooling, layout, Docker) — blocks Feature 1, the first thing to architect.
2. **Auth approach** (token vs cookie session, built-in vs custom, dual-role claims, own-store policies) — blocks Feature 4.
3. **Data model specifics** (concurrency token, keys, indexes) — blocks Feature 3.
4. **Stock locking mechanism** (reservation rows vs counter; Redis's role) — blocks Feature 11.
5. **Outbox + inbox design and operation idempotency keys** — blocks Feature 12.
6. **Payment state machine + PaymentIntent mapping** — blocks Feature 12.
7. **State machine implementation style** (state classes vs guarded methods vs a library) — blocks Feature 13.
8. **SignalR backplane** (needed once more than one instance runs) — blocks Feature 19 / multi-instance.
9. **Search mechanism** (SQL full-text vs an engine) — blocks Feature 17.
10. **Email provider** — blocks Feature 18.
11. **Cloud host + CI/CD specifics** — blocks Feature 19.
12. **Review gate** (Delivered vs Paid), **review edits/one-per-purchaser**, **admin moderation depth** — confirm before Features 7 and 16.

**Assumptions carried** (veto any): 15-minute reservation TTL · single currency · single-variant products · seller marks Delivered · stats computed on read · seller payouts deferred to Next · reviews gated on Delivered, one per purchaser, editable.

## Legend

**The decision box.** Every feature carries exactly one sub-task whose label ends with `(spec)`. Skills locate it by that `(spec)` suffix. Every other box is an execution box.

**Feature lifecycle:**

| State | Set by | The feature shows |
|---|---|---|
| `planned` · needs a decision | `/scope` | one box: `Design it (spec): /architect <feature>` |
| `in-progress` (designed) | `/architect` at spec capture | `Design it` ticked; spec linked; `Build it: /develop <feature>` + 2 to 5 milestones; the GA closing boxes (`Verify it`, `Test it`, `Review it`, `Document it`) |
| `in-progress` (building) | `/develop` | milestone sub-boxes tick one by one; code pointer filled |
| `in-progress` (verified) | `/check verify` | `Build it` + milestones ticked; `Verify it` ticked |
| `done` | you, when you decide it is; `/sync` reconciles | the GA last stage (`/test`, then review + document) is the suggested point to call it done |

- **Next step** = the first unticked box (always a command or a tracked milestone).
- **needs a decision** = run `/architect` first; the tag drops once the spec is captured.
- **Atomic build tasks live in the spec's `## Build plan`, not here.**
- **Status** `planned` → `in-progress` → `done`, plus `existing` (pre-workflow) and `dropped` (de-scoped, kept for history).
- **Workflow** (header line) is the project default, GA: after `/develop`, run `/check verify`, then `/test`, then a fresh-model `/check review`, then `/document`.
- **Pointer line** (`spec <n> · code in <path>`): the spec link added by `/architect`, the code path by `/develop`.
