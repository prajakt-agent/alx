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
┌──────────────────────────────────────────────────────┐
│                  STORE MANAGER                        │
│               (Orchestrator Agent)                    │
│                                                      │
│  • Monitors all customer conversations               │
│  • Routes to specialist agents                       │
│  • Escalates to store owner when needed              │
└──────┬────────┬────────┬────────┬────────┬───────────┘
       │        │        │        │        │
┌──────▼──┐ ┌──▼────┐ ┌─▼──────┐ ┌▼──────┐ ┌▼─────────┐
│  Sales  │ │Social │ │Catalog │ │Order  │ │Analytics │
│  Agent  │ │Media  │ │Agent   │ │Agent  │ │Agent     │
└────┬────┘ └──┬────┘ └──┬─────┘ └──┬────┘ └──┬───────┘
     │         │         │          │          │
  Shopify   Instagram  Shopify   Shopify    Shopify
  Products  Posts/     Inventory  Orders    Reports
  API       Stories   API        Webhooks   API
```

## 2. Agent Specifications

### 2.1 Store Manager Agent (Orchestrator)

| Property | Value |
|----------|-------|
| Role | Orchestrator — monitors all channels, delegates to specialists |
| Model | Claude Sonnet (planning quality) |
| Channels | Instagram DM webhooks, WhatsApp, Telephony |
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

## 3. Integration Architecture

### 3.1 Instagram DMs — Webhook Service

Instagram DMs require a **public HTTPS endpoint** for webhooks. No existing MCP server handles this end-to-end.

```
Instagram Platform → Webhook POST → ALX Webhook Service → Store Manager Agent
                                          ↓
                                    Message Queue (Redis/SQLite)
                                          ↓
                                    Agent processes & responds
                                          ↓
                                    Instagram Send API ← response
```

**Required Meta App Permissions:**
- `instagram_manage_messages` — read/send DMs
- `pages_messaging` — messaging via Page
- `pages_manage_metadata` — webhook subscriptions
- `instagram_content_publish` — posting (Marketing Agent)
- `instagram_basic` — profile info
- `pages_read_engagement` — Page insights

**Webhook Events to Subscribe:**
- `messages` — new DM received
- `messaging_postbacks` — quick reply button tapped
- `message_reactions` — customer reacts to message

**24h Messaging Window:**
- Customer initiates → 24h window opens
- Agent can send unlimited messages within window
- After 24h → can only send approved message templates (limited)
- No cold outreach to customers who haven't messaged first

### 3.2 Shopify Integration

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

**MCP Server Option:** `hcu7/shopify-mcp` (59 tools, most complete) or official Anthropic-verified Shopify connector.

### 3.3 WhatsApp Integration

Already set up via Hermes gateway. Used for:
- **Inbound:** Customer conversations (Store Manager routes)
- **Outbound to owner:** Order notifications, analytics reports, escalations

### 3.4 Telephony

Via Exotel (India) or Twilio/Vapi for incoming calls:
- Calls transcribed in real-time
- Store Manager processes transcript
- Delegates to Sales/Order Agent as needed
- Response spoken back or followed up via WhatsApp

## 4. Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Orchestration | Hermes Agent (delegate_task) | Agent coordination |
| Storefront | Shopify (GraphQL Admin API) | Products, orders, inventory |
| Customer Chat | Instagram Graph API + WhatsApp | DM monitoring & response |
| Social Media | Instagram Content Publishing API | Posts, stories, reels |
| Telephony | Exotel / Twilio + Vapi | Incoming calls |
| Webhook Service | FastAPI / Flask (public endpoint) | Instagram DM webhooks |
| Media Hosting | Cloudflare R2 / S3 | Public URLs for Instagram posts |
| Task Tracking | Linear (ALX team) | Development tracking |
| Notifications | WhatsApp (store owner) | Order alerts, reports |
| Data Parsing | Python (openpyxl/pandas) | Excel → Shopify import |

## 5. Deployment Architecture

### Phase 1: Mac + Webhook Tunnel

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
│  Webhook Service (FastAPI)                        │
│  └─ Cloudflare Tunnel / ngrok → public HTTPS      │
│                                                   │
│  Cron: analytics report (daily), token refresh    │
└──────────────────────────────────────────────────┘
```

### Phase 2: Cloud VPS

```
┌─────────────────────────────────────────┐
│  VPS (Ubuntu, $20-40/mo)                │
│                                         │
│  Docker Compose:                        │
│  ├─ hermes-gateway (all agent profiles) │
│  ├─ webhook-service (Instagram DMs)     │
│  ├─ redis (message queue)               │
│  └─ caddy (reverse proxy, auto-TLS)     │
│                                         │
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
- [ ] Instagram Business/Creator account linked to Facebook Page
- [ ] Facebook Developer App with required permissions
- [ ] Shopify store with Custom App (Admin API token)
- [ ] WhatsApp number for customer communication
- [ ] WhatsApp number for owner notifications (can be same)
- [ ] Media hosting (S3/R2) for Instagram post uploads
- [ ] Public domain/URL for webhook endpoint

### Technical Setup
- [ ] Meta App review & approval (instagram_manage_messages requires review)
- [ ] Shopify Custom App with 9 scopes
- [ ] Instagram webhook subscription configuration
- [ ] Shopify webhook registration (orders/create)
- [ ] Hermes profiles for each agent
- [ ] AgentMail inboxes (or shared inbox)
- [ ] Cron job for analytics agent
- [ ] Token refresh automation (Instagram: every 45 days)

## 8. Key Constraints & Risks

| Constraint | Impact | Mitigation |
|-----------|--------|-----------|
| Instagram 24h reply window | Must respond within 24h of customer message | Typing indicator + quick acknowledgment immediately |
| Instagram DM rate limit (~200/hr) | Can't spam responses | Queue + rate limiter |
| Meta App Review required | 2-4 weeks for messaging permission approval | Start review process early |
| Shopify GraphQL rate limits | Cost-based throttling | Batch operations, respect Retry-After |
| Image-to-product matching | No built-in visual search in Shopify | Use vision model to describe image → text search |
| Instagram posts need public URLs | Can't upload directly from disk | Use S3/R2 as media CDN |
| Long-lived tokens expire (60 days) | Service interruption if not refreshed | Cron job to refresh at day 45 |

## 9. References

- [Instagram Graph API Research](./docs/instagram-api.md)
- [Shopify API Research](./docs/shopify-api.md)
- [Instagram Messenger API Docs](https://developers.facebook.com/docs/messenger-platform/instagram)
- [Shopify Admin API Docs](https://shopify.dev/docs/api/admin)
- [Shopify Draft Orders](https://shopify.dev/docs/api/admin-rest/2024-10/resources/draftorder)
