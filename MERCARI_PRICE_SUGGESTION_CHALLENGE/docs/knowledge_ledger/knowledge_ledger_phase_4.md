# Knowledge Ledger — Phase 4
## Initial Baseline

**Purpose:** Establish simple, transparent predictive references before introducing machine-learning models or large feature representations.

---


## 4.1 Global Median Baseline

**Decision**

Use the median target from the training partition as the simplest global reference prediction.

**Key findings**

- Training-partition median price: 17.0
- Validation RMSLE: 0.75656

**Extraction**

- A constant median prediction provides the absolute reference point for subsequent models.
- The median must be computed from the training partition only.

**Status:** COMPLETE

---

## 4.2 Single-Feature Group Median Baselines

**Decision**

Evaluate validation-safe median prices grouped by individual categorical features, with the global median used for unseen groups.

**Key findings**

| Grouping | RMSLE | Fallback rate |
|---|---:|---:|
| `category_name` | 0.65865 | 0.010% |
| `brand_name` | 0.66613 | 0.097% |
| `shipping` | 0.73636 | 0.000% |
| `item_condition_id` | 0.75759 | 0.000% |

**Extraction**

- `category_name` is the strongest single categorical grouping.
- `brand_name` is also highly informative.
- `shipping` provides modest signal.
- `item_condition_id` does not improve the global median when used alone.
- Weak standalone performance does not imply that condition is useless in interactions.

**Status:** COMPLETE

---

## 4.3 Multi-Feature Group Median Baselines

**Decision**

Evaluate only a small set of empirically motivated categorical interactions using medians calculated from the training partition.

**Key findings**

| Grouping | RMSLE | Fallback rate |
|---|---:|---:|
| `category_name + brand_name` | 0.593385 | 1.525% |
| `category_name + item_condition_id` | 0.639301 | 0.059% |
| `brand_name + item_condition_id` | 0.654452 | 0.288% |

**Extraction**

- `category_name + brand_name` provides a substantial improvement over either feature alone.
- `item_condition_id` becomes useful when conditioned on category or brand.
- Categorical interactions carry important price information.

**Status:** COMPLETE

---

## 4.4 Baseline Comparison

**Decision**

Use the same fixed validation split and RMSLE implementation to compare every baseline.

**Key findings**

Current reference baseline:

```text
category_name + brand_name
RMSLE = 0.5933847755
```

Improvement over global median:

```text
0.7565639206 - 0.5933847755 = 0.1631791451
```

**Extraction**

- Simple category-brand conditional pricing explains a large amount of the predictable price variation.
- This becomes the benchmark that subsequent feature/model experiments must beat.
- The workflow should chase incremental signal rather than accumulate features without measured benefit.

**Status:** COMPLETE

---


---

# Phase 4 Outcome

Phase 4 established two reference points: a global median RMSLE of 0.756564 and a category+brand median RMSLE of 0.593385. The latter became the main benchmark because it demonstrated that categorical context and interactions already explain substantial predictable price variation.
