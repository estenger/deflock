# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is DeFlock

A crowdsourced tool for locating and reporting Automatic License Plate Readers (ALPRs) on a map. Data comes from OpenStreetMap (OSM) via the Overpass API.

## Repository Structure

```
deflock/
├── webapp/        # Vue3 frontend (main user-facing site)
├── api/           # Fastify API (geocoding, GitHub sponsors proxy)
├── cms/           # Directus CMS (Docker-based, SQLite)
├── serverless/    # AWS Lambda functions
│   ├── alpr_cache/   # Generates map clusters daily (Docker Lambda)
│   ├── alpr_counts/  # Counts total ALPRs hourly (zip Lambda)
│   └── blog_scraper/ # Python blog scraper
├── terraform/     # Infrastructure as code
└── scripts/       # Utility scripts (Directus backup)
```

## Commands

### Webapp (Vue3 + Vite)
```sh
cd webapp
npm install
npm run dev              # Dev server on port 5173
npm run build            # Production build
npm run type-check       # TypeScript type checking
npm run build-with-typecheck  # Type-check + build
```

### API (Bun + Fastify)
```sh
cd api
bun install
bun server.ts            # Dev server on port 3000
```

### CMS (Directus via Docker)
```sh
cd cms
docker compose up        # Runs Directus on port 8055
```

### Serverless (alpr_counts — Terraform deploy)
```sh
cd terraform
terraform apply -target=module.alpr_counts
```

### Serverless (alpr_cache — Docker Lambda)
```sh
cd serverless/alpr_cache
./deploy.sh
```

## Architecture

### Data Flow

1. **ALPR data** comes from OSM's Overpass API — not stored in any project-owned database.
2. **alpr_counts** Lambda queries Overpass hourly, writes `alpr-counts.json` to S3.
3. **alpr_cache** Lambda clusters ALPR locations daily, writes to S3.
4. **Webapp** reads static JSON from `https://cdn.deflock.me` (S3) and queries `https://api.deflock.org` (the Fastify API).
5. **CMS** at `https://cms.deflock.me` (Directus) serves vendor info, chapters, and blog content. Extensions in `cms/extensions/` include a Cloudflare cache purge hook.

### Webapp Architecture

**Services:**
- **`src/services/apiService.ts`** — Axios client for `api.deflock.org`; uses `localhost:3000` when on localhost. Contains `BoundingBox` class used for map viewport queries.
- **`src/services/cmsService.ts`** — Axios client for `cms.deflock.me`; fetches LPR vendor data, chapters, and non-LPR surveillance devices. Always hits production (no localhost detection).
- **`src/services/blogService.ts`** — Axios client for `cms.deflock.me`; fetches paginated blog posts and individual post content.
- **`src/services/deflockAppUrls.ts`** — Generates `deflockapp://` deep links for the DeFlock mobile app (profile import via base64-encoded JSON).

**Stores (Pinia):**
- **`src/stores/vendorStore.ts`** — Caches LPR vendor and device data from CMS (lazy-loaded).
- **`src/stores/global.ts`** — Geolocation state (browser Geolocation API).
- **`src/stores/tiles.ts`** — Manages ALPR tile data fetched from CDN/S3 (`cdn.deflock.me/regions/`). Tracks loaded tiles, prevents duplicate fetches.

**Other key files:**
- **`src/router/index.ts`** — Vue Router with all routes and `useHead` title updates on navigation.
- **`src/types.ts`** — Shared TypeScript interfaces: `ALPR`, `LprVendor`, `OtherSurveillanceDevice`.

### API (Fastify)
- Runs on port 3000
- CORS allows `localhost:5173`, `deflock.org`, and `*.deflock.pages.dev` (Cloudflare Pages previews)
- Services in `api/services/`: `NominatimClient` (geocoding, 24h disk cache), `GithubClient` (sponsors)
- Endpoints: `GET /geocode`, `GET /sponsors/github`, `HEAD /healthcheck`

### CMS (Directus)
- SQLite-backed Directus instance
- Extensions in `cms/extensions/` for Cloudflare cache purging
- Collections: `chapters`, `lprVendors`, `nonLprVendors`, `blog`, `flockWins`
- Requires `DIRECTUS_SECRET`, `CF_API_KEY`, `CF_ZONE_ID` env vars

### Serverless Functions
- **alpr_cache** — Daily Lambda; queries Overpass for all ALPRs, segments into 20-degree tiles, uploads to S3 (`regions/*.json` + `regions/index.json`)
- **alpr_counts** — Hourly Lambda; counts US ALPRs via Overpass, fetches wins from CMS, writes `alpr-counts.json` to S3
- **blog_scraper** — Syncs RSS feed from haveibeenflocked.com into CMS blog collection

## Environment Variables

### API (`api/.env`)
- `GITHUB_TOKEN` — GitHub Sponsors API access

### CMS (`cms/.env` or Docker env)
- `DIRECTUS_SECRET`
- `CF_API_KEY`
- `CF_ZONE_ID`

## Local Development Notes

- The webapp auto-detects `localhost` and routes API calls to `localhost:3000` instead of `api.deflock.org`
- The CMS service has **no localhost detection** — the webapp always hits `cms.deflock.me` (production). Running CMS locally is not needed for frontend development.
- To validate the local API is being used: open browser DevTools Network tab and use the map search bar — requests should go to `localhost:3000/geocode`

## Key External Services

- **Overpass API** — OSM ALPR data queries
- **Nominatim** — Geocoding (proxied through the API)
- **Cloudflare** — DNS, proxy, CDN cache (cdn.deflock.me)
- **AWS S3** — Static JSON storage for clusters/counts
- **AWS Lambda** — Serverless compute for data processing
- **GitHub Actions** — CI/CD; API deploys to production on push to `master`

## Additional Documentation

See `docs/architecture.md` for a detailed feature-to-service mapping, data flow diagrams, and user flow documentation.
