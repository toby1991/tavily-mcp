# Multi API Key Rotation Design

## Problem

Tavily single API key has rate limits. Heavy usage causes 429 errors that block all requests until the cooldown expires.

## Solution

Add a `KeyManager` class that manages multiple API keys with round-robin rotation and smart cooldown.

## Configuration

| Env Variable | Format | Description |
|---|---|---|
| `TAVILY_API_KEYS` | JSON array `["key1","key2","key3"]` | Multiple keys (preferred) |
| `TAVILY_API_KEY` | Single string | Backward compatible, single key |
| `TAVILY_KEY_COOLDOWN_MS` | Number (ms) | Base cooldown duration, default 60000 |

Priority: `TAVILY_API_KEYS` > `TAVILY_API_KEY`. Both work, fully backward compatible.

## Rotation Strategy

- **Round-Robin**: Each request cycles to the next available key.
- **429 (Rate Limit)**: Key enters cooldown with exponential backoff (base 60s, max ~64min).
- **401 (Invalid Key)**: Key permanently disabled until process restart.
- **Recovery**: Cooldown keys lazily restored in `getNextKey()` when their cooldown expires.
- **Success**: Resets consecutive failure count, keeps key active.

## Key Pinning

`research()` uses long-polling (POST + repeated GETs). The same key is used for the entire lifecycle of a research request to avoid issues with `request_id` being bound to a specific key.

## Architecture

```
loadApiKeys() → KeyManager → TavilyClient
                  ↕
          getNextKey()      (round-robin)
          markFailed(key)   (429/401 → cooldown)
          markSuccess(key)  (reset failures)
```

Each API method (search/extract/crawl/map/research) calls `getNextKey()` before the request and sets the `Authorization` header per-request instead of using a fixed header on the axios instance.

## Files Changed

- `src/index.ts`: Added `loadApiKeys()`, `KeyState`, `KeyManager` (~130 lines). Modified `TavilyClient` constructor and all 5 API methods. Total: +195 lines, -13 lines.
