---
id: BLFX-2
title: "Structured belief state schema and prompt"
status: Pending
type: implementation
blocked_by: []
unlocks: [BLFX-4]
confidence: high
---

## Problem

Our `ForecastState` is `{probability, summary}`. The paper's belief state is `{probability, confidence_level, key_evidence_for, key_evidence_against, open_questions}`. The paper's ablation in §1 shows removing the structured belief state degrades Brier Index by 3.0 — the second-largest non-search ablation effect.

## Context

Today's `ForecastState` lives in `mneme-substrate/src/activations/forecast/types.rs`. It's the per-trial response shape AND the aggregated state. The forecast SKILL.md prompt asks for `{probability, summary}` JSON.

The paper's `b_t` is updated at each step of the iterative loop (BLFX-4) — but the schema itself can land independently. Adding the fields now lets BLFX-4 just plug into the existing types.

## Evidence

The paper explicitly contrasts the structured belief state with two failure modes (§1, Method §3):
1. **Text accumulation** — appending all retrieved evidence to context (grows unbounded).
2. **Batch search then reason** — N parallel searches, then one reasoning step (no incremental belief update).

Their structured semi-JSON state is the alternative to both. The structure forces the model to maintain explicit pro/con evidence lists and open questions across iterations, rather than relying on context-window recall.

The aggregation impact: `summary` can stay a derived ConcatEvidence over the structured fields (e.g., concat all evidence_for entries across trials). The probability remains the aggregation target.

## Required behavior

`ForecastState` extended:

```rust
pub struct ForecastState {
    pub probability: f64,
    pub confidence: ConfidenceLevel,           // NEW: low | medium | high
    pub evidence_for: Vec<EvidenceItem>,       // NEW
    pub evidence_against: Vec<EvidenceItem>,   // NEW
    pub open_questions: Vec<String>,           // NEW
    pub summary: String,                       // KEEP — derived from evidence; backwards compat
    // existing aggregation metadata
    pub n_trials: u8,
    pub prior_used: Option<PriorRef>,
}

pub struct EvidenceItem {
    pub claim: String,                         // one-sentence claim
    pub source: Option<String>,                // URL or "training" or "user-provided"
    pub weight: f64,                           // model's self-rated impact in [0, 1]
}

pub enum ConfidenceLevel {
    Low, Medium, High,
}
```

Forecast SKILL.md updated to demand this shape. The fenced JSON block becomes:

```json
{
  "probability": 0.42,
  "confidence": "medium",
  "evidence_for": [
    {"claim": "...", "source": "https://...", "weight": 0.7}
  ],
  "evidence_against": [...],
  "open_questions": ["..."]
}
```

(`summary` is auto-generated from the structured fields by the substrate; the model doesn't need to produce it separately.)

## Risks

| Risk | Mitigation |
|------|-----------|
| Larger schema → more parse failures | The schema validator (mneme/respond/schema.rs) returns a clear error per missing/wrong-type field; trials retry up to 3 times |
| Summary auto-generation produces redundant prose | Define a deterministic template: "FOR: {evidence_for joined}; AGAINST: {evidence_against joined}; OPEN: {open_questions joined}" |
| Backward incompatibility breaks existing programs/<id>/artifact.json files | Add `belief_schema_version: "0.2.0"` field; readers branch on version |

## What must NOT change

- The `forecast.update` external API shape (parameters stay the same).
- Existing program directories on disk (older ones keep their old schema; new ones get v0.2.0).

## Acceptance criteria

1. `ForecastState` and the new `EvidenceItem` / `ConfidenceLevel` types in `forecast/types.rs`, all `Serialize + Deserialize + JsonSchema`.
2. Forecast SKILL.md updated to demand the new shape; old `{probability, summary}` shape is no longer accepted.
3. The aggregation in `forecast/activation.rs::run_update_in_background` produces the new shape: probability via LogitShrinkage, summary via deterministic template, evidence_for/against by concatenating across trials, open_questions by union.
4. Unit tests cover: round-trip serde, shape validation, summary template determinism, cross-trial evidence merging.
5. Live verification: `synapse forecast update` against a running substrate produces an artifact with all fields populated.
6. `cargo test --lib` passes green.

## Completion

Implementor runs the live test, inspects the resulting artifact.json, confirms all fields present and well-formed; status flips to `Complete` in the same commit as the code.
