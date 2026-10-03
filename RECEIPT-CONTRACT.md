# Payment Portal Receipt Contract

Canonical spec for making every Pentagon Games purchase show a **uniform receipt** on
payments.pentagon.games, regardless of which storefront took the payment.

Owner of the portal side: **Payments.Pentagon.Games** session (endpoints live, receipts render).
Owner of cross-storefront rollout: **Obsidian Vault & Master Coord**.
Status: ingest system LIVE as of 2026-10-01 (pg-payment-fe BUILD_ID 200wmA9qafl3JXpKPdn4m).

---

## What a receipt is

A receipt = **a payment record** + **its delivery proof**, joined and rendered on the portal
(Activity tab row, expandable; and a deep link `/?tab=activity&receipt=<tx_or_order_id>`).

Payment records come from **two sources** — a storefront uses whichever already applies:

- **Core ledger** (`cross_chain_purchase_log`): purchases through a Pentagon rail —
  the processor (`purchaseWithNative` / `spend_purchase`), custodial Points, Stripe top-ups.
  These are recorded automatically by `event_tracker` / the Stripe webhook. **No payment wiring
  needed.** (KEEPS TCG, BCSH heroes, Stores items are here.)
- **Ingest** (`payment_ingest`): payments on surfaces **outside** the core processor —
  on-chain spends on other contracts, or a project's own Stripe. These need a report (Path 1 or 2 below).

Delivery proof comes from **`project_orders`** (`allocation` jsonb) for vendor-delivered items,
or from `reward_log` for on-chain NFT/token drops (automatic), or from the report's `delivery` field.

---

## Which category is your store? (decide this first)

Every purchase falls into exactly one of three categories. The category decides what — if anything —
you build. Answer two questions: **how was it paid?** and **how is it delivered?**

- **Category 1 — Processor/Points-paid, delivered by PENTAGON'S `reward_service`.**
  Payment is recorded automatically in `cross_chain_purchase_log`; delivery is recorded automatically in
  `reward_log` **when Pentagon's `reward_service` performs it** — i.e. the SKU is configured in
  `cross_chain_payment_config` with a `mint_function_abi` (NFT) or as a token tier (PC/Points credit), so the
  worker mints/credits. **Then you build NOTHING** — receipts appear on their own. Optional: a label beacon
  (Path 1) for nicer names.
  *Examples: BCSH legendary heroes (reward_service mints), NPC/Points top-ups (reward_service credits).*
  ⚠️ The test is **who performs the delivery**, not whether it's on-chain. If **your own backend** mints/grants
  the item (even an on-chain 0x0 NFT mint) instead of `reward_service`, there is **no `reward_log` row** and the
  receipt would sit paid-undelivered — you are **Category 2**, not 1. (A SKU with purchases but no
  `cross_chain_payment_config` row is a tell that reward_service isn't delivering it.)

- **Category 2 — Processor/Points-paid, delivered by the VENDOR (a grant or mint the vendor performs itself).**
  Payment is already in `cross_chain_purchase_log`; **you report the delivery** via the integration API
  (Path 3: register-then-deliver → `project_orders`). The delivery can be off-chain (a grant) **or** an
  on-chain mint your own backend does — either way `reward_service` didn't record it, so you report it
  (put the mint tx in `delivery_ref`, the token id in `allocation`).
  *Examples: TCG packs (KEEPS, the template), Gunnies rolls, EF genesis character mint (EF's backend mints),
  AI-asset hosting-extend.*

- **Category 3 — Paid OFF-PROCESSOR (no core-ledger row at all).**
  The payment itself happens outside the Pentagon processor, so there's nothing in `cross_chain_purchase_log`.
  **You report the payment** via `/api/ingest` — a label beacon for on-chain contracts the sweep watches
  (Path 1), or an s2s relay for a project's own Stripe (Path 2).
  *Examples: AR GamePayHub, Gunnies PFP mint, populace DNA store, Gunnies Stripe store, PNS, VaultDrops.*

Quick test: **delivery on-chain?** → Cat 1, build nothing. **Paid through the processor but a vendor grants
the item?** → Cat 2, integration `/delivery`. **Payment never touches the processor?** → Cat 3, `/api/ingest`.

## The 3 reporting paths

### Path 1 — On-chain, auto-swept (+ label beacon)
The ingest sweep already watches these contracts and records every payment on-chain; you only add a
**fire-and-forget label beacon** so the receipt shows real item names, not just a tx hash.

```js
navigator.sendBeacon(
  "https://payments.pentagon.games/api/ingest/report",
  new Blob([JSON.stringify({
    source: "<source-key>", tx_hash: "0x…",
    item_code: "…", item_name: "…", qty: 1,
    buy_url: "https://…", order_ref: "…"
  })], { type: "text/plain" })   // text/plain = no CORS preflight
);
```
Nothing in the body is trusted for money — the tx is verified on-chain (watched contract, success,
payment event); the body only adds labels. A not-yet-mined tx is kept `pending` and re-checked.
Returns 202. Watched source keys: `ar-gamepayhub`, `gunnies-pfp`, `pns`, `vaultdrops`, `topup-pc-transfer`.

### Path 2 — Off-chain relay (Stripe / non-rail)
Your backend POSTs **after the webhook commits the order as paid** (from a background task, so the
webhook response is never delayed). Server-to-server, each key writes only its own source.

```
POST https://payments.pentagon.games/api/ingest/relay
X-Ingest-Key: <your source key>     # provisioned per source, kept a box secret
{
  "source": "gunnies-stripe", "id": "cs_live_…",   // id = idempotency key
  "paid_at": "ISO", "currency": "usd", "amount_total": 19.99,
  "wallet": "0x…", "pg_user_id": 123, "email": "…",
  "items": [{ "code": "…", "name": "…", "qty": 1, "amount": 19.99 }],
  "buy_url": "https://…", "order_ref": "…", "alt_refs": ["cs_live_…"],
  "delivery": { "status": "pending" | "delivered", "at": "ISO" }
}
```
Idempotent on (source, id, line) — re-send to update in place. Wrong key → 401, unknown source → 400.
Put the success-page session id in `alt_refs` so `/?receipt=<that>` resolves for the buyer.

### Path 3 — Vendor delivery proof (integration API → `project_orders`)
For items a game backend delivers (digital or physical). The payment is already recorded (core ledger
via the processor/Points, or a relay); you report **what was delivered** so the receipt shows
Delivered-to + allocation + balance. This is the **integration API on the accounts backend** — NOT
`/api/ingest` (that's only for off-processor surfaces). EF TCG (KEEPS) is the live template; copy it.

**Base:** `https://api.account.pentagon.games`  ·  **Auth:** `X-App-Key: <your project key>`
(server-to-server, sha256 → `project_api_keys`; invalid → 401). Keys are per-project (prj_tcg, prj_gunnies).

Two steps:

1. **Register the order** (once, at/after purchase — binds order_id ↔ on-chain tx):
```
POST /api/integration/orders
X-App-Key: <key>
{ "project_key": "prj_tcg", "order_id": "<your id>", "sku_code": "<sku>", "tx_hash": "0x…" }
→ 200 { "status": true, "result": { "payment_id": <int|null>, "order_id", … } }
```
`payment_id` resolves from `tx_hash` against `cross_chain_purchase_log` (null until the payment settles;
re-register or the delivery call backfills it). Idempotent on (project, order_id).

2. **Report delivery** (after fulfillment):
```
POST /api/integration/payments/<payment_id>/delivery
X-App-Key: <key>
{ "order_id": "<same id>", "delivered": true, "delivery_ref": "<grant tx / db ref>",
  "allocation": { "pack_ids": ["…"], "card_ids": ["…"], "db_ref": "…",
                  "item_type": "digital_item", "item_name": "KEEPS pack x5", "qty": 5,
                  "recipient": "0x… or @username", "timestamps": {…},
                  "balance": { "before": 100, "after": 105, "delta": 5 } } }
→ 200 { "status": true, "result": { payment_id, order_id, delivered, delivery_status,
                                    delivery_ref, allocation, order_state, … } }
```
- `allocation` is stored verbatim in `project_orders.allocation` (jsonb) and rendered on the receipt.
  It **must** contain at least one of `pack_ids` / `card_ids` / `db_ref` (else 400).
- The order must already be registered (step 1) or you get `404 order_not_found`.
- **Idempotency:** keyed on (project, order_id); re-posting is safe (delivery is set once, then returns
  the current order view). Retry until you get a 200 ack.
- Errors: `401 invalid_app_key`, `400 order_id_required` / `allocation_requires_pack_ids_card_ids_or_db_ref`,
  `404 order_not_found`, `409 payment_id_mismatch`.

Switching an existing vendor to live = point its `PG_BASE` at `https://api.account.pentagon.games` and use
its live `X-App-Key`. Nothing else changes. The receipt then renders from `cross_chain_purchase_log`
(payment) joined to `project_orders` (delivery + allocation); deep link `/?tab=activity&receipt=<tx_or_order_id>`.

---

## Final session → path mapping

| Storefront / session | Payment source | Path(s) to wire |
|---|---|---|
| **TCG game** (tcg.etherfantasy.com — pack sales) — *digital template* | core ledger (processor/Points) — auto | **Path 3** delivery proof + allocation. Likely also the ef-tcg-site storefront owner |
| **BCSH v2** (legendary heroes) | core ledger — auto | reward_log delivery is automatic; label beacon optional. Receipts flow once a purchase confirms |
| **Gunnies store** (own Stripe) | — | **Path 2** relay (`gunnies-stripe`) + `delivery` field (pending→delivered) |
| **Gunnies PFP mint** (on-chain) | ingest — swept | **Path 1** label beacon (`gunnies-pfp`) |
| **EF — AR in-game** (GamePayHub) | ingest — swept | **Path 1** label beacon (`ar-gamepayhub`) |
| **PFP Vault** (VaultDrops) | ingest — swept | **Path 1** label beacon (`vaultdrops`); **Path 3** only if prize delivered off-chain |
| **PNS** name mints | ingest — swept | **Path 1** label beacon (`pns`) |
| **pentagon.games/topup** (PC transfer) | ingest — report-driven | report `topup-pc-transfer` from the top-up page |
| **Physical NFC cards** (1,000-card run / shipping) — *KEEPS NFC fork + shipping backend* | core ledger / relay | **Path 3**, but **shipped-driven** (deliver/email on SHIPPED, not settle) — a separate physical-fulfillment receipt; out of scope until physical sale opens |

> Note: the **KEEPS TCG NFC-programming fork** owns physical NFC card programming + the pet-card backend only — it does **not** process digital pack orders, so it is not the digital-delivery template. Pack receipts belong to the TCG game session.

## Getting wired
- Need a relay key (Path 2)? Ask the Payments session — provisioned per source, handed off as a box
  secret (never pasted in chat).
- Ready to test? Ask the Payments session for a **test round-trip** — it confirms the row lands in
  `payment_ingest` / `project_orders` and renders as a receipt, and returns the receipt link.
