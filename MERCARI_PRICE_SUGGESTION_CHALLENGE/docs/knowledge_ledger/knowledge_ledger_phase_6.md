# Knowledge Ledger — Phase 6: Iterative Model Benchmarking

## Phase Goal

Phase 6 turns the prepared data and candidate feature families from Phase 5 into evidence.

The core rule is:

> **Change one meaningful thing → measure it → keep or reject it.**

All experiments use the same fixed 80/20 validation split and the same RMSLE definition, so scores remain directly comparable.

---

## 6.1 Benchmark Setup

**Decision**

Create a consistent experiment framework before comparing models.

**Key findings**

- Validation split: 80/20, `random_state=42`
- Training rows: 1,186,028
- Validation rows: 296,507
- Target for modeling: `log1p(price)`
- Evaluation: RMSLE
- Competition test remains isolated
- Experiment registry stores model name, RMSLE, runtime, and improvement versus the simple benchmark

**Extracted knowledge**

- A fixed validation set is essential for fair experimentation.
- RMSLE matches the competition objective and naturally emphasizes relative/log-scale errors.
- Experiment results should be recorded instead of relying on notebook memory.

**Status:** COMPLETE

---

## 6.2 TF-IDF + Ridge — Text Only

**Decision**

Use the existing word-level TF-IDF representation for `name` and `item_description`, then train Ridge on the log-transformed target.

**Configuration**

- Ridge: `alpha=1.0`
- Solver: `lsqr`
- Initial TF-IDF: 5,000 features for each text field

**Key result**

- Validation RMSLE: **0.532397**
- Training runtime: ~9 seconds

**Extracted knowledge**

- Text alone is highly predictive.
- A sparse linear model is a natural fit for high-dimensional TF-IDF data.
- The experiment clearly beats the simple category-brand median benchmark of **0.593385**.

**Status:** COMPLETE

---

## 6.3 TF-IDF + Structured + Ridge

**Decision**

Add the prepared structured feature representation to the text representation.

**Structured representation**

- 60,005 one-hot categorical columns
- 10 numeric columns
- 60,015 structured columns total
- 10,000 initial text columns
- 70,015 combined columns

**Included structured families**

- Original categorical features
- Category hierarchy
- Categorical interactions
- Frequency features
- Lightweight text statistics
- Missingness signal

**Key result**

- Validation RMSLE: **0.475911**
- Model training runtime: ~83 seconds
- Full preprocessing + model runtime: ~96 seconds

**Extracted knowledge**

- Structured information adds a large amount of signal beyond text alone.
- The strongest tested route became **text + structured + Ridge**.
- High dimensionality is manageable because the representation is sparse.

**Status:** COMPLETE

---

## 6.4 Feature-Family Ablation

**Decision**

Remove one feature family at a time from the 6.3 model to determine which families contribute meaningful signal.

**Results**

| Model variation | RMSLE |
|---|---:|
| Full model | **0.475911** |
| Remove categorical interactions | 0.485553 |
| Remove original structured | 0.483542 |
| Remove text statistics | 0.478969 |
| Remove missingness | 0.476666 |
| Remove hierarchy | 0.475877 |
| Remove frequency | 0.475568 |

**Extracted knowledge**

- Original structured information and categorical interactions matter noticeably.
- Text statistics provide smaller but measurable value.
- Hierarchy and frequency produced almost no change in this specific Ridge setup.
- A tiny ablation difference means weak evidence, not proof that a feature is useless everywhere.

**Decision**

Keep the full representation for subsequent experiments until stronger evidence justifies removing any family.

**Status:** COMPLETE

---

## 6.5 CatBoost Structured Benchmark

**Decision**

Test a different model family rather than assuming Ridge is automatically the right model.

**Approach**

Use structured features with a single sensible CatBoost configuration rather than a large tuning search.

**Key result**

- Validation RMSLE: **0.539761**
- Runtime: ~9 minutes
- Best iteration reached the end of the configured run

**Extracted knowledge**

- Structured features contain useful signal, but CatBoost does not exploit them as effectively as the sparse text + Ridge route in this project.
- A substantially slower model that is already behind the strongest benchmark does not justify a large tuning campaign.

**Decision**

Do not pursue CatBoost tuning without new evidence.

**Status:** COMPLETE — REJECTED AS PRIMARY ROUTE

---

## 6.6 Model Comparison

**Decision**

Compare the major model routes tested so far.

**Results**

| Model | RMSLE |
|---|---:|
| Category + Brand Median | 0.593385 |
| TF-IDF + Ridge | 0.532397 |
| CatBoost Structured | 0.539761 |
| **TF-IDF + Structured + Ridge** | **0.475911** |

**Extracted knowledge**

The important conclusion is directional:

> **Sparse text + structured features + Ridge is the strongest tested modelling route.**

This narrowed future experimentation toward the Ridge/text representation instead of broad model-family exploration.

**Status:** COMPLETE

---

## 6.7 Validation Error Analysis

**Decision**

Inspect where the 0.475911 model fails instead of relying only on the overall RMSLE.

**Key findings**

The worst 5% of validation rows contributed about **39.3% of total squared log error**.

Major weak areas:

- Zero-price listings
- High-price listings
- Missing-brand listings
- Rare groups
- Shipping = 1
- Condition = 5

Earlier price-band RMSLE:

| Price band | RMSLE |
|---|---:|
| Zero | 2.9449 |
| (0,10] | 0.4869 |
| (10,17] | 0.3435 |
| (17,29] | 0.3624 |
| (29,75] | 0.5194 |
| (75,170] | 0.8535 |
| >170 | 1.2914 |

**Extracted knowledge**

- Most remaining error is concentrated in the upper-price tail.
- Zero-price rows are extreme but very small in number.
- Missing and rare categorical states are somewhat harder, but are not large enough by themselves to justify complicated special handling.

**Status:** COMPLETE

---

## 6.8 OOF Target Statistics

**Decision**

Test leakage-safe target statistics for:

- `category_log_mean`
- `brand_log_mean`
- `category_brand_log_mean`

OOF means each training row receives a statistic calculated without using its own target.

**Key result**

- Validation RMSLE: **0.477509**
- Current model at the time: **0.475911**
- Worsening: **+0.001598**
- Total runtime: ~145 seconds

**Extracted knowledge**

- Target encoding is not automatically useful just because it contains direct price information.
- Proper leakage prevention is essential.
- The added complexity did not improve this Ridge pipeline.

**Decision**

Reject the tested target-statistic features. Do not add more target-encoding complexity without a concrete reason.

**Status:** COMPLETE — REJECTED

---

## 6.9 LightGBM Structured Benchmark

**Decision**

Give gradient-boosted trees one stronger benchmark using the existing structured features only.

**Configuration**

- Maximum estimators: 3,000
- Learning rate: 0.05
- `num_leaves=63`
- 4 CPU threads
- Early stopping

**Key result**

- Best iteration: **2,118**
- Runtime: **527 seconds**
- Validation RMSLE: **0.529431**
- Ridge reference at the time: **0.475911**

**Extracted knowledge**

- LightGBM is slightly better than the earlier CatBoost benchmark but remains far behind the text + Ridge route.
- The learning curve plateaued, so simply adding more trees is not the obvious solution.
- Structured-only tree models are not currently competitive.

**Decision**

Do not spend further tuning time on GBDT without new evidence.

**Status:** COMPLETE — REJECTED AS PRIMARY ROUTE

---

## 6.10 TF-IDF Capacity Benchmark — 10k

**Decision**

Test whether the initial 5,000-feature cap is limiting text signal.

**Only change**

- Name TF-IDF: 5,000 → 10,000
- Description TF-IDF: 5,000 → 10,000

All other settings remain unchanged.

**Key result**

- Name features: 10,000
- Description features: 10,000
- Total features: **80,015**
- Runtime: **115.7 seconds**
- Validation RMSLE: **0.466357**
- Improvement versus 0.475911: **−0.009554**

**Extracted knowledge**

- The original 5k vocabulary was genuinely capacity-limited.
- More retained text patterns uncover additional predictive information.

**Decision**

Accept the 10k representation.

**Status:** COMPLETE — ACCEPTED

---

## 6.11 TF-IDF Capacity Benchmark II — 20k

**Decision**

Test whether additional text capacity continues to improve the model.

**Only change**

- Name TF-IDF: 10,000 → 20,000
- Description TF-IDF: 10,000 → 20,000

**Key result**

- Name features: 20,000
- Description features: 20,000
- Total features: **100,015**
- Runtime: **203.8 seconds**
- Validation RMSLE: **0.458119**
- Improvement versus 0.466357: **−0.008238**

**Extracted knowledge**

- More text capacity still improves the model.
- The improvement is smaller than the previous jump, showing early diminishing returns.

**Decision**

Accept the 20k representation as the new text baseline.

**Status:** COMPLETE — ACCEPTED

---

## 6.12 Character TF-IDF on Name

**Decision**

Test whether character-level patterns add information beyond word-level TF-IDF.

**Configuration**

- Apply character TF-IDF only to `name`
- Analyzer: `char`
- Character n-grams: `(3,5)`
- Maximum features: 10,000

Keep unchanged:

- 20k word TF-IDF for name
- 20k word TF-IDF for description
- Structured features
- Ridge
- Same validation split

**Key result**

- Name word features: 20,000
- Description word features: 20,000
- Name character features: 10,000
- Structured features: 60,015
- Total features: **110,015**
- Runtime: **364.0 seconds**
- Validation RMSLE: **0.455065**
- Improvement versus 0.458119: **−0.003054**

**Extracted knowledge**

Character fragments provide useful additional information for product names, including partial matches, formatting variation, model-number fragments, and spelling variation.

Character TF-IDF on the description was abandoned because its computational cost was too high for the available resources.

**Decision**

Accept name character TF-IDF.

**Status:** COMPLETE — ACCEPTED

---

## 6.13 Updated Error Analysis

**Decision**

Rerun the original error analysis on the new 0.455065 model before moving into final optimization work.

**Key result**

- Validation RMSLE: **0.455065**
- Worst 5% rows: 14,826
- Worst 5% share of total squared log error: **39.3%**

Current problem areas:

| Group | RMSLE |
|---|---:|
| Zero price | 2.9514 |
| >170 | 1.1893 |
| 75–170 | 0.7851 |
| Missing brand | 0.4764 |
| Present brand | 0.4385 |
| Missing description | 0.4854 |
| Present description | 0.4532 |

**Extracted knowledge**

The broad error pattern did not fundamentally change:

- Upper-price listings remain the main weakness.
- Zero-price observations remain extreme but rare.
- Missing and rare groups remain somewhat harder.

There was not enough evidence to justify another broad feature-engineering round.

**Decision**

Finish Phase 6 diagnostics and move into:

> **Optimization, Ensembling & Calibration**

**Status:** COMPLETE

---

# Phase 6 Overall Decisions

## Model / Algorithm Decisions

**Kept**

- Ridge with sparse TF-IDF
- Structured + text combination
- 20k word TF-IDF for name
- 20k word TF-IDF for description
- 10k character TF-IDF for name

**Rejected**

- CatBoost as primary route
- LightGBM as primary route
- Tested OOF target statistics
- Further arbitrary feature expansion without evidence

## Engineering Decisions

- Keep the fixed validation split unchanged.
- Change one meaningful thing per experiment.
- Reuse persisted learned preprocessing where possible.
- Keep large text matrices sparse.
- Avoid broad hyperparameter searches after a model family loses the first serious comparison.
- Avoid expensive character TF-IDF on the description under the current runtime constraints.
- Persist learned artifacts/models/results, but do not blindly cache every large sparse matrix because of the 1 GB scratch/output limit.

---

# Phase 6 Progression

```text
Category + Brand Median
RMSLE = 0.593385
        ↓
TF-IDF + Ridge
RMSLE = 0.532397
        ↓
TF-IDF + Structured + Ridge
RMSLE = 0.475911
        ↓
10k Word TF-IDF
RMSLE = 0.466357
        ↓
20k Word TF-IDF
RMSLE = 0.458119
        ↓
+ 10k Name Character TF-IDF
RMSLE = 0.455065
```

---

# Current State

**Current best local model**

```text
20k name word TF-IDF
+
20k description word TF-IDF
+
10k name character TF-IDF
+
structured features
+
Ridge
```

**Current validation RMSLE**

> **0.4550653**

**Phase status**

> **Phase 6 diagnostics complete.**

The project is now ready for **Optimization, Ensembling & Calibration**, with the same experimental discipline:

> **One meaningful change → measure → keep or reject.**
