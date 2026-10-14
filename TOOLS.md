# Tools

Sixteen tools. "Commits" says whether the call can move money: **no** means read-only or reversible; **human** means the tool only creates an approval that the account owner must confirm by link with their two-factor code.

Identifiers for a trade (`trade_id`) accept either the numeric id or the public id shown in the trade URL.

## Account

### `get_readiness` — Account readiness · commits: no
Who the connected user is and whether the account can trade: email verified, identity verified, 2FA, and in which countries the agent may execute trades (not just read). Call this first.

No parameters.

### `get_balances` — My iEscrow balances · commits: no
Crypto balances held in the user's iEscrow account.

No parameters.

### `get_my_payment_accounts` — My payment accounts (masked) · commits: no
The user's saved payment accounts (bank transfer, Mercado Pago, PIX, etc.), masked. Needed to pick `payment_account_id` for `open_trade`. Adding or changing accounts is only possible on the web.

No parameters.

## Quotes and trades

### `get_quote` — Get executable offers · commits: no
Best executable offers on the P2P marketplace for buying or selling crypto with fiat, for the given crypto amount: price with the maker's margin, what you pay or receive all-in (platform fee included), whether it is instant, accepted payment methods and which of the user's saved accounts are compatible (`compatible_account_ids`). Also returns `routes` (comparable routes through external providers, without escrow protection) and a `recommendation` by all-in cost; a route marked `simulated` is not executable.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `side` | `buy` \| `sell` | yes | buy = the user pays fiat and receives crypto; sell = the user delivers crypto and receives fiat |
| `crypto` | string | yes | symbol, e.g. `USDT`, `BTC`, `ETH` |
| `fiat` | string | yes | ISO code, e.g. `ARS`, `USD`, `BRL`, `EUR` |
| `amount` | number | yes | amount in crypto units |
| `payment_method` | string | no | filter by method name, e.g. `Mercado Pago`, `PIX` |
| `limit` | integer 1–20 | no | |

### `open_trade` — Open a trade from an offer · commits: no (reserves the offer)
Takes an offer from `get_quote` and opens the trade. With `side=buy` the user will pay fiat and receive the crypto in their iEscrow balance. With `side=sell` the user sells crypto against a buy ad and then locks it with `fund_escrow`. Never delivers to an external wallet. Idempotent within 10 minutes. Call `get_trade` afterwards for the exact amount and account.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `offer_id` | integer | yes | the offer's `id` from `get_quote` |
| `side` | `buy` \| `sell` | no | default `buy`; must match the quote |
| `price_fiat_per_crypto` | number | for `sell` | the offer's price from `get_quote` |
| `amount` | number | yes | crypto units, within the offer's limits |
| `payment_account_id` | integer | for fiat legs | one of `compatible_account_ids` |
| `expected_rate` | number | no | for floating-price offers; the trade is rejected if the market moved more than 1 % |
| `idempotency_key` | string | no | same key on retries |

### `get_trade` — Trade status and next step · commits: no
State of one trade: roles, the exact fiat amount to pay and the account to pay to (once the crypto is locked), time windows and the next step. No chat messages.

| Parameter | Type | Required |
|---|---|---|
| `trade_id` | integer or public id | yes |

### `list_my_trades` — My trades · commits: no
The user's trades, most recent first, summarized.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `only_active` | boolean | no | default true |

### `wait_for_trade_update` — Wait for a trade change · commits: no
Long-polls up to `timeout_s` seconds (max 200) and returns as soon as the trade's state or `crypto_funded` changes from the values passed.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `trade_id` | integer or public id | yes | |
| `since_state` | string | yes | the state last seen, e.g. `active`, `declared` |
| `since_funded` | boolean | no | the `crypto_funded` value last seen |
| `timeout_s` | integer 1–200 | no | default 60 |

### `get_proof_upload_link` — Where the human uploads the receipt · commits: no
The payment receipt can only be uploaded by the human. Returns a dedicated link that shows the exact amount and the account to pay, with a single button to attach the receipt and declare the payment. Give it to the user once `crypto_funded` is true.

| Parameter | Type | Required |
|---|---|---|
| `trade_id` | integer or public id | yes |

### `cancel_trade` — Cancel a trade before paying · commits: no
Cancels a private escrow request the user created, only while the counterparty has not accepted it. A trade opened from a marketplace offer cannot be cancelled by the taker; if nobody locks or pays in time it expires by itself.

| Parameter | Type | Required |
|---|---|---|
| `trade_id` | integer or public id | yes |

## Money-moving actions (always approved by the human)

### `fund_escrow` — Lock crypto from balance · commits: human
Asks to lock the trade's crypto from the user's iEscrow balance into escrow. Creates an approval and returns `approve_url`. Only for trades where the user is the crypto seller.

| Parameter | Type | Required |
|---|---|---|
| `trade_id` | integer or public id | yes |

### `confirm_received` — Confirm the fiat arrived · commits: human
Asks to confirm that the fiat payment arrived, which releases the crypto to the other party. Creates an approval and returns `approve_url`; the human must first check their own account. Irreversible once approved.

| Parameter | Type | Required |
|---|---|---|
| `trade_id` | integer or public id | yes |

### `check_approval` — Status of a human approval · commits: no
Status of an approval created by `fund_escrow`, `confirm_received` or `publish_ad`: `pending`, `approved` (with the resulting trade state), `rejected`, `expired` or `failed`.

| Parameter | Type | Required |
|---|---|---|
| `public_id` | string | yes |

## Seller side (marketplace ads)

### `publish_ad` — Publish a marketplace ad · commits: human · scope `ads:write`
Prepares a marketplace ad. `side=sell`: the user sells crypto from their iEscrow balance for fiat (the crypto is reserved when approved). `side=buy`: the user buys crypto with fiat. Fixed price or floating margin over market, per-order limits in fiat, accepted payment accounts. Creates an approval and returns `approve_url`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `side` | `sell` \| `buy` | no | default `sell` |
| `crypto` | string | yes | e.g. `USDT` |
| `fiat` | string | yes | e.g. `ARS`, `USD`, `EUR` |
| `total_crypto_amount` | number | yes | total to sell or buy |
| `price_mode` | `fixed` \| `floating` | yes | |
| `fiat_per_crypto` | number | for `fixed` | price per 1 crypto |
| `margin_pct` | number −20…50 | no | for `floating`; default 0 |
| `min_order_fiat` | number | yes | minimum per order, in fiat |
| `max_order_fiat` | number | yes | maximum per order, in fiat (≤ total × price) |
| `payment_account_ids` | integer[] (1–10) | yes | accounts buyers can pay to (from `get_my_payment_accounts`) |
| `idempotency_key` | string | no | |

### `list_my_ads` — My marketplace ads · commits: no
The user's own ads: state, remaining liquidity, price mode, margin and per-order limits.

No parameters.

### `close_ad` — Close an ad · commits: no
Closes one of the user's open ads. Unsold liquidity returns to the iEscrow balance; trades already opened from it continue normally.

| Parameter | Type | Required |
|---|---|---|
| `ad_id` | integer | yes |
