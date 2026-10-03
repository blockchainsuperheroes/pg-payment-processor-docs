# Onboarding a new project — request your OWN clean rails

Read this before you write any payment code. The golden rule:

> **Every project gets its own clean rails.** Request your own payments **project** (`prj_<name>`) and your
> own **SKUs**. Do **NOT** reuse another project's SKUs, keys, actions, or rails to "save time." Reusing
> someone else's rail means: no revenue attribution to you, your sales lumped under their line, shared keys
> you can't rotate independently, and sku-id/price collisions. One project = one set of SKUs = clean
> attribution, receipts, and caps. (Worked example: the MOBA game didn't reuse the TCG or card rails — it
> got `prj_moba` + its own `moba_s2_ticket` SKU.)

## What to ask the payments team to enable (points spending)

Send the payments session this, and we set it up:

1. **A project.** We create `prj_<yourname>` (owner pg_id + your app domain) in the payments DB.
2. **Your SKU(s).** Tell us, per item: a code (e.g. `moba_s2_ticket`), the **price in PC** (1 PC = 1000
   Points; e.g. 5 Points = `0.005` PC), the `item_type` (`digital_item` for a ticket/entitlement/pack the
   vendor delivers; `token` for a Points credit; `nft` for a mint), and any supply **cap**. We:
   - collision‑check the sku_id on‑chain (both processors) so it can never clash,
   - register the `cross_chain_payment_config` row (inactive) + a `project_skus` row under your project.
3. **On‑chain price.** The owner (nftprof, hardware wallet) signs one `setPrice` per SKU on the processor
   `0x3930B34a…A0ba` — via the `/admin/payments` panel, one click per SKU. (We hand you the panel link.)
4. **Activate.** We flip `is_active=true` once the price is verified on‑chain.
5. **Keys** (only if you report orders/receipts or relay your own Stripe): your project `X-App-Key` and/or
   an ingest source key, handed over as a **box secret** — never in chat, never in a repo.
6. **AA2:** nothing for you to sign. The processor your Points spend hits is already on the AA2 allowlist
   (`purchaseWithNative` `0xc3c12e69`), so migrated users work with no new signature.

You do **not** touch Stripe keys, the processor owner wallet, or another project's config. You call the
spend endpoint with the **user's own login token**; identity signs from their canonical wallet.

## Sample — charge Points for one item/ticket (server‑side)

Custodial Points purchase of *your* SKU. Node/JS; adapt to any language. This is the whole charge.

```js
// POST from YOUR backend with the player's Pentagon login JWT (never a wallet you picked).
async function chargePoints({ jwt, skuId, idemKey }) {
  const r = await fetch("https://api.account.pentagon.games/user/npc/spend_purchase", {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${jwt}` },
    body: JSON.stringify({ sku_id: skuId, idempotency_key: idemKey }),  // reuse idemKey on retry — no double charge
  });
  const j = await r.json().catch(() => ({}));
  if (r.ok && j.status !== false && j.tx_hash) {
    return { ok: true, tx_hash: j.tx_hash };          // Points spent on-chain from the user's canonical wallet → deliver your item
  }
  // j.error: insufficient_npc_points → "Not enough Points"; sku_not_purchasable / simulation_reverted / tx_failed → retry/soft-fail
  return { ok: false, error: j.error || `http_${r.status}` };
}

// Example: a MOBA play ticket (sku 36 = moba_s2_ticket, 5 Points)
const res = await chargePoints({ jwt: playerJwt, skuId: 36, idemKey: `moba:${user}:${runId}` });
if (res.ok) issueTicket(user, res.tx_hash);
```

Read the player's balance first if you want to gate the UI:

```js
const w = await (await fetch("https://api.account.pentagon.games/user/walletinfo",
  { headers: { Authorization: `Bearer ${jwt}` } })).json();
const points = w.result?.npc_points ?? w.npc_points;   // already in Points (×1000 of PC)
```

Direct‑PC (connected wallet, optional): read `prices(skuId, 0x0)` on the processor and send
`purchaseWithNative(skuId){ value: price }` — same SKU, same receipt. Most integrations only need the
custodial Points call above (works with no wallet connected — PG login is enough).

## Then: receipts / delivery

Pick your category from [`RECEIPT-CONTRACT.md`](./RECEIPT-CONTRACT.md) (on‑chain mint = automatic;
you grant the item = report via the integration API; paid off‑processor = `/api/ingest`). Full worked
examples per flow in [`PAYMENT-FLOWS.md`](./PAYMENT-FLOWS.md); every endpoint in
[`INTEGRATION-GUIDE.md`](./INTEGRATION-GUIDE.md).

## Checklist for a new integrator

- [ ] Asked payments for `prj_<name>` + your SKU(s) (code, PC price, item_type, cap) — your own, not reused.
- [ ] Owner signed `setPrice`; SKU `is_active=true` (payments confirms).
- [ ] Charge via `spend_purchase` with the user's JWT + a stable idempotency key.
- [ ] Deliver your item on success; report the receipt per your category.
- [ ] Got your `X-App-Key` / ingest key as a box secret (if reporting).
- [ ] One Points test purchase verified end‑to‑end with payments.
