# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

LazyCache is a thread-safe in-memory caching library for .NET. The core value proposition is `GetOrAdd` / `GetOrAddAsync`: a factory delegate is guaranteed to execute **once** per key even under concurrent load. This is achieved by storing `Lazy<T>` / `AsyncLazy<T>` wrappers inside `Microsoft.Extensions.Caching.Memory.IMemoryCache` rather than the raw values.

Target framework is `net10.0` (upgraded from netstandard2.0 on the `clear-cache` branch). Depends on `Microsoft.Extensions.Caching.Memory`.

## Build & Test

```bash
dotnet restore
dotnet build --configuration Release
dotnet test                                    # all test projects
dotnet test LazyCache.UnitTests                # single project
dotnet test --filter "FullyQualifiedName~MethodName"   # single test
```

Tests use NUnit 4 + FluentAssertions.

## Architecture

Two layers, both injectable:

- **`IAppCache` / `CachingService`** — the consumer-facing API (`Get`, `GetOrAdd`, `GetOrAddAsync`, `Add`, `Remove`, `RemoveAll`). This layer wraps factory results in `Lazy<T>` / `AsyncLazy<T>` and handles single-evaluation semantics.
- **`ICacheProvider` / `MemoryCacheProvider`** — a thin adapter over `IMemoryCache`. Swap this to plug in a different backing store. `MemoryCacheProvider` is constructed with a `Func<IMemoryCache>` factory so `RemoveAll()` can atomically swap in a fresh `IMemoryCache` (via `Interlocked.Exchange`) and dispose the old one — `IMemoryCache` has no native clear-all operation.

Key implementation points:

- `CachingService` uses a process-wide `SemaphoreSlim` to serialize *insertions* into the provider. The factory delegate runs **outside** the lock — the lock only guards placing the `Lazy<T>` wrapper. Actual single-evaluation is enforced by `Lazy<T>` / `AsyncLazy<T>` themselves.
- `GetValueFromLazy` / `GetValueFromAsyncLazy` unwrap `Lazy<T>`, `AsyncLazy<T>`, `Task<T>`, or raw `T` — this lets callers `Add` a raw value and `GetOrAdd` it back, or mix sync/async on the same key.
- `EnsureEvictionCallbackDoesNotReturnTheAsyncOrLazy` rewrites user-supplied eviction callbacks so they receive the unwrapped value, not the `Lazy<T>` wrapper.
- If the factory throws, the key is removed from the cache — exceptions are never cached. Same for cancelled/faulted `Task`s in `GetOrAddAsync`.
- The default `CachingService()` constructor uses a static `DefaultCacheProvider` — this means **by default, all `new CachingService()` instances share one cache**. Tests and DI scenarios typically pass a provider explicitly.

## Projects

- `LazyCache/` — the library itself; only this is published to NuGet as `LazyCache`.
- `LazyCache.AspNetCore/` — `AddLazyCache()` DI extension that registers `IAppCache` and `ICacheProvider` as singletons. Namespace is `Microsoft.Extensions.DependencyInjection` by convention.
- `LazyCache.Ninject/` — Ninject module equivalent.
- `LazyCache.UnitTests/`, `LazyCache.Ninject.UnitTests/` — NUnit tests. Project names ending in `Tests.csproj` are picked up by `build.ps1`.
- `CacheDatabaseQueriesApiSample/` — ASP.NET Core sample caching EF queries.
- `Console.Net461/` — legacy-framework smoke test; kept to verify compatibility (relevant if framework targets are ever re-broadened).

## Mocks

`LazyCache.Mocks.MockCachingService` is a shipped implementation of `IAppCache` that always invokes the factory and never caches — use it in tests to take caching out of the picture without stubbing the whole interface.
