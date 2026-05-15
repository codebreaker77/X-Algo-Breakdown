# Chapter 4: Home Mixer — Feed Assembly & Serving

> The orchestration layer that wires 28+ services into a ranked feed

---

## Overview

Home Mixer is the **Rust gRPC server** that receives a user request and returns the final ranked feed. It's the integration point that ties together Phoenix ML, Thunder in-network, candidate pipeline, ads blending, and dozens of external service clients.

**Location:** `home-mixer/`

---

## Server Architecture

### Entry Points (from `server.rs`)

```rust
impl pb::scored_posts_service_server::ScoredPostsService for ScoredPostsServer {
    async fn get_scored_posts(&self, request) -> Result<ScoredPostsResponse, Status>;
    async fn get_debug_scored_posts(&self, request) -> Result<DebugScoredPostsResponse, Status>;
}

impl pb::for_you_feed_service_server::ForYouFeedService for ForYouFeedServer {
    async fn get_for_you_feed(&self, request) -> Result<ForYouFeedResponse, Status>;
    async fn get_for_you_feed_urt(&self, request) -> Result<ForYouFeedUrtResponse, Status>;
}
```

### Server Initialization

```rust
async fn build(ctx: ServiceContext<HomeMixerConfig>) -> Self {
    // 1. Create Gizmoduck client for viewer data
    let gizmoduck_client = ProdGizmoduckClient::new(shard_coordinate, datacenter).await;

    // 2. Build query builder with feature switches
    let query_builder = QueryBuilder { feature_switches, decider, datacenter, gizmoduck_client };

    // 3. Create Phoenix candidate pipeline (wires 28+ clients)
    let phoenix_candidate_pipeline = PhoenixCandidatePipeline::prod(shard_coordinate, datacenter).await;

    // 4. Create scored posts server
    let scored_posts = ScoredPostsServer::new(query_builder.clone(), phoenix_candidate_pipeline);

    // 5. Create For You pipeline (wraps scored posts + adds URT conversion)
    let for_you_pipeline = ForYouCandidatePipeline::new(scored_posts, datacenter).await;
}
```

---

## The 28+ External Service Clients

From `phoenix_candidate_pipeline.rs`, the `build_with_clients()` function wires these:

### ML Model Clients
| Client | Purpose |
|--------|---------|
| `PhoenixPredictionClient` | Main ML scorer — Grok transformer predictions |
| `PhoenixPredictionClient` (egress) | Sidecar fallback for predictions |
| `PhoenixRetrievalClient` | Two-tower retrieval for out-of-network |
| `VMRankerClient` | Alternative video-specific ranking |

### Feature Stores
| Client | Purpose |
|--------|---------|
| `UserActionAggregationClient` | User engagement history sequences |
| `StratoClient` | General-purpose feature store |
| `TESClient` | Tweet Engagement Store |

### Social Graph
| Client | Purpose |
|--------|---------|
| `SocialGraphClientOps` | Follow/block/mute relationships |
| `GizmoduckClient` | User profiles (roles, follower count, subscription) |

### Safety
| Client | Purpose |
|--------|---------|
| `SafetyLabelStoreClient` | Post-level safety labels |
| `VisibilityFilteringClient` | VF decisions (delete/spam/violence/gore) |
| `VfClient` | Safety labels for VF |

### User Signals
| Client | Purpose |
|--------|---------|
| `UserDemographicsClient` | User demographic data |
| `UserInferredGenderStoreClient` | Inferred gender store |
| `GenderPredictionGrpcClient` | Gender prediction gRPC |
| `ImpressedPostsClient` | Posts user has seen before |
| `ImpressionBloomFilterClient` | Bloom filter for dedup |

### Content/Topics
| Client | Purpose |
|--------|---------|
| `FollowedGrokTopicsStoreClient` | User's followed Grok topics |
| `FollowedStarterPacksStoreClient` | User's followed starter packs |
| `TweetMixerClient` | Tweet content mixing |
| `ThunderClient` | In-network post retrieval |
| `GeoIpLocationClient` | IP-based location |

### Infrastructure
| Client | Purpose |
|--------|---------|
| `RedisClient` (ATLA) | Request cache — Atlanta datacenter |
| `RedisClient` (PDXA) | Request cache — Portland datacenter |
| `KafkaPublisherClient` (phoenix) | Phoenix event publishing |
| `KafkaPublisherClient` (reranking) | Reranking event publishing |

---

## 12 Candidate Sources

From `home-mixer/sources/`:

| Source | File | Purpose |
|--------|------|---------|
| **Thunder** | `thunder_source.rs` | In-network posts from followed accounts |
| **Phoenix** | `phoenix_source.rs` | Out-of-network ML retrieval |
| **Phoenix Topics** | `phoenix_topics_source.rs` | Topic-based retrieval |
| **Phoenix MoE** | `phoenix_moe_source.rs` | Mixture-of-experts retrieval |
| **Ads** | `ads_source.rs` | Advertisement candidates |
| **Who to Follow** | `who_to_follow_source.rs` | Follow suggestions |
| **Prompts** | `prompts_source.rs` | Engagement prompts |
| **Push to Home** | `push_to_home_source.rs` | Push notification posts |
| **Tweet Mixer** | `tweet_mixer_source.rs` | Mixed tweet sources |
| **Scored Posts** | `scored_posts_source.rs` | Pre-scored posts cache |
| **Cached Posts** | `cached_posts_source.rs` | Request-level cache |

---

## Feature Switch System

The `QueryBuilder` evaluates feature switches for every request:

```rust
fn evaluate_feature_switches(&self, proto_query, user_roles, has_phone_number, fs_overrides) {
    let recipient = RecipientBuilder::new()
        .user_id(proto_query.viewer_id)
        .country(&proto_query.country_code)
        .language(&proto_query.language_code)
        .client_app_id(proto_query.client_app_id)
        .custom_string("datacenter", &self.datacenter)
        .custom_i64("account_age_days", days_since_creation(proto_query.viewer_id))
        .custom_bool("has_phone_number", has_phone_number)
        .user_roles(user_roles);

    let results = self.feature_switches.match_recipient(&recipient.build());

    // Debug overrides can be applied per-request
    for (key, value) in fs_overrides {
        results.override_fs(key, value);
    }
}
```

Feature switches can target by:
- User ID, country, language
- Client app ID, datacenter
- Account age, phone number status
- User roles (verified, staff, etc.)

---

## Ads Blending

### Safe Gap Blender (`ads/safe_gap_blender.rs`)

Ads are inserted at **safe gaps** — positions between organic posts where ad placement is appropriate (avoiding sensitive content boundaries).

```rust
fn blend_impl(scored_posts: Vec<ScoredPost>, ads: Vec<AdIndexInfo>, min_posts: usize) -> Vec<FeedItem> {
    // 1. Find safe gaps between organic posts
    let safe_gaps = find_safe_gaps(&scored_posts);

    // 2. Compute spacing based on ad metadata
    let spacing = compute_spacing(&ads);

    // 3. Assign each ad to the nearest safe gap
    let placements = assign_ads_to_gaps(&safe_gaps, ads.len(), &spacing, first_ideal);

    // 4. Interleave organic posts and ads
    interleave_and_finalize(scored_posts, ads, &placements)
}
```

The `find_best_gap()` function uses binary search (`partition_point`) to find the gap closest to the ideal position while respecting minimum spacing:

```rust
fn find_best_gap(gaps: &[usize], ideal: usize, min: usize) -> Option<(usize, usize)> {
    let min_offset = gaps.partition_point(|&g| g < min);
    let candidates = &gaps[min_offset..];
    let ideal_pos = candidates.partition_point(|&g| g < ideal);
    // Pick the gap closest to ideal (ties broken towards earlier position)
}
```

---

## URT (Unified Rich Timelines)

The `get_for_you_feed_urt` endpoint returns feeds in X's internal timeline format with cursor support:

```rust
// Cursor handling for pagination
if !cursor_str.is_empty() {
    match cursor_utils::decode_ordered_cursor(&cursor_str) {
        Ok(Some(c)) => {
            query.is_bottom_request = c.cursor_type == Some(CursorType::BOTTOM);
            query.is_top_request = c.cursor_type == Some(CursorType::TOP);
            query.cursor = Some(c);
        }
        ...
    }
}
```

---

## Next Chapter

→ [Chapter 5: Thunder — In-Network Post Store](./05-thunder.md)
