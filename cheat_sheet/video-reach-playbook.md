# Video Reach Optimization Playbook

Based on the X recommendation algorithm source code (`weighted_scorer.rs` and `multimodal_post_embedder_v2.py`), here is exactly how video content is processed and boosted.

## 1. The VQV (Video Quality View) Multiplier
The weighted scorer includes a specific multiplier just for video views: `VQV_WEIGHT`.

**The Catch:** The algorithm explicitly checks the length of your video before applying this weight.
```rust
// From weighted_scorer.rs
if candidate.video_duration_ms.is_some_and(|ms| ms > p::MIN_VIDEO_DURATION_MS) {
    p::VQV_WEIGHT
} else {
    0.0
}
```
*Tactic*: Do not post 2-second looping GIFs as videos. Ensure your videos are long enough to pass the `MIN_VIDEO_DURATION_MS` threshold, otherwise you lose the `VQV_WEIGHT` entirely.

## 2. Subtitles Are Algorithmic Food
The `MultimodalPostEmbedderV2` processes videos by sampling frames at regular intervals. Crucially, it maps subtitles directly to these frames:

```python
# From multimodal_post_embedder_v2.py
subtitle_str = f"with subtitle: {subtitle}" if subtitle else "(no subtitles)"
res.append(f"At {bucket_times[i]:.2f} seconds, {subtitle_str}.")
```
*Tactic*: **Always use subtitles/captions.** The algorithm's Vision-Language Model reads your subtitles frame-by-frame and embeds them into your post's vector representation. If you don't use subtitles, the model has to guess your video's context entirely from visual frames.
