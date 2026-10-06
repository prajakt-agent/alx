# ALX-2: eCommerce AI Team Architecture

## 1. Overview

An AI-powered team that runs an eCommerce store across Instagram, WhatsApp, and phone — with Shopify as the storefront. The **Store Manager** orchestrates specialist agents for sales, marketing, catalog management, orders, and analytics.

```
Customer Channels                    Store Owner
─────────────────                    ───────────
Instagram DM ──┐                     ┌── WhatsApp (notifications)
WhatsApp ──────┤                     │
Phone Call ────┘                     │
       ↓                             │
┌─────────────────────────────────┐  │
│          ZERNIO API             │  │  ← Unified social/messaging layer
│  (MCP Server + Webhooks)        │  │
│  Instagram DMs, WhatsApp,       │  │
│  Posts, Stories, Analytics       │  │
└──────────────┬──────────────────┘  │
               ↓                     │
┌──────────────────────────────────────────────────────┐
│                  STORE MANAGER                        │
│               (Orchestrator Agent)                    │
│                                                      │
│  • Receives messages via Zernio webhooks              │
│  • Routes to specialist agents                       │
│  • Responds via Zernio messaging API                 │
│  • Escalates to store owner when needed              │
└──────┬────────┬────────┬────────┬────────┬───────────┘
       │        │        │        │        │
┌──────▼──┐ ┌──▼────┐ ┌─▼──────┐ ┌▼──────┐ ┌▼─────────┐
│  Sales  │ │Social │ │Catalog │ │Order  │ │Analytics │
│  Agent  │ │Media  │ │Agent   │ │Agent  │ │Agent     │
└────┬────┘ └──┬────┘ └──┬─────┘ └──┬────┘ └──┬───────┘
     │         │         │          │          │
  Shopify   Zernio    Shopify   Shopify    Shopify
  Products  Posts/    Inventory  Orders    Reports
  API       Stories   API       Webhooks   API
            API
```

## 2. Agent Specifications

### 2.1 Store Manager Agent (Orchestrator)

| Property | Value |
|----------|-------|
| Role | Orchestrator — monitors all channels, delegates to specialists |
| Model | Claude Sonnet (planning quality) |
| Channels | Zernio webhooks (`message.received`), WhatsApp, Telephony |
| Key Logic | Classify intent → route to correct agent → synthesise response |

**Intent Classification:**
```
Customer message → Store Manager classifies:
├─ Product inquiry (availability, sizes, price, similar) → Sales Agent
├─ Order status / tracking → Order Agent
├─ Complaint / escalation → Store Owner (WhatsApp alert)
├─ General question → Store Manager handles directly
└─ Spam / irrelevant → Ignore (mark as read)
```

**24-Hour Rule (Instagram DM):**
Instagram enforces a 24h reply window from last customer message. Store Manager must:
1. Acknowledge within minutes (typing indicator + quick response)
2. Delegate to specialist immediately
3. Return response to customer within the window
4. If approaching 24h, send a proactive "we're looking into this" message

### 2.2 Sales Agent

| Property | Value |
|----------|-------|
| Role | Handle product queries, share payment links |
| Model | Gemini Flash (high volume, fast) |
| APIs | Shopify Products, Storefront Search, Draft Orders |
| Capabilities | Product lookup, image matching, size/variant info, discount lookup, payment link generation |

**Workflow — "Is this product available?"**
```
1. Customer sends image or text query via Instagram DM
2. Store Manager delegates to Sales Agent with message + context
3. Sales Agent:
   a. If image → describe product, search Shopify by keywords
   b. If text → search Shopify Products API (title, product_type, vendor)
   c. Return: product details, variants, sizes, prices, inventory status
   d. If customer wants to buy → create Draft Order → share invoiceUrl
4. Response sent back through Store Manager → Instagram DM
```

**Shopify Payment Links (Draft Orders):**
```graphql
mutation {
  draftOrderCreate(input: {
    lineItems: [{ variantId: "gid://shopify/ProductVariant/123", quantity: 1 }]
  }) {
    draftOrder {
      id
      invoiceUrl    # ← This is the payment link to share with customer
    }
  }
}
```

### 2.3 Social Media Marketing Agent

| Property | Value |
|----------|-------|
| Role | Post photos/videos to Instagram feed and stories |
| Model | Gemini Flash |
| APIs | Instagram Content Publishing API |
| Input | Photos/videos + caption from store owner |

**Publishing Flow (Two-Step):**
```
1. Create media container:
   POST /{IG_USER_ID}/media
     ?image_url={PUBLIC_URL}&caption={TEXT}&access_token={TOKEN}

2. Publish container:
   POST /{IG_USER_ID}/media_publish
     ?creation_id={CONTAINER_ID}&access_token={TOKEN}
```

**Supported content types:**
- Feed images (JPEG, max 8MB)
- Reels (MP4, 3-90s, max 1GB)
- Stories (image or video, 24h visibility)
- Carousels (2-10 items)

**Rate limit:** 100 posts per 24h rolling window.

**Note:** Media must be at a public URL — the agent needs to upload photos to a hosting service (S3, Cloudflare R2, or Shopify CDN) before publishing.

### 2.4 Store Catalog Agent

| Property | Value |
|----------|-------|
| Role | Keep Shopify store updated with new inventory |
| Model | Gemini Flash |
| APIs | Shopify Admin API (GraphQL), Excel parsing |
| Input | Excel/CSV files with product data |

**Workflow — Bulk Inventory Update:**
```
1. Store owner sends Excel file (via WhatsApp or email)
2. Agent parses Excel → validates data (title, price, SKU, variants, images)
3. Converts to JSONL format
4. Uploads via Shopify staged uploads:
   a. stagedUploadsCreate → get upload URL
   b. Upload JSONL to staged URL
   c. bulkOperationRunMutation → async product creation
5. Monitors bulk operation status
6. Reports results to store owner (created, updated, errors)
```

**Individual product updates:**
```graphql
mutation {
  productSet(synchronous: true, input: {
    title: "New Product",
    productOptions: [{ name: "Size", values: [{ name: "S" }, { name: "M" }, { name: "L" }] }],
    variants: [
      { optionValues: [{ optionName: "Size", name: "S" }], price: "29.99" }
    ]
  }) {
    product { id title }
    userErrors { field message }
  }
}
```

### 2.5 Order Agent

| Property | Value |
|----------|-------|
| Role | Monitor new orders, notify store owner |
| Model | Gemini Flash |
| APIs | Shopify Orders API, Webhooks |
| Output | WhatsApp notifications to store owner |

**Webhook-Driven (Preferred):**
```
Shopify webhook: orders/create
    ↓
Order Agent receives notification
    ↓
Formats order summary:
  - Customer name, items, total
  - Shipping address
  - Management URL: https://{store}.myshopify.com/admin/orders/{ORDER_ID}
    ↓
Sends to store owner via WhatsApp
```

**Polling Fallback:**
```
Every 5 minutes:
  GET /admin/api/2024-10/orders.json?status=any&created_at_min={LAST_CHECK}
```

**Order Status Queries (from customers):**
```
Customer asks "where is my order?" via DM
    → Store Manager delegates to Order Agent
    → Order Agent looks up by customer email/phone
    → Returns: order status, tracking number, estimated delivery
```

### 2.6 Analytics Agent

| Property | Value |
|----------|-------|
| Role | Generate daily reports for store owner |
| Model | Gemini Flash |
| APIs | Shopify Analytics, Orders, Products |
| Schedule | Daily at configured time (cron job) |

**Daily Report Contents:**
- Orders placed today (count, total revenue)
- Top selling products
- Instagram DM conversations (count, response time)
- Inventory alerts (low stock items)
- Comparison with previous day/week

**Delivery:** WhatsApp message to store owner with formatted report.

## 3. Integration Architecture — Zernio as Unified Layer

### 3.1 Why Zernio

Zernio replaces the entire custom Instagram/WhatsApp integration layer:

| Before (Custom) | After (Zernio) |
|-----------------|---------------|
| Build webhook service for Instagram DMs | ✅ Zernio `message.received` webhook |
| Meta App review (2-4 weeks) | ✅ Already approved |
| Public HTTPS endpoint for webhooks | ✅ Zernio handles it |
| Instagram token refresh (every 60 days) | ✅ Zernio manages OAuth |
| Media hosting (S3/R2) for Instagram posts | ✅ Zernio presigned uploads |
| Separate WhatsApp Business API setup | ✅ Unified via Zernio |
| Custom DM read/reply code | ✅ Zernio messaging inbox API |

### 3.2 Zernio MCP Server

Zernio exposes an MCP server at `https://mcp.zernio.com/mcp` that plugs directly into Hermes:

```yaml
# Add to Hermes MCP config
mcpServers:
  zernio:
    url: https://mcp.zernio.com/mcp
    transport: streamable-http
    env:
      ZERNIO_API_KEY: "${ZERNIO_API_KEY}"
```

**Available MCP Scopes:**
- `posts:read` / `posts:write` — Publishing (Social Media Agent)
- `messaging:write` — DMs and WhatsApp (Store Manager, Sales Agent)
- `analytics:read` — Engagement metrics (Analytics Agent)
- `accounts:read` — Connected account info
- `commerce:write` — Shopify integration (if used)

### 3.3 Instagram DMs via Zernio

**Inbound (customer → agent):**
```
Customer sends Instagram DM
    → Zernio receives via Meta webhook
    → Zernio fires `message.received` webhook to our endpoint
    → Store Manager Agent processes message
    → Delegates to Sales/Order Agent
    → Response sent via Zernio messaging API
    → Zernio delivers reply as Instagram DM
```

**Zernio Webhook Events for eCommerce:**
| Event | Use Case |
|-------|----------|
| `message.received` | New DM from customer (Instagram, WhatsApp) |
| `message.delivered` | Confirm delivery (WhatsApp) |
| `message.read` | Customer read our reply |
| `message.failed` | Delivery failed — retry or alert |
| `conversation.started` | New customer thread opened |
| `comment.received` | Comment on Instagram post |
| `reaction.received` | Emoji reaction on message |

**24h Reply Window:** Zernio handles this transparently — messages within 24h of customer's last message go through standard messaging API. Beyond 24h, WhatsApp requires approved templates.

**Message Types Supported:**
- Text (1000 chars for Instagram)
- Images (8MB)
- Video/Audio/PDF (25MB)
- Quick replies (up to 13 options) — great for product selection
- Product templates — for sharing product cards
- Typing indicators — show "agent is typing..."

### 3.4 Instagram Posting via Zernio (Social Media Agent)

```bash
# Upload media
curl -X POST https://zernio.com/api/v1/media/presign \
  -H "Authorization: Bearer $ZERNIO_API_KEY" \
  -d '{"filename": "product.jpg", "contentType": "image/jpeg"}'
# → returns uploadUrl + publicUrl

# Publish to Instagram
curl -X POST https://zernio.com/api/v1/posts \
  -H "Authorization: Bearer $ZERNIO_API_KEY" \
  -d '{
    "content": "New arrivals! 🛍️ #fashion #newcollection",
    "mediaItems": [{"type": "image", "url": "https://media.zernio.com/temp/..."}],
    "platforms": [{"platform": "instagram", "accountId": "IG_ACCOUNT_ID"}],
    "publishNow": true
  }'
```

**Supported Instagram content:**
- Feed posts (single image/video)
- Reels (3-90s video)
- Stories (image or video)
- Carousels (2-10 items)
- Scheduled posts (set `scheduledFor` instead of `publishNow`)

### 3.5 WhatsApp via Zernio

Same API for both customer communication and store owner notifications:

```bash
# Send message to store owner
curl -X POST https://zernio.com/api/v1/messaging/send \
  -H "Authorization: Bearer $ZERNIO_API_KEY" \
  -d '{
    "platform": "whatsapp",
    "accountId": "WA_ACCOUNT_ID",
    "to": "+91XXXXXXXXXX",
    "text": "🛒 New Order #1042 - ₹1,499 - Blue Floral Dress (M)"
  }'
```

### 3.6 Shopify Integration (unchanged)

**Authentication:** Custom App with Admin API access token.

**Required Scopes:**
```
read_products, write_products        # Sales + Catalog Agent
read_orders, write_orders            # Order Agent
read_inventory, write_inventory      # Catalog Agent
write_draft_orders                   # Sales Agent (payment links)
read_analytics                       # Analytics Agent
read_customers                       # Order Agent (lookup)
```

**MCP Server Option:** Official Anthropic-verified Shopify MCP connector or `hcu7/shopify-mcp` (59 tools).

### 3.7 Telephony (unchanged)

Via Exotel (India) or Twilio/Vapi for incoming calls. Zernio also supports phone numbers for calls/SMS if preferred as a unified solution.

## 4. Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Orchestration | Hermes Agent (delegate_task) | Agent coordination |
| Social & Messaging | **Zernio API + MCP Server** | Instagram DMs, WhatsApp, posting, analytics |
| Storefront | Shopify (GraphQL Admin API) | Products, orders, inventory |
| Telephony | Exotel / Twilio + Vapi | Incoming calls |
| Task Tracking | Linear (ALX team) | Development tracking |
| Data Parsing | Python (openpyxl/pandas) | Excel → Shopify import |

## 5. Deployment Architecture

### Phase 1: Mac + Zernio (No Custom Infra)

```
┌──────────────────────────────────────────────────┐
│  Mac (always-on)                                  │
│                                                   │
│  Hermes Gateway (multiplexed profiles)            │
│  ├─ store-manager profile                         │
│  ├─ sales-agent profile                           │
│  ├─ marketing-agent profile                       │
│  ├─ catalog-agent profile                         │
│  ├─ order-agent profile                           │
│  └─ analytics-agent profile                       │
│                                                   │
│  MCP Servers:                                     │
│  ├─ Zernio (Instagram DMs, WhatsApp, posting)     │
│  └─ Shopify (products, orders, inventory)         │
│                                                   │
│  Zernio webhook → Hermes webhook adapter          │
│  Cron: analytics report (daily), token refresh    │
└──────────────────────────────────────────────────┘
```

**No custom webhook service needed.** Zernio handles Meta/WhatsApp webhooks and forwards events to our Hermes webhook endpoint. No public URL, no tunnel, no FastAPI service.
```

### Phase 2: Cloud VPS

```
┌─────────────────────────────────────────┐
│  VPS (Ubuntu, $20-40/mo)                │
│                                         │
│  Docker Compose:                        │
│  ├─ hermes-gateway (all agent profiles) │
│  └─ caddy (reverse proxy, auto-TLS)     │
│                                         │
│  MCP: Zernio (hosted) + Shopify         │
│  Public domain: shop.yourdomain.com     │
└─────────────────────────────────────────┘
```

## 6. Data Flow Examples

### Customer Asks "Is this available in size M?"

```
1. Customer sends DM on Instagram with product image
2. Instagram webhook → Webhook Service → Store Manager
3. Store Manager classifies: product inquiry → delegates to Sales Agent
4. Sales Agent:
   a. Analyzes image → identifies product keywords
   b. Searches Shopify: products(query: "blue dress")
   c. Finds match → checks variant inventory for size M
   d. Formats response: "Yes! Blue Floral Dress is available in M.
      Price: ₹1,499. Shall I send you the payment link?"
5. Store Manager sends response via Instagram DM API
6. Customer says "Yes, send link"
7. Sales Agent creates Draft Order → gets invoiceUrl
8. Sends: "Here's your payment link: {invoiceUrl}"
```

### New Order Placed

```
1. Customer completes payment on Shopify
2. Shopify webhook: orders/create → Order Agent
3. Order Agent formats:
   "🛒 New Order #1042
    Customer: Priya S.
    Items: Blue Floral Dress (M) × 1
    Total: ₹1,499
    Address: Mumbai, MH
    Manage: https://store.myshopify.com/admin/orders/5678"
4. Sends to store owner via WhatsApp
```

### Store Owner Sends New Inventory Excel

```
1. Owner sends Excel file via WhatsApp/Email
2. Catalog Agent receives file
3. Parses Excel: validates columns (title, price, SKU, sizes, images)
4. Converts to JSONL, uploads via Shopify bulk operations
5. Reports: "Added 45 products, 3 errors (missing images for rows 12, 34, 67)"
6. Owner fixes → resends → Catalog Agent handles corrections
```

## 7. Setup Requirements

### From Store Owner
- [ ] Instagram Business/Creator account
- [ ] Shopify store with Custom App (Admin API token)
- [ ] WhatsApp Business number
- [ ] Zernio account (connect Instagram + WhatsApp in dashboard)

### Technical Setup
- [ ] Zernio API key + connect Instagram & WhatsApp accounts
- [ ] Zernio MCP server added to Hermes
- [ ] Shopify Custom App with 9 scopes
- [ ] Shopify MCP server added to Hermes
- [ ] Zernio webhook → Hermes webhook adapter configured
- [ ] Shopify webhook registration (orders/create)
- [ ] Hermes profiles for each agent
- [ ] Cron job for analytics agent

## 8. Key Constraints & Risks

| Constraint | Impact | Mitigation |
|-----------|--------|-----------|
| Instagram 24h reply window | Must respond within 24h of customer message | Zernio handles; agent sends quick acknowledgment immediately |
| Instagram DM rate limit (~200/hr) | Can't spam responses | Zernio manages rate limiting |
| Shopify GraphQL rate limits | Cost-based throttling | Batch operations, respect Retry-After |
| Image-to-product matching | No built-in visual search in Shopify | Use vision model to describe image → text search |
| Zernio dependency | Single vendor for social/messaging layer | REST API is standard; can migrate to direct Meta API if needed |
| Zernio pricing | $6/account for 1-10 accounts | First 2 free; minimal cost vs building custom infra |

## 9. References

- [Instagram Graph API Research](./docs/instagram-api.md) — Direct API reference (backup for Zernio)
- [Shopify API Research](./docs/shopify-api.md)
- [Zernio API Docs](https://docs.zernio.com)
- [Zernio LLM Docs](https://zernio.com/llms.txt)
- [Zernio MCP Server](https://mcp.zernio.com/mcp)
- [Shopify Admin API Docs](https://shopify.dev/docs/api/admin)
