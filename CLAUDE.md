# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## What is this

**RappiMCP** — an MCP server that lets Claude control Rappi Chile (rappi.cl): browse stores, search products, manage the cart, and open checkout. Uses Chrome DevTools Protocol (CDP) for authentication and checkout navigation; all other operations hit Rappi's internal HTTP API directly.

> **Region**: Rappi Chile only (`rappi.cl` / `services.rappi.cl`). Other Rappi country domains are not supported.

## Quick start

```bash
pip install -r requirements.txt
```

Then authenticate once from Claude:
```
rappi_auth()   # launches Chrome, waits for login, writes .env automatically
```

## Architecture

```
Claude ──tools──▶ rappi_mcp.py ──HTTP──▶ services.rappi.cl
                       │
                       └──CDP (port 9223)──▶ Chrome (auth + checkout)
```

- `rappi_mcp.py` — MCP server entry point (FastMCP), tool definitions, auth + CDP logic
- `rappi_client.py` — Authenticated HTTP client with automatic token refresh

## Tools

| Tool | Description |
|------|-------------|
| `rappi_auth` | Extract tokens from Chrome via CDP and write to `.env` |
| `rappi_reload` | Reload `.env` tokens into the running server without restart |
| `rappi_list_stores` | List stores by category (`market`, `restaurant`, `farma`, …) |
| `rappi_get_store` | Look up store metadata by `store_id` |
| `rappi_search_products` | Search products within a store |
| `rappi_get_cart` | Read current cart for a store type |
| `rappi_add_to_cart` | Add or increment an item in the cart |
| `rappi_remove_from_cart` | Remove an item from the cart |
| `rappi_clear_cart` | Empty the cart for a store |
| `rappi_checkout` | Navigate Chrome to the checkout page for a store |

## Typical order flow

```
1. rappi_list_stores(category="market")           # find a supermarket
2. rappi_search_products(store_id, "leche")        # search products
3. rappi_add_to_cart(store_id, store_type, ...)    # add to cart
4. rappi_get_cart(store_type)                      # review cart
5. rappi_checkout(store_type)                      # open checkout in Chrome
```

## Agent behavior rules

- **Always confirm before checkout.** Before calling `rappi_checkout`, show the user the cart contents and ask for explicit confirmation.
- **Always confirm before clearing the cart.** `rappi_clear_cart` is destructive — ask first.
- **Use `store_type` exactly as returned.** Slugs like `"turbo_rappidrinks_nc"` are case-sensitive and must match exactly across cart tools.
- **Search in Spanish.** Rappi Chile's catalog is in Spanish — queries like `"leche descremada"` return far better results than English equivalents.
- **Handle auth errors gracefully.** If any tool returns an auth/401 error, suggest calling `rappi_auth()` to refresh credentials.
- **Checkout is per-store.** Rappi does not support multi-store checkout. Each store cart must be checked out separately via its own `store_type`.

## Session management

- Credentials are stored in `.env` (git-ignored)
- Tokens rotate automatically — `RappiClient` refreshes them transparently on `x-refresh-token: true` responses
- If tokens expire, re-run `rappi_auth()`
- CDP runs on port `9223` (not the default `9222`) to avoid conflicts
