<img src="https://github.com/tripolskypetr/backtest-kit/raw/refs/heads/master/assets/consciousness.svg" height="45px" align="right">

# 🧿 backtest-monorepo-parallel

> A **TypeScript monorepo template** for [backtest-kit](https://github.com/tripolskypetr/backtest-kit) that runs **9 symbols in parallel** with shared infrastructure (Mongo + Redis) and a **self-enforcement runtime** — strategy files in `./content/` can reach the workspace DI container (`globalThis.core`) at strategy-evaluation time without imports, without bundler hooks, and without touching the strategy author's code.

![screenshot](./screencast.gif)

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/tripolskypetr/backtest-kit)
[![npm](https://img.shields.io/npm/v/backtest-kit.svg?style=flat-square)](https://npmjs.org/package/backtest-kit)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue)]()
[![Build](https://github.com/tripolskypetr/backtest-kit/actions/workflows/webpack.yml/badge.svg)](https://github.com/tripolskypetr/backtest-kit/actions/workflows/webpack.yml)

📚 **[API Reference](https://backtest-kit.github.io/documents/example_02_first_backtest.html)** | 🌟 **[Quick Start](https://github.com/tripolskypetr/backtest-kit/tree/master/example)** | **📰 [Article](https://backtest-kit.github.io/documents/article_07_ai_news_trading_signals.html)**

---

## ⚡ Measured Speed

> At ~6 300× real time, a full 27-day window across all 9 symbols (extrapolating linearly from the 34-minute slice) takes **on the order of a few minutes of wall time**, dominated by Mongo writes + first-touch candle fetches. The hot loop itself — `listenActivePing → getPositionEntries → commitAverageBuy` — runs at the **103 ev/s** rate measured above.

`apr_2026.strategy.ts` prints `active <SYMBOL> backtest=<historicalMs> now=<wallClockMs>` on every `listenActivePing`. The file in this repo captured **297 events across 9 symbols** in **a single Node process**, against a hot Mongo + Redis cache, on a **HP Victus 15-FA1022CI** laptop (mid-range gaming-class, i.e. commodity developer hardware — not a server).

### Measured numbers

| Metric | Value |
|---|---|
| Wall-clock span (first → last event) | `1779292952202 − 1779292949309` = **2 893 ms** (~2.9 s) |
| Total events captured | **297** |
| Symbols running in parallel | **9** (BTC, POL, ZEC, HYPE, XAUT, DOGE, SOL, PENGU, HBAR) |
| Historical time advanced per symbol | `1775003640000 − 1775001600000` = **2 040 000 ms** = **34 minutes** |
| **Per-symbol replay speed** | 34 min historical ÷ 2.9 s wall = **≈ 703×** real-time |
| **Aggregate replay speed (9 symbols)** | 9 × 703 = **≈ 6 326×** real-time |
| Event throughput | 297 ev / 2.893 s = **≈ 103 events/sec** (one Node process) |
| Frame coverage | `2026-04-01 → 2026-04-27` = 27 days × 1m candles = **38 880 candles/symbol × 9** = **~350 000 candle ticks** |

### Test bench

| Component | Spec |
|---|---|
| Model | HP Victus by HP Gaming Laptop 15-fa1xxx |
| CPU | **13th Gen Intel® Core™ i5-13420H** — 8 cores / 12 threads, base 2.10 GHz, 12 MB L3 |
| RAM | **16 GB** DDR4 (2× SK Hynix 8 GB @ 3200 MT/s) |
| GPU | Intel UHD Graphics (iGPU) + NVIDIA GeForce RTX 2050 4 GB (dGPU, **unused** by the runtime — Node only) |
| Storage | Samsung MZVL2512HCJQ NVMe SSD, 512 GB |
| OS | Windows 10 Enterprise LTSC 19044 (64-bit) |
| Node | single process; Mongo + Redis on the same machine via docker-compose |

### Why so fast?

1. **Single-process concurrency.** All 9 `Backtest.background(...)` calls share one Node event loop, one Mongo connection pool, and one Redis pool. No IPC, no subprocess fork overhead.
2. **Redis O(1) lookup cache.** Every `findByContext(...)` is a hot-path Redis `GET` (one round-trip on localhost) before falling back to Mongo. See [BaseMap](packages/core/src/lib/common/BaseMap.ts) — strings only, no JSON parse on the cache key.
3. **Atomic upserts.** Every `write*Data(...)` is a single `findOneAndUpdate({ filter:uniqueIndex, $set:payload }, { upsert, new })` — no read-modify-write, no application-side locks, no E11000 retry loop under concurrent symbol writes.
4. **Cached candles.** The `--cache` flag in Mode A pre-warms every symbol's candles into Mongo via [`cacheCandles`](packages/main/src/main/backtest.ts#L5) before the runners start, so the inner loop never blocks on ccxt HTTP. Pure CPU + local I/O from that point on.
5. **JIT-friendly hot path.** The per-tick body in [`apr_2026.strategy.ts`](content/apr_2026.strategy/apr_2026.strategy.ts) is ~30 lines of synchronous arithmetic + a few awaited helpers. V8 inlines aggressively.

---

## 🚀 Quick Start

### 1. Bring up infrastructure (Mongo + Redis)

```bash
docker-compose -f docker/mongodb/docker-compose.yaml up -d
docker-compose -f docker/redis/docker-compose.yaml up -d
```

### 2. Build the workspace packages

The host `package.json` `build` script delegates to per-platform scripts that walk every package in `./packages/*` and run `npm run build` inside each (which itself runs `rollup -c`):

```bash
npm run build           # → dotenv → ./scripts/linux/build.sh
npm run build:win       # → dotenv → ./scripts/win/build.bat
```

Each build emits two artifacts per package:

- `packages/<pkg>/build/index.cjs` — CommonJS bundle (consumed at runtime via [config/alias.config.ts](config/alias.config.ts))
- `packages/<pkg>/types.d.ts` — rolled-up `.d.ts` bundle (consumed at compile-time via [tsconfig.json](tsconfig.json) `paths`)

### 3. Run a backtest

**Mode A — parallel runner, entry in `packages/main/src/main/backtest.ts`:**

```bash
npm run start -- --backtest --entry --ui --cache ./content/apr_2026.strategy/apr_2026.strategy.ts
```

`--entry` activates the gate in [backtest.ts:23](packages/main/src/main/backtest.ts#L23), which then iterates [`CC_SYMBOL_LIST`](packages/main/src/config/params.ts#L13) — by default `BTCUSDT,POLUSDT,ZECUSDT,HYPEUSDT,XAUTUSDT,DOGEUSDT,SOLUSDT,PENGUUSDT,HBARUSDT` — and calls `Backtest.background(symbol, …)` for each. All 9 contexts run **concurrently in the same Node process**, sharing the same Mongo + Redis connections.

**Mode B — single-strategy runner, entry inside `@backtest-kit/cli`:**

```bash
npm run start -- --backtest --ui --noCache ./content/apr_2026.strategy/apr_2026.strategy.ts
```

No `--entry` → [`packages/main/src/main/backtest.ts`](packages/main/src/main/backtest.ts) early-returns, and the bundled CLI from `@backtest-kit/cli` takes over: it loads the strategy file, calls schemas registered inside it, and runs a single backtest on whatever symbol the strategy itself dispatches.

---

## 📈 Codebase Scaling Pattern

This template is designed to grow without restructuring. The recipe per new capability:

### Adding a new service to `@pro/core`

1. **File** — drop `XService.ts` under `packages/core/src/lib/services/<category>/`.
2. **Symbol** — add `xService: Symbol('xService')` to [packages/core/src/lib/core/types.ts](packages/core/src/lib/core/types.ts).
3. **Provider** — add `provide(TYPES.xService, () => new XService())` to [packages/core/src/lib/core/provide.ts](packages/core/src/lib/core/provide.ts).
4. **Expose** — add `xService: inject<XService>(TYPES.xService)` to the `ioc` object in [packages/core/src/lib/index.ts](packages/core/src/lib/index.ts).
5. **Build** — `npm run build` regenerates `types.d.ts`, and `core.xService` is now globally typed and runtime-callable from any file under `./content/` or `./config/`.

No file under `./content/` ever needs to change for new services to become available.

### Adding a new Mongo collection

1. Define the schema under `packages/core/src/schema/<Name>.schema.ts` with the same pattern as [Candle.schema.ts](packages/core/src/schema/Candle.schema.ts) — `mongoose.model(...)` + a compound unique index whose shape matches your context key.
2. Wrap it in a `<Name>DbService extends BaseCRUD(NameModel)` ([packages/core/src/lib/common/BaseCRUD.ts](packages/core/src/lib/common/BaseCRUD.ts) gives you `create / update / findById / findByFilter / findAll / iterate / paginate` for free).
3. Use `findOneAndUpdate(filter, { $set }, { upsert: true, new: true, setDefaultsOnInsert: true })` for writes — same atomicity contract as the 15 persist adapters in the upstream template.

### Adding a new Redis-cached lookup

Extend `BaseMap(REDIS_KEY, ttlOrMinusOne)` from [packages/core/src/lib/common/BaseMap.ts](packages/core/src/lib/common/BaseMap.ts) — same pattern as the upstream `*CacheService` classes. Pass `-1` for no TTL on cache-only-by-key entries, or a positive number of seconds for time-bounded caches.

### Adding a new strategy

Drop a new directory under `./content/<my>.strategy/` with a `.strategy.ts` file that calls `addStrategySchema(...)`. Wire its symbol list and frame in a sibling `modules/backtest.module.ts` (compare [apr_2026 modules](content/apr_2026.strategy/modules/backtest.module.ts)). No monorepo changes required — the CLI loads the file by path argument.

### Adding a new entry point (mode)

Mirror [packages/main/src/main/backtest.ts](packages/main/src/main/backtest.ts):

1. Gate on a new flag in [helpers/getArgs.ts](packages/main/src/helpers/getArgs.ts).
2. Add a file under `packages/main/src/main/<mode>.ts` that early-returns unless its flag is set.
3. Re-export it from [packages/main/src/index.ts](packages/main/src/index.ts).

The `@pro/main` rollup picks it up automatically; the CLI loads it via [config/loader.config.ts](config/loader.config.ts).

---

## 🛠️ Persistence Configuration

[config/setup.config.ts](config/setup.config.ts) declares which adapter each backtest-kit subsystem uses. The current setup runs all **Live** modes against Mongo via `@backtest-kit/mongo` and most **Backtest** modes against in-memory or local-file adapters — backtest replay benefits from never touching the disk for transient state, while live mode needs Mongo's durability:

| Subsystem | Live | Backtest |
|---|---|---|
| Session | Persist (Mongo) | Local file |
| Storage | Persist | Memory |
| Recent | Persist | Memory |
| Notification | Persist | Memory |
| Memory | Persist | Local file |
| State | Persist | Local file |
| Markdown | Dummy (no-op) | Dummy |
| Log | JSONL | JSONL |

Re-pointing any of these to a custom adapter is one line in [config/setup.config.ts](config/setup.config.ts).

---

## 🔑 Environment Variables

| Var | Default | Purpose |
|---|---|---|
| `CC_MONGO_CONNECTION_STRING` | `mongodb://localhost:27017/backtest-pro?wtimeoutMS=15000` | Mongo connection (writes from all DbServices) |
| `CC_REDIS_HOST` | `127.0.0.1` | Redis host (BaseMap lookups) |
| `CC_REDIS_PORT` | `6379` | Redis port |
| `CC_REDIS_USER` | `default` | Redis ACL user |
| `CC_REDIS_PASSWORD` | `mysecurepassword` | Redis ACL password |
| `CC_SYMBOL_LIST` | `BTCUSDT,POLUSDT,ZECUSDT,HYPEUSDT,XAUTUSDT,DOGEUSDT,SOLUSDT,PENGUUSDT,HBARUSDT` | Comma-separated symbols for Mode-A parallel runner |

---

## 🧩 Dependencies

Notable runtime deps from the root [package.json](package.json):

- **`backtest-kit` 9.8.2** — engine. [`@backtest-kit/cli`](https://npmjs.org/package/@backtest-kit/cli) is the runner; [`@backtest-kit/mongo`](https://npmjs.org/package/@backtest-kit/mongo) supplies `setup()` + persist adapters; [`@backtest-kit/ui`](https://npmjs.org/package/@backtest-kit/ui) is the optional web UI on `:60050`.
- **`di-kit`** — symbol-keyed IoC container used by `@pro/core` (re-exported as `createActivator("pro")`).
- **`di-factory`** — class-factory helper for `BaseCRUD` and `BaseMap`.
- **`mongoose` 8.x** — Mongo client + schemas.
- **`ioredis` 5.x** — Redis client (used through `getRedis()` from `@backtest-kit/mongo`).
- **`functools-kit`** — `singleshot`, `errorData`, `str.newline`, etc. — small utility belt used throughout.
- **`ccxt`** — exchange client for the `ccxt-exchange` schema in [modules/backtest.module.ts](content/apr_2026.strategy/modules/backtest.module.ts).
- **`telegram` + `qrcode-terminal`** — MTProto session for the `--session` mode (auth via QR scan).

---

## 📜 Summary

| Aspect | This monorepo |
|---|---|
| **Throughput** | ~6 300× real-time aggregate across 9 symbols, ~103 events/sec on a HP Victus 15-FA1022CI laptop (i5-13420H, 16 GB DDR4-3200, NVMe SSD) |
| **Concurrency model** | Single Node process, 9 parallel `Backtest.background(...)` contexts sharing one event loop |
| **DI surface** | `globalThis.core` typed via root `tsconfig.json` paths → rolled-up `types.d.ts` |
| **Strategy isolation** | Strategy files in `./content/` are loaded by `@backtest-kit/cli` at runtime, never bundled into `@pro/*` |
| **Build** | `rollup -c` per package, walked by `scripts/win/build.bat` / `scripts/linux/build.sh` |
| **Scaling** | New service = +1 symbol, +1 provider, +1 ioc entry. No churn under `./content/` |
| **Persistence** | Mongo (durable, live) + Redis (O(1) cache) + Local/Memory (backtest-fast) per-subsystem in `config/setup.config.ts` |
