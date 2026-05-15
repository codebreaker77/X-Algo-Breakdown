# Chapter 10: Key Design Patterns

> Recurring architectural patterns found throughout the X algorithm codebase

---

## 1. Enable-Gating (Feature Switches)

**Found in:** Every hydrator, filter, scorer, side-effect, and source.

Every pipeline component implements an `enable()` method:

```rust
fn enable(&self, query: &ScoredPostsQuery) -> bool {
    query.params.get(SomeFeatureSwitch)
}
```

This means any component can be turned on/off per-user via feature switches, enabling:
- **A/B testing** — roll out new filters/scorers to % of users
- **Kill switches** — instantly disable a broken component
- **Targeted experiments** — enable by country, language, account age, etc.
- **Debug overrides** — force-enable components for specific requests

The `RecipientBuilder` in `server.rs` shows the targeting dimensions:
- User ID, country, language
- Client app ID, datacenter
- Account age (days since creation)
- Phone number status
- User roles (verified, staff)

---

## 2. Trait-Object Pipeline Composition

**Found in:** `candidate-pipeline/`

All pipeline components are stored as `Box<dyn Trait>` trait objects:

```rust
pub struct PipelineComponents<Q, C> {
    pub sources: Vec<Box<dyn Source<Q, C>>>,
    pub hydrators: Vec<Box<dyn Hydrator<Q, C>>>,
    pub filters: Vec<Box<dyn Filter<Q, C>>>,
    pub scorers: Vec<Box<dyn Scorer<Q, C>>>,
    // ...
}
```

This allows **runtime composition** — the pipeline doesn't know at compile time which specific components will be used. Home Mixer constructs its pipeline by inserting concrete implementations:

```rust
let pipeline = PipelineComponents {
    sources: vec![
        Box::new(ThunderSource::new(...)),
        Box::new(PhoenixSource::new(...)),
        Box::new(AdsSource::new(...)),
    ],
    scorers: vec![
        Box::new(PhoenixScorer { phoenix_client, egress_client }),
        Box::new(WeightedScorer),
        Box::new(AuthorDiversityScorer::default()),
        Box::new(OONScorer),
    ],
    // ...
};
```

---

## 3. Multi-Hash Embeddings

**Found in:** `phoenix/run_pipeline.py`, `phoenix/recsys_retrieval_model.py`

Instead of maintaining a lookup table for every user/post/author ID:

```python
def _hash_ids(ids, scales, biases, modulus, num_buckets):
    """Linear congruential hash: (id * scale + bias) % modulus."""
    for i in range(n):
        for j in range(m):
            raw = (ids[i] * scales[j] + biases[j]) % modulus
            out[i, j] = int((int(raw) % (num_buckets - 1)) + 1)
```

Each entity is hashed with **2 independent hash functions**, and the resulting embeddings are combined. Benefits:
- **Cold-start handling** — new entities immediately get embeddings
- **Fixed memory** — table size is constant regardless of entity count
- **Collision resilience** — multiple hashes make accidental collision unlikely
- **Unified table** — users, items, and authors share one embedding table with offset sections

---

## 4. Bloom Filter Deduplication

**Found in:** `home-mixer/` (ImpressionBloomFilterClient)

The `ImpressionBloomFilterClient` maintains probabilistic sets of posts each user has already seen. This prevents re-serving content without the cost of exact set membership testing.

Used by `PreviouslySeenPostsFilter` and `PreviouslySeenPostsBackupFilter`.

---

## 5. Dual-Region Redis

**Found in:** `phoenix_candidate_pipeline.rs`

```rust
phoenix_request_cache_redis_atla_client: Arc<dyn RedisClient>,  // Atlanta DC
phoenix_request_cache_redis_pdxa_client: Arc<dyn RedisClient>,  // Portland DC
```

Two Redis clusters in different datacenters (ATLA = Atlanta, PDXA = Portland) for:
- Low-latency request caching regardless of user location
- Cross-datacenter redundancy
- Phoenix scoring sequence caching

---

## 6. Kafka as Event Bus + Side Effects

**Found in:** `thunder/`, `grox/`, `home-mixer/side_effects/`

Kafka serves three roles:
1. **Ingestion** — Thunder consumes post create/delete events
2. **Task routing** — Grox dispatcher consumes from 16 Kafka stream types
3. **Side effects** — Pipeline publishes cache entries and client events post-response

```rust
// From home-mixer/side_effects/client_events_kafka_side_effect.rs
fn build_empty_timeline_events(base: &ClientEventParams, post_count: i64) -> Vec<TimelineEvent>
```

---

## 7. LLM-as-Classifier

**Found in:** `grox/classifiers/content/`

Safety and quality decisions are made by prompting Grok vision models with structured output:

```python
# System prompt: detailed policy rules
convo.messages.append(Message(role=Role.SYSTEM, content=[SafetyPtos().render()]))

# User message: rendered post with media
convo.messages.append(user_msg_with_post)

# Assistant prefill: force structured JSON output
convo.messages.append(Message(role=Role.ASSISTANT, content=["<json>"], separator=""))
```

This pattern appears in all 6 classifiers. The LLM generates reasoning followed by `<json>...</json>` structured output.

### Fallback Chains
```
Mini VLM → Primary VLM (if mini fails)
Standard → Deluxe reasoning (for ambiguous cases)
EAPI 4.2 → Standard VLM (if reasoning model unavailable)
```

---

## 8. Multiprocess Async Architecture

**Found in:** `grox/`

Grox uses a **multi-process architecture** with inter-process communication via shared queues:

```python
context = {
    "task_queue": Queue[TaskPayload],       # Dispatcher → Engine
    "resp_queue": Queue[TaskResult],        # Engine → Dispatcher
    "shutdown_event": Event,                # Graceful shutdown
    "queue_connection_shutdown_event": Event # Queue drain signal
}

# Each component runs in its own process
engine_process = Process(target=engine.run, name="grox-engine")
dispatcher_process = Process(target=dispatcher.run, name="grox-dispatcher")
```

Within each process, **asyncio** handles concurrency — multiple LLM calls run concurrently via `asyncio.create_task()`.

---

## 9. Exponential Decay for Diversity

**Found in:** `home-mixer/scorers/author_diversity_scorer.rs`

```rust
fn multiplier(&self, position: usize) -> f64 {
    (1.0 - self.floor) * self.decay_factor.powf(position as f64) + self.floor
}
```

The Nth post from the same author gets multiplied by `decay^N`, asymptotically approaching `floor`. This guarantees diversity without hard caps.

---

## 10. Score Normalization with Negative Offset

**Found in:** `home-mixer/scorers/weighted_scorer.rs`

```rust
fn offset_score(combined_score: f64) -> f64 {
    if combined_score < 0.0 {
        (combined_score + NEGATIVE_WEIGHTS_SUM) / WEIGHTS_SUM * NEGATIVE_SCORES_OFFSET
    } else {
        combined_score + NEGATIVE_SCORES_OFFSET
    }
}
```

Negative signals (block, mute, report predictions) create negative weighted scores. The offset function compresses negatives into `[0, offset]` and shifts positives up by `offset`, ensuring all final scores are non-negative while preserving the signal from negative actions.

---

## 11. Candidate Isolation (Transformer Masking)

**Found in:** `phoenix/recsys_model.py`

The attention mask prevents candidates from seeing each other during transformer inference. This is critical because:
- Score for post A is **independent** of what other posts are in the batch
- Scores can be **cached** and reused across different request compositions
- Inference can be **parallelized** without cross-candidate dependencies

---

## 12. Graceful Shutdown with Drain Period

**Found in:** `grox/main.py`

```python
queue_connection_shutdown_context(context)  # Signal: stop accepting new tasks
await asyncio.sleep(300)                     # 5-minute drain period
shutdown_context(context)                    # Signal: stop processing
await asyncio.gather(
    grpc_server.stop(),
    dispatcher.stop(),
    engine.stop(),
)
```

A **300-second drain period** between stopping new task acceptance and actually shutting down, ensuring all in-flight tasks complete.

---

## Summary

| Pattern | Where | Why |
|---------|-------|-----|
| Enable-gating | All components | A/B testing, kill switches |
| Trait-object pipeline | candidate-pipeline | Runtime composition |
| Multi-hash embeddings | Phoenix | Scale, cold-start |
| Bloom filter dedup | Home Mixer | Efficient "already seen" |
| Dual-region Redis | Home Mixer | Global low-latency |
| Kafka event bus | Everywhere | Decoupled data flow |
| LLM-as-classifier | Grox | Flexible safety |
| Multiprocess async | Grox | Throughput |
| Exponential decay | Diversity scorer | Soft diversity |
| Score normalization | Weighted scorer | Handle negatives |
| Candidate isolation | Phoenix transformer | Cache consistency |
| Graceful shutdown | Grox | Operational safety |
