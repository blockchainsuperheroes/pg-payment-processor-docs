# Pentagon Payments — Complete Payment Flows (worked examples)

End-to-end, copy-ready examples for every real integration shape. Pick the flow that matches your product;
each lists the rail, the exact calls, and why it's chosen. Full endpoint/contract detail is in
[`INTEGRATION-GUIDE.md`](./INTEGRATION-GUIDE.md); delivery/receipt categories are in
[`RECEIPT-CONTRACT.md`](./RECEIPT-CONTRACT.md).

Bases: accounts/identity + payments backend `https://api.account.pentagon.games`; payments portal
`https://payments.pentagon.games`. **1 PC = 1000 Points** (fixed). Points are the user's own on-chain
custodial balance — spent on-chain through a sanctioned rail; the chain is the ledger.

The decision in one line: **custodial Points charge for a logged-in user = `spend_purchase` on a SKU.**
Everything below is a variation on that, or the fiat/guest path around it.

---

## Flow A — Logged-in player pays with Points (ar.etherfantasy.com, /stores)

The user is signed in with Pentagon; you spend their Points on an item. No wallet connection needed —
the charge is custodial (backend signs from the user's canonical wallet).

```
# 1. (once) read who they are + their balance
GET /user/info          Authorization: Bearer <JWT>   → { pg_user_id, username, aa_wallet_address, ... }
GET /user/walletinfo    Authorization: Bearer <JWT>   → { npc_points, ... }   # show npc_points

# 2. charge the item (on-chain, idempotent)
POST /user/npc/spend_purchase   Authorization: Bearer <JWT>
{ "sku_id": 24, "idempotency_key": "<stable per attempt>" }
→ 200 { "status": true, "tx_hash": "0x..." }        # signs purchaseWithNative(sku) from their wallet
# errors: insufficient_npc_points | sku_not_purchasable | simulation_reverted | tx_failed
```

Receipt: automatic if the item delivers on-chain (Category 1); if you grant it yourself, report delivery
(Category 2 — see Flow E). Settlement ~1–2 min.

---

## Flow B — Frictionless guest / email-only, Points hidden (gunnies.io rolls, pfp-maker)

No PG login. The buyer gives an email and pays once; Points are abstracted away. Because a Points top-up
and a Points spend are two separate on-chain steps (~1–2 min each), **deliver on fiat-paid and reconcile
behind it** — do NOT make the player wait for an on-chain round-trip.

```
# 1. (identity, s2s) provision/resolve the account from the email — magic-link / gunnies-activate pattern.
#    Gives the account a canonical wallet; the email is the delivery address. (Owner: identity.)

# 2. guest pays fiat — Stripe Checkout with the email (you price it in USD; buyer never sees "Points")
POST https://pentagon.games/api/stripe/checkout-sessions
{ skuId, wallet: <account aa_wallet_address>, lineItems:[{ name, priceId, quantity:1 }] }  → { url }

# 3. DELIVER THE GOOD THE MOMENT STRIPE SAYS PAID (webhook) — instant, no on-chain wait:
#    on checkout.session.completed / payment_intent.succeeded → grant the roll/pfp to the session/email.

# 4. RECORD the sale so it shows as a receipt (your own Stripe = off-processor = Category 3 relay):
POST https://payments.pentagon.games/api/ingest/relay   X-Ingest-Key: <your source key>
{ "source":"<your-source>", "id":"<stripe session id>", "paid_at":"<iso>", "currency":"usd",
  "amount_total":4.99, "email":"buyer@...", "pg_user_id":<if known>,
  "items":[{ "code":"roll_x1", "name":"1 Roll", "qty":1, "amount":4.99 }],
  "delivery":{ "status":"delivered", "at":"<iso>" } }
```

The Points top-up credit settles async in the background; the player already has their good. Use this for
cheap, instant-gratification, conversion-sensitive digital buys. (If a good MUST be gated on an on-chain
Points spend, use the resumable-order variant: fiat → top-up shortfall → `spend_purchase` → deliver on
settle — but the user waits ~2–4 min.)

---

## Flow C — MOBA play tickets, Points-only (Telegram) / Points-or-CT (web)

The user is PG-logged-in with **no wallet connected**, high volume, and you're **issuing a ticket**
(a consumable entitlement — not an NFT, not a game item, no DNA/Karma). Charge custodial Points with
**`spend_purchase` on a dedicated MOBA ticket SKU** — the same clean rail as a store item.

**Do NOT use `spend_ar` (GamePayHub) for this.** `spend_ar` is the AR game's rail: it credits Karma
watermark and feeds the EF-pet DNA economy (irrelevant and polluting for a MOBA ticket), needs a GamePayHub
gameId, sits on a contract whose ownership is disputed, and — critically — **GamePayHub spends are not
scanned into the payments ledger today**, so those charges would be invisible to receipts/revenue.
`spend_purchase` settles as a real confirmed purchase in `cross_chain_purchase_log` (visible, auditable,
receipt-able) and is already AA2-allowlisted (it hits the processor). For "issue a ticket like a mining
boost," a SKU-backed Points burn is exactly right.

```
# One-time setup (payments team): register a ticket SKU (e.g. moba_s2_ticket, item_type digital_item),
#   nftprof signs setPrice on the processor (e.g. 0.005 PC = 5 Points), then it's activated.

# Charge (server-side; Telegram = this only; web = this is the "Points" button):
POST https://api.account.pentagon.games/user/npc/spend_purchase
Authorization: Bearer <user login JWT>
{ "sku_id": <moba_ticket_sku>, "idempotency_key": "moba_s2:<user>:<run-or-attempt-id>" }
→ 200 { "status": true, "tx_hash": "0x..." }     # Points spent on-chain from the user's canonical wallet
# On 200 → ISSUE THE PLAY TICKET. Idempotency_key makes retries safe (no double-charge, no double-ticket).
# Map errors → user message: insufficient_npc_points → "Not enough Points"; others → retry/soft-fail.
```

- **Ticket issuance** happens on the 200 (the spend already settled on-chain for a digital_item). Store the
  ticket keyed by the idempotency_key / tx_hash so a client retry re-shows the same ticket.
- **Receipt** (optional but recommended at volume): report it as a vendor delivery (Category 2) so each
  ticket shows on payments — register the order + POST `/api/integration/payments/<id>/delivery` with
  `allocation:{ db_ref:"moba:ticket:<id>", item_type:"ticket", item_name:"MOBA S2 play ticket", qty:1 }`.
- **CT "chest" button (web only, later):** CT is an ecosystem token the user holds in a wallet — a separate
  rail (GamePayHub `payCT` or the MOBA's own CT sink), not a custodial Points spend. Ship the Points ticket
  first (works with no wallet); add the CT chest when wallets are connected.
- **AA2:** none needed — `spend_purchase` targets the processor, already on the AA2 allowlist
  (`purchaseWithNative` `0xc3c12e69`). Migrated users work with no new signature.

---

## Flow D — Top-up (buy Points with a card)

```
GET  https://payments.pentagon.games/api/topup/packages   → { packages:[{sku_id,name,points,price_usd}] }
POST https://pentagon.games/api/stripe/checkout-sessions   { skuId, wallet, lineItems:[{priceId}] } → { url }
# webhook credits Points to the user's canonical wallet ~1–2 min later; reflect the new balance (Flow A §1).
```

---

## Flow E — Vendor-delivered pack + receipt (KEEPS TCG template)

Processor/Points-paid, but YOU grant the item off-chain → you report the delivery (Category 2).

```
# charge: Flow A spend_purchase (or direct PC). Then:
POST /api/integration/orders  X-App-Key:<key>  { project_key, order_id, sku_code, tx_hash } → { payment_id }
POST /api/integration/payments/<payment_id>/delivery  X-App-Key:<key>
{ order_id, delivered:true, delivery_ref:"<grant tx/db ref>",
  allocation:{ pack_ids:[...], item_type:"digital_item", item_name:"KEEPS pack x10", qty:13,
               recipient:"<username>", balance:{before,after,delta} } } → { status:true }
```

---

## Flow F — On-chain NFT, delivery automatic (BCSH legendary heroes)

Processor/Points-paid, delivered by Pentagon's reward_service (the mint) → **you build nothing**; payment
and delivery are both recorded automatically (Category 1). Buy via Stripe (card), USDC on the ETH processor,
or Points (`spend_purchase`); poll `GET /api/integration/payments/<id>/status` for the mint tx.
Display price from `purchase_amount`; the on-chain pay amount comes from the processor at pay time.

---

## Choosing the rail (cheat sheet)

| Your case | Rail | Receipt |
|---|---|---|
| Logged-in user spends Points on an item/ticket | `spend_purchase` on a SKU | auto (Cat 1) or report (Cat 2) |
| Logged-in user pays PC from a connected wallet | `purchaseWithNative(sku)` on the processor | auto (Cat 1) or report (Cat 2) |
| Guest, email-only, instant digital good | Stripe → deliver-on-paid → ingest **relay** | relay (Cat 3) |
| Buy Points with a card | Stripe top-up (catalog + checkout) | auto |
| You mint/grant the good yourself | charge as above, then integration `/delivery` | report (Cat 2) |
| In-game item in the AR pet economy (Karma/DNA) | `spend_ar` (GamePayHub) | (scanner backlog) |

When unsure, it's almost always **`spend_purchase` on a SKU** — clean, custodial, on-chain, idempotent,
AA2-ready, and it lands in the ledger with a receipt.
