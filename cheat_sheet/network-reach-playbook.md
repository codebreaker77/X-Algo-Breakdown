# Network Reach Playbook (In-Network vs. Out-of-Network)

The algorithm treats your followers (In-Network) completely differently than non-followers (Out-of-Network).

## 1. The Out-Of-Network (OON) Penalty
When your post is shown to someone who doesn't follow you (via the Phoenix Retrieval model), the `OONScorer` (`oon_scorer.rs`) applies a penalty:
```rust
updated_score = base_score * p::OON_WEIGHT_FACTOR
```
Because `OON_WEIGHT_FACTOR` is less than 1.0, your post is actively handicapped when competing for space in a non-follower's feed.

## 2. The Thunder Advantage (In-Network)
When your followers load their feed, your posts are retrieved by the **Thunder** service (`thunder_service.rs`). 
- Thunder is incredibly fast (sub-millisecond, in-memory).
- Thunder sorts initially by **pure recency** (`score_recent`).

## The Playbook for Growth
- **Phase 1: Win Your Network:** Your post goes to your followers first via Thunder. Since there is no OON penalty, this is where you must accumulate high-weight engagements (Replies, Dwell Time, Retweets).
- **Phase 2: Break the Barrier:** To break into the Out-of-Network feed, your base score must be high enough that *even after* the `OON_WEIGHT_FACTOR` penalty is applied, it still outranks other posts. This requires a strong initial spike from your core audience.
