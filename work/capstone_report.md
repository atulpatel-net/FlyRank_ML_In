# Content Decline Prioritization from Historical Search Performance

**Author:** Atul Patel  
**Lane:** Refresh / Content Opportunity Scoring 
**Repo:** FlyRank_ML_In  
**Date:** August 2026

## Abstract

This study asks whether historical search-performance signals can help prioritize content that may warrant human review for declining organic search visibility. Using anonymized FlyRank ML Internship search-performance data, the analysis constructs previous-30-day impressions, clicks, and average-position features and defines an observed decline when subsequent 30-day impressions fall below 80% of the previous baseline. A Random Forest classifier was evaluated against a transparent Week-4 rule-based baseline using a client-grouped 75/25 validation split that prevents client overlap between training and test data. On this held-out split, the Random Forest measured an F1 score of 0.735 compared with 0.018 for the rule-based baseline, with measured precision of 0.670 and recall of 0.813. The resulting model output is treated as directional decision-support for prioritizing human content review, not as a causal prediction of future search performance or an automated recommendation to change content.

## 1. Problem framing

### Research question

Can historical search-performance signals be used to prioritize content that may warrant human review for declining organic search visibility?

### Decision supported

The unit of analysis is a **client-content pair**. The analysis produces a decline classification and ranking signal that can be used to prioritize which content should be reviewed first.

A FlyRank editor or SEO practitioner can use the ranked output to investigate high-priority content, review search intent and relevance, and decide whether to refresh, monitor, investigate further, or take no action.

### Why data and ML help

Large content portfolios can contain many pages whose search performance changes over time. A transparent rule can identify some cases, but it may not prioritize observations consistently across different historical performance patterns.

The model provides a measured way to combine a small set of historical search-performance signals into a review-prioritization score. The purpose is therefore not to automate editorial decisions, but to reduce the amount of manual screening required before human review.

### Cost of a wrong call

A false positive can cause an editor to spend time investigating content that does not require intervention. A false negative can cause potentially useful content to receive less attention than it otherwise would.

For this reason, the output is treated as **decision-support**, with human review required before any content action is taken.

## 2. Data safety

The analysis uses the FlyRank ML Internship dataset from the `FlyRank/internship-warehouse` release, specifically the `fact_content_daily_performance` data.

The available source data spans **27 January 2025 through 30 June 2026** and contains approximately **78.8 million daily records** across the available monthly files.

### Data used

The analysis uses:

- `report_date`
- `client_hash_id`
- `content_hash_id`
- `gsc_impressions`
- `gsc_clicks`
- `gsc_avg_position`

The final model features are previous-30-day impressions, clicks, and average position.

### Deliberate exclusions

`client_hash_id` and `content_hash_id` are **pseudonymous identifiers**. They are used for grouping, joins, and validation only; they are not model features.

Label-derived fields such as `trend_direction`, `trend_pct`, or equivalent target-period outcomes are not used as model features because they could leak information about the outcome being predicted.

The model also does not use client names, URLs, private search queries, or other client-identifying information.

### Leakage controls

The decline label is based on the subsequent 30-day impression window, while model features are constructed from the preceding 30-day window.

The primary validation uses a client-grouped split, ensuring that no client appears in both the training and test sets. The resulting client overlap was **0**.

Rows without positive previous-30-day impressions were excluded because a decline relative to a positive historical baseline cannot be meaningfully defined for those observations.

### Public safety

The underlying bulk dataset is not committed to the repository. Public outputs contain anonymized identifiers and aggregate/model results only. No client-identifying URLs, names, or private search queries are included in the research artifact.

## 3. Baseline

The initial benchmark was a transparent rule-based classifier developed before the Random Forest model.

The rule flags an observation when all three conditions are met:

- Previous-30-day impressions are at least 100.
- Previous-30-day clicks are zero.
- Previous-30-day average position is 10 or better.

An observation satisfying all three conditions is classified as declining.

This baseline provides a simple, interpretable comparison because it uses the same historical performance signals as the model and requires no learned parameters.

### Baseline performance

The rule was evaluated on the same client-grouped test split used for the Random Forest:

| Model | Accuracy | Precision | Recall | F1 |
|---|---:|---:|---:|---:|
| Week-4 Rule Baseline | 0.346 | 0.517 | 0.009 | 0.018 |
| Random Forest | 0.616 | 0.670 | 0.813 | 0.735 |

The baseline therefore provides a transparent reference point for evaluating whether the learned model adds useful measured classification signal beyond the initial rule.

## 4. Model / analysis

A Random Forest classifier was used to model the relationship between historical search-performance signals and the observed decline label.

### Features

The model uses three features, all calculated from the previous 30-day window:

| Feature | Description |
|---|---|
| `imp_prev30` | Total Google Search Console impressions |
| `clk_prev30` | Total Google Search Console clicks |
| `pos_prev30` | Average Google Search Console position |

Client and content identifiers were deliberately excluded from the feature set. They are used only for grouping, traceability, and validation.

### Target definition

The target is defined as:

`is_declining = 1` when `imp_last30 < 0.8 × imp_prev30`; otherwise `0`.

This represents an **observed 20% or greater reduction in impressions** between the previous and most recent 30-day windows.

### Model configuration

The Random Forest uses:

- 200 decision trees
- `random_state = 42`
- Parallel training using all available CPU cores

The model was selected as a practical nonlinear classifier that can capture interactions between the historical performance features without requiring a manually specified decision boundary.

### Analytical assumptions

The analysis assumes that historical impressions, clicks, and average position contain useful directional information for prioritizing content for review.

It does not assume that these features explain the cause of a decline, that the model represents a calibrated probability, or that a predicted decline implies a specific content intervention will be successful.

## 5. Evaluation

### Validation design

The primary evaluation uses a **client-grouped 75/25 train-test split** with `random_state = 42`.

The split produced:

- Training observations: **194,871**
- Test observations: **42,558**
- Unique clients: **56**
- Client overlap between train and test: **0**

This design is preferred over a simple row-level random split because observations from the same client can otherwise appear in both training and test data.

### Class distribution

The full modeling dataset contains:

- Declining: **169,864 observations (71.5%)**
- Non-declining: **67,565 observations (28.5%)**

The majority-class rate is therefore approximately **71.5%**. Accuracy should be interpreted alongside precision, recall, and F1 rather than on its own.

### Model vs baseline

Both approaches were evaluated on the same client-grouped test observations:

| Model | Accuracy | Precision | Recall | F1 |
|---|---:|---:|---:|---:|
| Week-4 Rule Baseline | 0.346 | 0.517 | 0.009 | 0.018 |
| Random Forest | 0.616 | 0.670 | 0.813 | 0.735 |

The Random Forest measured substantially higher precision, recall, and F1 than the transparent rule-based baseline on this split.

Its accuracy of 0.616 is below the 71.5% majority-class rate, which is an important reason not to present accuracy alone as evidence of model quality. The F1 result provides a more useful view of the model's classification performance on the defined task.

### Error analysis

On the 42,558-observation client-grouped test set, the Random Forest produced:

| Outcome | Count |
|---|---:|
| Correct classifications | 26,216 |
| False positives | 11,133 |
| False negatives | 5,209 |

The larger number of false positives indicates that the model sometimes flags observations as declining when they do not match the defined decline label. In this decision-support setting, such errors primarily represent additional human-review effort.

False negatives are fewer but represent observations matching the decline label that the model did not identify. These cases illustrate why the ranked output should not be treated as a complete or guaranteed detection system.

The error analysis therefore supports using the model as a **prioritization aid**, with human review remaining necessary before any content action.

### Validation interpretation

The client-grouped evaluation provides a more conservative assessment than the earlier row-level random split. The row-level split measured an F1 score of **0.781**, while the client-grouped split measured **0.735**.

This reduction is consistent with the stricter validation design and illustrates why the grouped result is used as the primary result in this paper.

The evaluation supports the conclusion that the Random Forest provides **measured classification signal for the defined decline label under this validation setup**. It does not establish future-period performance, causality, or guaranteed improvement from subsequent content actions.

## 6. Interpretation

The analysis suggests that recent historical search-performance signals contain useful directional information for distinguishing observations that match the defined decline label.

The Random Forest uses previous-30-day impressions, clicks, and average position jointly rather than relying on a single threshold. This allows the model to identify combinations of historical performance signals that the simple Week-4 rule does not capture.

### What the model output means

The model's output is most useful as a prioritization signal. A higher decline score indicates that an observation is more strongly associated with the learned patterns corresponding to the defined decline label in the evaluation data.

The score should not be interpreted as a calibrated probability or as evidence that a particular page will definitely decline.

### Interpretation for content review

The Week-7 action playbook translates the model output into human-readable reason codes and content archetypes. Examples include:

- High model risk with low prior impressions.
- High model risk with no prior clicks.
- High model risk with weaker prior position.
- High model risk among more established or higher-visibility content.

These signals can help determine what should be investigated first, but they do not explain why a page's performance changed.

### Important negative finding

The analysis does not establish that any individual historical feature causes search-performance decline. It also does not establish that refreshing a page will reverse an observed decline.

The most defensible interpretation is therefore that the model provides a **measured, directional signal for prioritizing human review** under the tested validation design.

# Random Forest feature importance

feature_importance = pd.DataFrame({
    "Feature": feature_cols,
    "Importance": rf.feature_importances_
}).sort_values("Importance", ascending=False)

display(feature_importance.round(3))

plt.figure(figsize=(7, 5))

plt.bar(
    feature_importance["Feature"],
    feature_importance["Importance"]
)

plt.ylabel("Feature importance")
plt.xlabel("Feature")
plt.title("Random Forest Feature Importance")

plt.tight_layout()

feature_importance_path = figures_dir / "capstone_feature_importance.png"

plt.savefig(
    feature_importance_path,
    dpi=200,
    bbox_inches="tight"
)

plt.show()

print("Figure saved:", feature_importance_path)

## 7. Recommendation

The model output should be used as a **ranked review queue**, not as an automated content-decision system.

### Ranked actions

| Priority | Situation | Recommended action |
|---|---|---|
| 1 | High model risk + high visibility | Investigate first and preserve valuable existing content before changes. |
| 2 | High model risk + weaker position | Review search intent, relevance, and on-page coverage. |
| 3 | High model risk + low prior visibility | Investigate search opportunity and business value before considering a refresh. |
| 4 | Established content with elevated model risk | Review recent performance and relevance before deciding on intervention. |

### Human review workflow

A FlyRank editor could use the queue as follows:

1. Start with the highest-ranked observations.
2. Review the model score and reason codes.
3. Check search intent, content relevance, historical context, and business value.
4. Decide whether to refresh, investigate further, monitor, or take no action.
5. Measure subsequent performance separately if an intervention is made.

### Confidence and limits

The recommendations are **directional decision-support signals** based on measured historical patterns. They should not be interpreted as proof that a page needs a refresh or that a refresh will improve performance.

### No-go cases

The model should **not** independently:

- Rewrite or publish content.
- Delete or redirect pages.
- Change technical SEO settings.
- Declare a page successful or unsuccessful.
- Make client-facing recommendations without human review.
- Treat a model score as a guaranteed future outcome.

The practical value of the system is therefore in **prioritizing human attention**, while the final content decision remains with a qualified reviewer.

## 8. Reproducibility

The analysis is organized as a sequence of research notebooks covering the progression from research question through validation and action recommendations.

### Research workflow

- [Week 1 — Research question](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w01_research_question.ipynb)
- [Week 2 — ML task framing](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w02_ml_task_framing.ipynb)
- [Week 3 — Data contract](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w03_data_contract.ipynb)
- [Week 3 — Feature leakage check](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w03_feature_leakage_check.ipynb)
- [Week 4 — Baseline score](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w04_baseline_score.ipynb)
- [Week 4 — Signal audit](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w04_signal_audit.ipynb)
- [Week 5 — Model](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w05_model.ipynb)
- [Week 6 — Validation audit](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w06_validation_audit.ipynb)
- [Week 7 — Action playbook](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/w07_action_playbook.ipynb)
- [Capstone analysis](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/work/notebooks/capstone.ipynb)

### Environment

The project environment is specified in [`requirements.txt`](https://github.com/atulpatel-net/FlyRank_ML_In/blob/main/requirements.txt). The analysis uses Python, DuckDB, pandas, NumPy, and scikit-learn.

The primary model evaluation uses `random_state = 42` and a 75/25 client-grouped split.

### Data access

The analysis references the FlyRank ML Internship dataset rather than committing the underlying bulk data to the repository.

The capstone notebook contains the feature construction, target definition, validation split, model training, baseline comparison, and artifact-generation steps needed to reproduce the reported results.

### Repository

The complete research workflow and supporting artifacts are available in the [FlyRank_ML_In GitHub repository](https://github.com/atulpatel-net/FlyRank_ML_In).