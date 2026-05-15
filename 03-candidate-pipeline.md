# Chapter 3: Candidate Pipeline — The Rust Engine

> The composable trait-object framework that processes every candidate in the feed

---

## Overview

The Candidate Pipeline is a **reusable Rust framework** that defines how recommendation candidates flow through processing stages. It's the execution backbone that Home Mixer plugs its specific sources, hydrators, filters, and scorers into.

**Location:** `candidate-pipeline/`

---

## Pipeline Stages

```
                    PipelineComponents<Q, C>
                    ┌────────────────────────┐
                    │ query_hydrators[]      │
                    │ dependent_query_hydr[] │
                    │ sources[]              │
                    │ hydrators[]            │
                    │ filters[]              │
                    │ post_filter_hydrators[]│
                    │ scorers[]              │
                    │ selector              │
                    │ post_selection_filters[]│
                    │ side_effects[]         │
                    └────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         hydrate_query  fetch_candidates  run_hydrators
              │               │               │
              ▼               ▼               ▼
    hydrate_dependent   run_filters      score → select
              │                               │
              ▼                               ▼
         side_effects                   PipelineResult
```

---

## Core Types

### `PipelineStage` (Enum)
From `candidate_pipeline.rs:22`:
```rust
pub enum PipelineStage {
    QueryHydration,
    DependentQueryHydration,
    CandidateFetching,
    Hydration,
    Filtering,
    PostFilterHydration,
    Scoring,
    Selection,
    PostSelectionFiltering,
    SideEffects,
}
```

### `PipelineComponents` (Struct)
From `candidate_pipeline.rs:35`:
```rust
pub struct PipelineComponents<Q, C> {
    // All component vectors — each element is a trait object
    pub query_hydrators: Vec<Box<dyn QueryHydrator<Q>>>,
    pub sources: Vec<Box<dyn Source<Q, C>>>,
    pub hydrators: Vec<Box<dyn Hydrator<Q, C>>>,
    pub filters: Vec<Box<dyn Filter<Q, C>>>,
    pub scorers: Vec<Box<dyn Scorer<Q, C>>>,
    pub selector: Box<dyn Selector<Q, C>>,
    pub side_effects: Vec<Box<dyn SideEffect<Q, C>>>,
    // ...
}
```

### `PipelineResult` (Struct)
From `candidate_pipeline.rs:40`:
```rust
pub struct PipelineResult<Q, C> {
    pub query: Q,
    pub candidates: Vec<C>,
    pub stage_timings: HashMap<PipelineStage, Duration>,
    pub component_timings: HashMap<String, Duration>,
}
```

---

## Component Traits

### Source (`source.rs`)
```rust
#[async_trait]
pub trait Source<Q, C>: Send + Sync {
    fn name(&self) -> &str;
    fn enable(&self, query: &Q) -> bool { true }
    async fn fetch(&self, query: &Q) -> Result<Vec<C>, String>;
}
```

### Hydrator (`hydrator.rs`)
```rust
#[async_trait]
pub trait Hydrator<Q, C>: Send + Sync {
    fn name(&self) -> &str;
    fn enable(&self, query: &Q) -> bool { true }
    async fn run(&self, query: &Q, candidates: &[C]) -> Result<Vec<C>, String>;
    fn update(&self, candidate: &mut C, hydrated: C);
    fn update_all(&self, candidates: &mut [C], hydrated: Vec<C>) { ... }
}
```

### Filter (`filter.rs`)
```rust
pub enum FilterResult {
    Keep,
    Drop(String),  // reason for dropping
}

#[async_trait]
pub trait Filter<Q, C>: Send + Sync {
    fn name(&self) -> &str;
    fn enable(&self, query: &Q) -> bool { true }
    async fn run(&self, query: &Q, candidates: &[C]) -> Result<Vec<FilterResult>, String>;
}
```

### Scorer (`scorer.rs`)
```rust
#[async_trait]
pub trait Scorer<Q, C>: Send + Sync {
    fn name(&self) -> &str { type_name::<Self>() }
    fn enable(&self, query: &Q) -> bool { true }
    async fn score(&self, query: &Q, candidates: &[C]) -> Result<Vec<C>, String>;
    fn update(&self, candidate: &mut C, scored: C);
    fn update_all(&self, candidates: &mut [C], scored: Vec<C>) { ... }
}
```

### Selector (`selector.rs`)
```rust
pub struct SelectResult<C> {
    pub selected: Vec<C>,
    pub remaining: Vec<C>,
}

#[async_trait]
pub trait Selector<Q, C>: Send + Sync {
    fn name(&self) -> &str { type_name::<Self>() }
    async fn select(&self, query: &Q, candidates: Vec<C>) -> SelectResult<C>;
}
```

### SideEffect (`side_effect.rs`)
```rust
pub struct SideEffectInput<'a, Q, C> {
    pub query: &'a Q,
    pub selected: &'a [C],
    pub remaining: &'a [C],
}

#[async_trait]
pub trait SideEffect<Q, C>: Send + Sync {
    fn name(&self) -> &str;
    fn enable(&self, query: &Q) -> bool { true }
    async fn run(&self, input: SideEffectInput<'_, Q, C>);
}
```

---

## Pipeline Execution Flow

From `candidate_pipeline.rs`, the `execute()` method runs the stages in order:

```rust
pub async fn execute(&self, query: Q) -> Result<PipelineResult<Q, C>, PipelineError> {
    // 1. Hydrate query context
    let query = self.hydrate_query(query).await?;
    let query = self.hydrate_dependent_query(query).await?;

    // 2. Fetch candidates from all sources (parallel)
    let candidates = self.fetch_candidates(&query).await?;

    // 3. Hydrate candidates with features
    let candidates = self.run_hydrators(&query, candidates).await?;

    // 4. Filter out ineligible candidates
    let candidates = self.filter(&query, candidates).await?;

    // 5. Score candidates
    let candidates = self.score(&query, candidates).await?;

    // 6. Select top-K
    let SelectResult { selected, remaining } = self.select(&query, candidates).await?;

    // 7. Post-selection filtering
    let selected = self.post_selection_filter(&query, selected).await?;

    // 8. Run side effects (async, non-blocking)
    self.run_side_effects(&query, &selected, &remaining).await;

    Ok(PipelineResult { query, candidates: selected, ... })
}
```

### Key Properties:
- **Enable-gating:** Every component has `enable(&self, query: &Q) -> bool` — if it returns false, the component is skipped. This powers A/B testing via feature switches.
- **Observability:** `record_enabled_components()` logs which components are active for each request.
- **Parallel sources:** Multiple `Source` implementations can run concurrently.
- **Error isolation:** Individual component failures don't crash the pipeline.

---

## Next Chapter

→ [Chapter 4: Home Mixer — Feed Assembly & Serving](./04-home-mixer.md)
