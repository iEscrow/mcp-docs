# Tools

Twenty tools. "Commits" says whether the call can move money: **no** means read-only or reversible; **human** means the tool only creates an approval that the account owner must confirm by link with their two-factor code.

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

### `get_swap_quote` — Get executable crypto-for-crypto offers · commits: no
Best executable offers to swap one crypto for another, for the given amount. Both legs are held in escrow and verified on chain, so there is no payment account, no bank and nobody confirming a transfer by hand. The platform fee is charged on **each leg separately**, so every offer states what the user delivers with its fee and what they receive net, each in its own currency. Give either `give_amount` or `get_amount`, not both. Each offer also carries `open_trade_args`: pass them to `open_trade` exactly as given.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `give_currency` | string | yes | crypto the user delivers, e.g. `USDT` |
| `get_currency` | string | yes | crypto the user receives, e.g. `BTC` |
| `give_amount` | number | one of the two | amount the user wants to deliver |
| `get_amount` | number | one of the two | amount the user wants to receive |
| `limit` | integer 1–20 | no | |

For crypto against government money use `get_quote` instead.

### `open_trade` — Open a trade from an offer · commits: no (reserves the offer)
Takes an offer from `get_quote` or `get_swap_quote` and opens the trade. For a crypto-for-crypto offer pass the `open_trade_args` from the quote and no payment account. With `side=buy` the user will pay fiat and receive the crypto in their iEscrow balance. With `side=sell` the user sells crypto against a buy ad and then locks it with `fund_escrow`. Never delivers to an external wallet. Idempotent within 10 minutes. Call `get_trade` afterwards for the exact amount and account.

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

### `publish_swap_ad` — Publish a crypto-for-crypto ad · commits: human · scope `ads:write`
Prepares an ad to swap one crypto for another: the user offers `give_currency` from their iEscrow balance and receives `get_currency` there. Approving reserves the full give amount plus its fee from the balance until the ad is sold or closed, so the advertised liquidity is always real. No payment account is involved. Creates an approval and returns `approve_url`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `give_currency` | string | yes | crypto the user offers, e.g. `USDT` |
| `get_currency` | string | yes | crypto the user wants in exchange, e.g. `BTC` |
| `total_give_amount` | number | yes | total to offer, in `give_currency` |
| `price_mode` | `fixed` \| `floating` | yes | |
| `get_per_give` | number | for `fixed` | how much `get_currency` per 1 `give_currency` |
| `margin_pct` | number −20…50 | no | for `floating`; default 0 |
| `min_order_get` | number | yes | minimum per order, in `get_currency` |
| `max_order_get` | number | yes | maximum per order, in `get_currency` |
| `idempotency_key` | string | no | |

### `list_my_ads` — My marketplace ads · commits: no
The user's own ads: state, remaining liquidity, price mode, margin and per-order limits.

No parameters.

### `close_ad` — Close an ad · commits: no
Closes one of the user's open ads. Unsold liquidity returns to the iEscrow balance; trades already opened from it continue normally.

| Parameter | Type | Required |
|---|---|---|
| `ad_id` | integer | yes |

## Private escrow with one specific person

These two do not go through the marketplace: they prepare a request addressed to one person, who then has 24 hours to accept it.

Both require that person's **reference number**, which they find in their own iEscrow account and share with the requester. A username alone is not enough. That is what stops an agent from firing off requests at a list of usernames, and it means the number has to come from the user: never guess it, ask for it. `get_readiness` returns the connected user's own number, to pass on to others.

Nothing is created when these are called. The whole request is held in the approval, and only the owner's confirmation by link sends it.

### `open_private_request` — Request an escrow for goods or services · commits: human · scope `trades:write`
A private escrow to buy or sell a physical product or a service paid in crypto. The buyer's crypto is held by iEscrow and released when the buyer confirms delivery, or milestone by milestone for a service.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `counterparty_username` | string | yes | the other person's iEscrow username |
| `counterparty_reference` | string | yes | their reference number, which they have to give you |
| `kind` | `product` \| `service` | yes | product releases in full on confirmation; service releases milestone by milestone |
| `role` | `buyer` \| `seller` | yes | what the connected user is in this deal |
| `title` | string | yes | what is being bought or sold |
| `description` | string | no | the terms: what is delivered, when, in what condition |
| `crypto` | string | yes | crypto the deal is priced and paid in |
| `amount` | number | yes | total in that crypto |
| `milestones` | array of `{title, percentage}` | for `service` | the percentages must add up to exactly 100 |

### `open_private_swap` — Request a swap with one specific person · commits: human · scope `trades:write`
A private escrow to exchange one asset for another at a price the two have already agreed: crypto for crypto, or crypto for fiat. To trade with the open market use `get_swap_quote` or `get_quote` instead.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `counterparty_username` | string | yes | the other person's iEscrow username |
| `counterparty_reference` | string | yes | their reference number, which they have to give you |
| `give_currency` | string | yes | what the connected user sends |
| `give_amount` | number | yes | how much they send |
| `get_currency` | string | yes | what they receive |
| `get_amount` | number | yes | how much they receive; the two amounts set the agreed price |

## Catalog of products and services

A public catalog of physical products and services sold for crypto. Every result carries the seller's **real track record** (identity verified, completed trades, success and dispute rates, star rating); there is no invented score. Buying holds the buyer's crypto with iEscrow and releases it only when the buyer confirms delivery, or when the delivery window plus seven days runs out without a dispute. Empty results mean nobody is selling that right now: never invent listings.

### `search_listings` — Search products and services for sale · commits: no
| Parameter | Type | Required | Notes |
|---|---|---|---|
| `query` | string | no | free text, e.g. `macbook pro m3` |
| `kind` | `product` \| `service` | no | |
| `category` | string | no | one of the categories returned in `categories` |
| `condition` | `new` \| `used` \| `refurbished` | no | products only |
| `min_price_usd`, `max_price_usd` | number | no | prices are in US dollars |
| `ships_to` | ISO-2 | no | the buyer's country; worldwide listings always match |
| `min_rating` | 1–5 | no | sellers with no ratings yet are excluded |
| `min_completed_trades` | integer | no | |
| `identity_verified_only` | boolean | no | recommended above a few hundred dollars |
| `sort` | `newest` \| `price_asc` \| `price_desc` \| `reputation` | no | |
| `limit` | 1–50 | no | |

### `get_listing` — One listing, with a quote for the connected user · commits: no
Full detail plus exactly how much crypto would be locked (USD converted at the current rate, plus iEscrow's fee) and how many days the seller has to deliver before the escrow releases on its own.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `listing_id` | number or string | yes | `listing_id` or `public_id` from `search_listings` |
| `quantity` | integer | no | default 1 |

### `get_seller_profile` — Reputation of a seller · commits: no
Completed trades, cancellations at fault, dispute rate, average star rating and whether the identity is verified. Use `seller.user_id` from `search_listings`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `user_id` | integer | yes | |

### `buy_listing` — Buy a listing, paying on delivery through escrow · commits: human · scope `trades:write`
Creates an approval. When the user approves it with their 2FA, iEscrow creates the escrow **and locks the crypto from the user's iEscrow balance in the same act**, so the balance must cover the quote from `get_listing`. The seller then ships with a tracking number (`get_trade` shows it); the money is released when the user confirms delivery with `confirm_received`, or by itself when the delivery window plus seven days runs out without a dispute.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `listing_id` | number or string | yes | `listing_id` or `public_id` |
| `quantity` | integer | no | default 1 |
| `shipping_address` | object | for products | `{ name, line1, line2?, city, state?, postal_code?, country (ISO-2), phone? }`; ask the user, never guess |
| `note` | string | no | a note for the seller |
