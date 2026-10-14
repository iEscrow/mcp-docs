# iEscrow MCP server

iEscrow is a cryptocurrency escrow platform: in every trade the crypto is held by iEscrow until both sides have delivered, so nobody has to hand over first. This repository documents the **iEscrow MCP server**, which lets an AI agent (Claude, ChatGPT or any MCP host) operate an iEscrow account on behalf of the user who authorized it.

- **Endpoint:** `https://api.iescrow.io/mcp`
- **Transport:** Streamable HTTP, stateless
- **Auth:** OAuth 2.1 (authorization server at `https://api.iescrow.io`), PKCE, dynamic client registration
- **Tools:** 16, listed in [TOOLS.md](TOOLS.md)
- **Website:** https://iescrow.io · **For AI agents:** https://iescrow.io/ai

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
| `ads:write` | prepare marketplace ads (always subject to the human's approval); not advertised in the resource metadata, requested explicitly by hosts that support it |

Scopes are chosen on the consent screen and can be revoked at any time from *Connected apps* in the user's iEscrow settings.

## Rate limits

Per connection: 60 calls per minute, 10 trade openings per hour, 20 money-moving approval requests per hour. One concurrent long-poll (`wait_for_trade_update`) per user. A limited call returns `error_key` and `retry_after_s`.

## A typical flow (buying crypto with a bank transfer)

1. `get_readiness` — confirm the account can trade.
2. `get_quote` with `side=buy`, the crypto, the fiat and the amount — pick an offer; note `compatible_account_ids`.
3. `open_trade` with the offer id, the amount and a compatible `payment_account_id`.
4. `get_trade` — wait until `crypto_funded` is true. **The human must not pay before that.**
5. `get_proof_upload_link` — give the user the link; it shows the exact amount (the cents are the payment key) and the account to pay. The user pays, attaches the receipt and declares the payment there.
6. `wait_for_trade_update` — the trade moves to `released` when the seller confirms. The crypto is now in the user's iEscrow balance.

Selling is the mirror image: `open_trade` with `side=sell`, then `fund_escrow` (the human approves the lock by link), then `confirm_received` once the fiat arrives (again approved by link).

## Errors

Every error is returned as structured content with `error: true`, an HTTP-like `status`, a stable `error_key` and a human-readable `message`. When an action needs the human, the response carries `approval_required: true` and `approval.approve_url`; poll `check_approval` with `approval.public_id` to learn the outcome.

## Security

The MCP server never opens the database and holds no secrets of its own: it verifies the user's access token (RS256, issuer and audience pinned) and forwards the same bearer to the API. Reports about the agent channel: contact@iescrow.io.

## About this repository

This repository holds the public documentation of the server and the manifest used to list it in MCP registries. The server's source code is not published here.
