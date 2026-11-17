# iEscrow MCP servers

iEscrow is an escrow for crypto, fiat, goods and services: in every deal the crypto is held by iEscrow until both sides have delivered, so nobody has to hand over first. People use it to buy and sell crypto P2P, to buy products and services paid in crypto, and to close deals between companies.

This repository documents the **two iEscrow MCP servers**:

| | Public, read-only | With an account |
|---|---|---|
| **Endpoint** | `https://api.iescrow.io/mcp/public` | `https://api.iescrow.io/mcp` |
| **Auth** | none | OAuth 2.1, PKCE, dynamic client registration |
| **What it does** | search the catalog of products and services, show a listing with its quote, show a seller's track record, quote crypto offers | everything the public one does, plus open trades, buy, publish ads and follow them on behalf of the user |
| **Moves money** | never | only when the human approves by link with their 2FA |
| **Tools** | 5 | 25, listed in [TOOLS.md](TOOLS.md) |

- **Transport:** Streamable HTTP, stateless, on both
- **Server cards:** `https://api.iescrow.io/mcp/public/server-card` and `https://api.iescrow.io/mcp/server-card`
- **Website:** https://iescrow.io · **For AI agents:** https://iescrow.io/ai

## The public server: look without an account

`https://api.iescrow.io/mcp/public` answers anyone, with no login and no token. It exposes five read-only tools, with the same names and the same response shape as on the server with an account:

| Tool | What it returns |
|---|---|
| `search_products_and_services` | Listings from the public catalog of products and services sold for crypto, with price in USD, shipping times and the seller's real track record |
| `get_product_or_service` | One listing in full, plus how much crypto a buyer would lock to buy it right now, iEscrow's fee included |
| `get_seller_profile` | Completed trades, cancellations at fault, dispute rate, star rating and whether the seller's identity is verified |
| `get_quote` | Live P2P offers to buy or sell a crypto with fiat money, with the all-in amounts |
| `get_swap_quote` | Live offers to swap one crypto for another |

It cannot open a trade, buy, publish, or see anyone's balance or accounts. Every listing carries `url`, the page on iescrow.io where a person completes the purchase with the payment held in escrow, and every response says how to continue: on the web, or by connecting the user's account to the server with an account.

Connect it like any remote MCP server without authentication:

```
claude mcp add --transport http iescrow-marketplace https://api.iescrow.io/mcp/public
```

In Claude, ChatGPT, Gemini or Perplexity, add a custom connector with that URL and no authentication.

Limits: 30 tool calls per minute per IP address. A limited call returns `error_key` and `retry_after_s`.

The rest of this document is about the server with an account.

## The principle: the agent prepares, the human commits

The agent can quote, open trades, read balances and states, and prepare marketplace ads. Anything that moves money is **not executed by the agent**: the tool creates an approval and returns a link. The account owner opens that link, sees exactly what is about to happen, and approves it with their two-factor code.

What the agent can never do, by design:

- withdraw funds from the account,
- add or change payment accounts,
- upload a payment receipt (only the human can, through a dedicated link),
- lock crypto into escrow or confirm that a payment arrived without the human's approval,
- deliver crypto to an external wallet (everything lands in the user's iEscrow balance).

The API enforces this with an allow-list of routes for agent tokens; the MCP server only wraps those routes.

## Requirements

An iEscrow account with:

1. a verified email,
2. approved identity verification (KYC),
3. two-factor authentication (TOTP) enabled.

The connection screen (`/connect`) checks the three before issuing a token. Trades can be executed by an agent only in the countries where iEscrow is enabled for agents; `get_readiness` tells you which.

## Connecting

### Claude (claude.ai, desktop, mobile)

Settings → Connectors → *Add custom connector* → name `iEscrow`, URL `https://api.iescrow.io/mcp`. Claude discovers the authorization server, opens the iEscrow consent screen, and the connection is ready once you approve it.

### Claude Code

```
claude mcp add --transport http iescrow https://api.iescrow.io/mcp
```

Then run `/mcp` inside Claude Code to complete the login.

### ChatGPT

Settings → Connectors → *Create* (developer mode) → name `iEscrow`, MCP server URL `https://api.iescrow.io/mcp`, authentication *OAuth*. ChatGPT registers itself as a client and sends you to the iEscrow consent screen.

### Any other MCP host

Point the host to the URL above with the Streamable HTTP transport. The server publishes `/.well-known/oauth-protected-resource`, and the authorization server publishes `/.well-known/oauth-authorization-server`; hosts that implement the MCP authorization spec need nothing else. See [examples/](examples/) for configuration snippets.

## Scopes

| Scope | Grants |
|---|---|
| `profile:read` | who the user is and whether the account is ready to trade |
| `quotes:read` | executable offers and prices on the marketplace |
| `wallet:read` | balances and masked payment accounts |
| `trades:read` | state of the user's trades and ads |
| `trades:write` | open trades, request approvals to lock crypto or confirm payments, cancel own requests |
| `ads:write` | prepare marketplace ads (always subject to the human's approval) |

Scopes are chosen on the consent screen and can be revoked at any time from *Connected apps* in the user's iEscrow settings.

## Rate limits

On the server with an account, per connection: 60 calls per minute, 10 openings per hour (taking a marketplace offer, requesting a private escrow or buying from the catalog), 20 money-moving approval requests per hour. One concurrent long-poll (`wait_for_trade_update`) per user. A limited call returns `error_key` and `retry_after_s`.

## A typical flow (buying crypto with a bank transfer)

1. `get_readiness` — confirm the account can trade.
2. `get_quote` with `side=buy`, the crypto, the fiat and the amount — pick an offer; note `compatible_account_ids`.
3. `open_trade` with the offer id, the amount and a compatible `payment_account_id`.
4. `get_trade` — wait until `crypto_funded` is true. **The human must not pay before that.**
5. `get_proof_upload_link` — give the user the link; it shows the exact amount (the cents are the payment key) and the account to pay. The user pays, attaches the receipt and declares the payment there.
6. `wait_for_trade_update` — the trade moves to `released` when the seller confirms. The crypto is now in the user's iEscrow balance.

Selling is the mirror image: `open_trade` with `side=sell`, then `fund_escrow` (the human approves the lock by link), then `confirm_received` once the fiat arrives (again approved by link).

Swapping one crypto for another is shorter, because no bank is involved and nobody has to confirm a transfer: `get_swap_quote`, then `open_trade` with the offer's `open_trade_args`, then `fund_escrow`. Both legs are verified on chain and the escrow settles by itself once each side has funded.

To deal with one specific person instead of the market, use `open_private_request` (a product or a service, paid milestone by milestone if you want) or `open_private_swap`. Both need that person's reference number, which they share with you from their own account, so ask the user for it rather than guessing: that requirement is what keeps an agent from firing requests at a list of usernames.

## Errors

Every error is returned as structured content with `error: true`, an HTTP-like `status`, a stable `error_key` and a human-readable `message`. When an action needs the human, the response carries `approval_required: true` and `approval.approve_url`; poll `check_approval` with `approval.public_id` to learn the outcome.

## Security

The MCP server never opens the database and holds no secrets of its own: it verifies the user's access token (RS256, issuer and audience pinned) and forwards the same bearer to the API. Reports about the agent channel: contact@iescrow.io.

## About this repository

This repository holds the public documentation of the two servers and the manifests used to list them in MCP registries (`server.json` for the server with an account, `server-marketplace.json` for the public one). The server's source code is not published here.
