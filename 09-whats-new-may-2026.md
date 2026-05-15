# Chapter 9: What's New — May 15, 2026 Release

> A detailed comparison of what changed from the original 2023 algorithm release to this update

---

## Context

X originally open-sourced a partial algorithm in March 2023. That release was limited — it showed some of the ranking logic but not the ML models, not the full serving stack, and none of the content understanding pipeline.

This **May 15, 2026 release** is a fundamentally different and far more comprehensive disclosure. Here's everything new.

---

## 1. 🧠 Grok-1 Replaces All Feature Engineering

**Before (2023):** The ranking model used hand-engineered features — follower counts, engagement rates, content type signals, etc. The ML model was a relatively simple classifier.

**Now (2026):** The **Grok-1 transformer** (xAI's open-source LLM architecture) has been adapted for recommendations. It processes raw user engagement sequences directly — no feature engineering at all.

> "We have eliminated every single hand-engineered feature and most heuristics from the system."
> — README.md

This is a paradigm shift. The transformer learns its own feature representations from the raw sequence of your interactions.

---

## 2. 🔍 Two-Tower Retrieval (Brand New)

**Before (2023):** The retrieval mechanism was not disclosed.

**Now (2026):** Full two-tower architecture with:
- **User Tower:** Grok transformer encodes engagement history → user vector
- **Candidate Tower:** MLP/mean-pool projects post+author → candidate vector
- **ANN Search:** Dot product similarity on L2-normalized vectors
- **Hash Embeddings:** Multi-hash (2 per entity) instead of lookup tables

### Open-Source Artifacts
- Pre-trained mini model (128-dim, 4 layers, 4 heads)
- 537K sports corpus for demo retrieval
- Example user action history

---

## 3. 🤖 Grox Content Understanding Pipeline (Entirely New)

**Before (2023):** Content understanding was not disclosed.

**Now (2026):** Complete Grox subsystem with:

| Component | What It Does |
|-----------|-------------|
| `Engine` + `Dispatcher` | Multiprocess async task execution system |
| `SafetyPtosCategoryClassifier` | LLM-based safety categorization |
| `SafetyPtosPolicyClassifier` | Per-policy violation detection (7 policies) |
| `BangerInitialScreenClassifier` | Quality/engagement potential scoring |
| `SpamEapiLowFollowerClassifier` | Spam detection |
| `ReplyScorer` | Reply quality ranking |
| `MultimodalPostEmbedderV2` | Text+image+video embedding (5 models) |
| 16 Task Generator types | Kafka-sourced task pipelines |

### Key Insights:
- **LLM-as-classifier:** Safety decisions are made by Grok vision models, not rule-based systems
- **Deluxe mode:** Enhanced reasoning with chain-of-thought for ambiguous cases
- **Slop detection:** The banger classifier includes `slop_score` — AI-generated content detection
- **Minor detection:** `has_minor_score` for child safety

---

## 4. 📺 Ads Blending System (New)

**Before (2023):** Ads integration was not disclosed.

**Now (2026):** Full ads module including:
- `SafeGapAdsBlender` — inserts ads at safe positions between organic posts
- `PartitionOrganicBlender` — alternative blending strategy
- Brand safety tracking that respects sensitive content boundaries
- Configurable spacing with `min` and `requested` gap sizes

---

## 5. 🔧 Expanded Query Hydration (New)

**Before (2023):** Limited user context.

**Now (2026):** Rich query hydration including:
- Followed Grok topics
- Followed starter packs
- Impression bloom filters (dedup)
- IP-based geolocation
- Mutual follow graphs
- Served history
- Inferred gender
- User demographics

---

## 6. 📊 Expanded Candidate Hydration (New)

**Before (2023):** Basic post data.

**Now (2026):** Extensive candidate enrichment:
- Engagement counts
- Brand safety signals
- Language codes
- Media detection (video duration, photo count)
- Quote post expansion
- Mutual follow scores
- Subscription status

---

## 7. 🗂️ 12 Candidate Sources (Expanded)

**Before (2023):** 2 sources (in-network, out-of-network).

**Now (2026):** 12 sources:

| New Source | Purpose |
|-----------|---------|
| Phoenix MoE | Mixture-of-experts retrieval |
| Phoenix Topics | Topic-based retrieval |
| Ads | Ad candidates |
| Who to Follow | Follow suggestions |
| Prompts | Engagement prompts |
| Push to Home | Push notification driven posts |
| Tweet Mixer | Mixed content sources |
| Cached Posts | Request-level cache |
| Scored Posts | Pre-scored cache |

---

## 8. ⚡ Thunder In-Memory Store (Detailed)

**Before (2023):** Brief mention of in-network candidate sourcing.

**Now (2026):** Full Thunder implementation:
- Kafka-based ingestion pipeline
- Per-user in-memory stores (original, replies/reposts, video)
- Semaphore-based back-pressure
- Rich metrics (freshness, reply ratio, author diversity, posts per author)
- `spawn_blocking` for async-safe memory access

---

## 9. 🎯 Enhanced Scoring (19 Actions)

**Before (2023):** ~8 engagement types.

**Now (2026):** 19 engagement predictions:

| New in 2026 | Action |
|-------------|--------|
| ✨ | `share_via_dm` — share via direct message |
| ✨ | `share_via_copy_link` — copy link to share |
| ✨ | `dwell_time` — continuous dwell time prediction |
| ✨ | `quoted_click` — click through quote-tweets |
| ✨ | `follow_author` — follow the post's author |

---

## 10. 🏗️ End-to-End Inference Pipeline (New)

**Before (2023):** No runnable code.

**Now (2026):** `run_pipeline.py` provides a single entry point that runs **retrieval → ranking** from exported checkpoints:

```bash
uv run run_pipeline.py --artifacts_dir artifacts/oss-phoenix-artifacts
```

This mirrors production composition: the same two stages (retrieval then ranking) are composed in a single script.

---

## 11. 🎭 Candidate Isolation (Newly Disclosed)

**Before (2023):** Attention masking not disclosed.

**Now (2026):** The critical **candidate isolation** attention mask is fully documented:
- Candidates can attend to user + history
- Candidates **cannot** attend to each other
- This ensures score consistency and cacheability

---

## 12. 📡 Multimodal Understanding (New)

**Before (2023):** Text-only.

**Now (2026):** Full multimodal pipeline:
- 5 embedding models (Qwen3 0.6B, Qwen3 8B, RecSys V4, default, video)
- Video frame sampling with subtitle alignment
- ASR (audio speech recognition) processing
- Grok-generated post summaries
- Media description hydration

---

## Summary: 2023 vs 2026

| Aspect | 2023 | 2026 |
|--------|------|------|
| **ML Model** | Feature-engineered classifier | Grok-1 transformer (no features) |
| **Retrieval** | Not disclosed | Two-tower ANN with hash embeddings |
| **Ranking actions** | ~8 | 19 |
| **Content understanding** | None | Full Grox pipeline (LLM classifiers) |
| **Ads** | None | SafeGap + Partition blenders |
| **Safety** | Rule-based | LLM-powered (7 policy categories) |
| **Sources** | 2 | 12 |
| **Filters** | ~5 | 18 |
| **Embedding** | None | 5-model multimodal pipeline |
| **Runnable code** | No | Yes (full pipeline + artifacts) |
| **Model weights** | No | Yes (3GB mini checkpoint) |
| **Candidate isolation** | Not shown | Fully documented |

---

## Next Chapter

→ [Chapter 10: Key Design Patterns](./10-design-patterns.md)
