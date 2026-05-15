# Chapter 5: Thunder — In-Network Post Store

> Sub-millisecond post retrieval from accounts you follow

---

## Overview

Thunder is an **in-memory real-time post store** that serves "in-network" candidate posts — posts from people you follow. It avoids database lookups entirely, consuming posts directly from Kafka and serving them from memory.

**Location:** `thunder/`  
**Language:** Rust

---

## Architecture

```
Kafka (post create/delete events)
          │
          ▼
  ┌──────────────────┐
  │   KafkaConsumer   │ ← Deserializes proto events
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │    PostStore      │ ← Per-user in-memory maps
  │                   │    • Original posts
  │   HashMap<u64,    │    • Replies / Reposts
  │     Vec<LightPost>│    • Video posts
  │   >               │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ ThunderServiceImpl│ ← gRPC server
  │                   │    GetInNetworkPosts
  └──────────────────┘
```

---

## The Service Implementation

From `thunder_service.rs`:

```rust
pub struct ThunderServiceImpl {
    post_store: Arc<PostStore>,          // In-memory post storage
    strato_client: Arc<StratoClient>,    // Fallback for following list
    request_semaphore: Arc<Semaphore>,   // Concurrency limiter
}
```

### Request Flow

```rust
async fn get_in_network_posts(&self, request) -> Result<GetInNetworkPostsResponse, Status> {
    // 1. Acquire semaphore — reject immediately if at capacity
    let _permit = match self.request_semaphore.try_acquire() {
        Ok(permit) => { IN_FLIGHT_REQUESTS.inc(); permit }
        Err(_) => {
            REJECTED_REQUESTS.inc();
            return Err(Status::resource_exhausted("Server at capacity"));
        }
    };

    // 2. Get following list (from request or Strato fallback)
    let following_user_ids = if req.following_user_ids.is_empty() && req.debug {
        self.strato_client.fetch_following_list(user_id, MAX_INPUT_LIST_SIZE).await
    } else {
        req.following_user_ids
    };

    // 3. Limit inputs
    let following_user_ids: Vec<u64> = following_user_ids.into_iter()
        .take(MAX_INPUT_LIST_SIZE).collect();

    // 4. Fetch posts from in-memory store (spawn_blocking to avoid async blocking)
    let proto_posts = tokio::task::spawn_blocking(move || {
        let all_posts = if req.is_video_request {
            post_store.get_videos_by_users(&following_user_ids, &exclude, ...)
        } else {
            post_store.get_all_posts_by_users(&following_user_ids, &exclude, ...)
        };

        // Score by recency and limit
        score_recent(all_posts, max_results)
    }).await;
}
```

### Recency Scoring

Thunder's scoring is intentionally simple — just recency:

```rust
fn score_recent(mut light_posts: Vec<LightPost>, max_results: usize) -> Vec<LightPost> {
    // Sort by creation time, newest first
    light_posts.sort_unstable_by_key(|post| Reverse(post.created_at));
    // Limit to max results
    light_posts.into_iter().take(max_results).collect()
}
```

This is by design — the heavy ranking is left to Phoenix. Thunder just provides the candidate pool.

---

## Post Statistics & Observability

Thunder tracks rich metrics for every request:

```rust
fn analyze_and_report_post_statistics(posts: &[LightPost], stage: &str) {
    // Freshness: time since most recent post
    // Time range: spread between newest and oldest
    // Reply ratio: replies / total
    // Unique authors: diversity measure
    // Posts per author: concentration measure
}
```

Metrics reported:
- `GET_IN_NETWORK_POSTS_FOUND_FRESHNESS_SECONDS` — how fresh are the posts
- `GET_IN_NETWORK_POSTS_FOUND_TIME_RANGE_SECONDS` — temporal spread
- `GET_IN_NETWORK_POSTS_FOUND_REPLY_RATIO` — reply vs original ratio
- `GET_IN_NETWORK_POSTS_FOUND_UNIQUE_AUTHORS` — author diversity
- `GET_IN_NETWORK_POSTS_FOUND_POSTS_PER_AUTHOR` — author concentration
- `IN_FLIGHT_REQUESTS` / `REJECTED_REQUESTS` — load management

---

## Key Design Decisions

1. **No database** — Purely in-memory with Kafka replay for durability
2. **Back-pressure** — Semaphore-based concurrency limiting with immediate rejection
3. **Separate video store** — Dedicated video post storage for `is_video_request`
4. **`spawn_blocking`** — Post store lookups use `tokio::task::spawn_blocking` to avoid blocking the async runtime
5. **Simple scoring** — Recency only; leaves complex ranking to the pipeline

---

## Next Chapter

→ [Chapter 6: Scoring & Ranking — How Posts Get Their Score](./06-scoring-and-ranking.md)
