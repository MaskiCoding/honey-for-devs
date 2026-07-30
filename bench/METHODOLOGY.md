# Methodology

Pre-registered before the next paid run. The point of writing it down is that the
endpoints can't be chosen after seeing the numbers.

## Why this exists

Two published "token saving" add-ons have now been measured by an independent paired
benchmark and both came in far below their claim — [caveman](https://blog.jetbrains.com/ai/2026/06/caveman-claude-code-token-savings/)
(advertised −65%, measured −8.5%) and [rtk](https://blog.jetbrains.com/ai/2026/07/rtk-claude-code-token-savings/)
(advertised −60–90%, measured **+7.6% more expensive** at low reasoning effort, p=0.004,
with quality tied).

rtk's own dashboard reported 96.2M tokens saved across the same trials where the invoice
went up. Three mechanisms did that, and all three are general:

1. **Wrong counterfactual.** It scored the full raw output as what "would have" been
   billed, including outputs the harness truncates anyway.
2. **Wrong token class.** It counted tokens at execution time; most of a session's input
   cost is cached re-reads billed at a tenth of the price.
3. **Wrong denominator.** The lever only ever touched a fraction of context.

Honey's lever is on the other side of the pipe — **output** tokens, uncached, billed at
5× input — so it is not exposed to (2) and (3) the way rtk was. But nothing about being
on the right side of the pipe protects a *claim*. The rules below exist so Honey's numbers
are the ones an independent paired benchmark would reproduce.

## Endpoints

**Primary**

| Endpoint | Definition |
|---|---|
| Δ output tokens | paired per-task median vs the control arm |
| Δ cost | paired per-task median vs the control arm, all four token classes |

**Secondary**

| Endpoint | Definition |
|---|---|
| Δ new-input | fresh + cache-creation tokens — the class where a skill prompt *costs* |
| Δ turns | paired median agent iterations (harness bench only — the retry tax) |

**Quality**

| Endpoint | Definition |
|---|---|
| Test pass-rate | primary, objective: extracted code run against a real unit test |
| Judge sign test | secondary: exact two-sided sign test over per-task wins/losses |

A skill that cuts output while adding agent turns has saved nothing. That is the failure
mode that made rtk net-negative, and `Δ turns` is the only endpoint that can see it — the
single-call bench (`src/run.js`) structurally cannot.

## Statistics

- **Paired only.** Every delta compares the same task across arms. A task missing or
  errored in either arm is excluded from **both**.
- **Runs collapse by median**, not mean, so one bad sample can't move a cell.
- **Never a ratio of arm totals.** `sum(honey)/sum(baseline)` is dominated by whichever
  task happens to be longest. Reported for volume only, never as the claim.
- **Wilcoxon signed-rank**, two-sided, for continuous endpoints. Below 6 non-tied tasks no
  p-value is reported at all — the exact two-sided p cannot reach 0.05 there.
- **Exact sign test** for judge scores: ordinal and noisy, and a mean-of-means hides
  per-task losses. (Worked example: on `full-opus48` caveman's *mean* judge ties baseline
  exactly — 94 vs 94 — while the sign test shows it losing 16 of 23 tasks, p=0.004.)
- **`(ns)` is printed on any delta that misses p<0.05.** A tie gets called a tie.

Implementation: [`src/paired.js`](src/paired.js), unit-tested in
[`../tests/paired.test.js`](../tests/paired.test.js) against exact enumeration.

## Run ladder

Never quote k=1. In order, cheapest first:

1. **Free replay** — recompute over committed stamps: `node src/report.js --stamp <stamp>`.
   No API spend. Every methodology change must be validated here first.
2. **Ceiling analysis** — before claiming a lever helps, compute from real transcripts what
   share of the bill it can even touch. This is the free step that predicted rtk's result,
   and it is the step that argues against building rtk-style Bash-output filtering here.
3. **Wiring check** — one cell, live, to prove the treatment actually fires.
4. **Smoke** — a task subset at `RUNS=1`. Diagnostic only, never quoted.
5. **`RUNS=3` on the same subset** — most k=1 scares are sampling noise.
6. **Full suite at `RUNS=3`**, per provider.

## Instrument the treatment, don't assume it

Every result set snapshots each variant's resolved system prompt plus its hash
(`results/<stamp>/systems/`, `meta.variant_hashes`), so "the skill didn't help" can always
be distinguished from "the skill never loaded".

## Known limits

Carried from [README](README.md#honest-limits), plus what this document adds:

- **23 tasks, author-written.** Enough to see an effect; not a leaderboard. The neutrality
  safeguards cover the judge, the metric and variant drift — not task selection. An
  independent external suite is the open gap.
- **Caching is modelled, not invoiced.** `pricing.json` rates are approximate and the
  cache multipliers are list prices, not a contract.
- **The single-call bench sees no agent loop.** Δ turns only exists on the Cline harness
  bench (`src/cline-bench.js`); treat single-call cost deltas as an upper bound on the
  benefit, since they cannot show a retry tax.
- **Provider caching differs and is not yet equalised.** On `full-opus48` the skill prompt
  lands in `cache_write` once and is read back cheaply, so the typical task pays ≈no extra
  new input (paired Δ new-input −1%) even though the sweep total is +38%. On `full-gpt55`
  every recorded `cache_read` is 0, so every task pays the prompt fresh: Δ new-input
  **+573%** and Δcost **+14% (ns)** despite Δoutput of −20% (p=0.004).
  The harness maps OpenAI's `input_tokens_details.cached_tokens` correctly
  ([`src/client.js:112`](src/client.js:112)), so the zero is what the API reported, not a
  recording bug — but *why* automatic caching never engaged across 23 tasks × 3 runs
  sharing one system prefix is **unresolved**. Until it is, the gpt-5.5 cost delta is
  evidence of neither a saving nor a penalty; the output delta stands on its own.
