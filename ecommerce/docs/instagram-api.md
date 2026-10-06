# ALX-2: Instagram Graph API Research for eCommerce AI Agents

> Last updated: 2026-10-06  
> Scope: DM monitoring/responding (Store Manager + Sales Agent), content publishing (Social Media Marketing Agent)

---

## Table of Contents
1. [Authentication & Setup](#1-authentication--setup)
2. [Instagram DMs — Messenger API](#2-instagram-dms--messenger-api)
3. [Content Publishing API](#3-content-publishing-api)
4. [Rate Limits & Quotas](#4-rate-limits--quotas)
5. [Existing MCP Servers & SDKs](#5-existing-mcp-servers--sdks)
6. [Architecture Recommendations for ALX-2](#6-architecture-recommendations-for-alx-2)

---

## 1. Authentication & Setup

### Prerequisites
- **Instagram Professional account** (Business or Creator) — personal accounts have zero API access since Dec 4, 2024 (Basic Display API is dead)
- **Facebook Developer App** at developers.facebook.com → Create App → type "Business" → add Instagram product
- For the FB Login path: Instagram account must be linked to a Facebook Page

### Two Authentication Paths

| | **Path A — Instagram Login** | **Path B — Facebook Login** |
|---|---|---|
| Base host | `graph.instagram.com` | `graph.facebook.com/v25.0` |
| User logs in with | Instagram credentials | Facebook credentials |
| Facebook Page required? | **No** | **Yes** |
| Publish permission | `instagram_business_content_publish` | `instagram_content_publish` |
| Basic permission | `instagram_business_basic` | `instagram_basic` + `pages_read_engagement` + `pages_show_list` |
| Messaging permission | `instagram_business_manage_messages` | `instagram_manage_messages` + `pages_manage_metadata` |
| Comments permission | `instagram_business_manage_comments` | `instagram_manage_comments` |
| Hashtag search / Discovery | ✗ Not available | ✓ Available |
| Resumable video upload | ✗ | ✓ (`rupload.facebook.com`) |
| Product tagging | ✗ | ✓ |
| Token type | Instagram User access token | Page access token |

**Recommendation for ALX-2:** Use **Path B (Facebook Login)** — it's needed for DM webhooks (which go through the linked Page) and gives access to discovery/hashtag features useful for the Marketing agent.

### OAuth 2.0 Token Flow

```
1. Authorization URL (browser redirect):
   https://www.facebook.com/v25.0/dialog/oauth
     ?client_id={FB_APP_ID}
     &redirect_uri={REDIRECT_URI}
     &response_type=code
     &scope=instagram_basic,instagram_content_publish,instagram_manage_messages,
            instagram_manage_comments,pages_read_engagement,pages_messaging

2. Exchange auth code → short-lived token (1 hour):
   POST https://graph.facebook.com/v25.0/oauth/access_token
     ?client_id={APP_ID}&client_secret={APP_SECRET}
     &grant_type=authorization_code&redirect_uri={URI}&code={CODE}

3. Exchange short-lived → long-lived token (60 days):
   GET https://graph.facebook.com/v25.0/oauth/access_token
     ?grant_type=fb_exchange_token
     &client_id={APP_ID}&client_secret={APP_SECRET}
     &fb_exchange_token={SHORT_TOKEN}

4. Refresh long-lived token (must be >24h old and not expired):
   GET https://graph.facebook.com/v25.0/oauth/access_token
     ?grant_type=fb_exchange_token
     &client_id={APP_ID}&client_secret={APP_SECRET}
     &fb_exchange_token={LONG_TOKEN}
```

For **Instagram Login path** (alternative):
```
Exchange:  GET https://graph.instagram.com/access_token
             ?grant_type=ig_exchange_token&client_secret={SECRET}&access_token={SHORT}
Refresh:   GET https://graph.instagram.com/refresh_access_token
             ?grant_type=ig_refresh_token&access_token={LONG_TOKEN}
```

### Token Lifecycle Rules
- Short-lived tokens: **1 hour** validity
- Long-lived tokens: **60 days** validity
- Refresh window: token must be **≥24 hours old** and **still valid** (not expired)
- **An expired token cannot be recovered** — user must re-authorize
- Best practice: refresh at day **45–50**, not day 59
- System-user tokens (Business Manager, Path B): can be **never-expiring**

### Get Instagram Business Account ID (Path B)
```python
# Step 1: Get Pages
GET /me/accounts?access_token={TOKEN}
# → returns pages with their IDs

# Step 2: Get IG account linked to a Page
GET /{PAGE_ID}?fields=instagram_business_account&access_token={TOKEN}
# → { "instagram_business_account": { "id": "17841405822304914" } }
```

---

## 2. Instagram DMs — Messenger API

### Overview
Instagram DMs run through the **Messenger Platform** infrastructure — same concepts (webhooks, messaging windows, send API) as Facebook Messenger. The "Instagram Messaging API" is a **subset of the Graph API**, not a separate API.

### Permissions Required for DMs

| Permission | Required | Purpose |
|---|---|---|
| `instagram_manage_messages` / `instagram_business_manage_messages` | ✅ Yes | Send/receive DMs, manage conversations |
| `pages_messaging` | ✅ Yes (Path B) | Webhook subscription for DM events |
| `pages_manage_metadata` | ✅ Yes (Path B) | Manage webhook subscriptions on the linked Page |
| `pages_read_engagement` | ✅ Yes (Path B) | Read IG Business Account info for webhook setup |

**App Review required** for Advanced Access to message accounts you don't control.

### Core Constraint: Customer-Initiated Only
- **You cannot message someone first** — the customer must DM you
- Their message opens a **24-hour standard messaging window** (resets on each customer message)
- Outside that window: only a **human agent** can reply, within **7 days** (`human_agent` tag, requires App Review + Business Verification)
- **Automated follow-up after 24h = blocked**
- **Cold outreach / bulk promotional sends = account disabled**

### Webhook Setup (Receiving DMs)

Subscribe to these webhook fields on the Instagram object:

| Webhook Field | Events |
|---|---|
| `messages` | DMs received, story replies, quick reply taps, media DMs |
| `messaging_referrals` | Story mentions (user tags your account in their story) |
| `messaging_optins` | Opt-in events for recurring notification widgets |

**Webhook endpoint requirements:**
- Must be HTTPS
- Must respond `200 OK` within **5 seconds**
- Must verify signature (`X-Hub-Signature-256` HMAC)
- Heavy processing must be **async** (queue it, acknowledge immediately)
- Meta retries and eventually backs off if you're slow

### Webhook Payload Examples

```json
// Text DM
{
  "object": "instagram",    // NOT "page" — critical distinction from Messenger
  "entry": [{
    "id": "987654321098765",  // your IG Business Account ID
    "messaging": [{
      "sender": { "id": "12345678901234" },      // user's IGSID (store as STRING)
      "recipient": { "id": "987654321098765" },   // your IG Biz Account ID
      "timestamp": 1747231892,
      "message": {
        "mid": "aWdtc2c...",
        "text": "Hey, do you ship internationally?"
      }
    }]
  }]
}

// Story Reply
{
  "message": {
    "mid": "aWdtc2c...",
    "text": "Wow, this looks amazing!",
    "reply_to": {
      "story": {
        "id": "17893310459840806",
        "url": "https://lookaside.fbsbx.com/..."  // ephemeral CDN URL
      }
    }
  }
}

// Image/Media DM
{
  "message": {
    "mid": "aWdtc2c...",
    "attachments": [{
      "type": "image",     // image | video | audio | file
      "payload": {
        "url": "https://..."  // TEMPORARY — download immediately
      }
    }]
  }
}
```

**Important:** IGSIDs (Instagram-Scoped IDs) are per-business-account, not global. Store as **string**, not integer (can exceed JS `Number.MAX_SAFE_INTEGER`).

### Sending DMs (Reply API)

**Endpoint:** `POST /v25.0/{IG_BUSINESS_ID}/messages`  
(or `POST /me/messages` with Page access token)

```python
import requests

def send_dm(igsid: str, text: str):
    """Send a text DM reply (within 24h window)."""
    resp = requests.post(
        f"https://graph.facebook.com/v25.0/{IG_BIZ_ID}/messages",
        params={"access_token": PAGE_TOKEN},
        json={
            "recipient": {"id": igsid},
            "message": {"text": text}   # max 1,000 chars
        }
    )
    return resp.json()  # {"recipient_id": "...", "message_id": "..."}
```

### What You Can Send in DMs

| Type | Details |
|---|---|
| Text | Max 1,000 characters |
| Images | Max 8 MB, up to 10 per message |
| Video, audio, PDF | Max 25 MB each |
| Generic template | Carousel of up to 10 cards (image, title, subtitle, buttons) |
| Button template | Text + up to 3 buttons |
| Product template | Product cards from your catalog |
| Quick replies | Up to 13 tappable option buttons |
| Sender actions | Typing indicator (`typing_on`), mark seen |
| Media share | Share an Instagram post in DM |
| Heart sticker | Pre-defined sticker |
| Reactions | Emoji reactions to messages |

```python
# Send image
requests.post(f"{GRAPH}/{IG_BIZ_ID}/messages",
    params={"access_token": TOKEN},
    json={
        "recipient": {"id": igsid},
        "message": {
            "attachment": {
                "type": "image",
                "payload": {"url": "https://example.com/product.jpg"}
            }
        }
    })

# Send quick replies (for Sales Agent product selection)
requests.post(f"{GRAPH}/{IG_BIZ_ID}/messages",
    params={"access_token": TOKEN},
    json={
        "recipient": {"id": igsid},
        "message": {
            "text": "Which product are you interested in?",
            "quick_replies": [
                {"content_type": "text", "title": "Shoes", "payload": "CAT_SHOES"},
                {"content_type": "text", "title": "Bags", "payload": "CAT_BAGS"},
                {"content_type": "text", "title": "Accessories", "payload": "CAT_ACC"}
            ]
        }
    })

# Private reply to a public comment (comment-to-DM flow)
requests.post(f"{GRAPH}/{COMMENT_ID}/private_replies",
    params={"access_token": TOKEN},
    json={"message": "Thanks for your interest! Here's the link: ..."})
```

### What You CANNOT Receive via Webhooks
- GIFs/stickers (do NOT trigger webhooks)
- View-once media (do NOT trigger webhooks)

### Additional DM Features
- **Ice breakers**: Up to 4 pre-set questions shown when a user opens a new conversation
- **Persistent menu**: Always-visible menu options
- **Thread Control API**: Pass/take conversations between multiple apps (handover protocol)
- **Message Requests**: Non-followers go to "Requests" folder; set "Others on Instagram → Don't require approval" for immediate webhook delivery

---

## 3. Content Publishing API

### Publishing Flow (2-step container model)

```
Step 1: Create media container → Step 2: Publish container
POST /{IG_ID}/media          → POST /{IG_ID}/media_publish
```

### Supported Content Types

| Type | `media_type` | Notes |
|---|---|---|
| Feed image | (omit or `IMAGE`) | JPEG only, max 8 MB, aspect 9:16 to 16:9 (4:5 recommended) |
| Reel | `REELS` | MP4/MOV, 3s–15 min, max 300 MB, 9:16 recommended |
| Story | `STORIES` | Image (8 MB) or video (100 MB, 3s–60s), disappears after 24h |
| Carousel | `CAROUSEL` | 2–10 images/videos, first item sets aspect ratio |

**Single videos are automatically published as Reels** (Instagram no longer supports standalone video posts).

### Publishing Examples

```python
import requests, time

GRAPH = "https://graph.facebook.com/v25.0"
IG_ID = "17841405822304914"
TOKEN = "EAAx..."

# === Post a single image ===
# Step 1: Create container
resp = requests.post(f"{GRAPH}/{IG_ID}/media", params={
    "image_url": "https://cdn.example.com/product.jpg",  # must be public URL
    "caption": "New arrival! 🛍️ #fashion #newin",
    "access_token": TOKEN
})
container_id = resp.json()["id"]

# Step 2: Publish
resp = requests.post(f"{GRAPH}/{IG_ID}/media_publish", params={
    "creation_id": container_id,
    "access_token": TOKEN
})
media_id = resp.json()["id"]

# === Post a Reel ===
resp = requests.post(f"{GRAPH}/{IG_ID}/media", params={
    "media_type": "REELS",
    "video_url": "https://cdn.example.com/reel.mp4",
    "caption": "Behind the scenes 🎬",
    "share_to_feed": "true",
    "access_token": TOKEN
})
container_id = resp.json()["id"]

# Poll for video processing (required for video/reels)
while True:
    status = requests.get(f"{GRAPH}/{container_id}",
        params={"fields": "status_code", "access_token": TOKEN}).json()
    if status["status_code"] == "FINISHED":
        break
    elif status["status_code"] == "ERROR":
        raise Exception("Video processing failed")
    time.sleep(5)

requests.post(f"{GRAPH}/{IG_ID}/media_publish", params={
    "creation_id": container_id, "access_token": TOKEN
})

# === Post a Story ===
resp = requests.post(f"{GRAPH}/{IG_ID}/media", params={
    "media_type": "STORIES",
    "image_url": "https://cdn.example.com/story.jpg",
    "access_token": TOKEN
})
# then media_publish same as above

# === Post a Carousel ===
# Create child containers first (2–10 items)
children = []
for url in ["https://cdn.example.com/1.jpg", "https://cdn.example.com/2.jpg"]:
    r = requests.post(f"{GRAPH}/{IG_ID}/media", params={
        "image_url": url,
        "is_carousel_item": "true",
        "access_token": TOKEN
    })
    children.append(r.json()["id"])

# Create carousel container
r = requests.post(f"{GRAPH}/{IG_ID}/media", params={
    "media_type": "CAROUSEL",
    "children": ",".join(children),
    "caption": "Our latest collection 🔥",
    "access_token": TOKEN
})
# then media_publish
```

### Key Publishing Parameters

| Parameter | Usage |
|---|---|
| `caption` | Max 2,200 chars, 30 hashtags, 20 @mentions |
| `location_id` | Facebook Page ID for a location tag |
| `user_tags` | Array of `{username, x, y}` for tagging users |
| `product_tags` | Array of `{product_id, x, y}` for shop products |
| `collaborators` | Up to 3 Instagram usernames (feed/reels/carousels) |
| `cover_url` | Custom cover image for Reels |
| `thumb_offset` | Milliseconds offset for video thumbnail |
| `share_to_feed` | Boolean, for Reels — appear in Feed tab too |
| `alt_text` | Alt text for images, max 1,000 chars |

### Container Lifetime
- Containers **expire after 24 hours** if not published
- Video containers go through processing states: `IN_PROGRESS` → `FINISHED` / `ERROR` / `EXPIRED`
- Always poll `GET /{container_id}?fields=status_code` for video/reels before publishing

### Check Publishing Quota
```
GET /{IG_ID}/content_publishing_limit?fields=quota_usage,config
```

---

## 4. Rate Limits & Quotas

### Publishing Limits

| Metric | Limit | Window |
|---|---|---|
| API-published posts | **100 posts** (Meta docs say 100; some sources cite 50 for carousel-publishing accounts) | Rolling 24 hours |
| Carousel items | Up to 10 per carousel | Per post |
| Container lifetime | Must publish within 24h | Per container |

Publishing limit error: subcode `2207042` — **do not retry** (24h rolling window).

### Graph API Call Limits

| Mechanism | Formula / Limit | Scope |
|---|---|---|
| Platform rate limit | `4800 × account_impressions_last_24h` calls | Per app + user pair, rolling 24h |
| General API calls | ~200 calls/hour/user-token | Per user per app |
| Burst allowance | 10–15 requests in a 10-second window | Then drops to steady rate |

**Headers to monitor:**
- `X-Business-Use-Case-Usage` — JSON with per-account usage percentages
- `X-App-Usage` — app-level throttle percentage

**Error codes:**
| Code | Meaning | Retry? |
|---|---|---|
| `4` | App-level throttle | Yes (backoff) |
| `17` | User-level throttle | Yes (backoff) |
| `9` | Rate limit | Yes (backoff) |
| `613` | Custom / per-resource cap | Yes |
| `2207042` (subcode) | 24h publishing cap | **No — terminal** |

### Messaging Limits

| Metric | Limit | Source |
|---|---|---|
| Text/link messages | 100/second | Meta (IG Login path) |
| Audio/video messages | 10/second | Meta |
| Private replies to comments | 750/hour | Meta |
| Practical DM pacing | ~200/hour/account | Tool-side (anti-spam safe zone) |
| Messaging window | 24 hours from user's last message | Meta policy |
| Human agent exception | 7 days from user's last message | Requires App Review |

**October 2025 change:** Meta reduced the per-account DM ceiling from 5,000/hour to ~200/hour. For an eCommerce support bot, 200/hour (~4,800/day) is workable.

### App Review Requirements
- **Standard Access**: Test with accounts that have a role on your app (admin/dev/tester)
- **Advanced Access**: Required to message/read data from accounts you don't control (real customers)
- Must pass **Meta App Review** with use-case justification
- **Business Verification** required for `human_agent` tag

---

## 5. Existing MCP Servers & SDKs

### 🏆 Recommended: `instagram-mcp-ai` (IvanBBaev)

The most production-ready, official-API-only MCP server for Instagram.

| Property | Value |
|---|---|
| **npm package** | `instagram-mcp-ai` |
| **GitHub** | [IvanBBaev/instagram-mcp](https://github.com/IvanBBaev/instagram-mcp) |
| **Language** | TypeScript (ESM), Node.js ≥ 22 |
| **Tools** | **28 tools** across 6 packages |
| **API version** | Pinned `v25.0` |
| **License** | MIT |
| **Transports** | stdio (default), Streamable HTTP |

**Packages:**
- `account` — profile, linked accounts, token status
- `media` — list/read media, toggle comments
- `publishing` — feed images, carousels, Reels, Stories (container→publish flow)
- `comments` — list, reply, hide/unhide, delete
- `insights` — account/media metrics, audience demographics, online followers
- `discovery` — hashtag search, business discovery (Path B only)

**Key safety features:**
- Preview-by-default writes (nothing posted without `apply: true`)
- Destructive ops double-gated (`IG_ALLOW_DESTRUCTIVE=true`)
- Audit journal (JSONL)
- Untrusted-text fencing (prevents prompt injection from captions/bios)
- Official endpoints only — no scraping

**⚠️ Limitation:** DM/messaging tools are **not yet implemented** — messaging is an explicit v1 non-goal (webhooks require a public endpoint; the server runs locally). The author's docs mark messaging as a future item with strict safety requirements.

**Setup with Claude Code:**
```bash
claude mcp add instagram -- npx -y instagram-mcp-ai \
  --env IG_ACCESS_TOKEN=EAAx... \
  --env IG_ACCOUNT_ID=17841405822304914
```

### Other MCP Servers

| Server | Package | Tools | Notes |
|---|---|---|---|
| **`instagram-mcp`** (AleemHaider) | `pip install instagram-mcp` | 24 | Python/FastMCP, official API. Published May 2026. Includes DM tools. **Yanked from PyPI.** |
| **`instamcp`** (mpython77) | `pip install instamcp` | 79 | **Unofficial private API** (curl_cffi + Chrome TLS impersonation). Scraping, DMs, scheduling. Risk of account ban. |
| **`instagram-personal-mcp`** | `pip install instagram-personal-mcp` | 24 | **Unofficial** (instagrapi). Personal accounts. High ban risk. |
| **`@socialapis/mcp`** | `npm install -g @socialapis/mcp` | 16 IG tools | Unified social API (paid service, $key required). Facebook + Instagram + TikTok. |

### Python SDKs (Direct API Access)

| Library | Notes |
|---|---|
| **`requests` / `httpx`** | Direct Graph API calls — simplest approach, full control |
| **`python-facebook-api`** | Thin wrapper around Graph API |
| **`instagrapi`** | **Unofficial** private API — powerful but violates ToS, ban risk |

### Recommendation for ALX-2
Since we need **DMs + Publishing + Insights** and the best MCP server (`instagram-mcp-ai`) doesn't cover DMs yet:

1. **Publishing + Insights + Comments → Use `instagram-mcp-ai`** (add as MCP server for Social Media Marketing Agent)
2. **DMs → Build a custom webhook service** using direct Graph API calls (Python/httpx)
3. Alternatively: build a thin MCP server wrapping the DM endpoints, modeled after `instagram-mcp-ai`'s safety patterns

---

## 6. Architecture Recommendations for ALX-2

### Agent-to-API Mapping

| Agent | Instagram Capabilities Needed | Implementation |
|---|---|---|
| **Store Manager** | Monitor incoming DMs, detect customer intent, route/escalate | Webhook listener → message queue → agent processing |
| **Sales Agent** | Reply to DMs with product info, images, quick replies, templates | Send API calls within 24h window |
| **Social Media Marketing** | Post photos, reels, stories, carousels; read insights | `instagram-mcp-ai` MCP server (publishing + insights packages) |

### DM Architecture (Store Manager + Sales Agent)

```
                                      ┌─────────────────┐
Instagram User DMs ──webhook──►      │  Webhook Server  │
                                      │  (public HTTPS)  │
                                      └────────┬────────┘
                                               │ verify signature
                                               │ parse, enqueue
                                               ▼
                                      ┌─────────────────┐
                                      │  Message Queue   │
                                      │  (Redis/SQS)     │
                                      └────────┬────────┘
                                               │
                              ┌────────────────┼────────────────┐
                              ▼                                  ▼
                    ┌──────────────────┐              ┌──────────────────┐
                    │  Store Manager   │              │   Sales Agent    │
                    │  (classify,      │──escalate──► │  (product lookup,│
                    │   route, triage) │              │   reply with     │
                    │                  │              │   images/cards)  │
                    └──────────────────┘              └──────────────────┘
                                                              │
                                                              ▼
                                                     POST /{IG_ID}/messages
                                                     (within 24h window)
```

### Key Implementation Notes

1. **Webhook server must be public HTTPS** — cannot run behind localhost. Use ngrok for dev, deploy to cloud for prod.
2. **Respond to webhooks within 5 seconds** — queue everything, process async.
3. **Never poll for messages** — use webhooks exclusively. Polling wastes rate limit budget and triggers anti-abuse detection.
4. **Enforce your own send rate** below 200 DMs/hour — don't discover the limit in production.
5. **Download media attachment URLs immediately** — they are temporary CDN URLs.
6. **Store IGSIDs as strings**, not integers.
7. **Token refresh cron job** — refresh at day 45–50, alert if refresh fails. An expired token requires user re-auth.
8. **Handle message types**: text DMs, story replies, story mentions, post/reel shares, comment-to-DM flows. A message with an attachment and 2 words of text is not a text question — detect the shape before routing.
9. **24h window tracking** — track last customer message timestamp per conversation. After 24h, only human agent path is available (if approved).
10. **Non-followers**: Set Instagram → Settings → Privacy → Messages → "Others on Instagram" to "Don't require approval" for immediate webhook delivery.

### Required Permissions (Combined)

```
# Path B (Facebook Login) — all agents
instagram_basic
instagram_content_publish
instagram_manage_messages
instagram_manage_comments
instagram_manage_insights
pages_messaging
pages_read_engagement
pages_manage_metadata
```

### Environment Variables Template

```bash
# Meta App
FB_APP_ID=your_app_id
FB_APP_SECRET=your_app_secret

# Instagram Account  
IG_BUSINESS_ACCOUNT_ID=17841405822304914
IG_PAGE_ACCESS_TOKEN=EAAx...  # long-lived, refresh every 45-50 days

# Webhook
WEBHOOK_VERIFY_TOKEN=your_random_verify_token
FB_APP_SECRET=your_app_secret  # for HMAC signature verification

# MCP Server (for publishing agent)
IG_AUTH_MODE=fb-login
IG_ACCESS_TOKEN=EAAx...
IG_APP_ID=your_app_id
IG_APP_SECRET=your_app_secret
IG_ACCOUNT_ID=17841405822304914
IG_TOOL_PACKAGES=core
IG_WRITE_MODE=apply
```

---

## References

- [Meta Messenger Platform — Instagram](https://developers.facebook.com/docs/messenger-platform/instagram)
- [Instagram Content Publishing API](https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/content-publishing/)
- [Business Login for Instagram](https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/business-login)
- [Graph API Rate Limiting](https://developers.facebook.com/docs/graph-api/overview/rate-limiting/)
- [instagram-mcp-ai (IvanBBaev)](https://github.com/IvanBBaev/instagram-mcp)
- [Send Messages — IG Login](https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/messaging-api/)
- [IG User Media endpoint](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/media/)
