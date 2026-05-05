# AGENTS.md

Repository-level instructions for human and LLM contributors.

## Scope

Applies to [example: the entire workspace].

## Mission

[mission]

## Guiding Philosophy

All code in this repository is governed by a single design principle:

**Algebraic, homomorphic, compositional — derived from denotational semantics.**

This means:

1. **Algebraic.** Every domain concept is modeled as an algebraic structure with explicit operations and laws. Entities form a set. Relations form a category. Embeddings are elements of a metric space with defined operations. If a concept does not have a clear algebraic identity, it is not ready to be coded.

2. **Homomorphic.** Transformations between representations must be structure-preserving. Example: `The map from triples to embeddings preserves relational ordering. The map from embeddings to materialized RDF preserves geometric predicates. The map from text to entities preserves referential identity. Every pipeline stage is a morphism, and composition of stages must equal direct transformation where both are defined.`

3. **Compositional.** Complex behavior arises from composing simple, well-defined parts. Example: `Each crate exposes a small algebra. Pipeline stages compose via shared types. No stage reaches into another stage's internals. If two stages need to share logic, extract it into a shared algebra in a lower crate.`

4. **Denotational.** Every type has a clear mathematical denotation — what it *means*, not just how it's implemented. Example: `An 'EmbeddingTable' denotes a function from entities to points in a metric space. A 'Triple' denotes a morphism in a knowledge category. A 'Pattern' denotes a context predicate over token streams. Code review asks: "does this implementation faithfully represent its denotation?"`

In practice this means:
- Prefer pure functions. Side effects (I/O, randomness) are pushed to the edges.
- Prefer types that enforce invariants (e.g., `PoincarePoint` that guarantees ‖x‖ < 1) over runtime checks on raw arrays.
- Prefer small composable transforms over monolithic processing functions.
- Every public function's type signature should be readable as a mathematical statement.
- When in doubt, ask: "what is the algebra here?" If there isn't one, introduce one before writing the implementation.

## Mandatory Context Bootstrap

Before making architectural or broad changes, read these in order:

1. `ARCHITECTURE.md` — full architectural spec, design patterns, and experiment protocols.
2. `docs/DECISIONS.md` — locked design decisions.
3. `docs/STATUS.md` — current implementation state.
4. `docs/EXECUTION_TRACKER.md` — `Done / Next / Needed` progression.

Do not infer status from memory. Use:
1. `docs/STATUS.md` for current implementation state.
2. `docs/EXECUTION_TRACKER.md` for `Done / Next / Needed` execution state.
3. Resume from highest-priority open item in `docs/EXECUTION_TRACKER.md` unless explicitly redirected.

## Non-Negotiable Constraints

[constraints]

## Algebraic Design Patterns

> [!NOTE] 
> The following sections are examples.

### Newtypes over raw arrays

Wrap arrays in domain types that enforce geometric invariants:

```rust
/// A point in the Poincaré ball model of hyperbolic space.
/// Invariant: ‖inner‖ < 1.
pub struct PoincarePoint(Array1<f32>);

/// A point in Minkowski space ℝ^{1,D-1}.
/// First component is temporal; remainder is spatial.
pub struct MinkowskiPoint(Array1<f32>);

/// A tangent vector at a point on a Riemannian manifold.
pub struct TangentVector { base_point: PoincarePoint, vector: Array1<f32> }
```

Construction enforces invariants. Operations preserve them. Raw array access is available but explicitly marked as such (e.g., `.as_array()` or `.into_inner()`).

### Morphisms as traits

Each pipeline stage is a morphism. Express this as a trait:

```rust
/// A structure-preserving map from one representation to another.
trait Morphism<From, To> {
    fn apply(&self, input: &From) -> To;
}
```

The bootstrap pattern extractor is a `Morphism<(EntityVocabulary, TokenStream), TypedTriples>`.
The Poincaré trainer is a `Morphism<TypedTriples, EmbeddingTable>`.
The geometric decoder is a `Morphism<EmbeddingTable, MaterializedOntology>`.

Composition of morphisms must be associative and well-typed.

### Algebras as modules

Each crate exposes an algebra:

- `geo`: the algebra of geometric spaces (distance, exp, log, transport, projection).
- `extract`: the algebra of linguistic evidence (entities, contexts, patterns, triples).
- `embed`: the algebra of training (loss functions, gradient steps, convergence).
- `decode`: the algebra of geometric predicates (cone membership, interval classification, threshold logic).
- `validate`: the algebra of empirical measurement (accuracy, stability, compression).

## Workspace Structure

```
geo/
├── Cargo.toml                    # virtual workspace manifest
├── AGENTS.md                     # this file
├── ARCHITECTURE.md               # pipeline spec (mathematical reference)
├── docs/
│   ├── DECISIONS.md              # locked design decisions
│   ├── STATUS.md                 # current implementation state
│   └── EXECUTION_TRACKER.md      # Done / Next / Needed
├── crates/
│   ├── geo/                      # geometric space primitives
│   ├── extract/                  # text → entities 
│   ├── embed/                    # entities → embeddings
│   ├── decode/                   # embeddings → materialized 
│   ├── validate/                 # validation + metrics
│   └── pipe/                     # pipeline orchestration + CLI
├── corpora/                      # test corpora (gitignored)
└── results/                      # pipeline outputs (gitignored)
```

## Quality Gates

### Required before finalizing any change

1. `cargo clippy --workspace --all-targets -- -D warnings` (pedantic where practical).
2. `cargo test --workspace` (all tests pass).
3. `cargo doc --workspace --no-deps` (no rustdoc warnings).

### Code style

1. `rustfmt` with nightly using repo `rustfmt.toml`.
2. No `unwrap()` in library code. Use typed errors.
3. No `clone()` in hot paths without explicit justification in a comment.
4. Every `pub fn` has a rustdoc comment that states the mathematical operation it performs.
5. Every module has a module-level doc comment stating the algebra it provides.

## Documentation Discipline

When implementation state changes, update docs in the same change set:

1. `docs/STATUS.md` for implementation truth.
2. `docs/EXECUTION_TRACKER.md` for execution progression.
3. Relevant rustdoc comments for API contract changes.
4. `ARCHITECTURE.md` only if the pipeline architecture itself changes (not for implementation details).

## Performance Expectations

[performance]

## Execution Terminology

[terminology]

## Development Sequence

[development]
