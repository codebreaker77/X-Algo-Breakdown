# Chapter 14: Phoenix JAX/Haiku Inference

While the serving pipeline is built in Rust, the actual neural network mathematical operations are built using DeepMind's **Haiku** over **JAX**.

This chapter breaks down the tensor math, the inference wrappers, and the multi-hash embedding logic found in `phoenix/runners.py` and `phoenix/run_pipeline.py`.

---

## 1. JAX and Haiku (`hk.transform`)

To optimize inference speeds for the Grok-1 transformer, X heavily utilizes JAX's Just-In-Time (`jit`) compilation. Because JAX requires pure functions, the X engineers wrap their model's forward passes using DeepMind's Haiku library (`hk.transform`).

```python
# From runners.py
def make_forward_fn(self):
    def forward(batch: RecsysBatch, recsys_embeddings: RecsysEmbeddings):
        out = self.model.make()(batch, recsys_embeddings)
        return out

    # Converts the object-oriented model into a pure function for JAX
    return hk.transform(forward)
```

By decoupling the state (`params`) from the logic (`forward`), they can distribute the compiled inference graph across large TPU/GPU clusters extremely efficiently.

---

## 2. Linear Congruential Hashing

One of the hardest problems in recommendation systems is dealing with a massive cardinality of entities. X has hundreds of millions of users and billions of tweets. Storing an explicit 768-dimensional embedding for every single one in memory is impossible.

X solves this using **Multi-Hash Embeddings**.

```python
# From run_pipeline.py
def _hash_ids(ids, scales, biases, modulus, num_buckets):
    # Linear congruential hash: (id * scale + bias) % modulus
    # This maps any sparse ID to a dense index between 1 and num_buckets
    out = np.empty((n, m), dtype=np.int32)
    for i in range(n):
        for j in range(m):
            raw = (ids[i] * scales[j] + biases[j]) % np.int64(modulus)
            out[i, j] = 0 if ids[i] == 0 else int((int(raw) % (num_buckets - 1)) + 1)
    return out
```

### How it works:
1. Instead of one massive embedding table, they have a smaller, dense `embedding_tables.npz`.
2. When a `user_id` comes in, it is hashed *multiple times* (e.g., 2 times) using different `scales` and `biases`.
3. This produces 2 dense indices. The model looks up the embeddings at those 2 indices and combines them to form the user's representation.

This allows infinite sparse IDs to map to a finite memory footprint.

---

## 3. The 19-Dimensional Sigmoid Output

When the Ranking model evaluates a candidate, it doesn't output a single "score". It outputs 19 different logits.

```python
# From runners.py
def hk_rank_candidates(batch, recsys_embeddings):
    output = hk_forward(batch, recsys_embeddings)
    logits = output.logits

    # Apply sigmoid to squash all logits into probabilities between 0 and 1
    probs = jax.nn.sigmoid(logits)

    return RankingOutput(
        scores=probs,
        p_favorite_score=probs[:, :, 0],
        p_reply_score=probs[:, :, 1],
        p_repost_score=probs[:, :, 2],
        p_vqv_score=probs[:, :, 6],
        p_dwell_score=probs[:, :, 10],
        # ... 19 total
    )
```

These 19 probability arrays are then passed back to the Rust `Home Mixer` where the `WeightedScorer` multiplies each probability by its current feature-flag weight to compute the final sort order.

## Summary
The Phoenix inference layer is a masterclass in optimization. By leveraging Haiku/JAX for compilation and linear congruential hashing to compress billions of users into a dense embedding space, X manages to run a massive Grok-1 transformer over millions of candidates per second.
