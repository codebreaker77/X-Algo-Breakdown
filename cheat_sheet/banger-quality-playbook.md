# Banger and Quality Scoring Playbook

The algorithm actively attempts to predict if your post will go viral the moment it is published.

## The Banger Initial Screen
The `BangerInitialScreenClassifier` (`banger_initial_screen.py`) passes your post to a Vision-Language Model with the goal of assigning a `quality_score` from 0.0 to 1.0.

```python
# From banger_initial_screen.py
banger_initial_positive = score >= 0.4
```

## How to Guarantee a High Quality Score
1. **Cross the 0.4 Threshold**: The system explicitly flags posts with a score of `0.4` or higher as "banger positive". 
2. **Descriptive Content**: The LLM must output a `description` and `tags` for your post. If your post is so vague that the LLM cannot summarize it or tag it effectively, your quality score will drop.
3. **Image Editability**: The model tracks `is_image_editable_by_grok`. High-quality, clear images that the system understands are favored over highly distorted or confusing visuals.
4. **Minor Safety**: The model tracks a `has_minor_score`. If your content features children in ambiguous or unsafe contexts, it will trigger safety throttles before it ever reaches the feed.
