# Capstone Report: Ranking Pages for Review

* Author: (type your name here)
* Lane: Lane 2, Refresh / Content Opportunity Scoring
* Repo: (paste your repo URL here)
* Date: 2026-11-08

## 1. Problem framing

The decision: which pages should a content reviewer open first? The unit of analysis is one page. The output is a ranked list with a reason code and a suggested action per page. A reviewer works down the list and decides to refresh, review, or monitor each page. A false alarm wastes limited review time, and a missed page keeps losing traffic unnoticed. Data helps because a reviewer cannot check every page, and a ranking built from evidence puts the most likely problems first.

## 2. Data safety

I used the anonymized starter dataset (`data/raw/content_refresh_anonymized.csv`): 30,000 pages, one row per page, 90-day signals. I excluded FlyRank product scores and flags (`health_score`, `priority_score`, `action_type`), the label `trend_direction` and its companion `trend_pct`, every `last_30d` and `prev_30d` column, and the identifiers `content_id` and `client_id` (used only for grouping). I tested one deliberate leak: a column copied from the label raised ROC AUC to 1.0, and removing it gave 0.723, which is how I confirmed the final features are clean. No client names, domains, URLs, or queries appear anywhere in `work/`.

## 3. Baseline

A transparent rule written before any model: score = percentile rank of days since last update x percentile rank of impressions, with one reason code (`stale_visible_page`: 180+ days since update and 500+ impressions) and one action (refresh or monitor). On the same client-held-out split as the models, it scored ROC AUC 0.475, average precision 0.472, precision@50 0.34, and precision@100 0.32, against a test base rate of 0.511. Its grouped 5-fold ROC AUC was 0.585 (sd 0.089).

## 4. Model / analysis

Task: classification used for ranking, with pages ranked by predicted probability of decline. Methods: logistic regression, decision tree (depth 5), random forest, and gradient boosting. Features: log impressions, log clicks, average position, log word count, log days since update, and one-hot content type. Missing word counts are filled with 0. The target is the proxy `trend_direction == "down"`, a bucket computed from the current window, not a future outcome.

## 5. Evaluation

Split: client-grouped holdout with 25 clients (23,837 pages) in training and 7 clients (6,163 pages) in testing, and zero clients in both. I also ran a grouped 5-fold check. All methods use the same split and metrics.

| Method | ROC AUC | Avg precision | P@50 | P@100 | Grouped 5-fold AUC |
|---|---|---|---|---|---|
| Baseline rule | 0.475 | 0.472 | 0.34 | 0.32 | 0.585 |
| Logistic regression | 0.610 | 0.607 | 0.76 | 0.77 | 0.662 |
| Decision tree | 0.585 | 0.562 | 0.62 | 0.58 | 0.628 |
| Random forest | 0.592 | 0.577 | 0.68 | 0.63 | 0.653 |
| Gradient boosting | 0.611 | 0.600 | 0.74 | 0.73 | 0.633 |

Test base rate: 0.511. Flagging the top 20 percent of test pages by predicted risk caught 770 declining pages, produced 466 false alarms, and missed 2,379 declining pages. False alarms are high-visibility pages (median 1,156 impressions, median position 15.4). Missed pages have lower traffic (median 309 impressions).

## 6. Interpretation

Every model beats the baseline, but the gains are modest, and the simple logistic regression matches the complex models. The baseline falls below 0.5 on the test clients, so staleness and visibility alone carried no usable signal there. Permutation importance (gradient boosting): impressions 0.070, clicks 0.064, average position 0.031, word count 0.003, days since update -0.013. This shows which signals the model used, not that any signal causes decline. The staleness assumption behind the refresh flags came out MIXED in my signal audit (for example, 0.512 declining in the freshest bucket versus 0.467 for 181-365 days). A caveat: the label compares the last 30 days with the previous 30, and my 90-day features cover the same days, so some skill may come from overlapping measurements.

## 7. Recommendation

Ranked by gradient boosting on the held-out clients, the top 20 pages declined 80 percent of the time and the top 100 declined 73 percent, against a 51 percent base rate. Of the top 100, 66 carry `visible_decline_risk` (500+ impressions and in the top 20 percent of risk), which means review first, and 34 carry `model_decline_risk`, which means monitor. A FlyRank editor would open the review-first pages and check whether the drop is real, seasonal, or absorbed by a related page before editing. A recommendation is wrong if the page is stable, seasonal, or a sibling took its demand. Confidence is moderate and directional: the sample is 30,000 pages from 32 clients, the test set has only 7 clients, and the label is a proxy.

## 8. Reproducibility

```
git clone <your-repo-url>
cd <your-repo>
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute work/notebooks/w04_baseline_score.ipynb
jupyter nbconvert --to notebook --execute work/notebooks/w05_model.ipynb
```

Random seed 42 is used for the split, the models, and permutation importance. Environment: Python 3 with pandas, numpy, scikit-learn, and matplotlib from `requirements.txt`. The deployed paper is `index.html` at the repo root.

Claims checklist: all claims are observed, measured, directional, or decision-support. There are no causal claims, no claims about Google's algorithm, and no client-identifying details. Metrics are reported next to the 0.511 base rate.
