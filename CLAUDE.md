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

- **`src/services/apiService.ts`** — Axios client for `api.deflock.org`; uses `localhost:3000` when on localhost. Contains `BoundingBox` class used for map viewport queries.
- **`src/services/cmsService.ts`** — Axios client for `cms.deflock.me`; fetches LPR vendor data, chapters, and non-LPR surveillance devices.
- **`src/stores/vendorStore.ts`** — Pinia store caching LPR vendor and device data from CMS.
- **`src/stores/global.ts`** — Pinia store for geolocation state.
- **`src/stores/tiles.ts`** — Pinia store for map tile configuration.
- **`src/services/deflockAppUrls.ts`** — Generates `deflockapp://` deep links for the DeFlock mobile app (profile import via base64-encoded JSON).
- **`src/router/index.ts`** — Vue Router with all routes and `useHead` title updates on navigation.
- **`src/types.ts`** — Shared TypeScript interfaces: `ALPR`, `LprVendor`, `OtherSurveillanceDevice`.

### API (Fastify)
- Runs on port 3000
- CORS allows `localhost:5173`, `deflock.org`, and `*.deflock.pages.dev` (Cloudflare Pages previews)
- Services in `api/services/`: `NominatimClient` (geocoding), `GithubClient` (sponsors)

### CMS (Directus)
- SQLite-backed Directus instance
- Extensions in `cms/extensions/` for Cloudflare cache purging
- Requires `DIRECTUS_SECRET`, `CF_API_KEY`, `CF_ZONE_ID` env vars

## Environment Variables

### API (`api/.env`)
- `GITHUB_TOKEN` — GitHub Sponsors API access

### CMS (`cms/.env` or Docker env)
- `DIRECTUS_SECRET`
- `CF_API_KEY`
- `CF_ZONE_ID`

## Key External Services

- **Overpass API** — OSM ALPR data queries
- **Nominatim** — Geocoding (proxied through the API)
- **Cloudflare** — DNS, proxy, CDN cache (cdn.deflock.me)
- **AWS S3** — Static JSON storage for clusters/counts
- **AWS Lambda** — Serverless compute for data processing
- **GitHub Actions** — CI/CD; API deploys to production on push to `master`
