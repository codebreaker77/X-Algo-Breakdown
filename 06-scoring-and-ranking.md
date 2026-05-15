# Chapter 6: Scoring & Ranking — How Posts Get Their Score

> The exact scoring formula that determines what appears in your feed

---

## Overview

After Phoenix predicts engagement probabilities for each candidate, the **Weighted Scorer** combines them into a single score. This chapter reveals the exact formula, the 19 action types, the diversity penalty, and the out-of-network adjustment.

---

## The Scoring Stack

Scores flow through 4 scorers in sequence:

```
Candidates
     │
     ▼
┌─────────────────────┐
│  1. Phoenix Scorer   │  ML predictions: P(action) for 19 actions
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  2. Weighted Scorer  │  Σ (weight_i × P(action_i)) → single score
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  3. Author Diversity │  Exponential decay for repeated authors
│     Scorer           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  4. OON Scorer       │  Penalty factor for out-of-network content
└──────────┬──────────┘
           │
           ▼
     Final Score
```

---

## 1. Phoenix Scorer

**File:** `home-mixer/scorers/phoenix_scorer.rs`

```rust
pub struct PhoenixScorer {
    pub phoenix_client: Arc<dyn PhoenixPredictionClient>,
    pub egress_client: Arc<dyn PhoenixPredictionClient>,  // Sidecar fallback
}
```

### Key behaviors:
- **Disabled for cached posts** — if we already have scores, skip ML
- **New user routing** — users with history below `PhoenixRankerNewUserHistoryThreshold` get routed to a specialized model cluster
- **Cluster overrides** — A/B experiments can route to different ML clusters via decider flags (`override_qf_use_lap7`, `override_qf_use_fou`)
- **Egress fallback** — first tries egress sidecar, falls back to main client on failure

```rust
fn resolve_cluster(query: &ScoredPostsQuery) -> PhoenixCluster {
    let threshold = query.params.get(PhoenixRankerNewUserHistoryThreshold);
    if threshold > 0 {
        let action_count = query.scoring_sequence.metadata.length;
        if action_count < threshold {
            return PhoenixCluster::parse(&query.params.get(PhoenixRankerNewUserInferenceClusterId));
        }
    }
    // ... decider overrides ...
    configured_cluster
}
```

### Output: 19 Engagement Predictions

```rust
pub struct PhoenixScores {
    pub favorite_score: Option<f64>,        // P(like)
    pub reply_score: Option<f64>,           // P(reply)
    pub retweet_score: Option<f64>,         // P(retweet)
    pub quote_score: Option<f64>,           // P(quote-tweet)
    pub click_score: Option<f64>,           // P(click to expand)
    pub profile_click_score: Option<f64>,   // P(visit author profile)
    pub vqv_score: Option<f64>,             // P(video quality view)
    pub photo_expand_score: Option<f64>,    // P(expand photo)
    pub share_score: Option<f64>,           // P(share)
    pub share_via_dm_score: Option<f64>,    // P(share via DM)
    pub share_via_copy_link_score: Option<f64>, // P(copy link)
    pub dwell_score: Option<f64>,           // P(stop scrolling)
    pub dwell_time: Option<f64>,            // Predicted dwell time (continuous)
    pub follow_author_score: Option<f64>,   // P(follow author)
    pub not_interested_score: Option<f64>,  // P(tap "Not interested")
    pub block_author_score: Option<f64>,    // P(block author)
    pub mute_author_score: Option<f64>,     // P(mute author)
    pub report_score: Option<f64>,          // P(report)
    pub quoted_click_score: Option<f64>,    // P(click through quote)
}
```

---

## 2. Weighted Scorer

**File:** `home-mixer/scorers/weighted_scorer.rs`

### The Formula

```rust
fn compute_weighted_score(candidate: &PostCandidate) -> f64 {
    let s: &PhoenixScores = &candidate.phoenix_scores;
    let vqv_weight = Self::vqv_weight_eligibility(candidate);

    let combined_score =
          apply(s.favorite_score,            FAVORITE_WEIGHT)           // POSITIVE
        + apply(s.reply_score,               REPLY_WEIGHT)              // POSITIVE
        + apply(s.retweet_score,             RETWEET_WEIGHT)            // POSITIVE
        + apply(s.photo_expand_score,        PHOTO_EXPAND_WEIGHT)       // POSITIVE
        + apply(s.click_score,               CLICK_WEIGHT)              // POSITIVE
        + apply(s.profile_click_score,       PROFILE_CLICK_WEIGHT)      // POSITIVE
        + apply(s.vqv_score,                 vqv_weight)                // CONDITIONAL
        + apply(s.share_score,               SHARE_WEIGHT)              // POSITIVE
        + apply(s.share_via_dm_score,        SHARE_VIA_DM_WEIGHT)       // POSITIVE
        + apply(s.share_via_copy_link_score, SHARE_VIA_COPY_LINK_WEIGHT) // POSITIVE
        + apply(s.dwell_score,               DWELL_WEIGHT)              // POSITIVE
        + apply(s.quote_score,               QUOTE_WEIGHT)              // POSITIVE
        + apply(s.quoted_click_score,        QUOTED_CLICK_WEIGHT)       // POSITIVE
        + apply(s.dwell_time,                CONT_DWELL_TIME_WEIGHT)    // POSITIVE (continuous)
        + apply(s.follow_author_score,       FOLLOW_AUTHOR_WEIGHT)      // POSITIVE
        + apply(s.not_interested_score,      NOT_INTERESTED_WEIGHT)     // NEGATIVE
        + apply(s.block_author_score,        BLOCK_AUTHOR_WEIGHT)       // NEGATIVE
        + apply(s.mute_author_score,         MUTE_AUTHOR_WEIGHT)        // NEGATIVE
        + apply(s.report_score,              REPORT_WEIGHT);            // NEGATIVE

    Self::offset_score(combined_score)
}
```

### Video Quality View — Conditional Weight

```rust
fn vqv_weight_eligibility(candidate: &PostCandidate) -> f64 {
    if candidate.video_duration_ms.is_some_and(|ms| ms > MIN_VIDEO_DURATION_MS) {
        VQV_WEIGHT  // Only apply VQV weight for actual videos above minimum duration
    } else {
        0.0  // No VQV weight for non-video or very short videos
    }
}
```

### Score Offsetting — Handling Negatives

```rust
fn offset_score(combined_score: f64) -> f64 {
    if WEIGHTS_SUM == 0.0 {
        combined_score.max(0.0)
    } else if combined_score < 0.0 {
        // Negative scores get compressed into the [0, NEGATIVE_SCORES_OFFSET] range
        (combined_score + NEGATIVE_WEIGHTS_SUM) / WEIGHTS_SUM * NEGATIVE_SCORES_OFFSET
    } else {
        // Positive scores get shifted up by the offset
        combined_score + NEGATIVE_SCORES_OFFSET
    }
}
```

This ensures all scores are non-negative while preserving the relative ordering, and gives negative signals (block/mute/report) room to pull scores down.

---

## 3. Author Diversity Scorer

**File:** `home-mixer/scorers/author_diversity_scorer.rs`

Prevents any single author from dominating your feed.

```rust
pub struct AuthorDiversityScorer {
    decay_factor: f64,  // AUTHOR_DIVERSITY_DECAY
    floor: f64,         // AUTHOR_DIVERSITY_FLOOR
}

fn multiplier(&self, position: usize) -> f64 {
    // Exponential decay: each subsequent post by same author gets lower multiplier
    (1.0 - self.floor) * self.decay_factor.powf(position as f64) + self.floor
}
```

**How it works:**
1. Sort candidates by weighted score (descending)
2. For each author, track how many posts we've seen
3. The Nth post from the same author gets multiplied by `decay^N`
4. The floor prevents complete suppression

**Example** (with decay=0.5, floor=0.1):
- Author's 1st post: score × 1.0
- Author's 2nd post: score × 0.55
- Author's 3rd post: score × 0.325
- Author's 4th post: score × 0.2125
- ...asymptotes to: score × 0.1

---

## 4. OON (Out-of-Network) Scorer

**File:** `home-mixer/scorers/oon_scorer.rs`

```rust
pub struct OONScorer;

fn score(&self, _query, candidates) -> Vec<PostCandidate> {
    candidates.iter().map(|c| {
        let updated_score = c.score.map(|base_score| match c.in_network {
            Some(false) => base_score * OON_WEIGHT_FACTOR,  // Penalize out-of-network
            _ => base_score,                                // Keep in-network as-is
        });
        PostCandidate { score: updated_score, .. }
    })
}
```

This applies a **flat multiplier** to all out-of-network content, giving a natural advantage to posts from accounts you follow. The `OON_WEIGHT_FACTOR` is configured via feature switches.

---

## Complete Score Flow

```
P(favorite)=0.8, P(reply)=0.1, P(repost)=0.3, ...
                    │
                    ▼
    Weighted Score = 0.8×W_fav + 0.1×W_reply + 0.3×W_rt + ...
                    = 1.45  (example)
                    │
                    ▼
    Normalized Score = normalize_score(candidate, 1.45)
                    = 1.45 + NEGATIVE_SCORES_OFFSET
                    │
                    ▼
    Diversity Score = 1.45 × multiplier(author_position)
                    = 1.45 × 0.55  (if 2nd post from this author)
                    = 0.80
                    │
                    ▼
    Final Score = 0.80 × OON_WEIGHT_FACTOR
                = 0.80 × 0.9  (if out-of-network)
                = 0.72
```

---

## Next Chapter

→ [Chapter 7: Grox — Real-Time AI Engine](./07-grox-ai-engine.md)
