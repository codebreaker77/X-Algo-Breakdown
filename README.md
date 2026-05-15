# X Algorithm Breakdown — Deep Technical Analysis

> **Source:** [xai-org/x-algorithm](https://github.com/xai-org/x-algorithm) · May 15, 2026 Release  
> **Analysis by:** [@codebreaker77](https://github.com/codebreaker77)

---

## What Is This?

This repository is an **in-depth, source-code-level breakdown** of X's "For You" feed recommendation algorithm — released on May 15, 2026. The original codebase spans **207 files** across Rust and Python, implementing a full production recommendation pipeline from candidate retrieval through feed assembly.

This is NOT a surface-level summary. Every chapter traces through real function signatures, actual data flow, scoring weights, and architectural decisions as found in the source code.

---

## 🗂️ Table of Contents

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 1 | [Architecture Overview](./01-architecture-overview.md) | End-to-end system map, the 4 layers, data flow diagram |
| 2 | [Phoenix — The Grok-Based ML Brain](./02-phoenix-ml-engine.md) | Two-tower retrieval, transformer ranker, candidate isolation, hash embeddings |
| 3 | [Candidate Pipeline — The Rust Engine](./03-candidate-pipeline.md) | Hydrators, scorers, filters, selectors — trait-object pipeline architecture |
| 4 | [Home Mixer — Feed Assembly & Serving](./04-home-mixer.md) | gRPC server, 28+ clients, feature switches, ads blending, URT format |
| 5 | [Thunder — In-Network Post Store](./05-thunder.md) | Kafka ingestion, in-memory store, sub-ms lookups, recency scoring |
| 6 | [Scoring & Ranking — How Posts Get Their Score](./06-scoring-and-ranking.md) | Weighted scorer formula, 19 engagement actions, author diversity, OON factor |
| 7 | [Grox — Real-Time AI Engine](./07-grox-ai-engine.md) | Engine/Dispatcher architecture, Kafka streams, LLM classifiers, multimodal embeddings |
| 8 | [Safety & Content Moderation](./08-safety-and-moderation.md) | PTOS classifier, policy violation detection, spam, banger quality, reply ranking |
| 9 | [What's New — May 2026 Release](./09-whats-new-may-2026.md) | Every new feature in this release vs the original 2023 algorithm |
| 10 | [Key Design Patterns](./10-design-patterns.md) | Enable-gating, hash embeddings, bloom filter dedup, LLM-as-classifier |

---

## 🏗️ Codebase Stats

| Metric | Value |
|--------|-------|
| Total Files | 207 |
| Rust Files | 139 |
| Python Files | 68 |
| Knowledge Graph Nodes | 1,418 |
| Knowledge Graph Edges | 5,365 |
| Engagement Action Types | 19 |
| External Service Clients | 28+ |
| Content Classifiers | 6 |
| Filter Stages | 18 |
| Candidate Sources | 12 |

---

## Quick Architecture

```
User Request
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│  1. PHOENIX RETRIEVAL (Two-Tower ANN search)               │
│     Millions of posts → Top ~1000 candidates               │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  2. PHOENIX RANKING (Grok-1 Transformer)                   │
│     Predicts P(like), P(reply), P(repost)... 19 actions    │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  3. CANDIDATE PIPELINE (Rust — hydrate → score → filter)   │
│     Feature enrichment, weighted scoring, diversity        │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  4. HOME MIXER (Rust — blend organic + ads, serve)         │
│     Final feed assembly → gRPC response                    │
└─────────────────────────────────────────────────────────────┘

       ⬇️ Meanwhile, running continuously in background ⬇️

┌─────────────────────────────────────────────────────────────┐
│  GROX (Python — real-time AI classifiers & embedders)      │
│  Safety labels, quality scores, multimodal embeddings      │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 The Single Most Important Thing

**X eliminated all hand-engineered features.** The Grok-1 transformer does everything — no manual content relevance features, no heuristic scoring rules. Your engagement history (likes, replies, shares, dwell time) is the input; the transformer learns what's relevant to you.

The weighted scoring formula:

```
Final Score = Σ (weight_i × P(action_i))
```

Where positive actions (like, share, repost) contribute positively and negative actions (block, mute, report) pull the score down. The exact weights are tunable via feature switches for A/B testing.

---

## License

This analysis is provided for educational and research purposes. The original X algorithm code is licensed under Apache License 2.0.
