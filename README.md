# Prompt: Design & Build a Tech Blog Aggregator API

You are acting as a principal software engineer. Design and implement a backend API that aggregates engineering/tech blog posts from multiple companies and communities into a single feed, so users can browse everything in one place and click through to read the full article on the original site.

## 1. Product Requirements

- Aggregate posts from engineering blogs of companies like Stripe, Google (AI/Cloud/Developers blogs), Netflix, Uber, Airbnb, Meta, Snapchat, Shopify, Spotify, Cloudflare, GitHub, plus community sources like Hacker News (top stories) and dev.to.
- Support **latest** (chronological) and **top/trending** (ranked by engagement signal) feeds.
- Each article record only needs metadata + a link out — we are NOT scraping/rehosting full article bodies (avoids copyright/legal issues and keeps this lightweight).
- Users can filter/search by source, tag/topic, and date range.
- Clicking an article takes the user to the original blog post (external link), not a mirrored copy.
- New articles should show up automatically without manual intervention (scheduled ingestion).

## 2. Non-Functional Requirements

- API should respond in well under 300ms for feed reads (cached/pre-aggregated, not live-fetched per request).
- Should be resilient to individual source failures (one broken RSS feed shouldn't break the whole feed).
- Should avoid duplicate articles (same post can appear via RSS + HN + dev.to).
- Should be easy to add a new source without code changes to the core pipeline (config-driven source list).
- Should respect each source's terms/robots.txt and only use official RSS/Atom feeds or public APIs — no scraping of blogs that disallow it.

## 3. Architecture to Design

Propose and implement:

1. **Ingestion layer** — a scheduled worker (cron job / queue consumer) that:
   - Reads a config file/table of sources (name, RSS/Atom URL or API endpoint, logo, category).
   - Fetches each source on an interval (e.g., every 15–30 min), parses RSS/Atom (or hits public APIs like HN's Firebase API, dev.to's API).
   - Normalizes each entry into a common `Article` schema.
   - Deduplicates (by canonical URL + fuzzy title match).
   - Upserts into the database.
2. **Storage layer** — a database schema for `sources` and `articles` (see schema below), with indexes for sorting by date and by a computed "score".
3. **Ranking layer** — a simple scoring function for "top" articles: combine recency decay with any engagement signal available (HN points, dev.to reactions; for sources with no engagement data, weight by recency + source authority tier).
4. **API layer** — REST (or GraphQL if you prefer — justify the choice) exposing read endpoints described below. This layer only ever reads from the database/cache — it never calls external sources synchronously.
5. **Caching layer** — cache hot feed queries (e.g., Redis or in-memory with TTL) since ingestion already runs on its own schedule.

## 4. Data Model (starting point — refine as needed)

```
Source {
  id, name, slug, homepage_url, feed_url, logo_url,
  category (e.g. "fintech", "social", "cloud"), is_active
}

Article {
  id, source_id, title, canonical_url, author,
  summary (short excerpt, not full body), published_at,
  fetched_at, tags[], engagement_score (nullable),
  computed_rank_score, dedupe_hash
}
```

## 5. API Endpoints to Design

- `GET /articles?feed=latest|top&page=&limit=&source=&tag=&from=&to=`
- `GET /articles/{id}`
- `GET /sources` — list all supported blogs (for a filter UI / logos)
- `GET /tags` — list available topics/tags for filtering
- (Optional) `GET /search?q=`

For each, specify request/response JSON shape, pagination style (cursor vs offset — pick one and justify), error format, and rate limiting approach.

## 6. Deliverables

1. A short architecture write-up (diagram in words is fine) explaining the ingestion → storage → ranking → API flow and why.
2. The source config format (JSON/YAML) with 8–10 real example sources and their actual public RSS/Atom feed URLs (verify these are real, official feeds — don't invent URLs).
3. Database schema (SQL DDL).
4. Working implementation of:
   - The ingestion worker/script.
   - The API server with the endpoints above.
5. Sample `curl` requests and responses for each endpoint.
6. Notes on what you'd add for production (auth/API keys for consumers, monitoring for failed feed fetches, retry/backoff, horizontal scaling of the ingestion worker).

## 7. Tech Stack

Use: **[FILL IN — e.g., Node.js + Express + PostgreSQL + Redis, or Python + FastAPI + Postgres]**. If I haven't specified, pick a sensible modern stack and briefly justify the choice.

## 8. Constraints

- Keep the first version deployable as a single small service + one scheduled job — don't over-engineer with microservices.
- Prioritize correctness of dedup and reliability of ingestion over fancy ranking algorithms initially.
- Write clean, commented, production-quality code, not a toy script.

Start by proposing the architecture and schema, then implement it step by step.
