# Latchkey Fast Cache

High-performance streaming cache for [Latchkey](https://latchkey.dev) managed runners. Saves and
restores dependency caches in a single streaming HTTP request — no chunked reserve/commit round
trips, no Node.js bootstrap.

**New to Latchkey?** Start at **[latchkey.dev/sign-in](https://latchkey.dev/sign-in)**: sign in with
GitHub, connect your organization, and pick your repositories. Then point a job at a Latchkey runner
and this action works with no further setup.

> On non-Latchkey runners the cache proxy is not available and this action will not work. Use
> [`actions/cache@v4`](https://github.com/actions/cache) instead.

## Usage

```yaml
jobs:
  build:
    runs-on: [self-hosted, latchkey-medium]
    steps:
      - uses: actions/checkout@v4

      - uses: latchkey-dev/cache-action@v1
        id: cache
        with:
          action: restore
          key: deps-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          path: node_modules

      - run: npm ci
        if: steps.cache.outputs.cache-hit != 'true'

      # ... build steps ...

      - uses: latchkey-dev/cache-action@v1
        if: steps.cache.outputs.cache-hit != 'true'
        with:
          action: save
          key: deps-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          path: node_modules
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `action` | `"save"` or `"restore"` | **Yes** | |
| `path` | Path(s) to cache (newline or space separated) | **Yes** | |
| `key` | Cache key | **Yes** | |
| `cache-url` | Cache proxy base URL | No | `http://localhost:8080` |

## Outputs

| Output | Description |
|--------|-------------|
| `cache-hit` | Whether the cache was restored successfully (`true`/`false`) |

## How it works

1. **Restore** — checks cache existence (HEAD), then streams a `GET` through `zstd -d` decompression and `tar` extraction
2. **Save** — checks whether the cache already exists (HEAD), then streams `tar | zstd | curl PUT` in a single pipeline, with no intermediate files
3. **Cache isolation** — keys are versioned by `RUNNER_OS` automatically. For finer isolation (different Ubuntu versions, say) encode it in your cache key

The cache proxy runs on each [Latchkey managed runner](https://latchkey.dev/documentation/runners-overview)
as a local service backed by S3, exposing a streaming V2 API. Throughput comes from:

- A single streaming HTTP request, so no per-chunk round trips
- Multi-threaded `zstd` compression and decompression (`-T0`)
- Parallel S3 multipart uploads — 16 concurrent 8MB parts
- Parallel S3 range-request downloads — 16 concurrent 64MB parts
- Data locality: S3 in the same region as the runner

Full details in the [dependency caching docs](https://latchkey.dev/documentation/dependency-caching).
Docker layer caching is handled separately by
[`latchkey-dev/docker-cache-action`](https://github.com/latchkey-dev/docker-cache-action) — see
[Docker layer caching](https://latchkey.dev/documentation/docker-layer-caching).

## About Latchkey

Managed, ephemeral GitHub Actions runners that repair transient build failures mid-run. A fresh
Ubuntu 24.04 machine per job, destroyed when the job ends, and a hosted MCP server so your coding
agent can triage failures directly.

- [Documentation](https://latchkey.dev/documentation) · [Quickstart](https://latchkey.dev/documentation/quickstart) · [Runners](https://latchkey.dev/github-actions-runners)
- [Self-healing: what it does](https://latchkey.dev/documentation/self-healing)
- [Connect your AI agent (MCP)](https://latchkey.dev/documentation/connect-your-ai-agent)
- [Free for open source](https://latchkey.dev/open-source) — a free Scale plan for public, openly licensed projects

## License

MIT
