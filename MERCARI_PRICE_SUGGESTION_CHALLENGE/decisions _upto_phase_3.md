# Project Decisions & Knowledge Log

## Project Context

**Problem family:** Supervised Learning  
**Objective:** Predict product `price` from structured and text fields  
**Evaluation metric:** RMSLE  
**Known data:** Stage 1 `train.tsv`, Stage 1 `test.tsv`, Stage 1 submission sample  
**Hidden evaluation:** Stage 2 contains hidden test data  
**Compute constraint:** 4 CPU cores, 16 GB RAM, 1 GB scratch/output disk, 60-minute runtime

### Working feature classification

| Field | Role | Decision |
|---|---|---|
| `train_id` / `test_id` | Identifier | Exclude from modeling |
| `item_condition_id` | Ordered categorical | Treat as categorical/ordered discrete information, not continuous |
| `category_name` | Hierarchical categorical | Preserve structure; defer representation decision |
| `brand_name` | High-cardinality categorical with heavy missingness | Preserve; defer missing-value treatment |
| `shipping` | Binary categorical | Preserve as binary feature |
| `name` | High-cardinality text | Treat as text, not ordinary categorical data |
| `item_description` | High-cardinality text | Treat as text, not ordinary categorical data |
| `price` | Continuous target | Predict using regression; evaluate with RMSLE |

The overall problem is treated as a **supervised regression problem with structured/tabular and text features**, rather than as a pure NLP problem.

---

# 1. Data Ingestion & Integrity Auditing

## 1.1 Schema & Column Structure

**Decision**

- Use the expected train/test schemas as the structural contract.
- `price` is expected only in training data.
- `train_id` and `test_id` are expected dataset-specific identifiers.

**Key findings**

- No missing expected columns.
- No unexpected columns.
- Train columns: `train_id`, `name`, `item_condition_id`, `category_name`, `brand_name`, `price`, `shipping`, `item_description`
- Test columns: `test_id`, `name`, `item_condition_id`, `category_name`, `brand_name`, `shipping`, `item_description`

**Extracted knowledge**

- Train and test share the same six predictive features.
- The only intentional structural difference is the target/identifier arrangement.

**Status:** PASS

---

## 1.2 Data Type Enforcement

**Decision**

Use explicit dtypes during ingestion where known:

- IDs -> `int32`
- `item_condition_id` -> `int8`
- `shipping` -> `int8`

Text/object columns remain `object`. `price` is `float64`.

**Key findings**

All actual dtypes matched the intended dtypes in both train and test.

**Extracted knowledge**

- No unwanted dtype inference or coercion was observed.
- The categorical/text columns are currently represented as Pandas `object` columns.

**Status:** PASS

---

## 1.3 Null / Missing Value Audit

**Decision**

- Detect and document missing values now.
- Do not impute, delete, or otherwise remediate them during the integrity audit.
- Defer treatment to later feature engineering.

**Key findings**

Train:
- `category_name`: 6,327 missing
- `brand_name`: 632,682 missing
- `item_description`: 6 missing
- No missing IDs, condition values, shipping values, or prices.

Test:
- `category_name`: 3,058 missing
- `brand_name`: 295,525 missing
- No missing IDs, condition values, shipping values, or descriptions.

Missingness rates:
- `brand_name`: ~42.7% train, ~42.6% test
- `category_name`: ~0.43% train, ~0.44% test
- `item_description`: effectively negligible

**Extracted knowledge**

- Missing `brand_name` is a major structural property of the dataset, not automatically invalid data.
- Train/test missingness rates are very similar.
- Missing values will be handled later rather than dropped.

**Status:** VALID WITH FINDINGS

---

## 1.4 Duplicate Row Audit

**Decision**

Check for completely duplicated rows; do not remove rows automatically without understanding their meaning.

**Key findings**

- Train duplicate rows: 0
- Test duplicate rows: 0

**Extracted knowledge**

- No exact duplicate rows exist in either dataset.

**Status:** PASS

---

## 1.5 Primary Key Integrity

**Decision**

Require IDs to be non-null and unique within their own dataset.

**Key findings**

Train:
- Null `train_id`: 0
- Unique: `True`
- Duplicate IDs: 0

Test:
- Null `test_id`: 0
- Unique: `True`
- Duplicate IDs: 0

**Extracted knowledge**

- Both identifiers behave as valid primary keys within their respective datasets.
- `train_id` and `test_id` are separate namespaces; equal numeric values across them are not primary-key collisions.

**Status:** PASS

---

## 1.6 Basic Value Validity

**Decision**

Validate constrained categorical/binary fields against their known valid domains.

**Key findings**

- `item_condition_id` values: `{1, 2, 3, 4, 5}` in both train and test.
- `shipping` values: `{0, 1}` in both train and test.

**Extracted knowledge**

- `item_condition_id` is an ordered condition scale from 1-5, not a review/star rating.
- `shipping` is a binary indicator where 1 means seller-paid shipping and 0 means buyer-paid shipping.

**Status:** PASS

---

## 1.7 Train-Test Structural Consistency

**Decision**

Require identical predictive feature sets between train and test, excluding dataset-specific IDs and the target.

**Key findings**

Shared modeling features:
- `name`
- `item_condition_id`
- `category_name`
- `brand_name`
- `shipping`
- `item_description`

Train-only columns:
- `train_id`
- `price`

Test-only column:
- `test_id`

**Extracted knowledge**

- No unexpected train/test structural mismatch exists.
- The model feature set is structurally consistent across train and test.

**Status:** PASS

---

# 2. Data Understanding & Validation

## 2.1 Target Distribution

**Decision**

- Analyze raw `price` and `log1p(price)`.
- Use RMSLE as the validation metric.
- Do not treat extreme values as errors merely because they are statistically unusual.

**Key findings**

Training rows: 1,482,535

Raw target:
- Mean: ~26.74
- Median: 17
- 75th percentile: 29
- 95th percentile: 75
- 99th percentile: 170
- Maximum: 2,009
- Skewness: ~11.39
- Kurtosis: ~283.82

Log-transformed target:
- Mean of `log1p(price)`: ~2.98
- Skewness: ~0.66
- Kurtosis: ~1.09

Target validity:
- Zero prices: 874
- Negative prices: 0
- Missing prices: 0

**Extracted knowledge**

- Raw `price` is extremely right-skewed and heavy-tailed.
- `log1p(price)` dramatically reduces skew and tail concentration.
- Zero prices are valid non-negative targets and are not automatically errors.
- The upper tail contains observations up to $2,009; no evidence yet establishes them as erroneous.

**Status:** VALID WITH FINDINGS

---

## 2.2 Feature Distribution & Cardinality

**Decision**

Profile cardinality, missingness, dominant values, rare categories, and text lengths before choosing representations.

**Key findings**

### `item_condition_id`
- 5 unique values.
- Distribution dominated by conditions 1, 3, and 2.
- Train/test frequencies are very similar.

### `shipping`
- 2 unique values.
- Approximately 55% buyer-paid (`0`) and 45% seller-paid (`1`) in both train and test.
- Train/test frequencies are very similar.

### `category_name`
- 1,288 unique in train
- 1,224 unique in test
- ~0.4% missing in both
- Rare-category tail exists but is limited.

### `brand_name`
- 4,810 unique in train
- 3,901 unique in test
- ~42.6% missing in both
- Strong rare-brand tail.
- Missing `brand_name` is the most frequent single value.

### `name`
- ~82.65% of training rows contain unique exact names.
- Very high-cardinality text.

### `item_description`
- ~86.43% of training rows contain unique exact descriptions.
- Very high-cardinality text.

### Text lengths

`name`:
- Mean ~25.8 characters
- Median 26
- Maximum ~43-44

`item_description`:
- Mean ~145.6-145.7 characters
- Median ~85-86
- Maximum ~1,046-1,049

Rare category counts in train:
- `category_name` frequency = 1: 84
- `category_name` frequency <= 5: 243
- `brand_name` frequency = 1: 1,243
- `brand_name` frequency <= 5: 2,614

**Extracted knowledge**

- `brand_name` is high-cardinality, heavily missing, and has a long rare-category tail.
- `name` and `item_description` are fundamentally text features with very high exact-string uniqueness.
- Exact categorical treatment of the text fields would have poor direct test coverage.
- Train/test text lengths are broadly aligned.
- Structured feature frequency distributions appear stable.

**Status:** COMPLETE WITH FINDINGS

---

## 2.3 Train-Test Distribution Consistency

**Decision**

Compare train/test marginal distributions and category coverage before deciding whether special validation schemes are required.

**Key findings**

Maximum observed train/test frequency difference:
- `item_condition_id`: ~0.073 percentage points
- `shipping`: ~0.044 percentage points
- `category_name`: ~0.035 percentage points
- `brand_name`: ~0.053 percentage points

Category coverage:
- `category_name`: 23 test-only, 87 train-only
- `brand_name`: 480 test-only, 1,389 train-only

Exact text overlap:
- `name`: 75,773 exact unique strings shared; 525,344 unique test names unseen in train.
- `item_description`: 28,944 exact unique strings shared; 580,611 unique test descriptions unseen in train.

**Extracted knowledge**

- No obvious marginal distribution shift was observed in structured variables.
- High-cardinality text has low exact-string overlap, which is expected and limits direct memorization coverage.
- Low exact-string overlap does not imply semantic dissimilarity.

**Status:** PASS FOR OBSERVED MARGINAL CHECKS

---

## 2.4 Data Generation & Split Mechanism

**Decision**

Investigate whether row order or IDs suggest temporal/sequential structure before selecting a validation split.

**Key findings**

### IDs
- Train IDs run from 0 to 1,482,534 with exactly one ID per row.
- Test IDs run from 0 to 693,358 with exactly one ID per row.
- IDs are monotonically increasing with mean/median absolute step = 1.
- Row-position/ID correlation is ~1.0 in both datasets.

**Conclusion:** IDs are effectively row positions and show no evidence of useful ordering information.

### Target by row-order block
Across ten training blocks:
- Mean price remains approximately 26.61-26.87.
- Median price is 17 throughout.

**Conclusion:** No obvious target drift over row order.

### Missingness by row-order block
- `brand_name`: roughly 42.5-42.9%
- `category_name`: roughly 0.41-0.46%
- `item_description`: effectively zero

**Conclusion:** No obvious row-order-dependent missingness.

### Diagnostic correction
The first category/brand shift diagnostic incorrectly mixed NaN-included and NaN-excluded distributions. The contaminated numerical shift values were not used as evidence.

**Extracted knowledge**

- No obvious temporal or sequential structure requiring a specialized time-based split has been identified.
- A random validation split is currently justified by observed evidence, although the exact hidden competition sampling mechanism is not proven.

**Status:** SUFFICIENT FOR CURRENT VALIDATION DECISION

---

## 2.5 Validation Strategy

**Decision**

Create one fixed random 80/20 validation split from labeled training data.

Configuration:
- Training portion: 1,186,028 rows
- Validation portion: 296,507 rows
- `shuffle=True`
- `random_state=42`

RMSLE was defined with non-negative prediction clipping.

**Key findings**

- Target is non-negative.
- Split is reproducible.
- Validation set is independent of competition test set.
- Original training dataframe remains available.

**Extracted knowledge**

- Local model comparisons can use a fixed, reproducible validation set.
- RMSLE is the consistent local evaluation metric.

**Status:** COMPLETE

---

## 2.6 Competition Test Set Isolation

**Decision**

Treat Stage 1 `test.tsv` as a pristine, untouched competition evaluation set.

A reference copy named `competition_test` was created.

**Key findings**
- Rows: 693,359
- Expected schema: `True`
- `test_id` unique: `True`
- Target `price` present: `False`
- Snapshot hash: `1477444354750877641`

**Extracted knowledge**

- Competition test set must not be used as a labeled validation set.
- Snapshot hash provides a reproducible fingerprint.
- Later transformations should be performed on copies rather than altering the pristine reference.

**Status:** PASS

---

# 3. Rapid EDA & Sanitization

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

**Extracted knowledge**

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

**Extracted knowledge**

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

**Extracted knowledge**

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

**Extracted knowledge**

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

**Extracted knowledge**

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

**Extracted knowledge**

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

**Extracted knowledge**

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

**Extracted knowledge**

- Sanitized copies can represent missing states explicitly without modifying the raw source data.
- The target and identifiers remain intact for evaluation and submission handling.
- The sanitized datasets are ready as inputs for subsequent feature construction, subject to later feature-engineering decisions.

**Status:** PASS

# Unresolved Tickets

## U1. Zero-price observations

- **Finding:** 874 training rows have `price = 0` (~0.059%).
- **Current decision:** Keep them unchanged.
- **Reason:** Zero is valid under RMSLE and no corruption pattern was found.
- **Later decision:** Revisit only if modeling/error analysis reveals a specific issue.

## U2. Heavy missingness in `brand_name`

- **Finding:** ~42.6-42.7% missing in both train and test.
- **Current decision:** Represent missing brands with a dedicated missing token in sanitized data.
- **Later decision:** Assess whether an additional missingness indicator adds predictive value during modeling.

## U3. Category hierarchy representation

- **Finding:** `category_name` is overwhelmingly three levels deep, with legitimate but very rare levels 4-5.
- **Current decision:** Preserve the hierarchy information.
- **Later decision:** Decide whether the model should use the original path, individual levels, or both.

## U4. Near-duplicate / repeated listing structure

- **Finding:** Large normalized duplicate groups exist in both text fields; validation contains substantial normalized name/description overlap with training.
- **Current decision:** Do not perform exhaustive semantic similarity matching.
- **Later decision:** Determine whether repeated underlying listings/products could create validation optimism or whether efficient deduplication/grouping is justified.

## U5. Extreme target tail

- **Finding:** Maximum price = 2,009; 99th percentile = 170; extreme observations correspond to plausible high-value listings.
- **Current decision:** No clipping, winsorization, or row deletion.
- **Later decision:** Revisit only through model error analysis.

## U6. `name` case variation

- **Finding:** 59,002 train and 25,366 test case-variant groups were detected after case-folding.
- **Current decision:** Raw values remain unchanged.
- **Later decision:** Select text normalization appropriate to the eventual representation/model.

## U7. Categorical row-order shift diagnostic

- **Finding:** An earlier categorical block-shift calculation mixed NaN-included and NaN-excluded distributions.
- **Current decision:** Do not use the contaminated numerical values.
- **Later decision:** Re-run only if stronger evidence about row-order structure is required.

## U8. Text placeholder handling

- **Finding:** `no description yet` occurs 82,494 times in train and 38,508 times in test.
- **Current decision:** Treat it as absence of useful description during text preprocessing, while preserving a missingness indicator.
- **Later decision:** Confirm the final text representation and whether the placeholder should be retained as a token for any specific model.

---

# Phase Status

```text
1. Data Ingestion & Integrity Auditing
   -> Complete

2. Data Understanding & Validation
   -> Complete for current scope

3. Rapid EDA & Sanitization
   -> Complete
   -> 3.1 Complete
   -> 3.2 Complete
   -> 3.3 Complete
   -> 3.4 Complete
   -> 3.5 Complete
   -> 3.6 Complete
   -> 3.7 Complete
   -> 3.8 Complete
```

