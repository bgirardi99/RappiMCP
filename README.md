# RappiMCP

```
Program : RappiMCP
Author  : Bastian Girardi
Date    : April 2, 2026
Purpose : Built to automate grocery and delivery purchases on Rappi Chile
          through Claude — eliminating the need to manually browse the app
          when you just want to tell an AI what you need and have it ordered.
```

---

An MCP (Model Context Protocol) server that gives Claude full control over Rappi Chile — browse stores, search products, manage the cart, and open checkout — all through natural language.

## How it works

The server is built with [FastMCP](https://github.com/jlowin/fastmcp) and exposes tools that Claude can call. It communicates with Rappi via their internal web API (`services.rappi.cl`), authenticated with tokens extracted directly from a Chrome browser session using the Chrome DevTools Protocol (CDP).

```
Claude ──tools──▶ rappi_mcp.py ──HTTP──▶ services.rappi.cl
                       │
                       └──CDP──▶ Chrome (auth + checkout navigation)
```

## Setup

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure Claude Code

Add this to your Claude Code MCP settings (`claude_desktop_config.json` or equivalent):

```json
{
  "mcpServers": {
    "RappiMCP": {
      "command": "python",
      "args": ["C:/path/to/RappiMCP/rappi_mcp.py"]
    }
  }
}
```

### 3. Authenticate

On first use, call `rappi_auth` to extract your Rappi credentials from Chrome and save them to `.env`. No manual token copying needed.

## Authentication

Rappi uses rotating JWT tokens stored as browser cookies. The auth flow:

1. **`rappi_auth(use_existing_chrome=False)`** — launches a dedicated Chrome window at rappi.cl (on CDP port `9223`) and waits up to 2 minutes for you to log in
2. Once logged in, it reads three cookies via CDP:
   - `rappi.data` (Base64 JSON) → `user_id`
   - `deviceid` → `device_id`
   - `rappi_refresh_token` → `refresh_token`
3. Exchanges the refresh token for a fresh `access_token` via `POST /api/rocket/refresh-token`
4. Writes all four values to `.env`

**`rappi_auth(use_existing_chrome=True)`** (default) — attaches to an already-running Chrome with `--remote-debugging-port=9223` and reads cookies immediately (you must already be logged in).

### Token refresh

`RappiClient` handles token rotation automatically. If any API response includes the header `x-refresh-token: true`, it transparently refreshes both tokens and retries the request.

### Environment variables

```
RAPPI_ACCESS_TOKEN   — Bearer token for API calls
RAPPI_REFRESH_TOKEN  — Used to obtain new access tokens
RAPPI_DEVICE_ID      — UUID identifying the browser session
RAPPI_USER_ID        — Numeric Rappi user ID
```

See `.env.example` for the expected format.

## Tools reference

### `rappi_auth(use_existing_chrome=True)`
Extract credentials from Chrome and write them to `.env`. Set `use_existing_chrome=False` to launch a fresh Chrome window and wait for login.

---

### `rappi_reload()`
Reload tokens from `.env` and restart the API client — without restarting the MCP server. Useful after manually editing `.env`.

---

### `rappi_list_stores(category, query, limit)`
List stores available for delivery near your saved address.

| Param | Default | Description |
|-------|---------|-------------|
| `category` | `"market"` | Store category: `"market"`, `"restaurant"`, `"farma"`, `"licores"`, `"express-big"`, or any Rappi store_type slug |
| `query` | `""` | Optional name filter |
| `limit` | `50` | Max results |

Returns: `store_id`, `name`, `store_type`, `lat`, `lng`, `eta`, `shipping_cost`.

> `store_id` is used by `rappi_search_products`. `store_type` is used by all cart tools.

---

### `rappi_get_store(store_id)`
Look up metadata for a specific store by its numeric ID. Returns `store_type` (the slug needed for cart operations).

---

### `rappi_search_products(store_id, query, size, offset)`
Search products within a store. Queries in Spanish work best.

| Param | Default | Description |
|-------|---------|-------------|
| `store_id` | required | From `rappi_list_stores` |
| `query` | required | Search term (e.g. `"cerveza"`, `"preservativos"`) |
| `size` | `40` | Results per page (max 40) |
| `offset` | `0` | Pagination offset |

Returns: `composite_id`, `name`, `trademark`, `price`, `real_price`, `discount`, `quantity`, `unit_type`, `sale_type`, `in_stock`.

> Use `composite_id` and `sale_type` when adding to cart.

---

### `rappi_get_cart(store_type)`
Read the current cart contents for a given store. Returns a list of items with `composite_id`, `units`, `sale_type`, `name`, `price`.

---

### `rappi_add_to_cart(store_id, store_type, composite_id, units, sale_type)`
Add a product to the cart (or increment its quantity if already present). Internally: reads current cart → appends/increments → writes back via a full PUT.

---

### `rappi_remove_from_cart(store_id, store_type, composite_id)`
Remove a specific product from the cart by its `composite_id`.

---

### `rappi_clear_cart(store_id, store_type)`
Empty the entire cart for a store.

---

### `rappi_checkout(store_type)`
Navigate Chrome to the checkout page for a store. Each store has its own checkout URL:

```
https://www.rappi.cl/checkout/{store_type}
```

Examples:
- `turbo_rappidrinks_nc` → `rappi.cl/checkout/turbo_rappidrinks_nc`
- `cruz_verde_rappido_nc` → `rappi.cl/checkout/cruz_verde_rappido_nc`
- `lider` → `rappi.cl/checkout/lider`

> Rappi does not support consolidated multi-store checkout. Each store cart must be checked out separately.

## Typical flow

```
1. rappi_auth()                          # authenticate once
2. rappi_list_stores(category="farma")   # find pharmacies
3. rappi_search_products(store_id, "preservativos")  # search
4. rappi_add_to_cart(store_id, store_type, composite_id, 1, "U")
5. rappi_checkout(store_type)            # open checkout in Chrome
```

## File structure

```
RappiMCP/
├── rappi_mcp.py      # MCP server — tool definitions, auth logic, CDP helpers
├── rappi_client.py   # Rappi API client with automatic token refresh
├── requirements.txt  # Python dependencies
├── .env              # Credentials (git-ignored)
└── .env.example      # Credentials template
```

## Notes

- The server uses CDP port `9223` (not the default `9222`) to avoid conflicts with other tools
- Checkout navigation also uses CDP — the same Chrome instance used for auth
- `store_type` slugs come from `rappi_list_stores` and must be passed exactly as returned (e.g. `"turbo_rappidrinks_nc"`, not `"turbo"`)
