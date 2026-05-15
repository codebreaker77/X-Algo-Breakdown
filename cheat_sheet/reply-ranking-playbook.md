# Reply Ranking Playbook

How to dominate the comment section and leverage large accounts for growth.

---

## The Death of "Bumping"

Historically, users could boost their visibility by being the first to reply to a massive account with a simple "Great post!" or "Following this."

This no longer works. The `ReplyScorer` (`reply_ranking.py`) uses a Vision-Language Model (`VLM_MINI_CRITICAL` or `VLM_PRIMARY_CRITICAL`) to read the parent post, read your reply, and output a specific quality score and reason.

```python
# From reply_ranking.py
class ReplyScoreResult(BaseModel):
    score: float
    reason: str
```

---

## How the VLM Scores Replies

The LLM is prompted to evaluate *relevance* and *value-add*. Because it has to generate a text `reason` for the numerical score it assigns, your reply must provide the AI with something to write about.

### Example 1: The Low-Effort Reply
*   **Parent Post:** "I just launched my new SaaS product after 6 months of coding in Rust. The memory safety features saved me from countless production crashes."
*   **Your Reply:** "Awesome job, congrats on the launch!"
*   **VLM Evaluation:** `Score: 0.1` | `Reason: The reply is a generic congratulation that adds no technical context or meaningful discussion to the original post about Rust or SaaS development.`
*   **Result:** Buried at the bottom of the thread.

### Example 2: The Value-Add Reply
*   **Parent Post:** "I just launched my new SaaS product after 6 months of coding in Rust. The memory safety features saved me from countless production crashes."
*   **Your Reply:** "Congrats on the launch! Did you end up using `Arc<Mutex<T>>` for your shared state, or did you rely on message passing via channels? I've found channels scale better in production."
*   **VLM Evaluation:** `Score: 0.9` | `Reason: The reply directly engages with the technical subject matter (Rust memory safety) and asks a highly relevant question about concurrency patterns (Mutex vs Channels), encouraging deeper discussion.`
*   **Result:** Pinned near the top of the thread.

---

## Tactics for Top Replies

1.  **Context is King:** The LLM evaluates the entire thread context. You must mention specific concepts or keywords used in the parent post to prove relevance.
2.  **Add a New Fact or Counter-Argument:** The highest scores are given to replies that introduce a new, verifiable piece of information or respectfully disagree with a well-reasoned argument. 
3.  **Use Media:** The scorer uses a *Vision* model. Replying with a highly relevant chart, screenshot, or meme gives the VLM visual data to analyze, which often boosts the quality score.
4.  **Be Fast but Substantive:** While being early helps, a highly substantive reply will mathematically outrank a fast, low-effort reply because the LLM score multiplier will continuously push the high-quality reply up the conversation tree as the post gains traction.
