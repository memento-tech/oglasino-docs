# Content Moderation v2 (Parked)

**Status:** parked
**Revisit:** 2–3 months post-launch, once real abuse data is available

This document preserves the policy-scoring engine proposal that was raised during the validation refactor. The current v1 system (binary `Signal { Pass | Violation }` from rule-based analyzers) ships and works. v2 is the right direction, but it's premature without production data.

---

## The proposal

Replace the current binary moderation pipeline with a policy-scoring engine that aggregates weighted signals into a single suspicion score, then maps that score to one of four decisions.

### Current architecture

```
Signal → Violation → Block
```

Each analyzer (`KeywordStuffingAnalyzer`, `SpammyDescriptionAnalyzer`, `RepeatingRule`, `GibberishAnalyzer`, `LanguageDetectorAnalyzer`) returns either a `Violation` or nothing. Any single violation blocks the content. No severity calibration. No cross-signal reasoning.

### Proposed architecture

```
Signal → Score → Aggregation → Policy Decision
```

#### Layer 1 — Feature extraction

Each analyzer emits:

```java
record Signal(
  SignalType type,
  double score,        // 0.0 – 1.0
  double confidence    // 0.0 – 1.0
) {}
```

No blocking logic at the analyzer level.

#### Layer 2 — Policy scoring engine

```
finalScore = Σ(signalScore × weight(signalType, field))
```

Example weights (illustrative; real weights would be calibrated from labelled data):

| Signal type | Weight |
|---|---|
| keyword stuffing | 0.40 |
| repetition | 0.30 |
| entropy anomaly | 0.20 |
| language mismatch | 0.10 |

#### Layer 3 — Decision engine

| Score range | Action |
|---|---|
| 0.00–0.40 | ALLOW |
| 0.40–0.70 | WARN |
| 0.70–0.90 | REVIEW |
| 0.90–1.00 | BLOCK |

---

## Why this is the right long-term direction

The current strengths to preserve:

- Strong signal diversity (lexical repetition, character-level spam, statistical randomness, language inference)
- Domain-aware constraints (name vs description, field-specific entropy, language-aware thresholds)
- Lightweight, deterministic, fully explainable, no ML dependency

The current limitation: every signal is a hard trigger. This causes:

- False positives on natural repetition ("laptop laptop charger")
- No severity calibration (one banned word = same impact as ten)
- No cross-signal reasoning (high entropy + banned word + repetition should compound)
- No context weighting (a banned word in a title is more suspicious than in a description)

The scoring layer fixes all four.

---

## Why we're not building it now

1. **No calibration data.** Oglasino isn't in production. The weights above are guesses. Real weights come from labelled examples — listings flagged or unflagged by moderators across actual user traffic.
2. **It invalidates Phase 2.5.** The just-shipped `Signal { Pass | Violation }` sealed interface, `LocalContentModerator`, every analyzer's return type, and validator emit-first-violation behavior would all change.
3. **WARN vs BLOCK is a product decision.** Does the user see WARN as a soft block they can override? A submission that goes through with a flag for human review? An invisible flag for analytics only? This needs a real product choice, which needs real UX context.
4. **REVIEW needs a moderator surface.** Today Oglasino has no moderator queue. Building one is a separate feature.
5. **Risks introducing regressions during launch.** The current binary system is tested (274 backend tests, 100/100 web tests, 37-case golden set). Replacing it during the most fragile period is bad timing.

---

## What needs to happen before unparking

Conditions for revisiting:

- At least 2 months of production traffic with content moderation events logged
- A labelled dataset: at least 500 listings tagged by Igor (or a moderator) as clean / borderline / spam / malicious
- A product decision documented in `decisions.md` answering: what does WARN do? Does the user know? Does it block submission? Does it route to a queue?
- A moderator queue exists (or a decision that REVIEW maps to BLOCK + email-to-Igor in v2.1)
- Calibration analysis: per-signal-per-field weights derived from labelled data using logistic regression or similar

When all five exist, this proposal moves out of `future/` and into `features/content-moderation-v2.md` as a planned feature.

---

## Migration sketch (when we eventually do it)

Roughly:

1. Introduce the `Signal { type, score, confidence }` record alongside the existing sealed interface. Don't remove the old one yet.
2. Update each analyzer to emit both: keep returning `Optional<Signal.Violation>` for the current pipeline, also emit a scored `Signal` for the new pipeline.
3. Introduce `ContentPolicyEngine` as a new service. Initially shadow-mode: it computes a score for every moderation call but doesn't act on it. Log scores alongside actual decisions.
4. Collect 2–4 weeks of shadow data. Calibrate weights so the engine's decisions match historical decisions on clean content and improve on borderline content.
5. Flip the engine to authoritative. Remove the binary path. Delete the old `Signal { Pass | Violation }` sealed interface.

Total estimated effort: 2–3 weeks of focused work, plus the calibration data collection window.

---

## Reference

Original analysis preserved as historical context — see the validation refactor session archives in `sessions/` once they're written up.
