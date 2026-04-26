# subtitle-delivery

Subtitle delivery API for HTML5 players, focused on safe ingestion, cache-backed delivery, and straightforward operational setup.

## Overview

This project exposes a small Go API that accepts subtitle URLs, validates downloaded subtitle content for safety, stores the latest valid source URL in cache, and returns that original URL to clients.

Current behavior is intentionally minimal: the API validates remote subtitle content for safety but stores and returns only the original URL.

Main capabilities:

- Accept subtitle URLs through `POST /subtitle`
- Return the latest valid subtitle through `GET /subtitle`
- Accept only subtitle file extensions: `.srt`, `.vtt`, `.webvtt`
- Reject malicious subtitle content
- Store only metadata with temporary cache expiration (no subtitle serving)
- Enforce file-size limits
- Apply rate limiting
- Restrict cross-origin access through configurable CORS

## Requirements

- Go `1.24.2`
- Docker, if you want to run the container image

## Project Structure

- `main.go`: entrypoint focused on process setup
- `internal/infrastructure/config/config.go`: runtime configuration loader (`.env.dev`, `.env.prod`, and env vars)
- `internal/domain/subtitle.go`: subtitle entity and pure domain rules such as validation and normalization
- `internal/service/subtitle_service.go`: use-case orchestration for create and retrieve subtitle flows
- `internal/controller/http_controller.go`: HTTP translation layer between requests and service calls
- `internal/httpapi/router.go`: HTTP route registration
- `internal/httpapi/middleware.go`: HTTP middlewares for CORS and rate limiting
- `internal/httpapi/api_integration_test.go`: end-to-end HTTP behavior tests close to transport implementation
- `internal/infrastructure/memory_store.go`: in-memory cache implementation with expiration and invalidation
- `internal/infrastructure/db/redis_store.go`: Redis-backed subtitle store and DB-related persistence details
- `internal/infrastructure/fetcher.go`: concrete HTTP and test fetchers
- `internal/infrastructure/*_test.go`: tests near infrastructure implementations
- `*_test.go`: HTTP and unit tests covering the acceptance criteria
- `.env.dev`: default local development configuration
- `.env.prod`: production template with empty values

## Architecture

The project now follows a more expressive layered structure inspired by Go project-layout conventions:

- `domain`: pure business rules and core subtitle model
- `service`: application use cases and orchestration between fetch and storage
- `controller`: HTTP-specific input and output handling
- `httpapi`: router and transport middleware composition
- `infrastructure`: adapters such as config, cache, db, and fetchers
- `main`: dependency wiring and server startup

This keeps the subtitle rules out of the transport layer and makes future storage or transport changes easy.

## Configuration

The application loads configuration in this order:

1. Process environment variables
2. `.env.prod` values when `APP_ENV=production` and the value is not empty
3. `.env.dev` fallback values
4. Internal defaults when needed

Available settings:

- `APP_ENV`: `development` or `production`
- `APP_PORT`: HTTP server port
- `STORAGE_BACKEND`: `memory_cache` or `redis`
- `REDIS_ADDR`: Redis host and port used when `STORAGE_BACKEND=redis`
- `REDIS_PASSWORD`: Redis password used when required by the server
- `REDIS_DB`: Redis logical database number
- `REDIS_KEY_PREFIX`: prefix applied to subtitle keys in Redis
- `MAX_SUBTITLE_SIZE_BYTES`: maximum accepted subtitle payload size
- `ALLOWED_ORIGINS`: comma-separated CORS allowlist
- `PROBE_ALLOWED_IPS`: comma-separated IP allowlist for `/health` (empty means no IP restriction)
- `HEALTH_PROTECTION_ENABLED`: enables or disables IP protection for `/health`
- `CACHE_TTL`: cache expiration duration, for example `10m`
- `RATE_LIMIT_BURST`: allowed requests per window per client
- `RATE_LIMIT_WINDOW`: rate limit window duration, for example `1m`

Default development values are stored in `.env.dev`. Production templates are stored in `.env.prod`.

## Running Locally

```bash
go run .
```

The API starts on the port defined by `APP_PORT`. With the default development configuration, the base URL is `http://localhost:8080`.

## Running Tests

```bash
go test ./... -count=1
```

The suite includes an operational scenario that validates a subtitle fetched from a local in-process HTTP source and verifies URL storage and retrieval without external network dependency.

## API Endpoints

### GET /health

Returns a lightweight liveness response.

Success response:

```json
{
	"status": "ok"
}
```

### POST /subtitle

Stores a subtitle referenced by URL.

Request:

```json
{
	"url": "https://example.com/subtitles/movie.srt"
}
```

Success response:

```json
{
	"id": "c8f8cc0f5b1d4e33ae52e2a763c4e81d",
	"url": "https://example.com/subtitles/movie.srt"
}
```

Possible errors:

- `400 Bad Request`: invalid JSON body, unsupported extension, or malicious subtitle content
- `403 Forbidden`: origin is not allowed by CORS
- `413 Request Entity Too Large`: subtitle content exceeds the configured maximum size
- `429 Too Many Requests`: rate limit exceeded
- `502 Bad Gateway`: the remote subtitle could not be fetched
- `500 Internal Server Error`: internal persistence or identifier generation failure

### GET /subtitle

Returns the latest valid subtitle URL stored in cache.

Success response:

```json
{
	"id": "c8f8cc0f5b1d4e33ae52e2a763c4e81d",
	"url": "https://example.com/subtitles/movie.srt"
}
```

Possible errors:

- `404 Not Found`: no valid subtitle is currently stored, or the cached subtitle expired
- `403 Forbidden`: origin is not allowed by CORS
- `429 Too Many Requests`: rate limit exceeded
- `500 Internal Server Error`: internal error

## Subtitle Validation Rules

- URLs must end with `.srt`, `.vtt`, or `.webvtt`
- Malicious markers such as `<script`, `javascript:`, `<iframe`, `onerror=`, and `onload=` are rejected
- The fetched subtitle body must not be empty
- The service validates downloaded content, but does not serve subtitle bodies to clients

## Cache Strategy

The current implementation supports two cache backends:

- `memory_cache`: default in-memory cache for local development and single-instance deployments
- `redis`: distributed cache backend for multi-instance deployments

Supported strategies today:

- Expiration through `CACHE_TTL`
- Automatic cleanup of expired entries during read and write operations in the in-memory backend
- Explicit invalidation support in the in-memory cache implementation for future administrative workflows
- Latest subtitle replication in Redis through a dedicated key with TTL

The API depends on a storage interface (`service.Store`), so the backend can be replaced with limited impact on handler code.

## CORS

Cross-origin access is controlled through `ALLOWED_ORIGINS`. The API:

- Accepts preflight requests from allowed origins
- Rejects requests from origins outside the allowlist
- Returns `Access-Control-Allow-Origin` only for accepted origins

## Rate Limiting

Rate limiting is applied per client identifier derived from the request remote address.

Implementation notes:

- Implementation: `RateLimiter` in `internal/httpapi/rate_limiter.go`

Behavior notes:

- `/health` is excluded from request rate limiting.
- `/subtitle` requests are rate limited per `RemoteAddr`.

Configuration:

- `RATE_LIMIT_BURST`: maximum number of requests in the configured window
- `RATE_LIMIT_WINDOW`: reset window duration

## Health Endpoint Protection

The `/health` endpoint supports optional IP-based protection.

- If `HEALTH_PROTECTION_ENABLED=false`, `/health` is not restricted by IP.
- If `HEALTH_PROTECTION_ENABLED=true` and `PROBE_ALLOWED_IPS` is empty, `/health` is not restricted by IP.
- If `HEALTH_PROTECTION_ENABLED=true` and `PROBE_ALLOWED_IPS` is configured, requests from non-listed IPs receive `403`.

## Docker

Build the image:

```bash
docker build -t subtitle-delivery:local .
```

Run the container:

```bash
docker run --rm -p 8080:8080 \
	-e APP_ENV=development \
	-e ALLOWED_ORIGINS=http://localhost:3000 \
	subtitle-delivery:local
```

Run the container with Redis as the storage backend:

```bash
docker run --rm -p 8080:8080 \
	-e APP_ENV=production \
	-e STORAGE_BACKEND=redis \
	-e REDIS_ADDR=redis:6379 \
	-e REDIS_DB=0 \
	-e REDIS_KEY_PREFIX=subtitle-delivery \
	-e ALLOWED_ORIGINS=http://localhost:3000 \
	subtitle-delivery:local
```

## Swapping the Storage Backend

The storage boundary is represented by `service.Store`. To add a new backend:

1. Create a new type implementing `Save` and `Latest`
2. Keep expiration and invalidation behavior explicit in the backend implementation
3. Extend `app.NewStorage` in `internal/app/setup.go` to select the backend by `STORAGE_BACKEND`
4. Add tests for the new backend behavior

## Contributing

1. Add or update tests first
2. Keep changes framework-free unless a library is clearly justified
3. Prefer focused files by responsibility
4. Keep documentation aligned with configuration and behavior changes
