# CLAUDE_SUMMARY — space_t_td_hard_negatives

## Discussion outcome

v14 norm-stratified sampling was safe but insufficient — es_clones confidence barely moved (0.9748 → 0.9736). The escalation is **child-neighborhood hard negatives**: sample negatives from the child's immediate geometric neighborhood, directly targeting the decoder-competitive candidate population.

## v14 result (baseline for this tranche)

| Metric | v13 (containment) | v14 (norm-stratified) |
|--------|-------------------|----------------------|
| CRAFT precision | 85.7% | 85.7% |
| es_clones confidence | 0.9748 | 0.9736 |
| TD parent confidence | 0.9360 | 0.9448 |
| SubClassOf volume | 1329 | 1327 |

Gap between es_clones and TD parent: 0.0388 → 0.0288. Narrowed but didn't flip.

## Why norm-stratified was insufficient

Norm-stratified draws from ~2,700 parent-zone entities. P(es_clones sampled at least once across training) ≈ 11%. Even when sampled, it's diluted among thousands of other parent-zone entities. The gradient pressure on es_clones specifically is near zero.

## The child-neighborhood approach

### Definition

For each positive pair (child C, parent P), maintain a cache of K=64 **nearest shallower entities** to C by Poincaré distance, excluding known parents. Sample hard negatives from this cache.

### Why child-neighborhood, not anchor-neighborhood

- **Anchor-neighborhood** (entities near TD parent): wrong target — pushes entities away from the correct parent's region, potentially harming legitimate ancestors
- **Child-neighborhood** (entities near the child): right target — pushes away entities the decoder would mistakenly select as parents. Directly mirrors the decoder's candidate set.

### Negative mix

| Child type | Norm-stratified | Child-hard | Uniform | Total |
|-----------|----------------|------------|---------|-------|
| TD-anchored | 5 | 3 | 2 | 10 |
| Non-TD | 8 | 0 | 2 | 10 |

Budget stays fixed at 10 negatives per positive — no loss landscape change.

### Expected effect on es_clones

es_clones IS one of the 64 nearest shallower entities to hair_follicle (this is the entire problem — it's geometrically close and shallower).

| Metric | v14 (norm-stratified) | v15 (child-hard) |
|--------|----------------------|------------------|
| Candidate pool size | ~2,700 | 64 |
| P(sampled at least once) | ~11% | ~99% |
| Expected sample count | ~0.3 | ~4.7 |

A **15× increase** in expected negative exposure for the specific entity causing the failure.

## Implementation

### ChildNeighborhoodCache

```rust
struct ChildNeighborhoodCache {
    candidates: HashMap<usize, Vec<usize>>,  // child → K nearest shallower entities
}

impl ChildNeighborhoodCache {
    fn refresh(
        &mut self,
        table: &EmbeddingTable,
        pairs: &[WeightedPair],
        exclusions: &HashMap<usize, HashSet<usize>>,
        k: usize,  // 64
    ) { ... }

    fn sample(&self, child: usize, n: usize, rng: &mut impl Rng) -> Vec<usize> { ... }
}
```

### Integration in `train_poincare()`

```rust
// Before epoch loop:
let mut hard_cache = ChildNeighborhoodCache::new();
let td_children: HashSet<usize> = /* children with TD parents */;

// At each epoch start:
hard_cache.refresh(&table, &pairs, &exclusions, 64);

// Per pair:
if td_children.contains(&child) {
    let norm_neg = norm_sampler.sample(5, child_norm, &exclude, rng);
    let hard_neg = hard_cache.sample(child, 3, rng);
    let unif_neg = uniform_sampler.sample(2, &exclude, rng);
    negatives = [norm_neg, hard_neg, unif_neg].concat();
} else {
    let norm_neg = norm_sampler.sample(8, child_norm, &exclude, rng);
    let unif_neg = uniform_sampler.sample(2, &exclude, rng);
    negatives = [norm_neg, unif_neg].concat();
}
```

### Refresh cost

20 pairs × 8930 entities × O(50) per Poincaré distance = ~9M FLOPs per refresh. Under 1ms. Refreshed every epoch — negligible overhead.

### Scope

- ~40-50 lines new code (ChildNeighborhoodCache + integration)
- No new loss terms
- No changes to RSGD, radial ordering, or existing samplers
- Only affects `train_poincare()` path

## Decision gate

| Outcome | Interpretation | Next move |
|---------|---------------|-----------|
| es_clones removed, rest stable | **Sampler frontier closed** | Next pipeline stage |
| es_clones survives, rest stable | **Sampler frontier exhausted** | TD-parent margin loss OR declare ceiling at 85.7% |
| Regression | Hard negatives too aggressive | Revert to v14 8/2 |

## Escalation path status

1. ~~v10 — Taxonomic calibration~~ ✓
2. ~~v11 — Parsimonious ancestry~~ ✓
3. ~~v12 — Branch consistency (overlap)~~ ✓
4. ~~v13 — Branch consistency (containment)~~ ✓
5. ~~v14 — Norm-stratified negatives (8/2)~~ ✓ safe, insufficient
6. **v15 — Child-neighborhood hard negatives (5/3/2)** ← this tranche
7. TD-parent margin loss ← if v15 fails
8. Declare embedding ceiling ← if v15 and margin loss both fail
