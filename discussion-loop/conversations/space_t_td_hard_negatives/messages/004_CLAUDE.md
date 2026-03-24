# Consensus — closing

Agreed on all points. GPT's refinement about per-epoch refresh is correct — the cost is negligible and eliminates a confound.

The one nuance GPT adds that I want to explicitly endorse: **the 8/2 law stays for children without TD parents, 5/3/2 only applies when a TD parent exists.** This is clean — children without TD parents have no training anchor to define "wrong-best-parent," so generic parent-zone pressure is the best we can do for them.

## Locked consensus

1. **5/3/2 mix** for TD-anchored children: 5 norm-stratified + 3 child-hard + 2 uniform
2. **8/2 mix** for non-TD children: 8 norm-stratified + 2 uniform (unchanged from v14)
3. Child-hard pool: 64 nearest shallower entities by Poincaré distance, excluding positive/transitive parents
4. Refresh: every epoch from live table
5. Fixed negative budget (10 per positive)
6. No new loss terms
7. Rerun synthetic + LLL + CRAFT

## Decision gate

| Outcome | Interpretation | Next move |
|---------|---------------|-----------|
| es_clones removed, rest stable | Sampler-side frontier closed | Next pipeline stage |
| es_clones survives, rest stable | Sampler-side exhausted | TD-parent margin loss OR declare ceiling |
| Regression | Hard negatives too aggressive | Revert to v14 8/2 |

If this fails, sampler-side escalation is exhausted. Ready for implementation.
