# DeFlock

Crowdsourced tool for locating and reporting ALPRs. [View Live Site](https://deflock.org).

![DeFlock Screenshot](./webapp/public/map-interface-nationwide.webp)

## Purpose

I created this project after noticing the mass deployment of ALPRs in cities, towns, and even rural areas in the recent years. It's a massive threat to privacy, and this projects helps shed a light on this issue as ALPRs continue to be deployed to thousands of cities across the US and possibly beyond.

## What it Does

### View ALPRs on a Map
Uses OpenStreetMap data to populate a map with crowdsourced locations of ALPRs, along with their type and direction they face.

### Report ALPRs
Provides OSM tags for easy reporting of ALPRs based on brand on OSM's editing site, or through the DeFlock mobile app.

### Learn About ALPRs
See photos of common ALPRs and other surveillance devices, and learn about their capabilities.

### Blog & News
Read news and articles about LPR surveillance, including syndicated content from haveibeenflocked.com.

### Find Local Groups
Browse a directory of local community groups working on surveillance issues.

## Tech Stack

### Frontend
* Vue3 + TypeScript
* Vuetify (UI component library)
* Leaflet (mapping library)
* Pinia (state management)
* Vite (build tool)

### API
* Bun + Fastify
* Proxies geocoding (Nominatim) and GitHub sponsors

### CMS
* Directus (Docker, SQLite-backed)
* Manages vendor info, chapters, blog posts, and win counts

### Cloud
* AWS Lambda (daily [ALPR tiling](serverless/alpr_cache), hourly [counts](serverless/alpr_counts), [blog scraping](serverless/blog_scraper))
* AWS S3 (static JSON served via `cdn.deflock.me`)
* Cloudflare (DNS, proxy, CDN cache)
* Terraform (infrastructure as code)

### External Services
* OpenStreetMap — Overpass API for ALPR data, map tiles
* Nominatim — Geocoding (proxied through the API)
* Wikimedia Commons — ALPR images linked from OSM tags

## Repository Structure

```
deflock/
├── webapp/           # Vue3 frontend (main user-facing site)
├── api/              # Fastify API (geocoding, GitHub sponsors proxy)
├── cms/              # Directus CMS (Docker-based, SQLite)
├── serverless/       # AWS Lambda functions
│   ├── alpr_cache/   # Generates map tiles daily (Docker Lambda)
│   ├── alpr_counts/  # Counts total ALPRs hourly (zip Lambda)
│   └── blog_scraper/ # Syncs RSS feed to CMS
├── terraform/        # Infrastructure as code
├── scripts/          # Utility scripts (Directus backup)
```

## Usage

### Requirements
* Node.js / npm (webapp)
* Bun (API)
* Docker (CMS — optional for frontend development)

### Running the Webapp

```sh
cd webapp
npm install
npm run dev    # Dev server on port 5173
```

### Running the API

```sh
cd api
bun install
bun server.ts  # Dev server on port 3000
```

Requires a `GITHUB_TOKEN` in `api/.env` for the sponsors endpoint.

### Running the CMS (optional)

```sh
cd cms
docker compose up  # Runs Directus on port 8055
```

Requires `DIRECTUS_SECRET`, `CF_API_KEY`, and `CF_ZONE_ID` environment variables.

The webapp always hits the production CMS (`cms.deflock.me`), so running it locally is not needed for frontend development.

### Building for Production

```sh
cd webapp
npm run build              # Production build
npm run build-with-typecheck  # Type-check + build
```

## Contributing

We welcome contributions from anyone. Here's how you can help:

### How to Contribute

1. Fork the Repository
2. Make Your Changes
3. Open a Pull Request against This Repo
