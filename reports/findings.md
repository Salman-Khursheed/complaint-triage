# Findings: routing consumer complaints with NLP

**Question.** Can a model read a consumer's complaint and route it to the right product team, sending only the uncertain ones to a person?

**Data.** 138,042 distinct complaint narratives from the CFPB Consumer Complaint Database (complaints received Jan-Feb 2025, from the CFPB's narratives archive), 10 product classes. Trained on Jan 1-24, threshold chosen on Jan 25-31, tested once on February (42,445 complaints).

## What we found

1. **A simple model does most of the work.** TF-IDF (words and word pairs) + logistic regression reached macro-F1 0.649 on February (majority-class baseline: 0.078 on validation). A pretrained sentence-embedding model (MiniLM) was worse on every product (validation macro-F1 0.598 vs 0.673) and took 49 minutes to encode the data on a CPU.

2. **Triage works, but drift costs coverage.** With a confidence threshold of 0.73 chosen on validation, the model routes **65.5% of February complaints automatically at 93.4% accuracy** and sends 14,626 (34.5%) to people. On validation the same threshold gave 72.6% coverage at 95.1%. The target of 95% was not met on test: February's product mix differed from January's (Money transfer fell from 23.5% to 6.8% of complaints).

3. **The threshold holds back the right complaints.** On the complaints sent to people, the model would have been right only 62.3% of the time, versus 93.4% on the ones it routed. Accuracy rises steadily with confidence (44.5% below 0.4, 98.4% above 0.95 on validation).

4. **The best model depends on the decision.** Balanced class weights gave the best macro-F1 on validation (0.673 vs 0.653), but made the model less confident: at 95% accuracy it could auto-route only 41.2% of complaints, against 72.6% for the default model. For triage, the default model wins.

5. **Big products are easy; small ones are not.** Credit reporting (58% of test) is auto-routed 84.7% of the time at 98.4% accuracy. Payday/personal loan (F1 0.346) and "Other" (F1 0.278) are rarely auto-routed, and when they are, they are usually wrong.

6. **Many "mistakes" are not really mistakes.** The most common confident error is Debt collection -> Credit reporting (791 complaints). In a hand-read sample of 30 confident errors, 13 were about two products at once (typically a loan or card account disputed on the credit report) and 7 were filed under a product the text doesn't match. Only 2 were clear model errors. (Small sample.)

## What this means for a complaints team

- Auto-routing about two-thirds of complaints at roughly 93% accuracy is realistic with a cheap, explainable model.
- The threshold should be re-checked every month: the share auto-routed and its accuracy move with the complaint mix.
- "Which team owns a credit-report dispute about a loan?" is a business rule, not a modelling problem. Fixing that rule would remove a large share of the errors.

## Limitations

- Narratives are opt-in (32.7% of complaints have one), so this is not every complaint.
- Product labels are chosen by consumers and are noisy; 175 identical texts were filed under different products.
- Near-duplicate template complaints survive exact deduplication and may make scores look better than they are.
- Only one two-month batch, US only. The CFPB renamed its products in 2026, and newer archive batches have almost no narratives (August 2026: 1 in 667,140), so the model can't simply be retrained on recent data.
- "Other" combines two unrelated small products (Prepaid card, Debt or credit management).

All numbers: `reports/results_summary.md`.
