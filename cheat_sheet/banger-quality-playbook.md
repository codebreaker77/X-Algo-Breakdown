# Banger & Quality Scoring Playbook

The algorithm actively attempts to predict if your post will be a high-quality "banger" the moment it is published, before any human ever sees it. 

---

## The Banger Initial Screen

The `BangerInitialScreenClassifier` (`banger_initial_screen.py`) passes your post to a Vision-Language Model (VLM) with the strict goal of assigning a `quality_score` from 0.0 to 1.0.

### The Magic Threshold
```python
# From banger_initial_screen.py
banger_initial_positive = score >= 0.4
```
If your post crosses the `0.4` threshold, the system flags it as "banger positive", dropping it into high-priority distribution pipelines.

---

## How to Guarantee a High Quality Score

The LLM is prompted to evaluate specific traits of your post to calculate the score. Here is how to optimize for them:

### 1. Be Highly Descriptive
The LLM must output a `description` and `tags` for your post. 
*   **Example (Low Score):** "Just thinking out loud today..." (The LLM cannot generate meaningful tags for this. The quality score drops).
*   **Example (High Score):** "Just thinking about how the new FDA regulations will impact seed-stage biotech startups. Here is my thesis..." (The LLM instantly tags this as `Biotech`, `Venture Capital`, `FDA`, and assigns a high quality score because the topic is defined).

### 2. Optimize Image Editability
The model specifically tracks a boolean flag called `is_image_editable_by_grok`. 
*   **What this means:** The vision model prefers clear, high-resolution images with distinct subjects or readable text (like a clean infographic).
*   **The Penalty:** If you upload a deep-fried, pixelated meme, or an image crammed with illegible text, the VLM flags it as uneditable/unreadable, which tanks the overall quality score of the post.

### 3. Clear the Minor Safety Check
The model tracks a `has_minor_score` to protect children.
*   **The Trap:** If your content features children in ambiguous contexts (even a harmless family beach photo), the VLM might assign a moderate `has_minor_score` out of caution.
*   **The Result:** Posts with a high minor score are heavily throttled or completely excluded from Out-of-Network (For You) feeds to prevent exploitation. Keep your professional/growth-focused posts strictly focused on your niche, free of ambiguous family photos.

---

## Summary of the LLM Checklist
Before you hit post, ask yourself:
1. Can an AI accurately tag the topic of this post in 3 words?
2. Is the image clear enough for an AI to parse the subject matter?
3. Does this sound like a human wrote it? (See the [Slop Score guide](./safety-shadowban-guide.md#3-the-slop-score))

If yes to all three, your post will clear the 0.4 threshold.
