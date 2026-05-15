# Chapter 7: Grox — Real-Time AI Engine

> The async Python daemon that continuously classifies, embeds, and labels every post on X

---

## Overview

Grox runs **independently of the serving path**. While Home Mixer handles user requests synchronously, Grox is a continuously running daemon that:
- Consumes posts from Kafka streams
- Classifies them for safety, quality, and topics via LLM calls
- Generates multimodal embeddings (text + image + video)
- Publishes results back to Kafka for the serving stack to consume

**Location:** `grox/`  
**Language:** Python (asyncio)

---

## System Architecture

### Entry Point (`main.py`)

```python
async def serve():
    context = new_context()         # Shared queues + shutdown events
    engine = Engine(context)        # Worker process — executes tasks
    dispatcher = Dispatcher(context) # Producer process — generates tasks
    grpc_server = GrpcServer(context) # External API

    await engine.start()            # Fork engine process
    await dispatcher.start()        # Fork dispatcher process
    await grpc_server.start()       # Start gRPC server

    await shutdown.wait()           # Wait for SIGINT/SIGTERM

    # Graceful shutdown: drain queues, then stop
    queue_connection_shutdown_context(context)
    await asyncio.sleep(300)        # 5-minute drain period
    shutdown_context(context)
    await asyncio.gather(grpc_server.stop(), dispatcher.stop(), engine.stop())
```

### Process Model

```
Main Process
├── Engine Process (grox-engine)
│   └── asyncio event loop
│       ├── _poll_task() ← reads from task_queue
│       └── _run_task() → PlanMaster.exec(task) → writes to resp_queue
│
├── Dispatcher Process (grox-dispatcher)
│   └── asyncio event loop
│       ├── _fill_loop() ← generates tasks from Kafka streams
│       └── _result_loop() ← reads results, handles retries
│
└── gRPC Server (for external requests)
```

---

## The Dispatcher — Task Generation & Routing

**File:** `grox/dispatcher.py`

### 16 Task Generator Types

The dispatcher supports **16 different Kafka stream sources**, each generating different types of tasks:

| Generator | Purpose |
|-----------|---------|
| `PostStreamTaskGenerator` | Main post stream |
| `PostStreamRecoveryTaskGenerator` | Recovery/replay for missed posts |
| `PostStreamTestTaskGenerator` | Test/staging stream |
| `PostStreamDelayedTaskGenerator` | Delayed processing stream |
| `PostSafetyStreamTaskGenerator` | Safety-focused post stream |
| `MinTractionPostStreamForGroxTaskGenerator` | Min-traction posts for Grox |
| `MinTractionPostStreamForGroxPtosTaskGenerator` | Min-traction for PTOS safety |
| `MinTractionPostStreamForGroxMultiModalTaskGenerator` | Min-traction multimodal |
| `PostEmbeddingRequestWithSummaryStreamTaskGenerator` | Embedding with summary |
| `PostEmbeddingRequestWithSummaryRecoveryStreamTaskGenerator` | Recovery for above |
| `PostEmbeddingRequestWithSummaryForReplyRecoveryStreamTaskGenerator` | Reply embedding recovery |
| `PostEmbeddingV5StreamTaskGenerator` | V5 embedding stream |
| `PostEmbeddingV5ForReplyStreamTaskGenerator` | V5 embeddings for replies |
| `ReplyRankingRecoveryTaskGenerator` | Reply ranking recovery |
| `SafetyPtosRecoveryStreamTaskGenerator` | PTOS recovery |
| `SafetyPtosDeluxeStreamTaskGenerator` | Deluxe PTOS stream |

Each generator has a `max_qps` (rate limit) and a `weight` for priority scheduling via `PriorityTaskGenerator`.

### In-Flight Management

```python
async def _fill_loop(self):
    while not self._is_shutdown():
        async for task_payload in self._task_generator.poll():
            # Back-pressure: wait if too many in-flight tasks
            while len(self._in_flights) >= self.config.max_in_flight:
                await asyncio.sleep(0.01)
            await self._submit_task(task_payload)

async def _result_loop(self):
    while not self._is_shutdown() or self._in_flights:
        result = await self._poll_result()
        if result.success:
            self._in_flights.remove(task.payload_id)
            await self._task_generator.ack(result)
        else:
            if task.attempt < max_attempts:
                task.attempt += 1
                await self._submit_task(task)  # Retry
            else:
                # Max retries exceeded — log and ack
                origin = self._task_generator.identify_task_origin(result)
                await self._task_generator.ack(result)
```

---

## The Engine — Task Execution

**File:** `grox/engine.py`

```python
class Engine:
    async def _process_task(self, task: TaskPayload) -> TaskResult:
        start = time.perf_counter()
        res = await PlanMaster.exec(task)  # Delegates to plan-based execution
        Metrics.histogram("engine.task.processing_time").record(end - start)
        return res

    async def _run(self, started_event):
        await self._init_run()
        MediaProcessor.start()    # Media processing worker
        ASRProcessor.start()      # Audio/Speech recognition worker
        started_event.set()

        while not self._is_shutdown() or not self._task_queue.empty():
            task = await self._poll_task()
            if task is None:
                await asyncio.sleep(0.1)
                continue
            asyncio.create_task(self._run_task(task))  # Concurrent execution
```

### Key detail: `asyncio.create_task`
Each task runs **concurrently** within the engine process. This means multiple LLM calls can be in-flight simultaneously, maximizing throughput.

---

## Multimodal Embedder V2

**File:** `grox/embedder/multimodal_post_embedder_v2.py`

### 5 Embedding Models

```python
class MultimodalPostEmbedderV2:
    def __init__(self, model="qwen3"):
        self._client = XaiEmbeddingClient(config=embed_config)              # Default
        self._video_client = XaiEmbeddingClient(config=video_embed_config)  # Video posts
        self._qwen_3_embed_06b_client = XaiEmbeddingClient(...)             # Qwen3 0.6B
        self._qwen_3_embed_8b_client = XaiEmbeddingClient(...)              # Qwen3 8B
        self._recsys_v4_embed_client = XaiEmbeddingClient(...)              # RecSys V4
```

### Model Selection Logic

```python
def _get_client(self, num_text, num_image, num_video) -> XaiEmbeddingClient:
    if self.use_custom_embed:
        return self._custom_embed_client
    if self.model == "qwen3":
        return self._qwen_3_embed_06b_client    # Default: small & fast
    if self.model == "qwen3_8b":
        return self._qwen_3_embed_8b_client     # Larger, more expressive
    if self.model == "v4":
        return self._recsys_v4_embed_client     # RecSys-specific
    if num_video > 0:
        return self._video_client               # Video-capable model
    return self._client                          # Standard text+image
```

### Rendering Modes

| Mode | Description |
|------|-------------|
| `lite` | Minimal text rendering for fast embedding |
| `eval` | Evaluation-specific rendering |
| `mmembed_summary` | Full multimodal with Grok summaries |

### Video Handling

```python
def document_original(self, content):
    for c in content:
        if isinstance(c, ConvoVideo):
            if c.combined_video_bytes:
                text += f"The video lasts for {c.total_duration:.2f} seconds."
                text += f"Frames sampled every {c.duration:.2f} second interval."
                for i, frame in enumerate(c.frames):
                    subtitle = c.subtitles[i] if available
                    text += f"At {bucket_times[i]:.2f} seconds, {subtitle_str}."
                document.append(("video", c.combined_video_bytes))
```

### Grok Summaries

Posts can be enriched with Grok-generated summaries before embedding:

```python
async def hydrate_grok_post_summary(self, post):
    query = StratoContentUnderstandingUnifiedPostAnnotations()
    res = await query.fetch(int(post.id))
    if res:
        post.summary = res["annotations"]["description"]
```

---

## Data Loaders

| Loader | Source | Purpose |
|--------|--------|---------|
| `KafkaPostEmbeddingRequestLoader` | Kafka | Post embedding requests |
| `KafkaTweetEmbeddingLoader` | Kafka | Raw tweet stream |
| `ASRProcessor` | Audio | Speech-to-text transcription |
| `MediaProcessor` | Media | Image/video processing |
| `StratoLoader` | Strato | Feature store queries |
| `MediaDescriptionLoader` | Strato | Media accessibility descriptions |

---

## Task Publishing

After processing, results flow back to Kafka:

```python
# From tasks/task_embedding_pub.py
@classmethod
async def _publish_to_kafka(cls, post: Post, embedding: list[float]) -> None:
    # Publishes computed embeddings to Kafka for downstream consumption
    # by the serving stack (Home Mixer, feature stores)
```

---

## Next Chapter

→ [Chapter 8: Safety & Content Moderation](./08-safety-and-moderation.md)
