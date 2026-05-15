# The "Going Viral" Cheat Sheet (Based on Real Source Code)

This cheat sheet is derived *strictly* from the actual source code of the X algorithm (May 15, 2026 release). It tells you exactly how the algorithm evaluates your posts, what actions are mathematically rewarded, and what will hurt your reach.

---

## 1. The Core Mathematics of Reach

The ranking of your post is determined by a single weighted score that predicts 19 different types of engagement.

**Reference:** [Scoring Formula (`weighted_scorer.rs`)](../06-scoring-and-ranking.md#2-weighted-scorer)

### What to Optimize For (The Positive Multipliers)
The algorithm explicitly predicts and rewards these actions:
1. **Dwell Time (`DWELL_TIME_WEIGHT`)**: The continuous time someone spends looking at your post.
   - *Tactic*: Write longer posts (but make them engaging). Use threads. Post compelling visual content that makes people stop scrolling.
2. **Replies (`REPLY_WEIGHT`)**: Predicting if someone will reply.
   - *Tactic*: End with a question or a polarizing/debatable stance to provoke discussion.
3. **Favorites & Retweets (`FAVORITE_WEIGHT`, `RETWEET_WEIGHT`)**: The standard engagement metrics.
   - *Tactic*: High-value, shareable insights or emotional resonance.
4. **Profile Clicks (`PROFILE_CLICK_WEIGHT`)**: Will they click to your profile?
   - *Tactic*: Create intrigue. "I just spent 50 hours researching X, here's the one thing that surprised me most..."
5. **Shares (`SHARE_WEIGHT`, `SHARE_VIA_DM_WEIGHT`, `SHARE_VIA_COPY_LINK_WEIGHT`)**: New in this release. Sharing off-platform or via DM is heavily tracked.
   - *Tactic*: Post highly actionable "save this for later" or "share with your team" resources.
6. **Follows (`FOLLOW_AUTHOR_WEIGHT`)**: Will reading this post make them follow you?
   - *Tactic*: Establish deep authority. Provide value that implies future value.
7. **Media Expansion (`PHOTO_EXPAND_WEIGHT`, `VQV_WEIGHT`)**: Will they click the photo or watch the video?
   - *Tactic*: Use high-quality, detailed images (infographics, charts) that require tapping to read fully. Use videos longer than the minimum duration.

### What Kills Your Reach (The Negative Multipliers)
These actions actively reduce your final score, pushing your post down the timeline:
1. **"Not Interested" (`NOT_INTERESTED_WEIGHT`)**
2. **Block Author (`BLOCK_AUTHOR_WEIGHT`)**
3. **Mute Author (`MUTE_AUTHOR_WEIGHT`)**
4. **Report Post (`REPORT_WEIGHT`)**

*Tactic*: Engagement bait or excessively controversial content that annoys people enough to block/mute you will permanently damage your reach.

---

## 2. Frequency & Author Diversity

**Reference:** [Author Diversity Scorer (`author_diversity_scorer.rs`)](../06-scoring-and-ranking.md#3-author-diversity-scorer)

**The Rule:** The algorithm applies an exponential decay penalty to multiple posts from the same author in a single feed load.
```
Penalty = (1.0 - floor) * decay_factor ^ position + floor
```

**What this means:** If you post 5 times in one hour, only your *best* post will get maximum reach. The 2nd, 3rd, and 4th posts will have their scores artificially reduced to prevent you from dominating someone's feed.

*Tactic*: 
- **Do not spam.** Space your posts out (e.g., morning, afternoon, evening) so they appear in different feed refresh sessions.
- **Focus on quality over quantity.** One high-scoring post is better than three average ones, because the average ones will be penalized anyway.

---

## 3. The Power of "In-Network" vs. "Out-of-Network"

**Reference:** [OON Scorer (`oon_scorer.rs`)](../06-scoring-and-ranking.md#4-oon-out-of-network-scorer)

**The Rule:** Posts shown to people who *do not* follow you (Out-Of-Network) have their final score multiplied by `OON_WEIGHT_FACTOR` (which is < 1.0).

**What this means:** It is mathematically harder to rank high in the feeds of non-followers.

*Tactic*: 
- Your first burst of engagement *must* come from your own followers (In-Network), where you have no penalty.
- To go viral, your post must score so high on the positive multipliers (replies, retweets, dwell time) that it overcomes the OON penalty when it spills into the broader For You feed.

---

## 4. Content Quality & The "Banger" Screen

**Reference:** [Banger Initial Screen (`banger_initial_screen.py`)](../08-safety-and-moderation.md#banger-initial-screen--quality-scoring)

**The Rule:** A Grok vision model reads your post and assigns a `quality_score` from 0.0 to 1.0. Posts scoring ≥0.4 are flagged as "bangers".

**What this means:** The AI literally reads your text and analyzes your images to judge quality.

*Tactic*: 
- **Avoid "Slop"**: The classifier specifically calculates a `slop_score` (AI-generated, low-effort garbage). Ensure your writing sounds human, original, and substantive.
- **Visuals Matter**: The model is a *Vision* model. High-quality accompanying images contribute to the quality score.

---

## 5. Be Mindful of Reply Quality

**Reference:** [Reply Ranking (`reply_ranking.py`)](../08-safety-and-moderation.md#reply-ranking--scoring-reply-quality)

**The Rule:** Replies are individually scored by a Grok VLM for quality before being ranked under a post.

*Tactic*: 
- "Great post!" or "Following" are low-quality replies and will be buried.
- If you are trying to grow by replying to larger accounts, write substantive, value-add replies. The AI model reads the parent post and your reply, and scores how relevant and constructive your contribution is.

---

## 6. Topic Adherence

**Reference:** [Query Hydration](../01-architecture-overview.md#step-5-candidate-hydration)

**The Rule:** The system hydrates queries with "Followed Grok topics" and filters out irrelevance.

*Tactic*: Consistency helps. If the system knows you write about a specific topic, and it knows a user engages with that topic, the two-tower retrieval model is highly likely to match you. Extreme pivots in topic confuse your historical embedding vector.
