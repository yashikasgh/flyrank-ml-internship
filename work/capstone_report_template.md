# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Yashika Singh
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/yashikasgh/flyrank-ml-internship
- **Date:** October 2026

## 0. Abstract

This project investigates whether anonymized content-level search signals can help prioritize pages for content-refresh review. The analysis uses a 30,000-row anonymized dataset and defines a down-trend label from `trend_direction == "down"`. A transparent rule-based baseline and a Logistic Regression model are evaluated using random and client-grouped train/test splits, with target-related fields, identifiers, and potentially overlapping recent-window metrics excluded from the feature set. On the client-grouped test split, the baseline outperformed Logistic Regression on Precision@20 (0.60 versus 0.30), Precision@50 (0.56 versus 0.38), and average precision (0.520 versus 0.514). The resulting action playbook is intended to support human review and monitoring decisions, not to automate content changes or establish causal effects.

## 1. Problem framing

### Decision to support

The project addresses the question: **Which content records should an editor review first for a potential refresh, and which should be monitored instead?**

- **Unit of analysis:** An anonymized content-level record.
- **Output:** A rule-based priority score, model evaluation results, and an action queue with reason codes.
- **Human action:** Review the evidence for a record, decide whether to refresh or improve it, and monitor records that do not currently warrant intervention.
- **Cost of a wrong call:** Unnecessary edits can waste editorial effort or harm useful content. Missing a genuine opportunity can leave a page's performance issues unresolved.

A transparent baseline provides an interpretable starting point. Machine learning is explored to test whether a learned ranking signal can improve prioritization beyond that baseline.

This is an offline decision-support analysis. It does not predict Google's ranking algorithm or establish that refreshing content causes improved search performance.

## 2. Data safety

The analysis uses an anonymized content-level CSV with 30,000 rows and 44 columns. The dataset contains records associated with 32 clients. It is the anonymized starter dataset used in this project, not a demonstrated analysis of the full 79-million-row production warehouse.

The following fields or field groups were excluded from the model features:

- `trend_direction`: used to construct the target label.
- `trend_pct`: potentially target-related and therefore excluded.
- Recent and previous 30-day performance metrics: excluded because their measurement windows and potential overlap with the target period were not independently verified.
- `content_id` and other identifiers: excluded as predictive features.
- Client and domain identifiers, where present: not used as model features; client identity was used for grouped validation where applicable.
- `baseline_score`: excluded to prevent the model from learning directly from the comparison baseline.

The model's candidate features were:

**Numeric features**
- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `char_count`
- `content_age_days`
- `days_since_last_update`

**Categorical features**
- `competition_level`
- `content_type`
- `main_intent`
- `age_tier`
- `freshness_tier`
- `word_count_tier`
- `char_count_tier`

The target was defined as `is_down = 1` when `trend_direction == "down"` and `0` otherwise.

Public artifacts should contain only aggregate results, anonymized examples where appropriate, and non-identifying visualizations. Client names, domains, private queries, credentials, and raw production exports must not be published.

**Leakage limitation:** Column-level exclusions reduce obvious leakage risks but do not prove the analysis is leakage-free. Exact label-generation timestamps and feature measurement windows still require verification.

## 3. Baseline

The project begins with an interpretable rule-based baseline.

The baseline assigns:
- **40 points** when `days_since_last_update >= 180`.
- **40 points** when `impressions_90d` is at or above the dataset median.
- **20 points** when `ctr` is below the dataset median.

The score is the sum of these conditions. It is easy to inspect and provides a transparent comparison for deciding which records to review first.

The baseline is evaluated using the same test split and ranking metrics as the model. The following results are from the client-grouped test split:

| Metric | Rule-based baseline | Logistic Regression |
|---|---:|---:|
| Precision@20 | 0.60 | 0.30 |
| Precision@50 | 0.56 | 0.38 |
| Average precision | 0.520 | 0.514 |

On this split, the baseline ranked down-labelled records more effectively at the evaluated cutoffs and achieved slightly higher average precision. The model did not demonstrate an improvement over the baseline on the grouped evaluation.

## 4. Model / analysis

The project uses Logistic Regression as an interpretable classification baseline for the down-trend label. The model uses numeric content/search signals and categorical content descriptors, while excluding direct target fields, identifiers, the baseline score, and potentially overlapping recent-window metrics.

The target is a proxy for the observed dataset label, not a verified future outcome. The evaluation therefore tests how well the features distinguish records labelled as down in the supplied dataset; it does not by itself establish prospective forecasting performance.

### Validation design

Two split strategies were explored:

1. **Random split:** 22,500 training rows and 7,500 test rows. There were 32 clients in training and 31 in testing, with 31 clients overlapping.
2. **Client-grouped split:** 22,885 training rows and 7,115 test rows. Training contained 24 clients and testing contained 8 clients, with zero client overlap.

The grouped split is the stronger test of generalization to unseen clients. However, it evaluates only eight held-out clients, so the results may vary with a different client sample.

## 5. Evaluation

### Random split results

| Metric | Rule-based baseline | Logistic Regression |
|---|---:|---:|
| Precision@20 | 0.75 | 0.80 |
| Precision@50 | 0.72 | 0.74 |
| Average precision | 0.583 | 0.651 |

The model performed better than the baseline on the random split. However, the split had substantial client overlap between training and testing, so these results may not represent performance on unseen clients.

### Client-grouped split results

| Metric | Rule-based baseline | Logistic Regression |
|---|---:|---:|
| Precision@20 | 0.60 | 0.30 |
| Precision@50 | 0.56 | 0.38 |
| Average precision | 0.520 | 0.514 |

The model's apparent advantage on the random split did not carry over to the client-grouped split. The baseline outperformed it at both evaluated precision cutoffs and had slightly higher average precision.

The down-label prevalence was approximately 54.2% in the random-split evaluation and 51.7% in the grouped test set. Precision should be interpreted alongside these base rates. The current results do not establish a robust improvement from Logistic Regression over the rule-based baseline.

### Error analysis

The difference between random and client-grouped evaluation suggests that the evaluation split materially affects the apparent usefulness of the learned ranking signal. Client overlap may contribute to the more favourable random-split results, but this experiment does not prove that the model memorizes client-specific patterns.

The grouped evaluation has only eight held-out clients, and temporal leakage has not been fully ruled out. Further evaluation should use verified feature timestamps, an appropriate time-aware split, and additional held-out clients.

## 6. Interpretation

The results support three main observations:

1. A simple, interpretable baseline is competitive and outperforms Logistic Regression on the client-grouped test split.
2. Random-split performance is more favourable to the model, but substantial client overlap limits how confidently those results can be generalized.
3. The current experiments are insufficient to claim reliable future prediction or causal effects from content refreshes.

The separate signal audit found weak positive Spearman associations with the down-trend label for `days_since_last_update` (ρ = 0.0495), `impressions_90d` (ρ = 0.1458), and `ctr` (ρ = 0.0272). These are unadjusted associations and should not be interpreted as causal effects.

In the rule-based flag audit, the down-label rate was 59.40% for the high-impressions group versus 49.01% for the unflagged group. The stale-content flag showed a smaller difference: 54.52% versus 52.32%. The low-CTR group had a lower down-label rate than the unflagged group: 51.97% versus 56.39%.

These exploratory findings are not proof that high impressions cause decline, that stale content should always be refreshed, or that low CTR is unimportant for other tasks.

## 7. Recommendation

The following actions are ranked by practical usefulness and the evidence available from this analysis.

1. **Use the transparent baseline to create a human-review queue.** It is easy to explain and outperformed Logistic Regression on the grouped test split. Treat its ranking as a review aid, not an automatic instruction to edit a page.
2. **Review high-impressions records with additional evidence.** The high-impressions flag had the largest observed difference in down-label rates. Editors should inspect query intent, page relevance, and available performance history before deciding on an intervention.
3. **Use staleness as supporting context, not a standalone trigger.** The observed down-label-rate difference was small, so the stale flag alone does not justify a refresh.
4. **Do not use the low-CTR flag as a standalone decline detector.** Its observed relationship with this down-trend label was negative. CTR may still be useful for a separate click-opportunity analysis.
5. **Improve validation before operational deployment.** Verify the timing of all features and the label, evaluate on a time-aware holdout, and repeat client-grouped evaluation across more clients.

**Confidence:** Moderate that the reported metrics describe the evaluated dataset and splits; low that the current results generalize to future periods or the full production warehouse. These recommendations support editorial investigation and human review, not automatic content modification.

## 8. Reproducibility

Repository: https://github.com/yashikasgh/flyrank-ml-internship

The analysis notebook is `work/notebooks/capstone.ipynb`. The related exploratory notebooks are:

- `work/notebooks/w03_feature_leakage_check.ipynb`
- `work/notebooks/w04_signal_audit.ipynb`

### Re-running the analysis

1. Clone the repository.
2. Open the notebooks in Google Colab or a compatible Jupyter environment.
3. Obtain the permitted anonymized starter CSV through the project's intended workflow. Do not commit private or raw production data.
4. Run the notebook cells in order.
5. Compare the freshly generated metrics and aggregate artifacts with the values reported here.

```bash
git clone https://github.com/yashikasgh/flyrank-ml-internship.git
cd flyrank-ml-internship
```

The exact random seed, complete environment versions, and a fully automated end-to-end command have not been independently verified for this report. They should be recorded from the actual notebook and environment before claiming fully deterministic reproduction. Any sealed or holdout evaluation should include both the code that constructs the evaluation frame and the resulting metrics artifact in the repository.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

