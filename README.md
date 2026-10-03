# Pentagon Games — Payments Integration & Payment Flows

Integration docs for any Pentagon Games storefront, game, or 3rd‑party partner that needs to take payment,
spend Points, or deliver goods through the Pentagon payment stack.

**Start here:**

| Doc | What it's for |
|---|---|
| [INTEGRATION-GUIDE.md](./INTEGRATION-GUIDE.md) | End‑to‑end integration: sign‑in → read Points wallet → top‑up → charge → receipt. The two integration shapes (logged‑in player vs frictionless guest/email). Every endpoint. |
| [PAYMENT-FLOWS.md](./PAYMENT-FLOWS.md) | **Complete worked examples** per real product: logged‑in Points (AR / Stores), guest email checkout (Gunnies rolls / pfp‑maker), MOBA play tickets, card top‑up, vendor‑delivered packs (KEEPS), on‑chain NFTs (BCSH heroes). Copy‑ready. |
| [RECEIPT-CONTRACT.md](./RECEIPT-CONTRACT.md) | Delivery proof + receipts: the 3‑category decision tree (on‑chain auto / vendor off‑chain / off‑processor) and the exact contracts. |
| [index.html](./index.html) | The processor reference (rails, contract addresses, selectors, spend_purchase spec). |

**The one rule to remember:** a custodial Points charge for a logged‑in user is `spend_purchase` on a SKU.
1 PC = 1000 Points (fixed). Points are the user's own on‑chain balance, spent through a sanctioned rail.

Keys (project `X-App-Key`, ingest source keys) are provisioned by the payments team and handed over as a
box secret — never in chat, never in a repo. If a call 404s, ownership may have shifted — ask the payments team.
