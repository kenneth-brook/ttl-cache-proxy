# ttl-cache-proxy

A production-minded Node.js/TypeScript proxy service that fronts the **NASA Image and Video Library API** and adds:

- **Cache-aside TTL caching** (default 10 minutes for search, 24 hours for assets)
- **Stampede protection** (single-flight: one upstream fetch per key at a time)
- Optional **stale-if-error** fallback (serve expired cache if upstream fails)
- Two cache modes: **in-memory** or **Redis** (shared cache for multi-instance scaling)
- Basic ops endpoints: **/healthz** and **/metrics**
- Docker + Compose friendly

Upstream API: https://images-api.nasa.gov

---

## Why this exists

This is a reusable pattern for reducing cost/latency when proxying a real upstream API:

- Prevent repeated upstream calls for identical requests (TTL caching)
- Prevent “thundering herd” refresh storms (single-flight)
- Keep service responsive during upstream turbulence (stale-if-error)
- Make behavior visible and demo-friendly (headers + metrics)

If you can ship this cleanly, you can ship API gateways, ingestion workers, and “real” production services.

---

## Features

### Proxy routes

- `GET /nasa/search?q=apollo&page=1&media_type=image`  
  Proxies: `GET https://images-api.nasa.gov/search?...`

- `GET /nasa/asset/:nasa_id`  
  Proxies: `GET https://images-api.nasa.gov/asset/{nasa_id}`

### Cache keys

- Search:
  `search:${q}:${page}:${media_type}:${year_start}:${year_end}:${page_size}`

- Asset:
  `asset:${nasa_id}`

### TTL strategy (intentional contrast)

- Search TTL: **10 minutes**
- Asset TTL: **24 hours**

Rationale: search results change more often than a specific asset’s metadata.

### Stampede protection (single-flight)

When many concurrent requests hit the same cache key while it’s missing/expired:
- only **one** upstream call is made
- other callers wait for that result and reuse it

### Optional stale-if-error fallback

If the upstream fails and a stale value exists:
- respond with the stale cached value
- set `X-Cache: STALE`
- log the upstream error
- keep the service responding instead of faceplanting

---

## Response headers (cache visibility)

Every successful proxy response includes:

- `X-Cache: HIT | MISS | STALE`
- `X-Cache-Age: <seconds>`
- `X-Upstream: images-api.nasa.gov`

---

## Ops endpoints

- `GET /healthz`  
  Basic health check. Returns `{ "ok": true }`

- `GET /metrics`  
  Simple counters (JSON), e.g.:
  - `cache_hit`
  - `cache_miss`
  - `cache_stale_served`
  - `upstream_calls`
  - `upstream_errors`

---

## Quickstart

### Run with Docker Compose (recommended)

```bash
docker compose up --build
