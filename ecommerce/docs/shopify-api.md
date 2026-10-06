# Shopify API Research for ALX-2 eCommerce Agent Integration

**Date:** 2026-10-06  
**Purpose:** API reference for Sales Agent, Store Catalog Agent, Order Agent, Analytics Agent

---

## 1. Products API (Sales Agent & Store Catalog Agent)

### Base URL
```
https://{store}.myshopify.com/admin/api/2024-10
```
**Note:** REST Admin API entered legacy status Oct 1 2024. Since Apr 1 2025, all new apps must use **GraphQL**. Existing REST apps keep working but get no new features. **Recommend building on GraphQL from day one.**

### REST Endpoints (Legacy but functional)

#### List Products
```bash
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/products.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"
```
**Query params:** `ids`, `limit` (max 250), `since_id`, `title`, `vendor`, `handle`, `product_type`, `collection_id`, `created_at_min/max`, `updated_at_min/max`, `published_status` (published|unpublished|any), `status` (active|archived|draft), `fields`

#### Get Single Product
```bash
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/products/{PRODUCT_ID}.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"
```

#### Product Count
```bash
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/products/count.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"
```

#### Create Product (with variants)
```bash
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/products.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "product": {
      "title": "Burton Custom Freestyle",
      "body_html": "<strong>Good snowboard!</strong>",
      "vendor": "Burton",
      "product_type": "Snowboard",
      "status": "draft",
      "variants": [{"price": "99.99", "sku": "BOARD-001"}]
    }
  }'
```

### GraphQL (Recommended)

#### productSet Mutation (create or update)
The `productSet` mutation is the recommended way to sync products from external sources. It can:
- Create new products or update existing ones in a single request
- Handle variants, options, images, inventory quantities
- Run synchronously or asynchronously (`synchronous: false` for large inputs)

```graphql
mutation createProduct($productSet: ProductSetInput!, $synchronous: Boolean!) {
  productSet(synchronous: $synchronous, input: $productSet) {
    product {
      id
      title
      variants(first: 10) {
        nodes { id title price inventoryQuantity }
      }
    }
    userErrors { field message }
  }
}
```

### Product Search
- REST: Use `title` query param for exact match filtering
- GraphQL: Use `products(query: "title:*keyword*")` for full-text search
- Storefront API: `products(query: "keyword")` for customer-facing search

### Product Status Values
| Status | Description |
|--------|-------------|
| `active` | Available for sale |
| `archived` | Hidden, no longer available |
| `draft` | Not ready for sale |

### Pagination
REST uses cursor-based pagination via `Link` header with `page_info`:
```
Link: <https://store.myshopify.com/admin/api/2024-10/products.json?page_info=abc123&limit=50>; rel="next"
```
**Ceiling:** Cannot page past 25,000 objects — use bulk operations or date filters beyond that.

---

## 2. Orders API (Order Agent)

### List Orders (monitor new orders)
```bash
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/orders.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"
```
**Query params:** `ids`, `limit`, `since_id`, `created_at_min/max`, `updated_at_min/max`, `processed_at_min/max`, `status` (open|closed|cancelled|any), `financial_status`, `fulfillment_status`, `fields`

**For monitoring new orders:** Use `created_at_min` with ISO timestamp, or `since_id` for incremental polling. **Better: use `orders/create` webhook** (see Section 8).

### Get Single Order
```bash
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/orders/{ORDER_ID}.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"
```

### Order Count
```bash
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/orders/count.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"
```

### Update Order
```bash
curl -X PUT "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/orders/{ORDER_ID}.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"order":{"id":{ORDER_ID},"note":"Updated note","tags":"priority,vip"}}'
```

### Get Customer Orders
```bash
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/customers/{CUSTOMER_ID}/orders.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"
```

### Order Status Fields
| Field | Values |
|-------|--------|
| `financial_status` | pending, authorized, partially_paid, paid, partially_refunded, refunded, voided |
| `fulfillment_status` | null (unfulfilled), partial, fulfilled, restocked |

### Order Management URLs
- Admin order URL: `https://{store}.myshopify.com/admin/orders/{ORDER_ID}`
- Order status page (customer-facing): available in order's `order_status_url` field

### Cancel Order
```bash
curl -X POST "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/orders/{ORDER_ID}/cancel.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reason":"customer","email":true,"restock":true}'
```
Cancel reasons: `customer`, `fraud`, `inventory`, `declined`, `other`

---

## 3. Payment/Checkout API (Sales Agent — Payment Links)

### Draft Orders = Payment Links
Draft orders are the primary mechanism for creating payment links in Shopify. When a draft order is created, it generates an `invoice_url` — a secure checkout link the customer can use to pay.

#### Create Draft Order
```bash
curl -X POST "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/draft_orders.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "draft_order": {
      "line_items": [
        {"variant_id": 12345, "quantity": 1}
      ],
      "customer": {"id": 67890},
      "use_customer_default_address": true
    }
  }'
```

#### Custom Line Items (products not in inventory)
```json
{
  "draft_order": {
    "line_items": [
      {"title": "Custom Service", "price": "150.00", "quantity": 1, "taxable": true}
    ]
  }
}
```

#### Send Invoice (email with checkout link)
```bash
curl -X POST "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/draft_orders/{DRAFT_ORDER_ID}/send_invoice.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"draft_order_invoice": {"to": "customer@example.com", "subject": "Your Invoice", "custom_message": "Please complete your purchase"}}'
```

#### Complete Draft Order (mark as paid)
```bash
curl -X PUT "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/draft_orders/{DRAFT_ORDER_ID}/complete.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"
```

#### GraphQL Draft Order (Recommended)
```graphql
mutation draftOrderCreate($input: DraftOrderInput!) {
  draftOrderCreate(input: $input) {
    draftOrder {
      id
      invoiceUrl          # <-- THIS IS THE PAYMENT LINK
      status
      totalPrice
    }
    userErrors { field message }
  }
}
```

The `invoiceUrl` field on the draft order response is the shareable payment/checkout link.

### Key Draft Order Capabilities for Sales Agent
- Create orders for phone/chat/in-person sales
- Send invoices with secure checkout links
- Custom line items for non-inventory products
- Discount/wholesale pricing
- Pre-orders
- Payment terms (pay later)
- Reserve inventory with `reserveInventoryUntil` input

---

## 4. Inventory Management API (Store Catalog Agent — Bulk from Excel/CSV)

### Inventory Levels
```bash
# List inventory levels
curl "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/inventory_levels.json?inventory_item_ids={ITEM_ID}" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN"

# Adjust inventory (relative change)
curl -X POST "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/inventory_levels/adjust.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"location_id":{LOCATION_ID},"inventory_item_id":{ITEM_ID},"available_adjustment":5}'

# Set inventory (absolute value)
curl -X POST "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/inventory_levels/set.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"location_id":{LOCATION_ID},"inventory_item_id":{ITEM_ID},"available":100}'

# Connect inventory item to a location
curl -X POST "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/inventory_levels/connect.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"location_id":{LOCATION_ID},"inventory_item_id":{ITEM_ID}}'
```

### Bulk Product Import from Excel/CSV — Recommended Workflow

**Step-by-step for Store Catalog Agent:**

1. **Parse Excel/CSV** → Convert rows to JSONL format (one product per line)
2. **Upload JSONL to Shopify** via `stagedUploadsCreate` mutation
3. **Run bulk mutation** via `bulkOperationRunMutation`
4. **Monitor completion** via webhook or polling
5. **Download results** from the returned URL

#### Step 1: Create JSONL file from Excel
Each line = one `ProductInput` object:
```jsonl
{"input":{"title":"Product A","variants":[{"price":"29.99","sku":"SKU-A","inventoryQuantities":[{"availableQuantity":100,"locationId":"gid://shopify/Location/123"}]}]}}
{"input":{"title":"Product B","variants":[{"price":"49.99","sku":"SKU-B","inventoryQuantities":[{"availableQuantity":50,"locationId":"gid://shopify/Location/123"}]}]}}
```

#### Step 2: Upload JSONL
```graphql
mutation {
  stagedUploadsCreate(input: [{
    resource: BULK_MUTATION_VARIABLES,
    filename: "bulk_op_vars",
    mimeType: "text/jsonl",
    httpMethod: POST
  }]) {
    stagedTargets {
      url
      resourceUrl
      parameters { name value }
    }
    userErrors { field message }
  }
}
```
Then POST the JSONL file to the returned URL as multipart form data.

#### Step 3: Run bulk mutation
```graphql
mutation {
  bulkOperationRunMutation(
    mutation: "mutation call($input: ProductInput!) { productCreate(input: $input) { product { id title variants(first: 10) { edges { node { id title inventoryQuantity } } } } userErrors { message field } } }",
    stagedUploadPath: "tmp/.../bulk_op_vars"
  ) {
    bulkOperation { id url status }
    userErrors { message field }
  }
}
```

#### Alternative: productSet for sync/upsert
For syncing from external data source (Excel), `productSet` is recommended over `productCreate` because it handles create-or-update in one call. Can also run asynchronously.

#### Shopify CLI shortcut
```bash
shopify app bulk execute --watch --variable-file products.jsonl --query \
  'mutation productCreate($input: ProductInput!) { productCreate(input: $input) { product { id title } userErrors { message field } } }'
```

### GraphQL Inventory Mutations
```graphql
# Set inventory quantities (recommended over REST)
mutation inventorySetQuantities($input: InventorySetQuantitiesInput!) {
  inventorySetQuantities(input: $input) {
    inventoryAdjustmentGroup { reason }
    userErrors { field message }
  }
}
```

---

## 5. Storefront API vs Admin API

| Dimension | Admin API | Storefront API |
|-----------|-----------|----------------|
| **Purpose** | Store management (backend) | Customer-facing (frontend) |
| **Protocol** | GraphQL (primary) + REST (legacy) | GraphQL only |
| **Auth** | OAuth / private access token | Public storefront access token |
| **Token safety** | Server-side ONLY, never expose | Safe for client-side code |
| **Auth header** | `X-Shopify-Access-Token` | `X-Shopify-Storefront-Access-Token` |
| **Read access** | Full: orders, customers, inventory, products, financials | Published products, collections, cart |
| **Write access** | Full CRUD on all resources | Cart + checkout only |
| **Rate limits** | Cost-based bucket (100-2000 pts/sec by plan) | Scales with buyer traffic |
| **Who calls it** | Backend server, app server | Browser, mobile app, headless storefront |
| **Scope type** | `read_products`, `write_orders`, etc. | `unauthenticated_read_product_listings`, etc. |

### Which to Use for ALX-2 Agents
- **Sales Agent:** Admin API (create draft orders, payment links, read full product data)
- **Store Catalog Agent:** Admin API (create/update products, manage inventory)
- **Order Agent:** Admin API (read orders, fulfillment, cancellations)
- **Analytics Agent:** Admin API (sales data, financial reports)
- **Customer-facing product browsing widget (if any):** Storefront API

**All four agents need the Admin API. Storefront API only if building a customer-facing frontend.**

---

## 6. Authentication

### Option A: Custom App (Recommended for single-store integration)
1. Go to Shopify Admin → Settings → Apps and sales channels → Develop apps
2. Create app → Configure Admin API scopes
3. Install → Get **Admin API access token** (`shpat_...`)
4. Use header: `X-Shopify-Access-Token: shpat_your_token`

### Option B: OAuth 2.0 (For multi-store / app store distribution)
- Standard OAuth 2.0 flow with PKCE (required for apps after Jan 2026)
- Merchant installs app → grants scopes → app stores resulting token
- Tokens must be stored securely server-side

### Required Access Scopes for ALX-2

| Scope | Agent(s) | Access |
|-------|----------|--------|
| `read_products` | Sales, Catalog, Analytics | Read products, variants, collections |
| `write_products` | Catalog | Create/update products from Excel |
| `read_orders` | Order, Analytics | Read order data |
| `write_orders` | Order, Sales | Update orders, create draft orders |
| `read_inventory` | Catalog, Analytics | Read stock levels |
| `write_inventory` | Catalog | Update stock levels from Excel |
| `read_customers` | Sales, Order | Customer info for orders |
| `write_customers` | Sales | Create/update customers |
| `read_all_orders` | Analytics | Orders older than 60 days (requires approval) |

### Environment Variables
```bash
SHOPIFY_STORE_DOMAIN=your-store.myshopify.com
SHOPIFY_ACCESS_TOKEN=shpat_xxxxxxxxxxxxx
SHOPIFY_API_VERSION=2024-10
```

### Security Rules
- **NEVER** expose Admin API tokens in client-side code
- Store tokens in environment variables, never in git
- Validate webhook signatures via `X-Shopify-Hmac-SHA256` header
- Rotate tokens after employee offboarding

---

## 7. Existing MCP Servers & Integrations

### Official Shopify MCP Connector (Anthropic Verified)
- **URL:** `https://setup.shopify.com/mcp`
- **Status:** Anthropic-verified, added April 2026
- **Capabilities:** Store builder, inventory management, discount codes, order review, analytics
- **Integration:** Add directly to Claude via connector URL
- **Limitation:** Designed for Claude.ai web interface, not for custom agent pipelines

### Open Source MCP Servers

#### 1. shopify-mcp by hcu7 (Most Complete)
- **URL:** https://github.com/hcu7/shopify-mcp
- **Features:** 59 built-in tools + 5 custom tools across 5 domains
- **Use as:** MCP server for AI agents OR standalone CLI
- **Compatible with:** Claude, Cursor, Windsurf

#### 2. shopify-store-mcp by xmqywx
- **URL:** https://github.com/xmqywx/shopify-mcp-server
- **Install:** `npm install -g @xmqywxkris/shopify-store-mcp`
- **Claude Code integration:**
  ```bash
  claude mcp add shopify -- npx -y shopify-store-mcp \
    --env SHOPIFY_STORE_DOMAIN=your-store.myshopify.com \
    --env SHOPIFY_ACCESS_TOKEN=shpat_your_token
  ```
- **Features:** Products, orders, inventory, customers, analytics
- **Scopes needed:** `read_products`, `write_products`, `read_orders`, `read_inventory`, `write_inventory`, `read_customers`

#### 3. shopify-mcp by ptrcole
- **URL:** https://github.com/ptrcole/shopify-mcp
- **Tools:** `get_products`, `get_product`, `create_product`, `update_product`, `get_orders`, `get_order`, `get_customers`, `search_products`
- **Claude Desktop config:**
  ```json
  {
    "mcpServers": {
      "shopify": {
        "command": "node",
        "args": ["/path/to/shopify-mcp-server/dist/index.js"],
        "env": {
          "SHOPIFY_SHOP_DOMAIN": "your-store.myshopify.com",
          "SHOPIFY_ACCESS_TOKEN": "your-access-token"
        }
      }
    }
  }
  ```

#### 4. shopify-mcp by dzunglaviet
- **URL:** https://github.com/dzunglaviet/shopify-mcp
- **Features:** Orders, customers, products, inventory via natural language

### No Existing Hermes Skills for Shopify
No Shopify-specific skills found in the current Hermes skills catalog. **Opportunity to create a `shopify` skill** for the ALX-2 project.

---

## 8. Webhooks for Order Notifications (Order Agent)

### Create Webhook (REST)
```bash
curl -X POST "https://$SHOPIFY_STORE_DOMAIN/admin/api/2024-10/webhooks.json" \
  -H "X-Shopify-Access-Token: $SHOPIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhook": {
      "topic": "orders/create",
      "address": "https://your-app.com/webhooks/orders",
      "format": "json"
    }
  }'
```

### Create Webhook (GraphQL — Recommended)
```graphql
mutation webhookSubscriptionCreate(
  $topic: WebhookSubscriptionTopic!,
  $webhookSubscription: WebhookSubscriptionInput!
) {
  webhookSubscriptionCreate(topic: $topic, webhookSubscription: $webhookSubscription) {
    webhookSubscription { id topic format uri }
  }
}
```

### Relevant Webhook Topics for ALX-2

| Topic | Agent | Use Case |
|-------|-------|----------|
| `orders/create` | Order Agent | **Primary** — trigger on new order |
| `orders/updated` | Order Agent | Track order status changes |
| `orders/fulfilled` | Order Agent | Fulfillment notifications |
| `orders/cancelled` | Order Agent | Cancellation alerts |
| `products/create` | Catalog Agent | Sync new products |
| `products/update` | Catalog Agent | Product change detection |
| `products/delete` | Catalog Agent | Product removal detection |
| `inventory_levels/update` | Catalog Agent | Stock level changes |
| `refunds/create` | Order/Analytics | Refund tracking |
| `customers/create` | Sales Agent | New customer detection |

### Webhook Security — HMAC Verification
Every webhook includes `X-Shopify-Hmac-SHA256` header. **Always verify** by computing HMAC-SHA256 of the request body using the app's API secret key.

```javascript
const crypto = require('crypto');

function verifyWebhook(body, hmacHeader, secret) {
  const hash = crypto.createHmac('sha256', secret)
    .update(body, 'utf8')
    .digest('base64');
  return crypto.timingSafeEqual(
    Buffer.from(hash),
    Buffer.from(hmacHeader)
  );
}
```

### Webhook Reliability
- Delivery is **best-effort**, not guaranteed
- Always implement a **reconciliation/polling pass** as safety net
- Implement **idempotency** — webhooks may deliver duplicates
- Respond with **200 status immediately**, process async
- Shopify retries failed deliveries (non-2xx) up to 19 times over 48 hours

### App Configuration (TOML)
```toml
[access_scopes]
scopes = "read_orders,write_orders,read_products,write_products,read_inventory,write_inventory,read_customers"

[webhooks]
api_version = "2024-10"

[[webhooks.subscriptions]]
topics = ["orders/create", "orders/updated", "orders/cancelled"]
uri = "/webhooks"

[[webhooks.subscriptions]]
topics = ["products/create", "products/update", "products/delete"]
uri = "/webhooks"

[[webhooks.subscriptions]]
topics = ["inventory_levels/update"]
uri = "/webhooks"
```

---

## 9. Rate Limits Summary

### GraphQL Admin API (Recommended)
| Plan | Points/second | Max single query |
|------|--------------|-----------------|
| Standard (Basic/Shopify) | 100 | 1,000 points |
| Advanced Shopify | 200 | 1,000 points |
| Shopify Plus | 1,000 | 1,000 points |
| Enterprise | 2,000 | 1,000 points |

### REST Admin API (Legacy)
| Plan | Bucket size | Leak rate |
|------|------------|-----------|
| Standard | 40 requests | 2/second |
| Shopify Plus | 80 requests (or 40 × 10) | 4/second |

### Rate Limit Response Headers
- REST: `X-Shopify-Shop-Api-Call-Limit` (e.g., `32/40`)
- GraphQL: `extensions.cost.throttleStatus` with `maximumAvailable`, `currentlyAvailable`, `restoreRate`

### Key Limits
| Limit | Value |
|-------|-------|
| Max single query cost | 1,000 points |
| Array input maximum | 250 items |
| Pagination ceiling | 25,000 objects |
| Concurrent bulk operations | 5 per app per shop |
| Bulk operation result URL lifetime | 1 week |

### Throttle Handling Pattern
```javascript
async function shopifyGraphQL(query, variables) {
  const res = await fetch(SHOPIFY_ENDPOINT, {
    method: "POST",
    headers: adminHeaders,
    body: JSON.stringify({ query, variables }),
  });
  const data = await res.json();
  const throttle = data.extensions?.cost?.throttleStatus;
  const wasThrottled = data.errors?.some(e => e.extensions?.code === "THROTTLED");
  if (wasThrottled && throttle) {
    const needed = data.extensions.cost.requestedQueryCost;
    const deficit = needed - throttle.currentlyAvailable;
    const waitMs = Math.ceil(deficit / throttle.restoreRate) * 1000;
    await new Promise(r => setTimeout(r, waitMs));
    return shopifyGraphQL(query, variables);
  }
  return data;
}
```

---

## 10. Architecture Recommendations for ALX-2

### Recommended Stack
- **API:** GraphQL Admin API (not REST — REST is legacy)
- **Auth:** Custom App with private access token (single store) or OAuth (multi-store)
- **Real-time:** Webhooks for order notifications + periodic reconciliation polling
- **Bulk ops:** `bulkOperationRunMutation` for Excel/CSV imports
- **MCP Server:** Use `shopify-mcp` (hcu7) or `shopify-store-mcp` (xmqywx) as starting point

### Agent → API Mapping

| Agent | Primary APIs | Key Operations |
|-------|-------------|----------------|
| **Sales Agent** | Products, Draft Orders | Search products, get details/pricing, create draft orders (payment links), send invoices |
| **Store Catalog Agent** | Products, Inventory, Bulk Operations | Bulk import from Excel → JSONL → `bulkOperationRunMutation`, update inventory levels |
| **Order Agent** | Orders, Webhooks | Subscribe to `orders/create` webhook, list/get orders, track fulfillment status |
| **Analytics Agent** | Orders, Products, Inventory | Query orders with date ranges, aggregate sales data, inventory reports |

### Integration Pattern
```
Excel/CSV → [Catalog Agent] → Parse to JSONL → stagedUploadsCreate → bulkOperationRunMutation → Shopify
Customer Chat → [Sales Agent] → Search Products → Create Draft Order → Send Invoice URL
Shopify Webhook → [Order Agent] → Process orders/create → Notify team → Track fulfillment
Scheduled Job → [Analytics Agent] → Query orders/products → Aggregate → Report
```

### Next Steps
1. Create Shopify custom app and obtain access token
2. Evaluate existing MCP servers (hcu7/shopify-mcp has 59 tools — most complete)
3. Build Excel→JSONL converter for Store Catalog Agent
4. Set up webhook endpoint for Order Agent
5. Create Hermes `shopify` skill with API patterns and auth setup
