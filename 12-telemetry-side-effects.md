# Chapter 12: Telemetry & The Side Effect Pipeline

Once the `Home Mixer` gRPC server successfully builds the final feed and returns it to the client, the work is not done. The Rust pipeline immediately spawns 16 asynchronous "Side Effects" in the background.

This chapter breaks down the most critical side effects and how they handle telemetry, caching, and serving history without increasing latency for the user.

---

## 1. The Async Architecture

Side Effects in the X algorithm are implemented via the `SideEffect` trait.

```rust
#[async_trait]
pub trait SideEffect<Q, C> {
    fn enable(&self, query: Arc<Q>) -> bool;
    async fn side_effect(&self, input: Arc<SideEffectInput<Q, C>>) -> Result<(), String>;
}
```

Because `side_effect` is an `async fn`, the Home Mixer can use `tokio::spawn` to fire off all 16 side effects concurrently *after* the HTTP response has been flushed to the client's phone. This allows the backend to do heavy logging and caching with **zero impact on user latency**.

---

## 2. Preventing Duplicate Feeds (Served History)

Have you ever refreshed your feed and seen the exact same posts? The `UpdateServedHistorySideEffect` exists to prevent this.

### How it works
1. It takes the final `selected_candidates` that were just sent to the user.
2. It builds a `ServedHistory` object containing the `RequestType` (e.g., `INITIAL`, `POLLING`, `NEWER`).
3. It makes a network call to the `ServedHistoryClient` to permanently log that the user has seen these specific `tweet_id`s.

Future requests to the Phoenix Retrieval layer will explicitly filter out these IDs (via `previously_served_posts_filter.rs`) so your feed stays fresh.

---

## 3. The 180-Second Multi-Region Cache

The `RedisPostCandidateCacheSideEffect` is one of the most critical scaling mechanisms in the architecture.

When Phoenix retrieves 1,000 candidates and ranks them, only the top ~40 make it into the final feed payload. The other 960 represent highly expensive ML compute that would normally be thrown away. 

### The Caching Strategy
Instead of discarding them, the side effect takes the top candidates (both selected and non-selected that have a score > 0.0), serializes them to JSON, and caches them in Redis with a 3-minute TTL (`REDIS_TTL_SECONDS: u64 = 180`).

### CPU Optimization via Tokio
Serialization and compression of thousands of JSON objects is CPU-bound, which can block the main async event loop. To prevent this, the engineers explicitly offload the compression to a dedicated thread pool:

```rust
let compressed_payload = tokio::task::spawn_blocking(move || {
    zstd::encode_all(json_payload.as_slice(), ZSTD_COMPRESSION_LEVEL)
}).await
```

By using Zstandard (`zstd`) compression on a blocking thread, they massively reduce the Redis network payload size without stalling the asynchronous web server.

---

## 4. Ads and Kafka Telemetry

Several side effects are dedicated purely to tracking what happened in the pipeline so ML models can be retrained.
*   **`AdsInjectionLoggingSideEffect`**: Tracks exactly which Ad was blended into the organic feed, its `impression_id`, and what index it was placed at (handled by the SafeGap blender).
*   **`ClientEventsKafkaSideEffect`**: Pushes the entire request metadata to Kafka so downstream Python consumers (like Grox) can analyze feed quality.
*   **`RerankingKafkaSideEffect`**: Logs instances where the initial Phoenix ranking was overridden by secondary business logic.

## Summary
The side effect pipeline demonstrates excellent systems engineering. By deferring caching, logging, and history updates to background Tokio tasks, the X algorithm handles massive secondary compute requirements without slowing down the primary `Home Mixer` user request.
