# Aggregation in logit space

## What this doc is

Mneme fans out K parallel reasoning trials per question. Each trial
returns its own probability `p_k`. Combining those K answers into a
single forecast is the **aggregation step**, and BLF specifies that it
happens in *logit space* — log-odds — rather than directly on the
probabilities.

This page explains why that matters, what the math looks like, and
where it lives in the substrate.

## The transform

For any probability `p ∈ (0, 1)`, the logit (a.k.a. log-odds) is

```
logit(p) = log( p / (1 − p) )
```

It maps `(0, 1) → (−∞, +∞)`, with `logit(0.5) = 0`.

Aggregation across K trials is then:

```
p_final = sigmoid( (1/K) · Σ_k logit(p_k) )
```

where `sigmoid` is the inverse of `logit`:

```
sigmoid(x) = 1 / (1 + e^−x)
```

So: take the K probabilities, transform each to log-odds, average them,
transform back. That's the entire operation.

## Why not just average the probabilities

Probabilities aren't a linear scale of evidence. Three reasons logit
space is the right one:

### 1. Distance is non-uniform on `[0, 1]`

The "size" of a step in probability depends on where you are. From 0.50
to 0.51 is a trivial shift in evidence (1.04× the odds). From 0.98 to
0.99 is enormous — odds went from 49:1 to 99:1, double the evidence.
Plain averaging treats those steps as equal and washes out the
high-confidence end.

### 2. Logit space is additive in evidence

Each independent piece of evidence — each "I read an article that said
X" — shifts the log-odds by an approximately constant amount. That's
because Bayes' rule in log-odds form is

```
log-odds(posterior) = log-odds(prior) + log( likelihood ratio )
```

If trials are independent estimates of the truth, averaging their
log-odds is the natural Bayesian combination. Averaging probabilities
is not what Bayes' rule says to do — it just happens to give an
answer-shaped number.

### 3. Symmetric around 0.5

The transform makes "very confident YES" and "very confident NO"
mirror images: `logit(0.99) = +4.60`, `logit(0.01) = −4.60`. Plain
averaging is symmetric only near 0.5 and gets increasingly biased near
the tails.

## Worked example

Three trials produce `p_1 = 0.50`, `p_2 = 0.50`, `p_3 = 0.99`.

**Plain mean:**

```
p_final = (0.50 + 0.50 + 0.99) / 3 = 0.663
```

The two indecisive trials water down the confident one.

**Logit mean:**

```
logit(0.50) = 0
logit(0.50) = 0
logit(0.99) = log(0.99 / 0.01) = log(99) ≈ 4.60

mean         = (0 + 0 + 4.60) / 3 = 1.53
p_final      = sigmoid(1.53) = 1 / (1 + e^-1.53) ≈ 0.823
```

The trial that actually saw evidence dominates, instead of being
diluted by the two that did not. **This is the point of the
transform.**

The intuition: trial 3 saw something that shifted its log-odds by 4.6
units of evidence; trials 1 and 2 saw nothing. Logit averaging keeps
the evidence and softens it across K; probability averaging discards
the evidence by the time it's done.

## The shrinkage parameter (λ)

Murphy 2026 generalizes the operation with a shrinkage / temperature
parameter `λ ∈ [0, 1]`:

```
p_final = sigmoid( (1 − λ) · mean(logit(p_k)) )
```

- `λ = 0` — no shrinkage. The pure logit mean.
- `λ = 1` — full shrinkage. Every output collapses to 0.5 regardless
  of the trials.
- `λ ∈ (0, 1)` — interpolate between them. Pulls the aggregated
  log-odds back toward 0 (probability 0.5) by a fixed factor.

Shrinkage is a regularizer for noisy trials. If your trials disagree
wildly because they're not actually independent samples of the truth,
small λ reduces overconfidence.

**Mneme's default: λ = 0.0**, per Murphy 2026 §4: optimal α = 1.0
(equivalently λ = 0) on ForecastBench. Setting λ > 0 systematically
blunts confident predictions; we measured this as net-negative against
the baseline. An earlier mneme default of `λ = 0.2` was the single
biggest algorithmic divergence from the paper and was the first thing
fixed when we started replicating bench-005.

LOO-CV (leave-one-out cross-validation) tuning of λ from the
calibration store — what the paper actually does — is BLFX-5; not yet
built. The current static-default approach is the right choice when
you have <100 resolved observations and noisy LOO estimates.

## Implementation

- Code: `mneme-substrate/src/mneme/swarm/aggregate/logit.rs`
- Default: `λ = 0` (Murphy 2026 §4)
- Tests: round-trip identities, edge cases at `p ∈ {0, 1}` clamp to
  `(ε, 1-ε)` to avoid `±∞`, three-trial sanity case.

The aggregation happens after K trials return their `ForecastState`
objects — see `mneme-substrate/src/activations/forecast/activation.rs`
for the call site (`run_aggregate` in the background task body).

## Outside-the-paper notes

- **Trials must be roughly independent.** If all K trials are forks of
  the same parent claudecode session and see identical evidence, the
  variance reduction from aggregation is small. Mneme uses the
  `diversify` parameter ("Reasoning style #%i — analytic / contrarian /
  base-rate-grounded") to inject orthogonality. Without diversification,
  K=2 ≈ K=1 in effective sample size.
- **Calibration is downstream of aggregation.** The aggregated
  probability is fed to Platt calibration (see
  `mneme/calibration/platt.rs`), which is fit from past
  `(predicted, actual)` rows. The two operations are independent: logit
  averaging is *between trials*; Platt is *between mneme and the world*.
- **The structured belief state is averaged separately.** Logit
  averaging applies to the scalar `probability` field. The
  `evidence_for` / `evidence_against` / `open_questions` arrays are
  unioned across trials and rendered into the final prose summary. The
  aggregated probability is the substrate's output; the structured
  fields are the substrate's *audit trail*.

## Reference

- Murphy 2026, *Bayesian Linguistic Forecaster*, §4 (aggregation):
  [arXiv:2604.18576](https://arxiv.org/abs/2604.18576)
- BLFX-5 (LOO-CV α tuning): `plans/BLFX/BLFX-5.md`
