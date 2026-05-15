# Network Reach Playbook (In-Network vs. Out-of-Network)

The algorithm treats your followers (In-Network) completely differently than non-followers (Out-of-Network). Understanding this dual-pipeline architecture is the key to consistent growth.

---

## 1. The Thunder Advantage (In-Network)

When your followers open the app and load their feed, your posts are retrieved by the **Thunder** service (`thunder_service.rs`).

### How Thunder Works
*   **Speed:** Thunder is an in-memory database that operates at sub-millisecond speeds.
*   **No ML Ranking (Initially):** Thunder's initial sort is based on **pure recency** (`score_recent`).
*   **The Benefit:** You have a guaranteed, un-penalized window to appear at the top of your followers' feeds the moment you post.

---

## 2. The Out-Of-Network (OON) Penalty

When the algorithm attempts to show your post to someone who doesn't follow you (via the Phoenix Retrieval model), the `OONScorer` (`oon_scorer.rs`) applies a strict mathematical penalty.

### The Math
```rust
updated_score = base_score * p::OON_WEIGHT_FACTOR
```
Because `OON_WEIGHT_FACTOR` is always less than 1.0 (e.g., 0.75), your post is actively handicapped when competing for space in a non-follower's feed. 

If an Out-of-Network post and an In-Network post both have a raw score of 10.0, the Out-of-Network post is artificially dropped to 7.5, ensuring the user sees the content from the person they actually follow.

---

## 3. The 2-Phase Playbook for Growth

To go viral, you must break the OON penalty barrier. You do this by engineering a massive initial spike from your own followers.

### Phase 1: Win Your Network (Minutes 0-60)
*   Your post goes to your followers first via Thunder. Since there is no OON penalty, this is your proving ground.
*   **Goal:** Accumulate high-weight engagements immediately.
*   **Tactics:** Ask a question to drive **Replies**. Format the post beautifully to drive **Dwell Time**. Make it highly relatable to drive **Retweets**.

### Phase 2: Break the Barrier (Hours 1-24)
*   As your followers engage, your post's raw score skyrockets (e.g., reaching a 25.0).
*   The Phoenix engine tests your post in an Out-of-Network feed. The OON penalty applies (25.0 * 0.75 = 18.75).
*   **The Breakthrough:** Because your raw score is so high, even your penalized score (18.75) is higher than the average posts in the non-follower's feed (which might hover around 12.0). 
*   **Result:** You enter the global algorithmic distribution loop.

**Summary:** You cannot go viral to strangers if your own followers do not care about your post. Optimize for your core audience first; the algorithm will handle the global distribution automatically once the math checks out.
