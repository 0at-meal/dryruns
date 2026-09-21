# Knowledge Ledger — Phase 1
## Data Ingestion & Integrity Auditing

**Purpose:** Establish that the competition data has the expected structure, types, identifiers, valid value domains, missingness profile, and train/test feature alignment before modeling begins.

---


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

**Extraction**

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

**Extraction**

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

**Extraction**

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

**Extraction**

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

**Extraction**

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

**Extraction**

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

**Extraction**

- No unexpected train/test structural mismatch exists.
- The model feature set is structurally consistent across train and test.

**Status:** PASS

---


---

# Phase 1 Outcome

Phase 1 established a clean raw-data contract. The datasets have the expected schemas and dtypes, valid unique identifiers, valid constrained values, no exact duplicate rows, an explicit missingness profile, and aligned predictive features. No raw data modification was required during the integrity audit.
