# Antigravity Agent Guidelines

## Absolute Rules (Strict Limitations)
1. **Infrastructure Minimization:** Do NOT add secondary data stores (Redis, RabbitMQ, MongoDB, Elasticsearch, etc.). Use PostgreSQL for RDBMS, `JSONB` for NoSQL, `UNLOGGED` tables for caching, `LISTEN/NOTIFY` for pub/sub queues, and `TimescaleDB` extension for time-series monitoring.
2. **Zero-Proxy Streaming:** Fastify backends must ONLY relay, resolve, or parse `.m3u8` playlist URLs. **NEVER proxy raw audio/video byte streams** through the Fastify backend server to keep bandwidth costs at zero.
3. **Client-Side Decoding:** Audio stream decoding and playback must be strictly handled by the clients (`hls.js` on Nuxt 3 Web, `just_audio` / `audio_service` on Flutter Mobile).
4. **Strict Schema & Type Safety:** Every Fastify API endpoint must define explicit request/response schemas using `@fastify/typebox` or `Zod`.
5. **Station Isolation:** Each broadcast station parser (KBS, MBC, SBS, TBS, CBS, etc.) must be isolated in its own strategy module. A failure in one station parser must never crash or block other station streams.

---

## Architecture & Tech Stack

### Backend & Infrastructure
- **Runtime & Language:** Node.js (v20+) with TypeScript
- **Framework:** Fastify (for high-performance, low-latency API delivery)
- **Database Architecture (PostgreSQL All-in-One):**
  - **Relational DB:** Station metadata, user preferences, static configurations
  - **NoSQL (`JSONB`):** Dynamic, schema-less raw broadcast API/HTML parsing responses
  - **Cache (`UNLOGGED` Tables):** In-memory level fast storage for short-lived `.m3u8` URLs
  - **Queue (`LISTEN / NOTIFY`):** Native event-driven background parsing & health checks
  - **Time-Series (`TimescaleDB`):** Stream availability, latency (ms), and HTTP status metric logs
- **Monitoring:** Grafana connected directly to PostgreSQL/TimescaleDB

### Client Applications
- **Web App:** Nuxt 3 (Vue 3, TypeScript, SSR enabled for SEO, `hls.js`)
- **Mobile App:** Flutter (Dart, iOS/Android background audio playback via `just_audio` & `audio_service`)

---

## Directory Layout

```text
├── apps/
│   ├── backend/               # Fastify API Server
│   │   ├── src/
│   │   │   ├── config/        # Environment & App Config
│   │   │   ├── plugins/       # Fastify DB (pg/Timescale), CORS, AutoLoad plugins
│   │   │   ├── routes/        # Versioned API Routes (/api/v1/...)
│   │   │   ├── services/      # Station Parsers (KBS, MBC, SBS, etc.) & Health Checkers
│   │   │   ├── types/         # TypeBox/Zod Schemas and TS Interfaces
│   │   │   └── index.ts       # Application Entry Point
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── web/                   # Nuxt 3 Web App
│   └── mobile/                # Flutter Mobile App
├── init-db/                   # Database Initialization Scripts
│   └── 01_init.sql            # Extensions, Hypertable, and Base Tables
├── compose.yml                # Container Orchestration (Postgres + Grafana + Backend)
└── AGENTS.md                  # Development Guidelines & AI Agent Rules
```

## Workflow

### 1. Database First

Always run docker-compose up postgres -d to ensure local PostgreSQL + TimescaleDB instance is healthy.

Database schema updates must be mirrored in init-db/01_init.sql.

### 2. Backend Development

Station parsing strategies must implement a unified interface (StationParserStrategy).

Parsed HLS stream URLs must be upserted to the stream_cache (UNLOGGED) table.

Health checking workers must write ping metrics directly to the stream_health_logs hypertable.

### 3. API Contracts

Define TypeScript types / TypeBox schemas in apps/backend/src/types/.

Ensure response payloads strictly match client requirements for both Nuxt 3 and Flutter apps.

## Test

### 1. Database & Schema Verification

- Verify TimescaleDB extension and stream_health_logs hypertable creation

    ```bash
    docker exec -it radio_postgres psql -U radio_user -d radio_db -c "\dx"
    docker exec -it radio_postgres psql -U radio_user -d radio_db -c "\d stream_health_logs"
    ```

### 2. Backend Integration Testing

- Run Fastify test runner (Vitest or Node Test Runner)

    ```bash
    cd apps/backend && npm test
    ```

- Test individual stream parsing modules to verify .m3u8 extraction accuracy

### 3. Health Check Metric Log Verification

- Ensure stream latency and HTTP status logs are being written to TimescaleDB

    ```SQL
    SELECT time, station_id, status_code, latency_ms FROM stream_health_logs ORDER BY time DESC LIMIT 10;
    ```