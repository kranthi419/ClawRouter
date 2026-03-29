# ClawRouter Multi-Factor Scoring System

The rule-based classifier scores each request across **15 weighted dimensions** and maps the aggregate to a routing tier in under 1ms, entirely locally. Confidence is calibrated via sigmoid — low confidence triggers a fallback LLM classifier.

> All dimensions score against the **user prompt only**. System prompts are excluded to prevent boilerplate tool definitions (skill descriptions, behavioral rules) from dominating the score.

---

## Scoring Dimensions

| # | Dimension | Weight | Score Range | File |
|---|-----------|--------|-------------|------|
| 1 | `reasoningMarkers` | **0.18** | 0 / 0.7 / 1.0 | `src/router/rules.ts`, `src/router/config.ts` |
| 2 | `codePresence` | **0.15** | 0 / 0.5 / 1.0 | `src/router/rules.ts`, `src/router/config.ts` |
| 3 | `multiStepPatterns` | **0.12** | 0 / 0.5 | `src/router/rules.ts` |
| 4 | `technicalTerms` | **0.10** | 0 / 0.5 / 1.0 | `src/router/rules.ts`, `src/router/config.ts` |
| 5 | `tokenCount` | **0.08** | -1.0 / 0 / 1.0 | `src/router/rules.ts`, `src/router/config.ts` |
| 6 | `creativeMarkers` | **0.05** | 0 / 0.5 / 0.7 | `src/router/rules.ts`, `src/router/config.ts` |
| 7 | `questionComplexity` | **0.05** | 0 / 0.5 | `src/router/rules.ts` |
| 8 | `agenticTask` | **0.04** | 0 / 0.2 / 0.6 / 1.0 | `src/router/rules.ts`, `src/router/config.ts` |
| 9 | `constraintCount` | **0.04** | 0 / 0.3 / 0.7 | `src/router/rules.ts`, `src/router/config.ts` |
| 10 | `imperativeVerbs` | **0.03** | 0 / 0.3 / 0.5 | `src/router/rules.ts`, `src/router/config.ts` |
| 11 | `outputFormat` | **0.03** | 0 / 0.4 / 0.7 | `src/router/rules.ts`, `src/router/config.ts` |
| 12 | `domainSpecificity` | **0.02** | 0 / 0.5 / 0.8 | `src/router/rules.ts`, `src/router/config.ts` |
| 13 | `referenceComplexity` | **0.02** | 0 / 0.3 / 0.5 | `src/router/rules.ts`, `src/router/config.ts` |
| 14 | `simpleIndicators` | **0.02** | 0 / -1.0 | `src/router/rules.ts`, `src/router/config.ts` |
| 15 | `negationComplexity` | **0.01** | 0 / 0.3 / 0.5 | `src/router/rules.ts`, `src/router/config.ts` |

---

## Dimension Details

### 1. `reasoningMarkers` — weight 0.18
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.reasoningKeywords` in `src/router/config.ts:126`

Detects requests that require formal deductive or mathematical reasoning.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1 | 0.7 |
| 2+ | 1.0 → **direct REASONING tier override** (bypasses tier boundaries) |

Sample keywords: `prove`, `theorem`, `step by step`, `chain of thought`, `mathematical`, `logically` — in 9 languages (EN, ZH, JA, RU, DE, ES, PT, KO, AR).

---

### 2. `codePresence` — weight 0.15
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.codeKeywords` in `src/router/config.ts:27`

Detects code blocks or programming constructs in the prompt.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1 | 0.5 |
| 2+ | 1.0 |

Sample keywords: `function`, `class`, `import`, `SELECT`, `async`, `await`, ` ``` ` — in 9 languages.

---

### 3. `multiStepPatterns` — weight 0.12
**Scoring function:** `scoreMultiStep()` in `src/router/rules.ts:57`
**No external keyword list** — uses inline regex patterns.

Detects sequential instruction patterns using regular expressions.

| Pattern hit | Score |
|-------------|-------|
| None | 0 |
| Any match | 0.5 |

Patterns: `/first.*then/i`, `/step \d/i`, `/\d\.\s/`

---

### 4. `technicalTerms` — weight 0.10
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.technicalKeywords` in `src/router/config.ts:307`

Detects systems/infrastructure/engineering terminology.

| Matches | Score |
|---------|-------|
| 0–1 | 0 |
| 2–3 | 0.5 |
| 4+ | 1.0 |

Sample keywords: `algorithm`, `optimize`, `kubernetes`, `microservice`, `distributed`, `database` — in 9 languages.

---

### 5. `tokenCount` — weight 0.08
**Scoring function:** `scoreTokenCount()` in `src/router/rules.ts:18`
**Thresholds:** `scoring.tokenCountThresholds` in `src/router/config.ts:24`

Uses total estimated tokens (system + user) — context size informs model selection.

| Token count | Score |
|-------------|-------|
| < 50 | -1.0 (pushes toward SIMPLE) |
| 50–500 | 0 (neutral) |
| > 500 | 1.0 (pushes toward COMPLEX) |

---

### 6. `creativeMarkers` — weight 0.05
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.creativeKeywords` in `src/router/config.ts:384`

Detects open-ended generative requests.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1 | 0.5 |
| 2+ | 0.7 |

Sample keywords: `story`, `poem`, `brainstorm`, `creative`, `imagine`, `write a` — in 9 languages.

---

### 7. `questionComplexity` — weight 0.05
**Scoring function:** `scoreQuestionComplexity()` in `src/router/rules.ts:66`
**No external keyword list** — counts `?` characters directly.

Treats a high number of questions as a signal of multi-faceted complexity.

| `?` count | Score |
|-----------|-------|
| ≤ 3 | 0 |
| > 3 | 0.5 |

---

### 8. `agenticTask` — weight 0.04
**Scoring function:** `scoreAgenticTask()` in `src/router/rules.ts:83`
**Keyword list:** `scoring.agenticTaskKeywords` in `src/router/config.ts:903`

Detects multi-step autonomous task patterns (file ops, execution, iteration). Also produces a standalone `agenticScore` used separately for agentic mode activation.

| Matches | Dimension score | Agentic score |
|---------|----------------|---------------|
| 0 | 0 | 0 |
| 1–2 | 0.2 | 0.2 |
| 3 | 0.6 | 0.6 (triggers auto-agentic mode) |
| 4+ | 1.0 | 1.0 |

Sample keywords: `read file`, `edit`, `deploy`, `debug`, `iterate`, `step 1`, `until it works` — in 7 languages.

---

### 9. `constraintCount` — weight 0.04
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.constraintIndicators` in `src/router/config.ts:557`

Detects explicit constraints that imply more careful model reasoning.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1–2 | 0.3 |
| 3+ | 0.7 |

Sample keywords: `at most`, `maximum`, `minimum`, `limit`, `budget`, `within`, `no more than` — in 9 languages.

---

### 10. `imperativeVerbs` — weight 0.03
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.imperativeVerbs` in `src/router/config.ts:462`

Detects action-oriented commands indicating construction or deployment tasks.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1 | 0.3 |
| 2+ | 0.5 |

Sample keywords: `build`, `create`, `implement`, `design`, `develop`, `deploy`, `configure` — in 9 languages.

---

### 11. `outputFormat` — weight 0.03
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.outputFormatKeywords` in `src/router/config.ts:637`

Detects structured output requirements that push toward more capable models.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1 | 0.4 |
| 2+ | 0.7 |

Sample keywords: `json`, `yaml`, `xml`, `table`, `csv`, `markdown`, `schema`, `structured` — in 9 languages.

---

### 12. `domainSpecificity` — weight 0.02
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.domainSpecificKeywords` in `src/router/config.ts:827`

Detects highly specialized scientific or cryptographic domains that require expert-level knowledge.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1 | 0.5 |
| 2+ | 0.8 |

Sample keywords: `quantum`, `fpga`, `genomics`, `zero-knowledge`, `homomorphic`, `lattice-based`, `photonics` — in 9 languages.

---

### 13. `referenceComplexity` — weight 0.02
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.referenceKeywords` in `src/router/config.ts:681`

Detects prompts that reference prior context, docs, or attachments — indicating multi-turn or document-grounded complexity.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1 | 0.3 |
| 2+ | 0.5 |

Sample keywords: `above`, `below`, `previous`, `the docs`, `the code`, `attached`, `earlier` — in 9 languages.

---

### 14. `simpleIndicators` — weight 0.02
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.simpleKeywords` in `src/router/config.ts:216`

Always returns a **negative** score — actively pushes the weighted total toward SIMPLE.

| Matches | Score |
|---------|-------|
| 0 | 0 |
| 1+ | -1.0 |

Sample keywords: `what is`, `define`, `translate`, `hello`, `yes or no`, `capital of`, `who is` — in 9 languages.

---

### 15. `negationComplexity` — weight 0.01
**Scoring function:** `scoreKeywordMatch()` in `src/router/rules.ts:31`
**Keyword list:** `scoring.negationKeywords` in `src/router/config.ts:758`

Detects negation constraints that add specification complexity.

| Matches | Score |
|---------|-------|
| 0–1 | 0 |
| 2 | 0.3 |
| 3+ | 0.5 |

Sample keywords: `don't`, `avoid`, `never`, `without`, `except`, `exclude` — in 9 languages.

---

## Tier Mapping

After all 15 dimensions are scored, a weighted sum is computed and mapped to a tier:

```
weighted_score = Σ (dimension_score × dimension_weight)
```

| Weighted Score | Tier |
|----------------|------|
| `< 0.0` | **SIMPLE** |
| `0.0 – 0.3` | **MEDIUM** |
| `0.3 – 0.5` | **COMPLEX** |
| `≥ 0.5` | **REASONING** |

Tier boundaries are defined in `scoring.tierBoundaries` in `src/router/config.ts:1029`.

---

## Confidence Calibration

**Function:** `calibrateConfidence()` in `src/router/rules.ts:324`

Confidence is calculated using a sigmoid of the distance from the nearest tier boundary:

```
confidence = 1 / (1 + e^(-steepness × distance))
```

- **Steepness:** 12 (`scoring.confidenceSteepness` in `src/router/config.ts:1036`)
- **Threshold:** 0.7 — below this, tier is `null` (ambiguous) and a fallback LLM classifier is invoked (`scoring.confidenceThreshold` in `src/router/config.ts:1038`)

---

## Special Overrides

| Override | Value | Source |
|----------|-------|--------|
| 2+ `reasoningMarkers` matches | Force **REASONING** tier, confidence ≥ 0.85 | `src/router/rules.ts:272` |
| `agenticScore` ≥ 0.6 (3+ agentic matches) | Activate agentic model tier | `src/router/rules.ts:109` |
| Ambiguous result (confidence < 0.7) | Default to **MEDIUM** tier | `overrides.ambiguousDefaultTier` in `src/router/config.ts:1215` |
| `estimatedTokens > 100,000` | Force **COMPLEX** tier minimum | `overrides.maxTokensForceComplex` in `src/router/config.ts:1213` |
| Structured output detected | Minimum **MEDIUM** tier | `overrides.structuredOutputMinTier` in `src/router/config.ts:1214` |
