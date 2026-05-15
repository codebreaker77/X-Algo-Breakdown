# Chapter 8: Safety & Content Moderation

> LLM-powered safety classifiers, spam detection, quality scoring, and policy enforcement

---

## Overview

Content moderation on X is powered by **LLM-as-classifier** — Grok vision models analyze every post for safety violations, quality scores, and spam signals. This runs continuously via the Grox engine, separate from the serving path.

**Location:** `grox/classifiers/content/`

---

## The 6 Classifiers

| Classifier | File | Model | Purpose |
|------------|------|-------|---------|
| `SafetyPtosCategoryClassifier` | `safety_ptos.py` | VLM_SAFETY | Categorizes posts by safety violation type |
| `SafetyPtosPolicyClassifier` | `safety_ptos.py` | VLM_PRIMARY_CRITICAL | Detailed policy violation analysis |
| `BangerInitialScreenClassifier` | `banger_initial_screen.py` | VLM_PRIMARY | Quality/engagement potential scoring |
| `SpamEapiLowFollowerClassifier` | `spam.py` | EAPI | Spam detection for low-follower accounts |
| `PostSafetyScreenDeluxe` | `post_safety_screen_deluxe.py` | Multiple | Multi-label comprehensive safety |
| `ReplyScorer` | `reply_ranking.py` | VLM_MINI_CRITICAL | Reply quality scoring |

---

## Safety PTOS — Category Classification

**File:** `safety_ptos.py`

The primary safety system. **PTOS = Platform Terms of Service.**

### How It Works

```python
class SafetyPtosCategoryClassifier(ContentClassifier):
    result_pattern = re.compile(r"(.*)<json>(.*)</json>", re.DOTALL)

    def __init__(self, model_name=ModelName.VLM_SAFETY, deluxe=False):
        vlm_config = grox_config.get_model(model_name)
        vlm_config.temperature = 0.000001  # Near-deterministic sampling
        vlm = VisionSampler(GrokModelConfig(**vlm_config.model_dump()))
```

### Conversation Structure

```python
def build_convo(self, post: Post) -> Conversation:
    convo = Conversation(conversation_id=uuid.uuid4().hex)

    # System prompt: PTOS policy rules
    if self.deluxe:
        # Deluxe mode: strip thinking restrictions for chain-of-thought reasoning
        convo.messages.append(Message(role=Role.SYSTEM,
            content=[_render_safety_ptos_for_reasoning()]))
    else:
        convo.messages.append(Message(role=Role.SYSTEM,
            content=[SafetyPtos().render()]))

    # User message: render the post with full context
    user_msg = Message(role=Role.USER, content=[])
    user_msg.content.extend(UserRenderer.render(post.user))    # Author info
    user_msg.content.extend(PostRenderer.render(post, include_reply_to=True))  # Post content + media
    user_msg.content.append(f"Analyze the post {post.id} and provide the requested JSON object.")

    # Assistant prefill for structured output
    convo.messages.append(Message(role=Role.ASSISTANT, content=["<json>"], separator=""))
```

### Output Parsing

The LLM returns reasoning + structured JSON:

```
<reasoning>
The post contains ... analysis ...
</reasoning>
<json>
{
  "violations": [...],
  "category": "...",
  "severity": "..."
}
</json>
```

```python
async def classify_post(self, post: Post) -> SafetyPostAnnotations:
    convo = await self._to_convo(post)
    result = await self._sample(convo, post)
    match = self.result_pattern.search(result)
    json_str = match.group(2).strip()
    return SafetyPostAnnotations.model_validate_json(json_str)
```

---

## Safety PTOS — Policy Violation Detection

The second-stage classifier performs **per-policy deep analysis** for posts flagged by the category classifier.

### 7 Supported Safety Policies

```python
SUPPORTED_POLICY_CATEGORIES = {
    SafetyPolicyCategory.ViolentMedia,
    SafetyPolicyCategory.AdultContent,
    SafetyPolicyCategory.Spam,
    SafetyPolicyCategory.IllegalAndRegulatedBehaviors,
    SafetyPolicyCategory.HateOrAbuse,
    SafetyPolicyCategory.ViolentSpeech,
    SafetyPolicyCategory.SuicideOrSelfHarm,
}
```

Each policy has its own detailed prompt template:
- `ViolentMediaPolicy` — graphic violence, gore
- `AdultContentPolicy` — NSFW content
- `SpamPolicy` — coordinated spam, bots
- `IllegalAndRegulatedBehaviorsPolicy` — illegal activity
- `HateOrAbusePolicy` — hate speech, harassment
- `ViolentSpeechPolicy` — threats of violence
- `SuicideOrSelfHarmPolicy` — self-harm content

### Deluxe Mode — Enhanced Reasoning

```python
class SafetyPtosPolicyClassifier(ContentClassifier):
    def __init__(self, deluxe=False):
        if deluxe:
            # Deluxe mode uses two additional reasoning models:
            # 1. EAPI_REASONING_INTERNAL — internal reasoning model
            self.eapi_reasoning = EapiSampler(eapi_config_reasoning)
            # 2. EAPI_REASONING — external reasoning model
            self.eapi_reasoning_x_algo = EapiSampler(eapi_config_reasoning_x_algo)

    DELUXE_4_2_CATEGORIES = {
        SafetyPolicyCategory.AdultContent,    # Needs visual reasoning
        SafetyPolicyCategory.ViolentMedia,    # Needs visual reasoning
    }
```

For Adult Content and Violent Media, the deluxe mode uses a specialized **4.2 reasoning model** with fallback:

```python
async def _sample_4_2(self, convo, post) -> str:
    try:
        return await self.eapi_reasoning_x_algo.sample(
            convo.interleaveToEapi(), conversation_id=convo.conversation_id)
    except Exception:
        # Fallback to standard VLM
        return await self.llm.sample(
            convo.interleave(), conversation_id=convo.conversation_id)
```

### Thinking Restriction Stripping

```python
def _strip_thinking_restrictions(text: str) -> str:
    """Remove thinking restriction markers from prompts.
    
    When using reasoning models (deluxe mode), thinking restrictions
    are stripped to allow full chain-of-thought reasoning in <think> blocks.
    """
    lines = text.splitlines(keepends=True)
    return "".join(
        line for line in lines if line.strip() not in _THINKING_RESTRICTION_LINES
    ).lstrip("\n")
```

---

## Banger Initial Screen — Quality Scoring

**File:** `banger_initial_screen.py`

Scores posts for **engagement potential** ("banger" = high-quality, likely to go viral).

### Output Structure

```python
class BangerInitialScreenResult(BaseModel):
    quality_score: float           # 0.0 - 1.0 quality assessment
    description: str               # Text description of the post
    tags: list[str]                # Content tags
    taxonomy_categories: list[dict] | None  # Category taxonomy
    tweet_bool_metadata: TweetBoolMetadata | None  # Boolean metadata flags
    is_image_editable_by_grok: bool | None   # Can Grok edit this image?
    slop_score: int | None         # AI slop detection score
    has_minor_score: float | None  # Minor/child presence score
```

### Quality Threshold

```python
banger_initial_positive = score >= 0.4  # Posts scoring ≥0.4 are "bangers"
```

### Metrics

Quality scores are tracked with fine-grained histograms:
```python
Metrics.histogram("banger_initial_screen_score",
    explicit_bucket_boundaries=[0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1]
).record(score)
```

---

## Reply Ranking — Scoring Reply Quality

**File:** `reply_ranking.py`

Scores replies to determine their ranking within conversation threads.

### Two-Model Architecture

```python
class ReplyScorer:
    def __init__(self):
        # Primary: fast mini model
        self.vlm = VisionSampler(GrokModelConfig(model=VLM_MINI_CRITICAL))
        # Fallback: larger model if mini fails
        self.vlm_fallback = VisionSampler(GrokModelConfig(model=VLM_PRIMARY_CRITICAL))
```

### Fallback Logic

```python
async def _sample(self, convo, post) -> str:
    output = await self.vlm.sample(convo.interleave(),
        json_schema=json.dumps(ReplyScoreResult.model_json_schema()))

    match = re.search(r"\{.*\}", output, re.DOTALL)
    if not (match and "score" in match.group(0)):
        # Mini model failed → retry with larger model + non-reasoning mode
        fallback_convo = await self._to_convo(post, non_reasoning=True)
        output = await self.vlm_fallback.sample(fallback_convo.interleave(), ...)
    return output
```

### Robust JSON Parsing

The reply ranker has a 4-layer parsing fallback:

```python
async def _parse(self, output) -> list[ReplyScoreResult]:
    # Layer 1: Standard Pydantic parsing
    result = ReplyScoreResult.model_validate_json(cleaned_result)

    # Layer 2: json_repair library
    repaired = json_repair.repair_json(cleaned_result, return_objects=True)

    # Layer 3: Regex extraction of "score" field
    score_match = re.search(r'"score":\s*([\d.]+)', cleaned_result)

    # Layer 4: Regex extraction of "reason" field
    reason_match = re.search(r'"reason":\s*"((?:[^"\\]|\\.)*)"', cleaned_result)
```

---

## 18 Feed Filters

From `home-mixer/filters/`:

### Pre-Scoring Filters
| Filter | Purpose |
|--------|---------|
| `DropDuplicatesFilter` | Remove duplicate post IDs |
| `CoreDataHydrationFilter` | Remove posts that failed hydration |
| `AgeFilter` | Remove posts older than threshold |
| `SelfTweetFilter` | Remove viewer's own posts |
| `RetweetDeduplicationFilter` | Dedupe reposts of same content |
| `IneligibleSubscriptionFilter` | Remove paywalled content |
| `PreviouslySeenPostsFilter` | Remove already-seen posts |
| `PreviouslySeenPostsBackupFilter` | Backup dedup filter |
| `PreviouslyServedPostsFilter` | Remove already-served posts |
| `MutedKeywordFilter` | Remove posts with muted keywords |
| `AuthorSocialgraphFilter` | Remove blocked/muted authors |
| `NewUserTopicIdsFilter` | Filter by new user topics |
| `TopicIdsFilter` | Filter by requested topics |
| `VideoFilter` | Filter video-specific requests |

### Post-Selection Filters
| Filter | Purpose |
|--------|---------|
| `VFFilter` | Visibility filtering (deleted/spam/violence/gore) |
| `AncillaryVFFilter` | Additional visibility checks |
| `DedupConversationFilter` | Deduplicate conversation threads |

---

## Next Chapter

→ [Chapter 9: What's New — May 2026 Release](./09-whats-new-may-2026.md)
