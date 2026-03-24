# Response

## The v14 result confirms the escalation

The numbers are diagnostic:

- es_clones confidence: 0.9748 → 0.9736 (barely moved)
- TD parent confidence: 0.9360 → 0.9448 (slightly improved)
- Gap narrowed from 0.0388 to 0.0288 but didn't flip

Norm-stratified sampling improved the general parent-zone pressure (the TD parent gained confidence), but es_clones specifically was either not sampled often enough or is embedded close enough that the few samples it received didn't move it out of range.

This confirms: generic parent-zone pressure is insufficient. We need **targeted repulsion from the child's immediate geometric neighborhood**.

## Answering the four questions

### Q1: What is the cleanest definition of a TD-anchored hard negative?

**Option 2 — child-neighborhood negatives**. Not anchor-neighborhood.

The reason: we want to push away entities that are near the child and could be mistaken for parents. We do NOT want to push entities away from the training parent's neighborhood (that could harm entities that legitimately belong near the training parent).

The definition: **entities that are among the K nearest shallower neighbors of the child in the current Poincaré embedding, excluding known parents**.

This directly mirrors the decoder's candidate set: the decoder looks at the 64 nearest shallower entities and picks parents from them. The hard negatives should come from exactly that population.

### Q2: How many, and how?

**Replace 3 of the 8 norm-stratified with child-neighborhood hard negatives.**

New mix: **5 norm-stratified + 3 child-hard + 2 uniform = 10 total**

Why replace rather than add:
- Adding increases the total negative count, changing the loss landscape for ALL pairs
- Replacing keeps the budget constant — the only change is WHERE the negatives come from
- Simpler to interpret: same loss magnitude, different negative distribution

Why 3:
- With K=64 candidate pool, 3 draws per epoch across 100 epochs = 300 draws
- P(es_clones in the 64-nearest) ≈ very high (it IS one of the nearest)
- P(es_clones sampled at least once) ≈ 1 - (1 - 3/64)^100 ≈ 99%
- Compare to v14: ~89% probability of NEVER being sampled

### Q3: Replace part of the 8/2 or add as a third component?

**Replace part of the 8.** The new mix is a three-component split: 5 + 3 + 2.

The 5 norm-stratified still provide broad parent-zone pressure. The 3 child-hard provide targeted neighborhood pressure. The 2 uniform provide regularization. Clean three-way decomposition.

### Q4: Smallest implementation?

```rust
struct ChildNeighborhoodCache {
    /// For each child entity: its K nearest shallower entities by Poincaré distance.
    candidates: HashMap<usize, Vec<usize>>,
}

impl ChildNeighborhoodCache {
    /// Refresh from current embedding table.
    /// Only computes for entities that appear as children in positive pairs.
    fn refresh(
        &mut self,
        table: &EmbeddingTable,
        pairs: &[WeightedPair],
        exclusions: &HashMap<usize, HashSet<usize>>,
        k: usize,  // 64
    ) {
        for pair in pairs {
            let child = pair.subject;
            let child_emb = table.get(child);
            let child_norm = norm(child_emb);
            // Collect (entity, poincare_distance) for shallower non-excluded entities
            // Sort by distance, take top K
            // Store in self.candidates[child]
        }
    }

    /// Sample n hard negatives for a child from its cached neighborhood.
    fn sample(&self, child: usize, n: usize, rng: &mut impl Rng) -> Vec<usize> {
        // Draw uniformly from self.candidates[child], up to n
    }
}
```

**Refresh frequency**: every 10 epochs. With 20 pairs and 8930 entities, refresh cost is 20 × 8930 = 178,600 Poincaré distance computations, each O(d=50) → ~9M FLOPs. Under 1ms. Could refresh every epoch at negligible cost.

**K = 64**: matches the decoder's `max_candidate_neighbors`. The training and decode stages look at the same candidate population.

**Integration in `train_poincare()`**:

```rust
// Before epoch loop:
let mut hard_cache = ChildNeighborhoodCache::new();

// At epoch start (or every 10 epochs):
hard_cache.refresh(&table, &pairs, &exclusions, 64);

// Per pair:
let norm_neg = norm_sampler.sample(5, child_norm, &exclude, rng);
let hard_neg = hard_cache.sample(child, 3, rng);
let unif_neg = uniform_sampler.sample(2, &exclude, rng);
let negatives = [norm_neg, hard_neg, unif_neg].concat();
```

~40-50 lines of new code. No new loss terms. No changes to RSGD, radial ordering, or the existing sampler infrastructure.

## Why this should work for `hair_follicle → es_clones`

es_clones is shallower than hair_follicle and geometrically close — it is almost certainly in the 64-nearest shallower neighbors. With 3 draws per epoch from a pool of 64, across 100 epochs:

- P(es_clones sampled at least once) ≈ 99%
- Expected times sampled ≈ 100 × 3/64 ≈ 4.7

Compare to v14 (norm-stratified): es_clones was in a pool of ~2700, sampled ~0.3 times expected. Now it's in a pool of 64, sampled ~4.7 times. That's a **15× increase in expected negative exposure** for the specific entity causing the failure.

Each time es_clones is sampled as a negative, the loss says: "push hair_follicle away from es_clones and toward the training parent." After ~5 such gradient steps, the embedding should learn to separate them.

## Risk assessment

**Low**: the change only affects how 3 of 10 negatives are chosen. The loss function, RSGD, and radial ordering are all unchanged. The 5 norm-stratified + 2 uniform provide the same broad coverage as before. The 3 child-hard negatives add concentrated pressure in the specific region that matters.

**Regression path**: if results degrade, revert to v14's 8/2 split. The change is cleanly separable.

## My stance

Child-neighborhood hard negatives at 5/3/2 mix, K=64, refreshed every 10 epochs. This is the smallest sampler-side move that directly addresses the observed underconstraint. If this doesn't work, we've exhausted sampler-side options and should either declare the embedding ceiling or move to a parent-margin loss term.
