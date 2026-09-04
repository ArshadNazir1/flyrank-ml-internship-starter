# Capstone Report

- **Author:** Arshad Nazir
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/ArshadNazir1/flyrank-ml-internship-starter
- **Date:** 2026-09-04

## 1. Problem framing

This capstone asks: **Which content pages should a content team review first based on observable search and content signals?**

The decision is content-review prioritization. The unit of analysis is an anonymized content page. The output is a model score and a ranked, reason-coded review queue. A content editor or SEO analyst can use the queue to decide which pages to check first rather than manually reviewing every page.

After a page is flagged, the human reviewer still decides whether the appropriate action is to refresh, expand, protect, prune, or monitor it. The model does not make publishing or editing decisions.

The cost of a wrong call is asymmetric. Flagging a page that is actually fine mainly costs reviewer time. Missing a page that needs attention can leave it unreviewed. For that reason, recall is considered alongside precision and accuracy.

ML helps because the potential review signal combines several observable factors, including position, freshness, visibility, content characteristics, search demand, competition, intent, and related fields. A model can combine these signals into a consistent ranking rather than relying on one manually written rule. The model is only useful if it improves on a transparent baseline on the same evaluation data.

## 2. Data safety

This study uses the **FlyRank ML Internship anonymized starter release**, specifically the `content_refresh_anonymized.csv` content-refresh table. It contains a single approximately 90-day performance snapshot rather than a multi-period time series.

- **Total pages:** 30,000
- **Pages used for modeling:** 28,795
- **Pages without a ranking position excluded:** 1,205
- **Validated client groups:** 31
- `client_id` and `content_id` are hashed pseudonyms.
- No client names, domains, URLs, or raw search queries are used in the paper.

The model features include:

- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `char_count`
- `avg_position`
- `impressions_90d`
- `impressions_last_30d`
- `impressions_prev_30d`
- `days_since_last_update`
- `content_age_days`
- `days_with_impressions`
- `has_word_count`
- `has_char_count`
- `has_search_volume`
- `has_competition`
- `has_cpc`
- `content_type`
- `main_intent`
- `competition_level`
- `position_tier`
- `provider_used`
- `model_used`

The following outcome-adjacent fields were deliberately excluded from model inputs because the target is built directly from CTR:

`ctr`, `clicks`, `sessions`, `pageviews`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`, `trend_direction`, and `trend_pct`.

`client_id` is used only for grouped validation and is never a model feature. `content_id` is an identifier only. FlyRank product decision flags such as health scores and quick-win tags are not used.

The leakage audit confirmed that the excluded outcome-adjacent fields were absent from the final feature list used for training.

## 3. Baseline

The fair baseline is a transparent **position-vs-tier-median rule**. A page is flagged when its average position is worse than the median position for its position tier.

An earlier CTR-based staleness rule was not used as the final model baseline because it directly uses CTR, while the model target is constructed from CTR. Using that rule for comparison would allow the baseline to see the answer and would make the comparison circular.

On the same evaluated data, the position baseline achieved:

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC |
|---|---:|---:|---:|---:|---:|
| Position baseline (client-grouped) | 0.547 | 0.502 | 0.544 | 0.522 | — |

## 4. Model / analysis

### Target / proxy

There is no ground-truth manual “needs refresh” label in this dataset. The target is therefore a proxy: **`ctr_underperforming` is true when a page's CTR is below the median CTR of other pages in its own position tier.**

### Model

The model is **Logistic Regression** with `class_weight="balanced"` inside a preprocessing pipeline.

Numeric features are median-imputed and scaled. Categorical features are one-hot encoded.

The validation design uses `StratifiedGroupKFold` with five folds, grouped by `client_id`. This keeps pages from the same client on one side of each fold and reduces the risk that client-specific patterns appear in both training and test data. A time-aware split was not possible because the dataset is a single snapshot.

### Leakage controls

The target uses CTR, so CTR and other outcome-adjacent fields were kept out of the feature set. The leakage audit checked the final feature list before training. `client_id` was retained only as a grouping variable for validation.

## 5. Evaluation

The primary evaluation uses a **five-fold client-grouped validation design**. This is the more conservative and trusted result because the model is evaluated on held-out client groups rather than rows from clients that may also appear in training.

The base rate is **45.4%** of ranked pages being `ctr_underperforming`. This is reported so the classification metrics are interpreted against the task's class balance.

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC |
|---|---:|---:|---:|---:|---:|
| Position baseline (client-grouped) | 0.547 | 0.502 | 0.544 | 0.522 | — |
| Logistic Regression — random row split | 0.752 | 0.736 | 0.709 | 0.722 | 0.834 |
| Logistic Regression — client-grouped split (honest) | **0.737** | **0.774** | **0.593** | **0.672** | **0.804** |

The client-grouped result is the one to trust for the main claim. The random-row result is included only to show why grouping matters: pages from the same client can share patterns, so a random row split can produce more optimistic results.

The grouped model beats the position baseline on the reported classification metrics. Its precision of 0.774 is higher than its recall of 0.593. This means the model is better at avoiding false flags than at finding every underperforming page, so a low model score must not be interpreted as proof that a page is fine.

### Error analysis

The validation errors show that the model does not perfectly separate the proxy target. False positives represent pages the model flags even though they do not meet the proxy definition, while false negatives represent pages that meet the proxy definition but receive a lower score. This supports using the output as a review-prioritization signal rather than as an automatic decision.

## 6. Interpretation

The model finds a multi-factor relationship between the available content/search features and the CTR-underperformance proxy. The score is useful as a ranking signal, but it is not a calibrated probability.

In the model analysis, permutation importance highlighted several variables, with `days_with_impressions` and `position_tier` among the strongest contributors. Other contributors included `model_used`, `avg_position`, `competition_level`, `main_intent`, `content_type`, and recent-impression fields. These are model associations, not causal effects.

A notable observed pattern is freshness. Pages untouched for 181+ days had the highest mean model score in the freshness comparison. This is an observed pattern in the single snapshot, not evidence that staleness causes underperformance or that refreshing a page will improve its search performance.

The negative result is equally important: the model does not eliminate false negatives. Recall remains well below 1.0, so the absence of a high score cannot be treated as evidence that a page is healthy.

## 7. Recommendation

The model output is converted into a five-band ranked action queue. Thresholds are based on the observed model-score distribution, using the top decile and top quartile of model score plus the top quartile of 90-day impressions rather than arbitrary hand-picked cutoffs.

| Priority | Recommended action | Reason code | Pages |
|---|---|---|---:|
| 1 | Refresh Content — Human Review | `RC_HIGH_MODEL_AND_STALE` | 83 |
| 2 | Review Content and Search Alignment | `RC_HIGH_MODEL_RISK` | 2,797 |
| 3 | Review Search Result Presentation | `RC_HIGH_VISIBILITY_CTR_GAP` | 1,580 |
| 4 | Review Content Candidate | `RC_MODEL_REVIEW` | 4,301 |
| 5 | Monitor | `RC_MONITOR` | 20,034 |

The intended workflow is:

1. Start with Priority 1 and review the highest-priority pages first.
2. Confirm that the stated reason actually matches the page's real situation.
3. Check the page's context, including whether it has meaningful impressions.
4. Consider seasonality because the analysis uses an approximately 90-day window.
5. Decide manually whether to refresh, expand, protect, prune, or monitor the page.

The output should **not** be used for automatic publishing, editing, removing, or pruning. A CTR gap alone is not sufficient evidence for automatically changing a title or snippet.

Confidence is limited to directional decision support in this evaluated dataset. The model does not prove that refreshing a flagged page will improve performance, does not predict Google's algorithm, and does not guarantee the same performance on future or unseen data.

## 8. Reproducibility

The capstone notebook is the source of the reported analysis and charts. The work uses a fixed random seed of **42**.

From a fresh clone:

```bash
git clone https://github.com/ArshadNazir1/flyrank-ml-internship-starter.git
cd flyrank-ml-internship-starter
pip install -r requirements.txt
jupyter notebook work/notebooks/capstone.ipynb
```

Run the capstone notebook from start to finish. The notebook contains the data preparation, feature construction, target definition, baseline comparison, grouped validation, leakage checks, action queue, and figures.

The weekly notebooks supporting this capstone are:

- `work/notebooks/w01_research_question.ipynb`
- `work/notebooks/w02_ml_task_framing.ipynb`
- `work/notebooks/w03_data_contract.ipynb`
- `work/notebooks/w04_baseline_score.ipynb`
- `work/notebooks/w05_model.ipynb`
- `work/notebooks/w06_validation_audit.ipynb`
- `work/notebooks/w07_action_playbook.ipynb`
- `work/notebooks/capstone.ipynb`

The public paper is deployed through GitHub Pages from `docs/index.html`, with the figures stored under `docs/img/`.

---

**Claims checklist**

- Claims are framed as observed, measured, directional, or decision-support findings.
- The task base rate (45.4%) is reported alongside the classification metrics.
- The client-grouped evaluation is treated as the primary result.
- No causal claims are made.
- No claim is made to predict Google's algorithm.
- No client-identifying details are included.
- The model output is described as a ranking/review signal, not an automated publishing decision.
- The limitations of the single approximately 90-day snapshot, grouped validation scope, proxy target, and recall are stated explicitly.

**Data credit:** Built on the FlyRank ML Internship dataset — https://flyrank.ai
