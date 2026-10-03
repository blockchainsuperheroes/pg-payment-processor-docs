# Pentagon Payments — 3rd‑Party Integration Guide

One place for a partner storefront or game to integrate the Pentagon payment stack end to end:
**sign‑in → read the Points wallet → let users top up → charge for an item → show a receipt.**

Two host bases are referenced throughout:
- **Accounts/identity + payments backend:** `https://api.account.pentagon.games`
- **Payments portal (receipts, catalog):** `https://payments.pentagon.games`

Everything is **server‑to‑server or user‑token**; never embed a secret key in a browser. The one
browser‑safe call is the receipt label *beacon* (it carries no secret and is verified on‑chain).

Key facts to internalise first:
- **Points = PC.** 1 PC = **1000 Points**. Points are a user's own on‑chain custodial balance ×1000 —
  non‑transferable, spend‑only, no cash‑out. The chain is the ledger; there is no off‑chain balance.
- **Canonical wallet.** Each user has one canonical custodial wallet (`aa_wallet_address`). After the AA2
  migration that's their AA2 account. **Always address a user by their Pentagon identity (user token /
  pg_user_id / username), never by a raw wallet you picked** — the backend resolves the right wallet.

---

## Two integration shapes — pick one

**Shape A — Logged‑in player (e.g. ar.etherfantasy.com).** The user signs in with Pentagon, you show
their Points balance, they top up if short, and they spend Points on items. This is §1–5 below, in order.
Best when the user has an ongoing account and balance: games, repeat purchases, anything with a wallet UI.

**Shape B — Frictionless guest / email‑only (e.g. gunnies.io rolls, pfp‑maker).** No PG login. The buyer
gives an email (the delivery address + their later account handle) and pays **once**; Points are abstracted
away — the fiat payment provisions/ tops‑up the buyer's account and the good is delivered. Best for one‑off
digital buys where a login step would kill conversion.

### The catch that decides Shape B's UX
Points are on‑chain PC, so a **top‑up credit (~1–2 min)** and a **Points spend (~1–2 min)** are two separate
on‑chain steps — **not atomic, not instant**. So "pay → Points appear → deducted → good appears" cannot be
truly instant *if* you gate the good on an on‑chain Points spend. Two honest ways to build Shape B:

- **B1 — deliver‑on‑paid (recommended for rolls/pfp; genuinely minimal‑click).** Guest pays fiat (Stripe) →
  you **deliver the digital good the moment Stripe says paid** (`checkout.session.completed` /
  `payment_intent.succeeded`), with no wait → you **record the sale to payments** for revenue + receipt
  (the ingest *relay*, §5 category 3). Points are the accounting layer and reconcile behind the scenes; the
  good never waits on an on‑chain round‑trip. This is what makes gunnies‑style "pay, it's there" possible.
- **B2 — Points strictly in the middle (fully on‑chain, auditable, but ~2–4 min).** fiat → auto top‑up (§3)
  → auto `spend_purchase` (§4) → deliver on settle. Use only when the good *must* be gated on an on‑chain
  Points spend; it's a resumable order (quote → pay → top‑up‑shortfall → credited → spend → deliver,
  idempotency‑keyed), not a click‑and‑it's‑there.

### What Shape B composes (no single turnkey endpoint — it's these pieces)
1. **Email → account (identity‑owned).** The magic‑link / auto‑provision flow (the `gunnies-activate`
   pattern): a scoped s2s call creates or resolves a Pentagon account from the email and gives it a
   canonical wallet; a magic link lets the buyer claim the account later. The email is the delivery address.
   You do **not** collect a password or run a login UI. (Owner: identity — ask them for the current
   `gunnies-activate` endpoint + scoped key.)
2. **Pay (Stripe, guest).** A Stripe Checkout session with the buyer's email (§3). Price it in your own
   currency/USD; the buyer never sees "Points."
3. **Credit + (optionally) spend.** The top‑up webhook credits Points to that account's canonical wallet
   (§3). In **B1** you deliver immediately on paid and let the credit settle; in **B2** you then
   `spend_purchase` the SKU and deliver on settle.
4. **Deliver + record.** Deliver the digital good to the on‑screen session / email, and record the sale so
   it shows as a receipt (§5): for B1 that's the ingest **relay** (your own Stripe is off‑processor →
   category 3); for B2 the processor spend is auto‑recorded (category 1/2).

Rule of thumb: **digital, cheap, instant‑gratification, conversion‑sensitive → Shape B1.** Account‑bound,
balance‑visible, repeat spend → Shape A. The sections below detail every endpoint both shapes use.

---

## 1. Sign‑in (identity)

Use **Sign in with Pentagon** (the login widget / pill). It returns a login JWT for the user; validate it
and read the profile:

```
GET https://api.account.pentagon.games/user/info
Authorization: Bearer <user login JWT>
→ { pg_user_id, username, email, mm_address, aa_wallet_address, has_shippable_address, ... }
```

- `/user/login` also accepts an email **or** PNS name **or** username in one field (it resolves them).
- Client id for your site is registered with identity (e.g. `pg-payments-site`). Register yours with them.
- Treat the JWT as the user's — forward it on their calls; don't store it beyond the session.

## 2. Read the Points wallet (balance)

```
GET https://api.account.pentagon.games/user/walletinfo
Authorization: Bearer <user login JWT>
→ { npc_points, migration_state, aa2_address, has_aa2, npc_points_legacy, npc_points_aa2, spend_restricted, ... }
```

- `npc_points` is the spendable balance **in Points** (already ×1000 of PC). Display it directly.
- **Do not hardcode the rate anywhere except 1 PC = 1000 Points** (that ratio is fixed; the USD price of a
  PC is what changes). If you must show a rate, read it from the catalog (§3), don't assume a USD figure.
- `migration_state`/`has_aa2` tell you whether the user is on AA2; you don't need to branch on it —
  the spend endpoints wrap both modes. Never key your UI off a specific wallet address.

## 3. Top‑up (let the user buy Points)

**Catalog (public, CORS‑enabled — safe from the browser):**
```
GET https://payments.pentagon.games/api/topup/packages
→ { ok, points_per_pc: 1000, usd_per_1000_points: <rate>, packages:[ { sku_id, name, points, price_usd }, ... ] }
```
Load prices from here live — don't hardcode tiers (they drift). Today: 100/$5, 425/$20, 1000/$45,
2500/$100, 5500/$200 (the 1000 tier is a discount off the per‑1000 list rate).

**Card checkout (Stripe):** create a checkout session on the accounts side and redirect the user to it.
```
POST https://pentagon.games/api/stripe/checkout-sessions
{ skuId: <sku_id>, wallet: <user aa_wallet_address>, lineItems: [ { name, priceId, quantity: 1 } ] }
→ { url }   // redirect the user here
```
- The sku to **credit** is carried in the session metadata (`skuId`); the webhook credits off it.
  (The platform derives the sku server‑side from the price to prevent price/SKU mismatch — don't rely on
  a client‑supplied sku being trusted.)
- **Crediting is asynchronous and on‑chain.** After the card payment settles, a worker credits the user's
  canonical wallet (~1–2 min). You don't deliver Points yourself — just reflect the new balance (§2) and
  the receipt (§5).
- Points top‑ups never go to a wallet you pass blindly; they land in the user's own canonical wallet.

**Crypto top‑up** also exists (PC on Pentagon Chain, or USDC/PC‑ERC20 on **Ethereum only**) — same credit
result. Most integrations only need the card path above.

## 4. Charge for an item (spend)

Every item buys through its **original rail**; there is no single "charge" call. Pick by item type:

| Item sold through | Rail | How to charge |
|---|---|---|
| The Pentagon **processor** (SKUs: packs, drops, downloads, heroes) | Points (custodial) | `POST /user/npc/spend_purchase` |
| | or direct PC (wallet) | `purchaseWithNative(skuId)` on the processor, `value = prices(skuId, 0x0)` |
| In‑game action priced per use | PayHub | `POST /user/npc/spend_ar` |
| Card/earn‑back split | CardSpendSplitter | `POST /user/npc/spend` |

**Custodial Points charge (the common one):**
```
POST https://api.account.pentagon.games/user/npc/spend_purchase
Authorization: Bearer <user login JWT>
{ "sku_id": <id>, "idempotency_key": "<stable per attempt>" }
→ { status, tx_hash, ... }   // signs purchaseWithNative(skuId) from the user's canonical wallet
```
- On‑chain price, balance‑gated, simulate‑first, **idempotent on `idempotency_key`** (reuse the same key
  on retry — it won't double‑charge).
- Direct‑PC path: read `prices(skuId, ZERO_ADDRESS)` on the processor and send `purchaseWithNative(skuId)`
  with that `value` from the user's connected wallet.
- Processor (Pentagon): `0x3930B34a524170Cc8966859Da167DB7B5413A0ba`; ETH processor:
  `0xe6bde156369d209c4d420e966541ee17093705b5` (USDC + PC‑ERC20).

## 5. Order binding + receipt (delivery proof)

A **receipt** = the payment record + its delivery proof, shown on `payments.pentagon.games`
(Activity tab; deep link `/?tab=activity&receipt=<tx_or_order_id>`). How you report delivery depends on
your category — **decide this once** (full detail + the exact Path contracts are in `RECEIPT-CONTRACT.md`):

1. **On‑chain delivery** (an NFT mint or a PC/Points credit): payment and delivery are both recorded
   automatically (settlement log + reward log). **You build nothing**; a receipt appears on its own.
   *(e.g. NFT heroes, Points top‑ups.)* Optional: a label beacon for nicer item names.
2. **Vendor off‑chain grant** (you grant something off‑chain, e.g. card packs): payment is auto‑recorded;
   **you report the delivery** via the integration API:
   ```
   POST https://api.account.pentagon.games/api/integration/orders
   X-App-Key: <your project key>
   { project_key, order_id, sku_code, tx_hash }            // register once

   POST https://api.account.pentagon.games/api/integration/payments/<payment_id>/delivery
   X-App-Key: <your project key>
   { order_id, delivered: true, delivery_ref, allocation: { pack_ids|card_ids|db_ref (≥1 required),
     item_type, item_name, qty, recipient, balance:{before,after,delta} } }
   ```
   Idempotent on (project, order_id); `allocation` is stored verbatim and rendered on the receipt.
3. **Off‑processor payment** (the money never touched the Pentagon processor — your own Stripe, or an
   on‑chain buy on a contract we don't settle): report the payment itself to the ingest system —
   a browser **label beacon** for watched on‑chain contracts, or a server **relay** for your own Stripe:
   ```
   POST https://payments.pentagon.games/api/ingest/report      // browser, no secret, tx verified on-chain
   { source, tx_hash, item_code, item_name, qty, buy_url, order_ref }

   POST https://payments.pentagon.games/api/ingest/relay       // server-to-server
   X-Ingest-Key: <your source key>
   { source, id, paid_at, currency, amount_total, wallet, pg_user_id, items:[...], delivery:{status} }
   ```

Quick test: **delivery on‑chain?** → category 1 (build nothing). **Paid through the processor but you grant
the item?** → category 2 (integration `/delivery`). **Payment never touches the processor?** → category 3
(`/api/ingest`).

## Keys & onboarding
- A project (`project_key`) + its `X-App-Key` (for the integration API) and/or an ingest source key are
  provisioned by the payments team and handed over as a box secret — never in chat, never in a repo.
- Tell the payments team which category (§5) your store is, and whether you sell via the processor
  (Points/PC) or your own rail, and you'll get the keys + a test round‑trip.

## Checklist for a new partner
1. Sign‑in wired (`/user/info`), user token forwarded on their calls.
2. Balance shown from `/user/walletinfo` (`npc_points`), rate only as 1 PC = 1000 Points.
3. Top‑up uses the live catalog (`/api/topup/packages`) + the Stripe checkout; balance reflects async credit.
4. Charge via the item's rail (`spend_purchase` for Points, `purchaseWithNative` for PC).
5. Receipts: pick your category; wire nothing / integration `/delivery` / `/api/ingest` accordingly.
6. Get your keys from the payments team and run one Points **and** one PC test to confirm the receipt flips
   to delivered.

*Contracts in this guide were verified against the live backend on 2026‑10‑01. `/user/*` endpoints are the
accounts/identity backend; `/api/topup`, `/api/ingest` and receipts are the payments portal; `/api/stripe/*`
is the website. Ownership can shift — if a call 404s, check with the payments team.*
