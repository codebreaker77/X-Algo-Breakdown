# Chapter 13: Grox Execution DAGs

The Grox AI Engine is responsible for running all ML classifications and embedding extractions asynchronously. But how does it guarantee that tasks execute in the right order? (e.g., You cannot extract an embedding from a video until you have run the audio through the speech-to-text model).

Grox solves this using **Directed Acyclic Graphs (DAGs)** defined in `grox/plans/`.

---

## 1. The Plan Master

When a post enters the Grox daemon, it is picked up by `plan_master.py`. The `PlanMaster` does not execute individual tasks; it executes 9 concurrent "Plans" via `asyncio.gather()`.

```python
# From plan_master.py
class PlanMaster:
    ALL_PLANS: list[Plan] = [
        PlanInitialBanger(),
        PlanPostSafety(),
        PlanSpamComment(),
        PlanPostEmbeddingWithSummary(),
        PlanReplyRanking(),
        PlanSafetyPtos(),
        # ...
    ]

    @classmethod
    async def exec(cls, task: TaskPayload):
        results = await asyncio.gather(*[p.execute(task) for p in cls.ALL_PLANS])
        return cls.merge_results(task, results)
```

By running the plans concurrently, Grox ensures that the safety check (`PlanSafetyPtos`) does not block the vector embedding extraction (`PlanPostEmbedding`).

---

## 2. Anatomy of a Plan (The DAG)

Each plan defines a precise execution order using a dictionary of `TASK_DEPENDENCIES`. The base `Plan` class acts as a topological sorter and async executor.

Let's look at `PlanPostEmbeddingWithSummary` to see how X constructs a multimodal vector for a post containing a video.

```python
# From plan_post_embedding_with_summary.py
class PlanPostEmbeddingWithSummary(Plan):
    TASKS = {
        "rate_limit": TaskRateLimitEmbeddingWithPostSummary,
        "filter": TaskPostEmbeddingWithSummaryFilter,
        "media_hydration": TaskMediaHydration,
        "summarizer": TaskPostEmbeddingSummarizer,
        "multimodal_embedding": TaskMultimodalPostEmbeddingWithSummary,
        "sink": TaskWriteMMEmbeddingSinkV3,
    }

    TASK_DEPENDENCIES = {
        "rate_limit": set(),
        "filter": {"rate_limit"},
        "media_hydration": {"filter"},
        "summarizer": {"media_hydration"},
        "multimodal_embedding": {"summarizer"},
        "sink": {"multimodal_embedding"},
    }
```

### The Execution Flow
Because of the `TASK_DEPENDENCIES` sets, the execution engine forces this strict sequence:
1.  **Rate Limit & Filter:** Ensure this post hasn't already been embedded recently to save GPU compute.
2.  **Media Hydration:** Download the raw video frames and run the ASR (Automatic Speech Recognition) to extract the audio subtitles.
3.  **Summarizer:** Take the extracted subtitles and text, and ask a Grok VLM to generate a dense, contextual summary of the post.
4.  **Multimodal Embedding:** Pass the visual frames, the audio transcript, *and* the AI-generated summary into the final Qwen3 embedding model to produce a 768-dimensional vector.
5.  **Sink:** Write the final vector back to the primary vector database so the two-tower retrieval system can find it.

## Summary
By abstracting complex ML workflows into topological DAGs, Grox allows X's machine learning engineers to easily plug in new AI models (like a new audio-transcription model) without breaking the strict dependency chain required to generate accurate multimodal embeddings.
