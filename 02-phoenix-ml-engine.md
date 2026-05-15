# Chapter 2: Phoenix — The Grok-Based ML Brain

> The machine learning engine that decides what you see — two-tower retrieval and transformer ranking

---

## Overview

Phoenix is the ML backbone of the For You feed. It's a **two-stage system**:

1. **Retrieval** — Narrow millions of posts down to ~1000 using a two-tower model and ANN search
2. **Ranking** — Score those ~1000 candidates using a Grok-1 transformer that predicts 19 engagement types

The transformer architecture is **ported from Grok-1** (xAI's open-source LLM), adapted for recommendation with custom input embeddings and a critical attention masking scheme called **candidate isolation**.

---

## Stage 1: Two-Tower Retrieval

**File:** `phoenix/recsys_retrieval_model.py`

### Architecture

```
User Tower                              Candidate Tower
┌──────────────────────────┐            ┌──────────────────────────┐
│ User hash embeddings     │            │ Post hash embeddings     │
│ + History post embeddings│            │ + Author hash embeddings │
│ + History author embs    │            │                          │
│ + Action embeddings      │            │ Concat → MLP (SiLU)     │
│ + Product surface embs   │            │   or Mean Pool           │
│                          │            │                          │
│ Concat → Transformer     │            │ L2 Normalize             │
│ (Grok architecture)      │            │         ↓                │
│                          │            │ Candidate vector [D]     │
│ Masked mean pool         │            └──────────────────────────┘
│ L2 Normalize             │
│         ↓                │
│ User vector [D]          │
└──────────────────────────┘
           ↓
    dot product with corpus
           ↓
    top-K candidates
```

### User Tower — How Your Profile Becomes a Vector

```python
# From build_user_representation() in recsys_retrieval_model.py

# 1. Lookup user hash embeddings
user_embeddings, user_padding_mask = block_user_reduce(
    batch.user_hashes, recsys_embeddings.user_embeddings,
    hash_config.num_user_hashes, config.emb_size, 1.0
)

# 2. Lookup history post + author embeddings
history_embeddings, history_padding_mask = block_history_reduce(
    batch.history_post_hashes, recsys_embeddings.history_post_embeddings,
    recsys_embeddings.history_author_embeddings,
    history_product_surface_embeddings, history_actions_embeddings,
    hash_config.num_item_hashes, hash_config.num_author_hashes, 1.0
)

# 3. Concatenate user + history
embeddings = jnp.concatenate([user_embeddings, history_embeddings], axis=1)

# 4. Run through Grok transformer (same architecture as the LLM)
model_output = self.model(embeddings, padding_mask, candidate_start_offset=None)

# 5. Masked mean-pool → L2 normalize
user_representation = user_embedding_sum / jnp.maximum(mask_sum, 1.0)
user_representation = user_representation / user_norm
```

The key insight: your **entire engagement history** (what you liked, replied to, retweeted) is encoded into a single D-dimensional vector using the same transformer architecture as Grok-1.

### Candidate Tower — How Posts Become Vectors

```python
# From CandidateTower.__call__() in recsys_retrieval_model.py

# Two modes:
# Mode 1 (enable_linear_proj=True): Two-layer MLP with SiLU
proj_1 = hk.get_parameter("candidate_tower_projection_1",
    [post_author_embedding.shape[-1], self.emb_size * 2])
proj_2 = hk.get_parameter("candidate_tower_projection_2",
    [self.emb_size * 2, self.emb_size])

hidden = jnp.dot(post_author_embedding, proj_1)
hidden = jax.nn.silu(hidden)  # SiLU activation
candidate_embeddings = jnp.dot(hidden, proj_2)

# Mode 2 (enable_linear_proj=False): Simple mean pool
candidate_representation = jnp.mean(post_author_embedding, axis=-2)

# Both modes: L2 normalize
candidate_representation = candidate_embeddings / candidate_norm
```

### Retrieval — Finding Your Posts

```python
# From _retrieve_top_k() in recsys_retrieval_model.py

# Simple dot product between user vector and all corpus vectors
scores = jnp.matmul(user_representation, corpus_embeddings.T)  # [B, N]

# Apply mask for invalid entries
if corpus_mask is not None:
    scores = jnp.where(corpus_mask[None, :], scores, -INF)

# JAX top-k operation
top_k_scores, top_k_indices = jax.lax.top_k(scores, top_k)
```

Because both user and candidate representations are L2-normalized, **dot product = cosine similarity**. This enables efficient ANN libraries in production.

---

## Stage 2: Transformer Ranking

**File:** `phoenix/recsys_model.py`

### The Grok-1 Architecture Applied to Recommendations

The ranking model takes the user's engagement history and a batch of candidate posts, then predicts the probability of **19 different engagement actions** for each candidate.

### Input Structure

```
Position:  [User] [History_1, History_2, ..., History_127] [Cand_1, ..., Cand_64]
              ↓           ↓                                        ↓
         User hashes  Post+Author hashes                   Post+Author hashes
                     + Action embeddings                   + Product surface
                     + Product surface
```

- **History sequence length:** 127 positions
- **Candidate sequence length:** 64 positions
- **Total sequence:** 1 (user) + 127 (history) + 64 (candidates) = 192

### Candidate Isolation — The Critical Design Decision

This is the most important architectural decision in the entire system.

```
            ATTENTION MASK

       Keys (what we attend TO)
       ──────────────────────────────────────────▶

       │ User │    History (127)      │  Candidates (64)        │
  ┌────┼──────┼───────────────────────┼─────────────────────────┤
  │ U  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✗   ✗   ✗   ...  ✗    │
  ├────┼──────┼───────────────────────┼─────────────────────────┤
Q │ H  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✗   ✗   ✗   ...  ✗    │
u │ i  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✗   ✗   ✗   ...  ✗    │
e │ s  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✗   ✗   ✗   ...  ✗    │
r │ t  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✗   ✗   ✗   ...  ✗    │
i ├────┼──────┼───────────────────────┼─────────────────────────┤
e │ C  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✓   ✗   ✗   ...  ✗    │
s │ a  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✗   ✓   ✗   ...  ✗    │
  │ n  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✗   ✗   ✓   ...  ✗    │
  │ d  │  ✓   │  ✓   ✓   ✓  ...  ✓  │  ✗   ✗   ✗   ...  ✓    │
  └────┴──────┴───────────────────────┴─────────────────────────┘

  ✓ = Can attend     ✗ = Cannot attend
```

**Why this matters:**
- Candidates can see the user and their history (to determine relevance)
- Candidates **cannot see each other** (diagonal-only self-attention)
- This means the score for post A doesn't depend on whether post B is also in the batch
- Scores are therefore **consistent and cacheable** — you get the same score regardless of what other candidates are being evaluated simultaneously

### Output — 19 Engagement Predictions

```
Output shape: [B, num_candidates, num_actions]

Predictions per candidate:
├── P(favorite)          ← will they like it?
├── P(reply)             ← will they reply?
├── P(repost)            ← will they retweet?
├── P(quote)             ← will they quote-tweet?
├── P(click)             ← will they click to expand?
├── P(profile_click)     ← will they visit the author's profile?
├── P(video_view)        ← will they watch the video (quality view)?
├── P(photo_expand)      ← will they expand the photo?
├── P(share)             ← will they share?
├── P(share_via_dm)      ← will they share via DM?
├── P(share_copy_link)   ← will they copy the link?
├── P(dwell)             ← will they dwell (stop scrolling)?
├── P(dwell_time)        ← predicted dwell time (continuous)
├── P(follow_author)     ← will they follow the author?
├── P(not_interested)    ← will they tap "Not interested"?
├── P(block_author)      ← will they block the author?
├── P(mute_author)       ← will they mute the author?
├── P(report)            ← will they report the post?
└── P(quoted_click)      ← will they click through a quote-tweet?
```

---

## Hash-Based Embeddings

Instead of maintaining a lookup table for every user ID and post ID (which would be billions of entries), Phoenix uses **multi-hash embeddings**:

```python
# From run_pipeline.py

def _hash_ids(ids, scales, biases, modulus, num_buckets):
    """Hash IDs using linear congruential hash."""
    for i in range(n):
        for j in range(m):
            raw = (ids[i] * scales[j] + biases[j]) % modulus
            out[i, j] = 0 if ids[i] == 0 else int((int(raw) % (num_buckets - 1)) + 1)
```

Each entity (user, post, author) is hashed with multiple hash functions (2 per entity type), and the resulting embeddings are combined. This approach:
- Handles cold-start (new users/posts immediately get embeddings via hash)
- Scales without growing the embedding table
- Tolerates hash collisions through multiple independent hashes

### Unified Embedding Table Layout

```
┌─────────────┬────────────────────────────────────────────┐
│ Section     │ Indices                                     │
├─────────────┼────────────────────────────────────────────┤
│ Padding     │ [0, 65)                                     │
│ User embs   │ [65, 65 + user_vocab_size)                  │
│ Item embs   │ [65 + uv, 65 + uv + item_vocab_size)       │
│ Author embs │ [65 + uv + iv, 65 + uv + iv + author_vs)   │
└─────────────┴────────────────────────────────────────────┘
```

Default vocab sizes: 1,000,000 each for users, items, authors.

---

## Model Config (Mini / Open-Source Release)

| Parameter | Value |
|-----------|-------|
| Embedding dimension | 128 |
| Transformer layers | 4 |
| Attention heads | 4 |
| Key size | 32 |
| Widening factor | 2 |
| History sequence length | 127 |
| Candidate sequence length | 64 |
| User/Item/Author vocab | 1,000,000 each |
| Hashes per entity | 2 |
| Action types | 19 |
| Product surface vocab | 16 |

Production uses a larger model with more layers and wider embeddings. The released model is a frozen checkpoint from continuous training.

---

## The Pipeline Script

**File:** `phoenix/run_pipeline.py`

The unified pipeline script replaces the older separate `run_ranker.py` and `run_retrieval.py`:

```python
# Simplified flow from main()

# 1. Load models
ret_params = load_model_params("retrieval/model_params.npz")
rank_params = load_model_params("ranker/model_params.npz")

# 2. Load corpus (537K sports posts)
corpus_repr = np.load("sports_corpus.npz")["candidate_representations"]

# 3. Build user representation
user_repr = ret_fn.apply(ret_params, batch, emb_batch)

# 4. Retrieve top-K
scores = corpus_repr @ user_repr[0]
top_idx = np.argpartition(scores, -TOP_K)[-TOP_K:]

# 5. Rank top-K
for chunk in chunked(top_k_candidates, 64):
    probs = jax.nn.sigmoid(rank_fn.apply(rank_params, batch, embs).logits)

# 6. Weighted final score
weighted = (
    probs[:, IDX_FAV] * 1.0    # Favorite weight: 1.0
    + probs[:, IDX_REPLY] * 0.5  # Reply weight: 0.5
    + probs[:, IDX_RT] * 0.3     # Retweet weight: 0.3
    + probs[:, IDX_DWELL] * 0.2  # Dwell weight: 0.2
)
```

---

## Next Chapter

→ [Chapter 3: Candidate Pipeline — The Rust Engine](./03-candidate-pipeline.md)
