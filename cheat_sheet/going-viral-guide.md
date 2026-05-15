# The "Going Viral" Cheat Sheet: A Deep Technical Guide

This guide is derived *strictly* from the actual source code of the X algorithm (May 15, 2026 release). It tells you exactly how the algorithm evaluates your posts, what actions are mathematically rewarded, and what will hurt your reach.

---

## 1. The Core Mathematics of Reach

The ranking of your post is determined by a single weighted score that predicts 19 different types of engagement via the `weighted_scorer.rs` component.

### The Positive Multipliers (What to Optimize For)

The algorithm explicitly predicts and rewards these actions. Here is how to trigger them:

#### A. Dwell Time (`CONT_DWELL_TIME_WEIGHT`)
This measures the continuous time someone spends looking at your post.
*   **The Code:** The transformer predicts a continuous value for how long a user will stop scrolling.
*   **Example (Bad):** A single sentence like "Good morning everyone." (Users read it in 1 second and scroll past).
*   **Example (Good):** A compelling story formatted with line breaks, or an intricate chart. "I spent 50 hours analyzing the new iPhone sales data. Here are the 3 anomalies nobody noticed..." (Requires the user to stop, read, and analyze the chart).

#### B. Profile Clicks (`PROFILE_CLICK_WEIGHT`)
The algorithm predicts if someone will click your avatar to see your profile.
*   **The Code:** High weights are given to posts that imply the author has more valuable content.
*   **Example (Bad):** "Here is a fact about marketing."
*   **Example (Good):** "I've scaled 3 startups to $1M ARR using this exact marketing framework. Here is step 1..." (Users click your profile to see if you are credible and to read the rest of your content).

#### C. Private Shares (`SHARE_VIA_DM_WEIGHT`, `SHARE_VIA_COPY_LINK_WEIGHT`)
New in this release, off-platform sharing and DM sharing are heavily tracked and rewarded.
*   **Example (Good):** Highly actionable templates, cheat sheets, or tool lists. People DM these to their colleagues or copy the link to drop in a Slack channel. "Here are 5 free ChatGPT prompts that replace a junior copywriter."

### The Negative Multipliers (What Kills Your Reach)
These actions actively reduce your final score:
*   `NOT_INTERESTED_WEIGHT`
*   `BLOCK_AUTHOR_WEIGHT`
*   `MUTE_AUTHOR_WEIGHT`
*   `REPORT_WEIGHT`

**The Math:** When these actions are predicted, the `offset_score` function compresses your post's score into a lower percentile, pushing it down the timeline.
**Tactic:** Engagement bait ("Like if you agree, ignore if you hate puppies") might get likes, but it severely annoys people, causing mutes. A single "Mute Author" penalty mathematically destroys the benefit of 10 likes.

---

## 2. Frequency & Author Diversity

**Reference:** `author_diversity_scorer.rs`

### The Rule
The algorithm applies an exponential decay penalty to multiple posts from the same author in a single feed load.
`Penalty = (1.0 - floor) * decay_factor ^ position + floor`

### The Math Explained
If the `decay_factor` is 0.5, here is how your posts are treated if you post a thread or spam multiple standalone posts in one hour:
*   **1st Post seen:** 100% of its normal score.
*   **2nd Post seen:** 55% of its normal score.
*   **3rd Post seen:** 32.5% of its normal score.
*   **4th Post seen:** 21% of its normal score.

### Tactic
*   **Space your posts.** If you post at 9 AM and 3 PM, they appear in different feed refresh sessions, avoiding the decay penalty.
*   **Do not post 5 standalone posts back-to-back.** Only the strongest one will survive; the others will be mathematically buried.

---

## 3. In-Network vs. Out-of-Network Penalties

**Reference:** `oon_scorer.rs`

### The Rule
Posts shown to people who *do not* follow you (Out-Of-Network) have their final score multiplied by an `OON_WEIGHT_FACTOR` (which is < 1.0).

### Example
Let's say `OON_WEIGHT_FACTOR` is 0.8.
*   Your post has a raw score of **10.0**.
*   In a follower's feed, it competes as a **10.0**.
*   In a non-follower's feed, it is forcefully reduced to **8.0**.

### Tactic
To go viral, your post must score so high on the positive multipliers (replies, retweets, dwell time) that an 8.0 still outranks the competition's 10.0. This means your core followers *must* engage heavily first to boost the base score before the algorithm attempts to test it on out-of-network users.

---

## 4. Avoiding AI "Slop"

**Reference:** `banger_initial_screen.py`

### The Rule
A Grok vision model reads your post and calculates a `slop_score` to detect AI-generated garbage.

### Example (High Slop Score):
"🚀 Unlock your full potential today! In this ever-evolving digital landscape, synergy is key. Let's delve into the multifaceted paradigms of success!"
*Why it fails:* It uses classic LLM vocabulary ("delve", "landscape", excessive emojis). The model easily flags this as low-effort AI generation.

### Example (Low Slop Score):
"We ripped out our entire Redis caching layer yesterday. Latency dropped by 40ms, but we immediately hit a CPU bottleneck on the primary DB. Here is how we fixed it."
*Why it wins:* It uses specific, human experiences, concrete numbers, and conversational syntax. It bypasses the `slop_score` penalty entirely.
