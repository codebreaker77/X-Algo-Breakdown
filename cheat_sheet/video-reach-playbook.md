# Video Reach Optimization Playbook

Based on the X recommendation algorithm source code (`weighted_scorer.rs` and `multimodal_post_embedder_v2.py`), here is exactly how video content is processed, analyzed, and boosted.

---

## 1. The VQV (Video Quality View) Multiplier

The weighted scorer includes a specific multiplier just for video views: `VQV_WEIGHT`. This weight is highly valuable because it rewards users simply for *watching* your content, even if they don't explicitly hit the "like" button.

### The Catch: Minimum Duration Checks
The algorithm explicitly checks the length of your video before allowing the `VQV_WEIGHT` to be applied.

```rust
// From weighted_scorer.rs
if candidate.video_duration_ms.is_some_and(|ms| ms > p::MIN_VIDEO_DURATION_MS) {
    p::VQV_WEIGHT
} else {
    0.0
}
```

### Strategic Implementation
*   **The Mistake:** Posting a 2-second looping GIF or a rapid 4-second meme video. The system classifies this as too short, dropping your VQV multiplier to 0.0. You lose the algorithmic advantage of posting a video entirely.
*   **The Fix:** Ensure your videos are substantive. If you have a short animation, loop it internally within a 15-second video file to ensure it clears the minimum duration threshold and qualifies for the VQV multiplier.

---

## 2. Subtitles Are Algorithmic Food

The `MultimodalPostEmbedderV2` processes videos by feeding them to a Vision-Language Model (VLM). But it doesn't just look at the pixels—it directly processes the audio track via an ASR (Automated Speech Recognition) pipeline and maps subtitles to specific visual frames.

### How the Code Extracts Frames
The algorithm buckets the video into intervals and creates a dense text representation:

```python
# From multimodal_post_embedder_v2.py
subtitle_str = f"with subtitle: {subtitle}" if subtitle else "(no subtitles)"
res.append(f"At {bucket_times[i]:.2f} seconds, {subtitle_str}.")
```

### Why This Matters for Reach
The VLM is trying to figure out what your video is about so the two-tower retrieval system can match it to interested users.
*   **Without Subtitles/Speech:** The model relies *only* on the visual frames. If you post a video of yourself talking to a camera, the visual frame is just "a person talking." The model has no idea what topic you are discussing, meaning your retrieval embedding will be incredibly weak.
*   **With Speech/Subtitles:** The text is literally injected into the model ("At 12.5 seconds, with subtitle: 'The new GPU architecture increases memory bandwidth by 50%'"). Now, the model explicitly maps your video to the "GPU Architecture" topic space.

### Strategic Implementation
1.  **Always speak clearly or use hardcoded/platform captions.** The ASR pipeline needs to convert your audio to text.
2.  **Front-load keywords verbally.** Say your primary topic in the first 3 seconds of the video. The frame bucket extraction will pick this up immediately, giving the algorithm high confidence in your content's topic.

---

## 3. Grok Summarization and Media Descriptions

The embedder can optionally request a Grok-generated summary of your post to help classify it.

```python
async def hydrate_grok_post_summary(self, post):
    # Fetches an AI-generated description of your content
    post.summary = res["annotations"]["description"]
```

**Tactic:** Don't post a video with a blank text caption. Always provide a clear, descriptive text body alongside your video. This text gives the Grok summarizer perfect context, which it then uses to build a highly accurate vector embedding, pushing your video to the exact right audience.
