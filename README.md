<div align="center">
  <h1>X Algorithm Breakdown</h1>
  <p><strong>A Deep-Dive Technical Analysis of the May 2026 X "For You" Recommendation Engine</strong></p>

  <p>
    <a href="https://github.com/codebreaker77/X-Algo-Breakdown/stargazers"><img src="https://img.shields.io/github/stars/codebreaker77/X-Algo-Breakdown?style=for-the-badge&color=2b2b2b" alt="Stars" /></a>
    <a href="https://github.com/codebreaker77/X-Algo-Breakdown/network/members"><img src="https://img.shields.io/github/forks/codebreaker77/X-Algo-Breakdown?style=for-the-badge&color=2b2b2b" alt="Forks" /></a>
    <a href="https://github.com/codebreaker77/X-Algo-Breakdown/blob/master/LICENSE"><img src="https://img.shields.io/github/license/codebreaker77/X-Algo-Breakdown?style=for-the-badge&color=2b2b2b" alt="License" /></a>
    <br/><br/>
    <a href="https://ko-fi.com/U7U31YJNBN"><img src="https://ko-fi.com/img/githubbutton_sm.svg" alt="ko-fi" /></a>
  </p>
</div>

---

## Executive Summary

This repository provides an **in-depth, source-code-level architectural breakdown** of the X "For You" recommendation algorithm, based on the codebase released on May 15, 2026. 

Unlike previous iterations that relied heavily on heuristic, rule-based systems, this release reveals a fundamentally modernized architecture. The system now leverages the **Grok-1 transformer** for 19-dimensional engagement prediction, a sub-millisecond in-memory post store (**Thunder**), and a distributed async Python daemon (**Grox**) for real-time, LLM-powered content moderation and multimodal embedding extraction.

This documentation serves as a comprehensive guide for systems engineers, ML researchers, and product builders aiming to understand how one of the world's largest recommendation feeds is engineered at scale.

---

## Core System Architecture

The recommendation engine is orchestrated across four primary subsystems:

1. **Phoenix ML Engine:** A two-tower retrieval and transformer-based ranking system. It utilizes a novel "candidate isolation" attention mask to independently evaluate posts against a user's historical interaction sequence.
2. **The Rust Candidate Pipeline:** A highly concurrent, trait-object-driven pipeline that dynamically hydrates, scores, and filters candidates at runtime.
3. **Home Mixer (gRPC):** The central nervous system of the feed. It manages feature flags, coordinates over 28 external service clients, blends organic posts with advertisements via the SafeGap blender, and serves the final URT payload.
4. **Grox AI Daemon:** A Kafka-driven, multi-process Python engine that continuously processes incoming posts, running Vision-Language Models (VLMs) to detect policy violations, assign quality scores, and extract text/video embeddings.

---

## Technical Chapters

The breakdown is structured into 10 detailed chapters, tracing the execution path directly through the original Rust and Python source code.

| Section | Analysis Link | Core Concepts Covered |
|---------|---------------|-----------------------|
| **01** | [Architecture Overview](./01-architecture-overview.md) | End-to-end request lifecycle, microservice topography, and data flow. |
| **02** | [Phoenix ML Brain](./02-phoenix-ml-engine.md) | Two-tower retrieval, transformer ranking, and multi-hash embeddings. |
| **03** | [Candidate Pipeline](./03-candidate-pipeline.md) | Dynamic trait-object orchestration (Hydrators, Filters, Scorers). |
| **04** | [Home Mixer Orchestration](./04-home-mixer.md) | gRPC serving, feature-flag targeting, and the SafeGap ad-blending algorithm. |
| **05** | [Thunder Post Store](./05-thunder.md) | Sub-millisecond in-memory storage, Kafka ingestion, and back-pressure. |
| **06** | [Scoring & Ranking Math](./06-scoring-and-ranking.md) | The 19-dimensional weighted formula, author diversity decay, and OON penalties. |
| **07** | [Grox AI Engine](./07-grox-ai-engine.md) | Multiprocess event loops, `asyncio` dispatcher, and multimodal frame extraction. |
| **08** | [Safety & Content Moderation](./08-safety-and-moderation.md) | LLM-based policy classifiers, "Deluxe" reasoning modes, and reply quality ranking. |
| **09** | [What's New (May 2026)](./09-whats-new-may-2026.md) | Architectural diffs and deprecations compared to the 2023 algorithm release. |
| **10** | [Key Design Patterns](./10-design-patterns.md) | Systems engineering patterns: bloom filters, multi-region caching, and enable-gating. |

---

## Algorithmic Playbooks

Derived strictly from the source code's mathematical multipliers and LLM classifier prompts, these playbooks translate the backend architecture into actionable strategies for content optimization.

| Playbook | Tactical Focus |
|----------|----------------|
| [The Core Growth Guide](./cheat_sheet/going-viral-guide.md) | Exploiting the 19 engagement multipliers and navigating the exponential diversity penalty. |
| [Video Reach Optimization](./cheat_sheet/video-reach-playbook.md) | Passing the `MIN_VIDEO_DURATION_MS` check and optimizing subtitles for VLM extraction. |
| [Safety & Visibility Filtering](./cheat_sheet/safety-shadowban-guide.md) | Bypassing the 7 PTOS safety flags and avoiding the algorithmic `slop_score`. |
| [Reply Ranking Mastery](./cheat_sheet/reply-ranking-playbook.md) | Dominating conversation threads using the `ReplyScorer` VLM logic. |
| [Network Reach Strategy](./cheat_sheet/network-reach-playbook.md) | Leveraging Thunder's recency sort to overcome the Out-of-Network (`OON_WEIGHT_FACTOR`) penalty. |
| [Banger Quality Scoring](./cheat_sheet/banger-quality-playbook.md) | Clearing the `0.4` VLM quality threshold for priority algorithmic distribution. |

---

## Codebase Statistics

| Component | Metric |
|-----------|--------|
| **Total Source Files** | 207 (139 Rust, 68 Python) |
| **Microservice Clients** | 28+ Integrated External Services |
| **Engagement Vectors** | 19 Distinct Predictive Action Weights |
| **Candidate Sources** | 12 Routing Paths (In-Network, Topics, MoE, Ads, etc.) |
| **Filter Stages** | 18 Specialized Post-Retrieval Filters |
| **AI Classifiers** | 6 Continuous Grox VLM/EAPI Classifiers |

---

## Support & Attribution

This documentation requires extensive source code analysis and system mapping. If you found this breakdown valuable for your research or engineering work, consider supporting the repository.

<a href="https://ko-fi.com/U7U31YJNBN"><img src="https://ko-fi.com/img/githubbutton_sm.svg" alt="ko-fi" /></a>

**License:** This analytical documentation is provided for educational and research purposes. The original X algorithm source code is licensed under the Apache License 2.0.
