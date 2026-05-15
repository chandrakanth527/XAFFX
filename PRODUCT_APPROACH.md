# Affiliate Product Discovery Platform — Approach Document

## Concept

A TikTok/YouTube Shorts-style vertical video feed that matches existing YouTube creator content to Meesho affiliate products. Users scroll product videos, swipe left to see product details, and buy through affiliate links.

**We don't create content.** We match YouTube Shorts (product reviews, unboxings, demos) to Meesho products at scale, verify quality through comments/transcripts, and human-review matches before going live.

---

## Target Categories

Starting with **home products** and **kids toys** because:
- Distinct product names, shapes, brand names → easier to match
- Creators often say the product name clearly in video/description
- Brand names appear in titles: "Hot Wheels", "Prestige", "Milton"
- Kids products get emotional comments → easy sentiment signal
- Home products have model numbers
- Lower SKU ambiguity vs fashion ("blue kurta" matches 10,000 listings)

---

## Data Pipeline

```
┌──────────────────┐     ┌──────────────────┐
│  YouTube API      │     │  Meesho Scraper   │
│  Search Shorts    │     │  Toys + Home      │
│  (batch/nightly)  │     │  (nightly crawl)  │
└────────┬─────────┘     └────────┬─────────┘
         │                        │
         ▼                        ▼
┌─────────────────────────────────────────────┐
│  Auto-Match Engine                           │
│  Video metadata → Product catalog            │
│  (keyword/title/description matching)        │
└────────────────────┬────────────────────────┘
                     ▼
┌─────────────────────────────────────────────┐
│  Quality Gate                                │
│  - Top 3 YouTube comments: sentiment check   │
│  - Video transcript: does creator recommend? │
│  - Score: 0-100                              │
│  - Reject if negative sentiment detected     │
└────────────────────┬────────────────────────┘
                     ▼
┌─────────────────────────────────────────────┐
│  Human Review Dashboard                      │
│  Reviewer sees: [Video] [Matched Product]    │
│  Keyboard shortcuts: A = Approve, R = Reject │
│  Target: 500+ reviews per hour per reviewer  │
└────────────────────┬────────────────────────┘
                     ▼
┌─────────────────────────────────────────────┐
│  Production Database                         │
│  Approved video-product pairs → User Feed    │
└─────────────────────────────────────────────┘
```

---

## Data Sources

### YouTube Shorts (YouTube Data API v3)

- **Rate limit**: 10,000 quota units/day (free tier)
- Search = 100 units/call → **100 searches/day**
- 100 searches × 50 results = **5,000 new videos/day**
- ~150,000 new videos/month into pipeline
- **Use ONLY at pipeline time (offline batch), never at runtime**
- Get: video ID, title, description, tags, view count, channel, transcript, top comments
- Filter: duration < 60s, language, recency, category

### Meesho Product Catalog

- **No public API exists**
- Options for scraping:
  - Apify Meesho Scraper (paid, managed)
  - ScrapingBee (paid, real-time)
  - DIY with Playwright/Puppeteer (free, maintenance heavy)
- Extract: product name, price, MRP, discount %, images, rating, review count, product URL, category
- Run nightly to catch price changes and new listings
- Store in own database

### Meesho Affiliate Links

- Join Meesho Creator Club (https://affiliate.meesho.com/)
- Commission: 4-25% per sale depending on category
- Generate tracking links per product
- Payout: 7-14 days after return window closes

---

## Matching Engine

### Phase 1 — Keyword matching (MVP)
- Parse video title + description for product names, brands, model numbers
- Match against Meesho product catalog using keyword overlap
- Works well for toys/home products with distinct names
- Example: Video "Hot Wheels Track Builder Set Review" → Meesho product "Hot Wheels Track Builder"

### Phase 2 — Human verification
- Auto-matched pairs queued for human review
- Reviewer confirms if video actually shows the matched product
- Approved matches go to production
- Rejected matches feed back to improve matching

### Phase 3 — AI visual matching (future)
- Extract video thumbnails/frames
- Compare against Meesho product images
- Semantic similarity using embeddings
- Fully automated pipeline with human spot-checks

---

## Quality Gate

### Comment Sentiment Analysis
```
Top 3 comments on video:
  ✅ "Bought this, my kid plays with it daily"     → positive
  ✅ "Great quality for the price"                  → positive
  ❌ "Broke in 2 days"                              → REJECT

Keywords that boost score:
  "worth it", "recommend", "my kid loves", "good quality", "value for money"

Keywords that reject:
  "waste", "broke", "cheap quality", "don't buy", "scam", "fake"
```

### Transcript Analysis
- Use YouTube transcript/captions API
- Check if creator recommends the product
- Flag sponsored/affiliate content from creator side
- Extract specific product attributes mentioned (color, size, age range)

### Quality Score Formula
```
quality_score = (comment_sentiment × 0.4) + (transcript_sentiment × 0.3) +
                (video_engagement_ratio × 0.2) + (creator_credibility × 0.1)

Threshold: quality_score >= 60 → queue for human review
           quality_score < 60  → auto-reject
```

---

## User Feed Algorithm

**No external API calls at runtime. Everything served from our database.**

### Feed Composition
```
70% — Similar to what user lingered on (tag-based, from our DB)
20% — Popular across all users (high linger time + quality score)
10% — Random/explore (new categories to test user interest)
```

### Linger-Based Personalization (Tag Scoring)

Track user behavior client-side (localStorage, no auth needed):

| Signal | Meaning | Score Impact |
|--------|---------|-------------|
| Watch time > 70% | Interested | +2 to tags |
| Paused video | Looking closely | +1 to tags |
| Swiped left (product card) | Purchase intent | +3 to tags |
| Scrolled back up | Wanted to rewatch | +2 to tags |
| Skipped in < 2s | Not interested | -1 to tags |
| Tapped Buy | High intent | +5 to tags |

```
User profile (localStorage):
{
  "toy-car": +3,
  "building-blocks": +2,
  "doll": -1,
  "kitchen-set": 0
}
```

### Feed Query Logic
```sql
SELECT * FROM matched_products
WHERE category IN (user_preferred_tags)
AND video_id NOT IN (already_seen)
AND quality_score >= 60
ORDER BY
  tag_match_score * 0.7 +
  global_popularity * 0.2 +
  RANDOM() * 0.1
LIMIT 20
```

### Global Popularity Score
```
feed_order = (avg_linger_time × 0.6) + (quality_score × 0.4)
```
Popularity-based ranking beats personalization for small catalogs and new users.

### Same Product, Multiple Videos
- Different videos of the SAME product = valuable (like reading multiple reviews)
- But only show multiple if user swiped left (research mode)
- If just browsing, don't repeat the same product

---

## UI/UX Design

### Main Feed (Vertical Scroll)
- YouTube Shorts-style snap scroll
- Videos embedded via YouTube IFrame API
- Floating price tag overlay: ₹299 ~~₹799~~ 63% off
- Right-side action buttons: Heart, Share, Cart
- Category chips at top for filtering

### Swipe-Left Product Card (60% screen overlay)
```
┌──────────────────────────┐
│                  ┌───────┤
│   Video still    │ Price │
│   visible        │ ★★★★☆ │
│   (blurred)      │ Image │
│                  │ Image │
│                  │       │
│                  │ 324   │
│                  │bought │
│                  │ today │
│                  │       │
│                  │[Buy ▶]│
│                  │       │
│                  │ More  │
│                  │videos │
└──────────────────┴───────┘
```

**Product card contains:**
- Product price (big, bold) with MRP strikethrough
- 2-3 product images (swipeable)
- Rating + review count
- Social proof: "X people bought today"
- "Buy on Meesho" CTA → affiliate link
- "More videos" → other videos of same product
- COD badge (important for Meesho audience)
- WhatsApp share (critical for tier 2/3 India)

### Gestures
```
Vertical scroll  = Browse products (snap scroll)
Swipe left       = Reveal product card
Swipe right      = Save to wishlist
Tap              = Pause/play video
Long press       = Share (WhatsApp priority)
```

---

## Technical Architecture

### Data Pipeline (Offline/Batch)
- **Language**: Python
- **YouTube data**: YouTube Data API v3
- **Meesho scraping**: Playwright (headless browser, JS-rendered pages)
- **Matching**: Keyword matching → future: OpenAI/Claude embeddings
- **Transcript**: YouTube Transcript API
- **Comment sentiment**: Simple keyword scoring → future: LLM-based

### Backend
- **Framework**: Node.js or Python FastAPI
- **Database**: PostgreSQL (product catalog, matched pairs, user events)
- **Search**: PostgreSQL full-text search → future: Elasticsearch
- **Cache**: Redis (feed caching, popular products)
- **Embeddings** (future): pgvector extension for PostgreSQL

### Frontend
- **MVP**: Vanilla HTML/CSS/JS (current prototype)
- **Production**: React or Next.js
- **Video player**: YouTube IFrame API
- **Personalization**: Client-side localStorage (no auth for MVP)

### Infrastructure
- Hosting: Vercel (frontend) + Railway/Render (backend)
- Database: Supabase (PostgreSQL + auth when needed)
- Scraping: Dedicated server or cloud functions on schedule

---

## Data Model

### products (Meesho catalog)
```json
{
  "id": "uuid",
  "name": "Hot Wheels Track Builder Set",
  "price": 299,
  "mrp": 799,
  "discount_pct": 63,
  "rating": 4.2,
  "review_count": 1243,
  "category": "toys",
  "sub_category": "cars-vehicles",
  "age_group": "3-5",
  "brand": "Hot Wheels",
  "images": ["url1", "url2"],
  "meesho_url": "https://meesho.com/...",
  "affiliate_url": "https://...",
  "scraped_at": "2026-05-14"
}
```

### videos (YouTube Shorts)
```json
{
  "id": "uuid",
  "youtube_id": "abc123",
  "title": "Best toy car for kids under ₹300",
  "description": "...",
  "tags": ["toy", "car", "kids", "review"],
  "channel": "ToyReviewIndia",
  "view_count": 45000,
  "transcript_summary": "Creator recommends, good quality",
  "top_comments": ["Great toy!", "My son loves it", "Fast delivery"],
  "comment_sentiment": 0.85,
  "quality_score": 82,
  "fetched_at": "2026-05-14"
}
```

### matched_pairs (approved video-product links)
```json
{
  "id": "uuid",
  "video_id": "uuid",
  "product_id": "uuid",
  "match_confidence": 0.78,
  "match_method": "keyword",
  "status": "approved",
  "reviewed_by": "reviewer_1",
  "reviewed_at": "2026-05-14",
  "global_linger_avg": 12.5,
  "global_click_count": 234
}
```

### user_events (analytics, no auth)
```json
{
  "session_id": "anonymous_uuid",
  "matched_pair_id": "uuid",
  "event": "linger|skip|swipe_left|buy_click|save",
  "duration_seconds": 8.5,
  "timestamp": "2026-05-14T10:30:00Z"
}
```

---

## Build Order

1. **Meesho scraper** — build product catalog (toys + home)
2. **YouTube pipeline** — search + get transcripts + top comments
3. **Auto-matcher** — keyword matching between video metadata and product names
4. **Quality scorer** — sentiment analysis on comments + transcript
5. **Review dashboard** — human approval UI (keyboard shortcuts for speed)
6. **Production database** — approved matches with all metadata
7. **Feed app** — current prototype evolved with swipe-left product card
8. **Linger tracking** — client-side behavior tracking + tag scoring
9. **Feed personalization** — tag-based feed ordering from own DB
10. **Analytics** — track which products convert, which videos perform

---

## Key Decisions

- **YouTube API is a data collection tool, NOT a runtime recommendation engine** (rate limits make runtime usage impossible at scale)
- **All recommendations served from our own database** — zero external API calls at runtime
- **Human-in-the-loop verification** before any match goes live
- **Quality over quantity** — reject negative-sentiment products, only show what creators genuinely recommend
- **Start with toys + home → expand categories** after matching pipeline is proven
- **No user auth for MVP** — localStorage-based personalization, add accounts later
- **Meesho-first** but data model supports adding Amazon/Flipkart affiliates later
