# Knowledge Ledger — Phase 5
## Feature Engineering & Construction

**Purpose:** Build candidate feature families under a strict leakage boundary and make them available for controlled benchmarking without assuming that every constructed feature is useful.

---


## 5.1 Leakage-Safe Preprocessing Boundary

**Decision**

Create explicit training, validation, and competition-test feature matrices. Any transformation that learns parameters from data must fit on `X_train` only.

**Key findings**

- `X_train`: 1,186,028 rows × 7 features
- `X_val`: 296,507 rows × 7 features
- `X_test`: 693,359 rows × 7 features
- Feature schemas align across all three datasets.
- `price` is absent from all feature matrices.
- Train/validation row overlap: 0.

**Extraction**

- Training data is the sole fitting source for learned transformations.
- Validation is reserved for evaluation.
- Competition test is reserved for final prediction.

**Status:** PASS

---

## 5.2 Category Hierarchy Features

**Decision**

Decompose `category_name` into up to five levels while retaining the original full path and using an explicit missing token for absent levels.

**Key findings**

- Level 1: 11 unique values in all splits.
- Level 2: 114 unique values in all splits.
- Level 3: 859 train / 780 validation / 834 test unique values.
- Levels 4–5 are extremely sparse but retained.
- Missing category paths are represented explicitly.
- Train/validation/test feature structures align.

**Extraction**

- Most category paths are three levels deep.
- Rare deeper levels are structurally valid and should not be discarded purely due to frequency.
- Hierarchy decomposition provides different granularities of the same categorical information.

**Status:** COMPLETE

---

## 5.3 Categorical Interaction Features

**Decision**

Construct only the three interactions already supported by baseline evidence:
- `category_brand`
- `category_condition`
- `brand_condition`

Use `|` as the composite separator.

**Key findings**

Training-partition cardinalities:
- `category_brand`: 38,278
- `category_condition`: 4,336
- `brand_condition`: 10,566

All interactions are aligned across train, validation, and test.

**Extraction**

- The interaction features expose categorical combinations already shown to carry predictive signal.
- The separator was changed from `__` to `|` to avoid visually ambiguous concatenation with the `__MISSING__` token.

**Status:** COMPLETE

---

## 5.4 Frequency Features

**Decision**

Calculate training-partition occurrence counts for:
- `brand_name`
- `category_name`
- `category_brand`

Map validation/test values to those training counts; unseen groups receive 0.

**Key findings**

| Feature | Train median | Train maximum | Validation unseen rows | Test unseen rows |
|---|---:|---:|---:|---:|
| `brand_count` | 38,489 | 506,058 | 288 | 728 |
| `category_count` | 6,903 | 48,164 | 30 | 45 |
| `category_brand_count` | 1,107 | 18,985 | 4,523 | 10,849 |

**Extraction**

- Frequency is highly concentrated, largely because missing brand is itself a very common state.
- Frequency statistics are leakage-safe because only `X_train` contributes to the counts.
- Unseen groups are explicitly represented by count 0.

**Status:** COMPLETE

---

## 5.5 Lightweight Text Statistics

**Decision**

Add only cheap text-derived numerical features:
- character count
- word count
- digit count

for both `name` and `item_description`.

**Key findings**

Train/validation/test means are highly similar across all six statistics.

Examples from training:
- `name_char_count`: mean ~25.78
- `name_word_count`: mean ~4.40
- `item_description_char_count`: mean ~144.64
- `item_description_word_count`: mean ~25.51
- `item_description_digit_count`: mean ~2.12

**Extraction**

- Basic text statistics are stable across splits.
- They may capture cheap signals such as listing length and embedded numeric information without a large feature-engineering cost.
- No obvious malformed text-statistic behavior was detected.

**Status:** COMPLETE

---

## 5.6 TF-IDF Text Representation

**Decision**

Use an initial sparse TF-IDF representation for `name` and `item_description`, with separate 5,000-feature vocabularies and unigram/bigram features.

Initial configuration:
- `ngram_range=(1, 2)`
- `max_features=5000`
- `min_df=2`
- `max_df=0.95`
- `dtype=float32`

Fit both vectorizers using training text only, then transform validation/test.

**Key findings**

- `name`: 5,000 features; 5,352,764 non-zero training values.
- `item_description`: 5,000 features; 29,070,845 non-zero training values.
- Combined: 10,000 features; 34,423,609 non-zero training values.
- All matrices have aligned train/validation/test shapes.

**Extraction**

- The text representation is sparse and therefore feasible under the dataset scale.
- Descriptions activate substantially more vocabulary terms per row than names.
- The initial vocabulary cap is a resource-conscious benchmark, not a final hyperparameter decision.

**Status:** COMPLETE

---

## 5.7 Feature-Family Readiness

**Decision**

Treat all constructed feature families as candidates. Do not assume they belong in the final model; evaluate incremental value against the established 0.593385 category-brand baseline.

**Key findings**

All planned families are available across train/validation/test:
- Original structured features
- Category hierarchy
- Categorical interactions
- Frequency features
- Lightweight text statistics
- TF-IDF text representation
- Missingness signal

**Extraction**

- The feature space is ready for controlled benchmarking.
- `category_name + brand_name` remains the current predictive reference.
- Feature families should earn inclusion through measurable validation improvement rather than theoretical plausibility alone.

**Status:** COMPLETE



---

# Phase 5 Outcome

Phase 5 created the candidate feature space: original structured variables, category hierarchy, categorical interactions, frequency features, lightweight text statistics, TF-IDF text, and explicit missingness. The key methodological decision was to treat these as candidates whose inclusion must be earned through validation improvement.
