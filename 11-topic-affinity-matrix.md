# Chapter 11: The Topic Affinity Matrix

The X algorithm does not just rely on author-follower graphs; it relies heavily on **Topic Affinity**. When a user requests a "Topic" feed or explores a "Starter Pack", the `TopicIdsFilter` is invoked. This chapter breaks down how the algorithm maps content to the specific interests you follow.

---

## 1. The Topic Expansion Logic

When you follow a specific topic (e.g., "SpaceX"), the algorithm doesn't just look for posts explicitly tagged with "SpaceX". It uses an expansion matrix to pull in related content.

### `TopicIdExpansion::expand_supertopic()`
The codebase maps hundreds of granular topics into massive "Supertopics". This is how exploration is built into the feed. 

```rust
pub fn supertopic(topic_id: i64) -> i64 {
    match topic_id {
        XAI_POP | XAI_K_POP | XAI_COUNTRY_MUSIC | XAI_ROCK | XAI_HIP_HOP => XAI_MUSIC,
        XAI_BIOTECH | XAI_SPACE => XAI_SCIENCE,
        XAI_AI | XAI_SOFTWARE_DEVELOPMENT | XAI_ROBOTICS => XAI_TECHNOLOGY,
        XAI_NFL | XAI_NCAA_FOOTBALL => XAI_AMERICAN_FOOTBALL,
        _ => topic_id,
    }
}
```

**How it works:**
If you follow the topic `XAI_ROBOTICS`, the algorithm resolves its supertopic as `XAI_TECHNOLOGY`. 
The `TopicIdsFilter` will then allow posts from `XAI_AI` and `XAI_SOFTWARE_DEVELOPMENT` to bleed into your feed, even if you never explicitly followed them, because they share the same supertopic cluster.

---

## 2. The Filter Pipeline

The actual filtering happens inside `home-mixer/filters/topic_ids_filter.rs`. It operates differently depending on whether it is a standard topic request or a "bulk" topic request.

### Standard Request Filtering
For a normal topic request, a post is kept if it satisfies **either** of these conditions:
1. **Direct Match:** The post's `filtered_topic_ids` (topics explicitly identified by Grok) contain the exact topic you requested.
2. **Supertopic Bleed:** The post's `unfiltered_topic_ids` (broader keyword matches) match the requested topic, **AND** its `filtered_topic_ids` contain a related supertopic.

This guarantees high relevance while still allowing the system to test new topics in your feed.

### Bulk Exclusion
If a user explicitly marks a topic as "Not Interested", the algorithm doesn't just block that single topic. It expands the excluded topic into all of its sub-categories.

```rust
let excluded_ids: HashSet<i64> = query.excluded_topic_ids.iter().copied().collect();
let mut excluded_expanded = TopicIdExpansion::expand(&excluded_ids);
```
If you block `XAI_SPORTS_REAL`, the `expand` function recursively blocks all 30+ sub-sports (Soccer, Basketball, NFL, Olympics, etc.).

---

## 3. Topic Overrides and A/B Testing

The filter includes a `TopicFilteringOverrideMap` which is parsed dynamically from feature switches.

```rust
// Parse string like "123=ExperimentA, 456=ExperimentB"
pub fn parse(raw: &str) -> Self
```

This allows X engineers to run A/B tests on specific topics. For example, they can dynamically loosen the filtering strictness for `XAI_ELECTIONS` during an election week to increase political reach, without deploying new code.

---

## Summary
The Topic Affinity Matrix is what prevents the For You feed from becoming an echo chamber. By utilizing the `supertopic` hierarchy, the algorithm constantly introduces you to adjacent communities (e.g., bleeding `XAI_BIOTECH` into an `XAI_SPACE` feed) to test if your engagement will expand the graph.
