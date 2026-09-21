# Knowledge Ledger — Phase 3
## Rapid EDA & Sanitization

**Purpose:** Determine modeling eligibility, understand category hierarchy and text repetition, define explicit missing-value representations, inspect target extremes, and construct modeling-ready sanitized copies without altering the raw source data.

---


## 3.1 Feature Eligibility & Leakage Audit

**Decision**

Model-eligible feature set:
- `name`
- `item_condition_id`
- `category_name`
- `brand_name`
- `shipping`
- `item_description`

Exclude:
- `train_id`
- `test_id`
- `price` from predictors

**Key findings**

- No obvious target-proxy columns identified from the known schema or column names.
- No unexpected train-only predictive features.
- No unexpected test-only predictive features.
- Train/test feature sets match.

**Extraction**

- IDs are excluded from modeling.
- `price` is exclusively the target.
- No obvious post-outcome or proxy-target field is present.

**Limitation**

Column-name inspection cannot prove absence of semantic leakage by itself; the current schema and field meanings provide no obvious leakage candidate.

**Status:** PASS

---

## 3.2 Variance & Constant-Feature Audit

**Decision**

Check constant and practical near-zero-information features using value concentration.

**Key findings**

All six modeling features:
- `zero_variance = False`
- `near_zero_variance = False`

Notable observations:
- `shipping` is reasonably balanced.
- `item_condition_id` contains all five conditions.
- `category_name` has substantial categorical diversity.
- `brand_name` has high missingness, but is not constant.
- `name` and `item_description` have extremely high uniqueness.

**Extraction**

- No feature needs to be removed on constant/near-zero-variance grounds.

**Status:** PASS

---

## 3.3 Category Hierarchy Analysis

**Decision**

Parse `category_name` into hierarchical levels for understanding, but do not yet decide how the hierarchy will be represented to the model.

**Key findings**
- Level 1: 11 unique values
- Level 2: 114 unique values
- Level 3: 871 train / 834 test unique values
- Level 4: 8 unique values, ~99.7% structurally absent
- Level 5: 4 unique values, ~99.8% structurally absent

Depth distribution:
- No category: ~0.43-0.44%
- 3 levels: ~99.26-99.28%
- 4 levels: ~0.09%
- 5 levels: ~0.20%

Structural checks:
- Maximum depth in both datasets: 5
- Empty category components: 0 in both
- Missing intermediate levels: 0 in both

**Extraction**

- Category hierarchy is overwhelmingly three levels deep.
- Levels 4-5 are legitimate but extremely rare.
- Missing levels 4-5 usually mean the category path ends earlier, not that information was lost.
- Train/test hierarchy depth proportions are highly consistent.

**Decision**

Do not discard levels 4-5 merely because they are rare. Preserve available hierarchy information until feature representation is designed.

**Status:** PASS

---

## 3.4 Text / Categorical Sanitization Audit

**Decision**

Audit text/categorical fields for representation anomalies before normalization, but do not modify raw data yet.

**Key findings**

- No non-string values among non-null text/categorical entries.
- No leading/trailing whitespace padding detected.
- Empty/whitespace counts correspond to previously observed missing values rather than additional hidden empty-string cases.

### Case variation

`name` contains many case-sensitive variants that collapse after case-folding:
- Train: 59,002 case-variant groups
- Test: 25,366 case-variant groups

No such case-variant groups were found for:
- `category_name`
- `brand_name`

**Extraction**

- `name` has substantial casing variation and may benefit from normalization later.
- No whitespace-cleaning issue has been detected.
- Raw data remains unchanged.

**Status:** COMPLETE WITH FINDING

---

---

## 3.5 Near-Duplicate Investigation

**Decision**

Use normalized exact-text repetition as a computationally cheap first-pass audit rather than exhaustive pairwise semantic similarity.

**Key findings**

### `name`
- 1,482,516 non-empty normalized values.
- 469,283 rows belong to normalized duplicate groups.
- 101,539 normalized duplicate groups.
- Largest normalized duplicate group: `bundle` with 3,370 rows.
- Other highly repeated names include `lularoe tc leggings`, `reserved`, `coach purse`, and `michael kors purse`.

### `item_description`
- 1,481,967 non-empty normalized values.
- 260,172 rows belong to normalized duplicate groups.
- 35,162 normalized duplicate groups.
- Largest normalized duplicate group: `no description yet` with 82,498 normalized occurrences.
- Other highly repeated descriptions include generic/template phrases such as `brand new`, `new`, `great condition`, and `good condition`.

### Train-validation normalized overlap
- `name`: 18.56% of unique validation normalized names are also present in training.
- `item_description`: 6.07% of unique validation normalized descriptions are also present in training.

**Extraction**

- Marketplace text contains substantial repeated templates and standardized wording.
- Exact normalized repetition is much higher than raw exact-string overlap.
- The observed overlap is not itself evidence of leakage.
- Repeated listings/templates could potentially make a random validation score optimistic if the same underlying listing/product appears across the split, but this has not been established.

**Status:** COMPLETE WITH SIGNIFICANT FINDINGS

---

## 3.6 Missing-Value Representation

**Decision**

Do not alter the raw datasets. Establish explicit missing-value representations in sanitized copies for later modeling.

Planned representation:
- `category_name` missing -> dedicated missing token
- `brand_name` missing -> dedicated missing token
- `item_description` missing -> explicit missing state
- `item_description_is_missing` -> binary indicator
- Known placeholder `no description yet` -> missing description state during preprocessing

**Key findings**

Raw null counts:
- `category_name`: 6,327 train / 3,058 test
- `brand_name`: 632,682 train / 295,525 test
- `item_description`: 6 train / 0 test

Effective description absence:
- `no description yet`: 82,494 train / 38,508 test
- Plus 6 literal train nulls
- Sanitized missing-description indicator: 82,500 train / 38,508 test

Effective description missingness is therefore approximately 5.56% in train and 5.55% in test.

**Extraction**

- Literal null counts substantially understate missing description information.
- Train/test effective description missingness is closely aligned.
- Missingness is represented explicitly rather than deleting observations.

**Status:** COMPLETE

---

## 3.7 Target Tail & Zero-Price Review

**Decision**

Keep the target exactly as provided. Do not drop, winsorize, or clip target observations based on the current evidence.

**Key findings**

Upper tail:
- 90th percentile: 51
- 95th percentile: 75
- 99th percentile: 170
- 99.5th percentile: ~230
- 99.9th percentile: 450
- 99.95th percentile: ~603
- 99.99th percentile: 1,015
- Maximum: 2,009

Zero prices:
- 874 rows
- 0.05895% of training data
- 0 negative prices
- 0 missing prices

Zero-price records do not show an obvious corruption pattern:
- 39.59% have missing `brand_name`
- 1.49% have missing `category_name`
- 0% have missing `item_description`
- They span diverse categories and brands.

Highest-price observations correspond to economically plausible marketplace items including luxury bags, jewelry, watches, and electronics.

**Extraction**

- The target has a very small but genuine high-price tail.
- Extreme prices are not demonstrated to be erroneous.
- Zero-price observations are rare and currently appear to be legitimate target values.
- RMSLE and the log-scale target view further support avoiding arbitrary target clipping.

**Status:** COMPLETE WITH DECISION

---

## 3.8 Sanitized Dataset Construction

**Decision**

Construct modeling-ready copies while preserving the original raw `train` and `test` data.

Transformations applied to sanitized copies:
- `category_name` null -> `__MISSING__`
- `brand_name` null -> `__MISSING__`
- `item_description` null -> empty text state
- `item_description == "no description yet"` -> empty text state
- Add `item_description_is_missing` as an `int8` indicator
- Preserve `price` unchanged
- Preserve IDs unchanged
- Preserve the competition test reference separately

**Key findings**

Sanitized shapes:
- Train: 1,482,535 rows × 9 columns
- Test: 693,359 rows × 8 columns

Post-sanitization null counts:
- 0 across all columns in both datasets.

Missing-description indicator:
- Train: 82,500 missing-description rows
- Test: 38,508 missing-description rows

Integrity checks:
- Original `price` unchanged: `True`
- Original train IDs unchanged: `True`
- Original test IDs unchanged: `True`

Sanitized modeling features:
- `name`
- `item_condition_id`
- `category_name`
- `brand_name`
- `shipping`
- `item_description`
- `item_description_is_missing`

**Extraction**

- Sanitized copies can represent missing states explicitly without modifying the raw source data.
- The target and identifiers remain intact for evaluation and submission handling.
- The sanitized datasets are ready as inputs for subsequent feature construction, subject to later feature-engineering decisions.

**Status:** PASS


---

# Phase 3 Outcome

Phase 3 established the first modeling-ready data layer. No obvious leakage or constant-feature issue was found; category hierarchy is structurally valid; text contains substantial normalized repetition; exhaustive semantic deduplication was not justified; missing values and description placeholders received explicit representations; and the target remained untouched.
