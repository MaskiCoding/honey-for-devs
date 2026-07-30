# Honey benchmark results

model: `claude-fable-5` · judge: `claude-haiku-4-5` · tasks: 23 · runs: 3

## Paired (headline)

| Variant | n | Δ output | p | Δ LOC | p | Δ new-input | p | Δ cost | p | Judge W/L/T | sign p |
|---------|--:|---------:|--:|------:|--:|------------:|--:|-------:|--:|------------:|-------:|
| honey | 23 | -32% | 0.005 | -44% | 0.003 | -1% | <0.001 | -13% (ns) | 0.110 | 5/4/9 | 1.000 |
| honey-lean | 23 | -9% (ns) | 0.616 | -26% (ns) | 0.112 | -1% | <0.001 | -7% (ns) | 0.843 | 7/2/9 | 0.180 |

Every Δ is a **per-task** delta vs `baseline`: runs collapse by median, tasks pair up,
and the column is the median of those paired deltas. `p` is a two-sided Wilcoxon
signed-rank test (`—` when fewer than 6 non-tied tasks — no significance is claimed there).
`(ns)` marks a delta that misses p<0.05: a tie, not a win. **Judge W/L/T** counts tasks
where the variant scored above / below / level with baseline, tested with an exact sign
test — the right test for ordinal, noisy judge scores, and one a mean-of-means can hide.

- **Δ output** — the headline lever: volume each skill directly controls, caching-independent.
- **Δ new-input** — fresh + cache-creation tokens, the class where a skill prompt *costs*.
  Cache reads are excluded: they bill at a tenth and dominate a long session regardless.
- **Δ cost** — all four token classes at `bench/pricing.json` rates.

## Arm totals

| Variant | Tests pass | Judge ±sd | Judge vs base | Output tok | Output vs base | New input tok | $ (cached) | $ (cold) | CO₂ (g) |
|---------|-----------:|----------:|--------------:|-----------:|---------------:|--------------:|-----------:|---------:|--------:|
| baseline | 71% | 90 ±23 | 100% | 102,095 | +0% | 13,683 | $5.242 | $5.242 | 2244.4 |
| honey | 97% | 94 ±10 | 105% | 80,163 | -21% | 18,576 | $4.549 | $7.615 | 1762.2 |
| honey-lean | 93% | 92 ±21 | 102% | 104,252 | +2% | 14,645 | $5.437 | $6.107 | 2291.8 |

Volumes and absolute costs. The *vs base* column here is a ratio of sums and is
outlier-sensitive — one long task can drive it; prefer the paired table above.

- **Tests pass** — objective: extracted code run against unit tests.
- **Judge ±sd** — LLM-as-judge (0-100, panel median) with per-record stdev. A judge gap
  inside ±sd is noise, not a quality win. Rubric: `plain`.
- **$ (cached)** — steady state: cache reads at ≈10% of input, cache *creation* charged.
  **$ (cold)** — first-turn worst case: every input token billed fresh. Real cost sits
  between, nearer cached as a session lengthens. Rates in `bench/pricing.json`.
- **CO₂** via EcoLogits port (`hooks/eco.js`), from output tokens.
