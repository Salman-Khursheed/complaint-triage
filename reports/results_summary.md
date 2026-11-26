# Results summary (single source of truth)

Every number in the README, findings and resume bullets must come from this file. Each section names the file or notebook it was copied from. Values are rounded to 3 decimals unless shown otherwise.

Data: CFPB Consumer Complaint Database Narratives Archive, batch `CCDB_Export_9_January_2025_through_February_2025.zip` (complaints received 2025-01-01 to 2025-02-28). Narratives were removed from the live CFPB database in Sept 2026; an Aug 2026 archive batch had 1 narrative in 667,140 rows.

## 1. Data and cleaning (source: `reports/cleaning_log.csv`, notebooks 01-02)

| Step | Rows left |
|---|---|
| Raw rows | 811,019 |
| Has a narrative | 265,405 (32.7%) |
| Not empty after removing redaction | 265,392 |
| Drop texts filed under more than one product (175 texts) | 239,885 |
| Exact dedupe on cleaned text, keep earliest | **138,042** |

- Distinct raw narratives: 147,596 (notebook 01). The most repeated template appeared 7,410 times (raw, before cleaning).
- Narrative length (before dedupe): median 102 words, 90th percentile 318, 99th percentile 803. 50.3% contain `XXXX` redaction.
- Classes: 11 CFPB products; "Debt or credit management" and "Prepaid card" (< 500 training texts each) folded into "Other" -> **10 classes**.

## 2. Time split (source: `reports/cleaning_log.csv`, notebook 02)

| Split | Dates (received) | Rows |
|---|---|---|
| Train | 2025-01-01 to 2025-01-24 | 77,504 |
| Validation | 2025-01-25 to 2025-01-31 | 18,093 |
| Test | 2025-02-01 to 2025-02-28 | 42,445 |

- Identical cleaned texts shared between any two splits: 0 (asserted).
- Drift (share of split, notebook 02): Money transfer 23.5% of train vs 6.8% of test; Credit reporting 49.1% vs 58.3%; Debt collection 7.2% vs 12.3%.

## 3. Model comparison on VALIDATION (sources: `reports/03_valid_scores.csv`, `reports/04_valid_scores.csv`, `reports/04_valid_C_search.csv`)

| Model | Macro-F1 | Weighted F1 | Accuracy |
|---|---|---|---|
| Majority class | 0.078 | 0.499 | 0.639 |
| TF-IDF (1-2 grams) + logistic regression | 0.653 | 0.853 | 0.862 |
| TF-IDF + logistic regression, balanced class weights | **0.673** | 0.836 | 0.826 |
| MiniLM sentence embeddings + logistic regression, balanced (C = 1.0) | 0.598 | 0.798 | 0.771 |

- TF-IDF vocabulary: 151,922 terms (min_df = 5).
- MiniLM (`all-MiniLM-L6-v2`, 384 dims, 256-token limit): encoding 138,042 complaints took 49.3 min on CPU; 27.3% of a 2,000-complaint sample exceeded 256 tokens.
- TF-IDF balanced beat MiniLM on all 10 classes. On validation, both were wrong on 14.3% of complaints; MiniLM was right where TF-IDF was wrong on 3.2%.

## 4. Triage threshold, chosen on VALIDATION (source: `reports/05_triage_test.json`, notebook 05)

Rule: auto-route when the top probability >= t. Target: >= 95% accuracy on auto-routed complaints (and >= 1,000 auto-routed), maximum coverage.

| Model | t | Coverage (valid) | Auto-routed accuracy (valid) |
|---|---|---|---|
| TF-IDF + LR, default weights | **0.73** | **72.6%** | 95.1% |
| TF-IDF + LR, balanced weights | 0.85 | 41.2% | 95.4% |

Chosen: default weights, t = 0.73 (balanced weights win on macro-F1 but give less confident probabilities, so far lower coverage).

## 5. Final TEST results, run once (source: `reports/05_triage_test.json`, notebook 05)

Model: TF-IDF + logistic regression (default weights), t = 0.73. Test = February 2025, 42,445 complaints.

| Metric | Validation | Test |
|---|---|---|
| Macro-F1 | 0.653 | **0.649** |
| Weighted F1 | 0.853 | 0.817 |
| Accuracy (all complaints) | 0.862 | 0.827 |
| Coverage (auto-routed) | 72.6% | **65.5%** (27,819) |
| Auto-routed accuracy | 95.1% | **93.4%** (target 95% not met on test) |
| Sent to humans | | 14,626 |
| Model accuracy on the human queue | | 62.3% |

Per class on TEST (all complaints; source: notebook 05 output):

| Product | Precision | Recall | F1 | Support | Auto-routed share at t | Accuracy when auto-routed |
|---|---|---|---|---|---|---|
| Credit reporting | 0.864 | 0.946 | 0.903 | 24,740 | 84.7% | 98.4% |
| Debt collection | 0.735 | 0.616 | 0.670 | 5,240 | 43.5% | 64.7% |
| Checking/savings | 0.724 | 0.802 | 0.761 | 3,101 | 39.4% | 90.9% |
| Money transfer | 0.802 | 0.811 | 0.807 | 2,906 | 57.7% | 96.4% |
| Credit card | 0.756 | 0.656 | 0.703 | 2,710 | 28.9% | 70.6% |
| Mortgage | 0.840 | 0.785 | 0.812 | 1,005 | 34.2% | 95.6% |
| Student loan | 0.864 | 0.494 | 0.628 | 964 | 29.5% | 68.3% |
| Vehicle loan | 0.737 | 0.485 | 0.585 | 802 | 16.7% | 45.5% |
| Other | 0.935 | 0.163 | 0.278 | 526 | 21.3% | 19.6% |
| Payday/personal loan | 0.734 | 0.226 | 0.346 | 451 | 7.5% | 0.0% |

"Auto-routed share" and "accuracy when auto-routed" are by TRUE product.

## 6. Errors (source: notebook 06, `reports/06_error_sample.csv`)

- Confident (auto-routed) mistakes on test: 1,834 of 27,819 (6.6%).
- Most common: Debt collection -> Credit reporting 791; Credit reporting -> Debt collection 248; Credit card -> Credit reporting 198; Student loan -> Credit reporting 87; Checking/savings -> Money transfer 72.
- Hand-read sample: 30 confident test mistakes (up to 3 per true product, seed 42), each given one reason:

| Reason | Complaints (of 30) |
|---|---|
| multi-product (e.g. a loan or card account disputed on the credit report) | 13 |
| label noise (text fits the predicted product better than the filed one) | 7 |
| template (generic copy-paste dispute letter) | 4 |
| ambiguous | 3 |
| model error (clear text, wrong prediction) | 2 |
| truncated/short | 1 |

- In this sample, 20 of 30 (multi-product + label noise) are cases where the model's answer is defensible; only 2 of 30 are clear model errors. This is a small sample of only 30, so it's an indication, not a measured rate.

## 7. Known limitations (numbers above where available)

- Narratives are opt-in (32.7% of complaints in this batch), so the data is not representative of all complaints.
- Labels are chosen by consumers: 175 identical texts were filed under different products; many confident "mistakes" involve two products (a collection on a credit report).
- Near-duplicate templates survive exact dedupe (cosine similarity 0.998-0.999 found in notebook 04) and may inflate scores.
- Time drift: the 95% auto-routed target set on validation fell to 93.4% on test.
- One 2-month batch, US only; CFPB product names changed in 2026, so the model would need relabelling for newer complaints.
- "Other" mixes two unrelated products.
