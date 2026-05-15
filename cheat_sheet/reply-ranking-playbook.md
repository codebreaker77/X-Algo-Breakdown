# Reply Ranking Playbook

How to dominate the comment section and leverage large accounts for growth.

## The VLM Reply Scorer
Replies aren't ranked purely by likes anymore. The `ReplyScorer` (`reply_ranking.py`) uses a Vision-Language Model (`VLM_MINI_CRITICAL` or `VLM_PRIMARY_CRITICAL`) to read the parent post and your reply, and then outputs a specific quality score and reason.

```python
# From reply_ranking.py
class ReplyScoreResult(BaseModel):
    score: float
    reason: str
```

## Tactics for Top Replies
1. **Context is King**: The LLM evaluates the entire thread. "Great post!" gets a low score because it adds zero context. 
2. **Effort Gets Rewarded**: The model generates a `reason` for its score. Your reply must provide a reason for the LLM to justify a high score (e.g., adding a new fact, a high-quality counter-argument, or a relevant image).
3. **Be Fast but Substantive**: While being early helps, a highly substantive reply will outrank a fast, low-effort reply because the LLM score multiplier will push it up the conversation tree.
