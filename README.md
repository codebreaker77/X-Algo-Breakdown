<div align="center">
  <h1>X Algorithm Breakdown</h1>
  <p><strong>Deep Technical Analysis of the X "For You" Feed</strong></p>

  <p>
    <a href="https://github.com/codebreaker77/X-Algo-Breakdown/stargazers"><img src="https://img.shields.io/github/stars/codebreaker77/X-Algo-Breakdown?style=for-the-badge&color=2b2b2b" alt="Stars" /></a>
    <a href="https://github.com/codebreaker77/X-Algo-Breakdown/network/members"><img src="https://img.shields.io/github/forks/codebreaker77/X-Algo-Breakdown?style=for-the-badge&color=2b2b2b" alt="Forks" /></a>
    <a href="https://github.com/codebreaker77/X-Algo-Breakdown/blob/master/LICENSE"><img src="https://img.shields.io/github/license/codebreaker77/X-Algo-Breakdown?style=for-the-badge&color=2b2b2b" alt="License" /></a>
    <br/><br/>
    <a href="https://ko-fi.com/U7U31YJNBN"><img src="https://ko-fi.com/img/githubbutton_sm.svg" alt="ko-fi" /></a>
  </p>
</div>

---

## What Is This?

This repository is an **in-depth, source-code-level breakdown** of X's "For You" feed recommendation algorithm released on May 15, 2026. The original codebase spans **207 files** across Rust and Python, implementing a full production recommendation pipeline from candidate retrieval through feed assembly.

This is NOT a surface-level summary. Every chapter traces through real function signatures, actual data flow, scoring weights, and architectural decisions directly from the source code.

---

## Table of Contents

| Chapter | Description |
|---------|-------------|
| [01: Architecture Overview](./01-architecture-overview.md) | End-to-end system map, the 4 layers, data flow diagram |
| [02: Phoenix ML Brain](./02-phoenix-ml-engine.md) | Two-tower retrieval, transformer ranker, candidate isolation, hash embeddings |
| [03: Candidate Pipeline](./03-candidate-pipeline.md) | Hydrators, scorers, filters, selectors within the Rust trait-object architecture |
| [04: Home Mixer](./04-home-mixer.md) | gRPC server, 28+ service clients, feature switches, ads blending, URT format |
| [05: Thunder](./05-thunder.md) | Kafka ingestion, in-memory store, sub-millisecond lookups, recency scoring |
| [06: Scoring & Ranking](./06-scoring-and-ranking.md) | Weighted scorer formula, 19 engagement actions, author diversity, OON factor |
| [07: Grox AI Engine](./07-grox-ai-engine.md) | Engine/Dispatcher multiprocess architecture, Kafka streams, multimodal embeddings |
| [08: Safety & Moderation](./08-safety-and-moderation.md) | PTOS classifier, policy violation detection, spam, banger quality, reply ranking |
| [09: What's New (May 2026)](./09-whats-new-may-2026.md) | Every new feature in this release versus the original 2023 algorithm |
| [10: Key Design Patterns](./10-design-patterns.md) | Enable-gating, hash embeddings, bloom filter dedup, LLM-as-classifier |

---

## Cheat Sheets & Playbooks

Derived strictly from mathematical multipliers and LLM scoring logic in the source code.

| Playbook | Description |
|----------|-------------|
| [The "Going Viral" Cheat Sheet](./cheat_sheet/going-viral-guide.md) | Actionable advice derived directly from the ranking code |
| [Video Reach Optimization Playbook](./cheat_sheet/video-reach-playbook.md) | Maximizing VQV weights and subtitle embeddings |
| [Safety & Shadowban Survival Guide](./cheat_sheet/safety-shadowban-guide.md) | Avoiding PTOS policy flags, slop scores, and spam filters |
| [Reply Ranking Playbook](./cheat_sheet/reply-ranking-playbook.md) | Mastering the VLM reply scorer for conversation dominance |
| [Network Reach Playbook](./cheat_sheet/network-reach-playbook.md) | Understanding the OON penalty and Thunder in-network advantage |
| [Banger & Quality Scoring Playbook](./cheat_sheet/banger-quality-playbook.md) | Crossing the 0.4 quality threshold for viral distribution |

---

## Codebase Stats

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

```text
User Request
     |
     v
+-------------------------------------------------------------+
|  1. PHOENIX RETRIEVAL (Two-Tower ANN search)                |
|     Millions of posts -> Top ~1000 candidates               |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|  2. PHOENIX RANKING (Grok-1 Transformer)                    |
|     Predicts P(like), P(reply), P(repost)... 19 actions     |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|  3. CANDIDATE PIPELINE (Rust - hydrate -> score -> filter)  |
|     Feature enrichment, weighted scoring, diversity         |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|  4. HOME MIXER (Rust - blend organic + ads, serve)          |
|     Final feed assembly -> gRPC response                    |
+-------------------------------------------------------------+

       v Meanwhile, running continuously in background v

+-------------------------------------------------------------+
|  GROX (Python - real-time AI classifiers & embedders)       |
|  Safety labels, quality scores, multimodal embeddings       |
+-------------------------------------------------------------+
```

---

## The Single Most Important Thing

**X eliminated all hand-engineered features.** The Grok-1 transformer does everything. There are no manual content relevance features and no heuristic scoring rules. Your engagement history (likes, replies, shares, dwell time) is the input; the transformer learns what is relevant to you.

The weighted scoring formula:

```text
Final Score = SUM (weight_i * P(action_i))
```

Positive actions (like, share, repost) contribute positively and negative actions (block, mute, report) pull the score down. The exact weights are tunable via feature switches for experiments and testing.

---

## Support the Project

If you found this deep dive useful and it helped you understand modern recommendation systems, consider supporting my work! It takes dozens of hours to read through hundreds of raw source code files and synthesize them into clean architectural documentation.
[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/U7U31YJNBN)

---

## License

This analysis is provided for educational and research purposes. The original X algorithm code is licensed under Apache License 2.0.
