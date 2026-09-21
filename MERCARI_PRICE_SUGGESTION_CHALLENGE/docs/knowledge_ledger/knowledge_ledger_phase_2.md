# Knowledge Ledger — Phase 2
## Data Understanding & Validation

**Purpose:** Understand the target distribution and feature structure, test train/test consistency, investigate whether row order suggests temporal structure, fix the validation protocol, and isolate the competition test set.

---


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

**Extraction**

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

**Extraction**

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

**Extraction**

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

**Extraction**

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

**Extraction**

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

**Extraction**

- Competition test set must not be used as a labeled validation set.
- Snapshot hash provides a reproducible fingerprint.
- Later transformations should be performed on copies rather than altering the pristine reference.

**Status:** PASS

---


---

# Phase 2 Outcome

Phase 2 established the statistical and evaluation foundation: raw price is highly right-skewed, log1p(price) is substantially more regular, structured train/test marginals are stable, text has low exact-string overlap, no obvious row-order drift was found, the validation split is fixed at 80/20 with seed 42, and the competition test set is kept isolated.
