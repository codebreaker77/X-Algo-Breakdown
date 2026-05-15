# Chapter 1: Architecture Overview

> How X's For You feed is assembled — from user request to ranked timeline

---

## The Big Picture

When you open X and scroll through your For You feed, the system behind that simple action involves **5 distinct services** spanning two languages, 207 source files, and dozens of external microservice dependencies. Here's the full end-to-end picture.

---

## System Map

```
                         ┌────────────────────────┐
                         │     Mobile / Web App    │
                         │      (User opens X)     │
                         └────────────┬───────────┘
                                      │ gRPC Request
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HOME MIXER (Rust gRPC)                           │
│                          Orchestration Layer                                │
│                                                                             │
│  1. Query Hydration ────────┐                                              │
│     • User engagement hist  │                                              │
│     • Following list        │                                              │
│     • Muted keywords        │                                              │
│     • Impression bloom      │                                              │
│                             ▼                                              │
│  2. Candidate Sourcing ─────┤                                              │
│     ┌──────────────────┐    │    ┌────────────────────────────┐            │
│     │     THUNDER      │    │    │   PHOENIX RETRIEVAL        │            │
│     │ (In-Network)     │    │    │   (Out-of-Network)         │            │
│     │ Sub-ms in-memory │    │    │   Two-tower ANN search     │            │
│     │ Kafka-ingested   │    │    │   against 537K+ corpus     │            │
│     └──────────────────┘    │    └────────────────────────────┘            │
│                             ▼                                              │
│  3. CANDIDATE PIPELINE (Rust)                                              │
│     hydrate → filter → score (Phoenix ML) → weighted_score → diversity    │
│                             │                                              │
│  4. Selection ──────────────┤                                              │
│     Sort by score, top K    │                                              │
│                             ▼                                              │
│  5. Post-Selection ─────────┤                                              │
│     Visibility filtering    │                                              │
│     Ads blending            │                                              │
│                             ▼                                              │
│  6. Side Effects ───────────┘                                              │
│     Kafka publish, cache    │                                              │
│                             ▼                                              │
│                     Ranked Feed Response                                   │
└─────────────────────────────────────────────────────────────────────────────┘

       ⬇ Meanwhile, running continuously in parallel ⬇

┌─────────────────────────────────────────────────────────────────────────────┐
│                         GROX (Python Async)                                 │
│                                                                             │
│  Kafka ──▶ Dispatcher ──▶ Engine ──▶ PlanMaster                           │
│                                         │                                  │
│  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────────┐   │
│  │ LLM Classifiers  │   │ Multimodal       │   │ Reply Ranking        │   │
│  │ • Safety PTOS    │   │ Embedder V2      │   │ • Grok VLM scoring   │   │
│  │ • Spam           │   │ • Qwen3 / v4     │   │                      │   │
│  │ • Banger Screen  │   │ • Text/Image/Vid │   │                      │   │
│  └──────────────────┘   └──────────────────┘   └──────────────────────┘   │
│                                                                             │
│  Results ──▶ Kafka ──▶ Feature Store ──▶ Used by Home Mixer at serve time  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The Five Services

### 1. Home Mixer (Rust)
**Role:** The **orchestration brain** — receives gRPC requests, assembles the complete pipeline, returns the ranked feed.

- **Location:** `home-mixer/`
- **Language:** Rust (tonic gRPC)
- **Key file:** `server.rs` — the `HomeMixerServer` that implements `XService`
- **Endpoints:**
  - `get_scored_posts` — raw ranked post IDs + scores
  - `get_debug_scored_posts` — same, with debug JSON for pipeline introspection
  - `get_for_you_feed` — proto-format feed items
  - `get_for_you_feed_urt` — URT (Unified Rich Timelines) format with cursors

### 2. Thunder (Rust)
**Role:** **In-network post store** — sub-millisecond lookups for posts from accounts you follow.

- **Location:** `thunder/`
- **How it works:** Consumes Kafka post create/delete events, maintains per-user in-memory stores, serves `GetInNetworkPosts` via gRPC
- **Key:** No database — purely in-memory with Kafka replay for durability

### 3. Phoenix (Python / JAX)
**Role:** The **ML engine** — retrieval (two-tower ANN) and ranking (Grok-1 transformer).

- **Location:** `phoenix/`
- **Two sub-models:**
  - Retrieval: user tower + candidate tower → dot product → top-K
  - Ranking: transformer with candidate isolation → per-action engagement probabilities

### 4. Candidate Pipeline (Rust)
**Role:** **Composable pipeline framework** — the traits and execution engine that Home Mixer uses.

- **Location:** `candidate-pipeline/`
- **Traits:** `Source`, `Hydrator`, `QueryHydrator`, `Filter`, `Scorer`, `Selector`, `SideEffect`
- Used by Home Mixer to compose its specific For You pipeline

### 5. Grox (Python)
**Role:** **Real-time AI processing** — continuously classifies, embeds, and labels new content.

- **Location:** `grox/`
- Runs as a separate daemon (Engine + Dispatcher + gRPC server)
- Results flow to Kafka → consumed by serving stack

---

## Language Split

| Language | Files | Role |
|----------|-------|------|
| **Rust** | 139 | All serving-path code (Home Mixer, Thunder, Candidate Pipeline) |
| **Python** | 68 | All ML code (Phoenix models, Grox classifiers/embedders) |

The split is deliberate: Rust for latency-critical serving, Python for ML flexibility.

---

## Request Lifecycle (Detailed)

Here's what happens when you open your For You feed, step by step:

### Step 1: gRPC Request Arrives
`HomeMixerServer::build()` in `server.rs` receives the request. The `QueryBuilder` extracts:
- Viewer ID, country code, language code
- Device info (IP, user agent, timezone)
- Previously seen/served post IDs
- Feature switch overrides

### Step 2: Feature Switch Evaluation
`evaluate_feature_switches()` builds a `RecipientBuilder` with user properties and evaluates all active experiments. This is the A/B testing layer — every pipeline component can be enabled/disabled via feature switches.

### Step 3: Viewer Data Fetch
`fetch_viewer_data()` calls the Gizmoduck client (200ms timeout) to get:
- User roles (e.g., verified, staff)
- Muted keywords
- Follower count
- Subscription level
- `allow_for_you_recommendations` preference
- Age

### Step 4: Candidate Sourcing (Parallel)
Multiple sources fire in parallel:
- **Thunder Source** — in-network posts from followed accounts
- **Phoenix Source** — out-of-network ML-retrieved posts
- **Phoenix Topics Source** — topic-based retrieval
- **Phoenix MoE Source** — mixture-of-experts retrieval
- **Ads Source** — ad candidates
- **Who to Follow Source** — follow suggestions
- **Prompts Source** — engagement prompts
- **Push to Home Source** — push notification driven posts

### Step 5: Candidate Hydration
Each candidate gets enriched with data from external services:
- Core post metadata (text, media, timestamps)
- Author info (from Gizmoduck)
- Engagement counts
- Brand safety signals
- Language codes, media detection
- Mutual follow scores

### Step 6: Pre-Scoring Filters (18 filters)
Filter out ineligible candidates before spending ML compute:
- Duplicates, old posts, self-posts
- Blocked/muted authors
- Muted keywords
- Previously seen/served posts
- Ineligible subscription content

### Step 7: ML Scoring (Phoenix Scorer)
Send surviving candidates to the Phoenix prediction service. The Grok-based transformer returns probabilities for 19 engagement actions.

### Step 8: Weighted Scoring
Combine ML predictions into a single score using configured weights (see Chapter 6 for the exact formula).

### Step 9: Author Diversity
Apply exponential decay to repeated authors so no single person dominates your feed.

### Step 10: Selection
Sort by final score, select top K.

### Step 11: Post-Selection Filtering
- Visibility filtering (deleted, spam, violence, gore)
- Conversation deduplication

### Step 12: Ads Blending
Interleave ad candidates at safe gaps using the `SafeGapAdsBlender`.

### Step 13: Side Effects
Publish cache entries and client events to Kafka for future use.

### Step 14: Response
Return the ranked feed as either proto or URT format.

---

## Next Chapter

→ [Chapter 2: Phoenix — The Grok-Based ML Brain](./02-phoenix-ml-engine.md)
