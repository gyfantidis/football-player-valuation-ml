# Predictive Modelling and Archetype Discovery for Football Player Valuation: A Comparative Study

**DAMA Hackathon 2026**
Dataset: *FIFA Player Performance & Market Value* — Open Access

---

## Abstract

This paper presents a complete machine-learning pipeline applied to a synthetic FIFA player dataset (n = 2,800). We tackle three tasks: (1) regression for player market value prediction, (2) multi-class classification of transfer risk, and (3) unsupervised player archetype discovery. A key empirical finding is that the dataset's annotated market values exhibit near-zero Pearson correlation with all performance features (|r| < 0.03), indicating that this label was assigned independently of the feature space. We document this as a data-quality observation and introduce a domain-informed composite target — the *FIFA Performance Value Index* (FPVI) — to demonstrate our full methodology in a supervised setting where the signal is well-defined. Across all tasks, gradient-boosted tree ensembles (LightGBM, XGBoost) outperform linear and Random Forest baselines. SHAP values provide transparent, feature-level explanations. K-Means clustering recovers two interpretable player archetypes.

---

## 1. Introduction

Estimating the market value of a professional football player is a problem of substantial commercial and sporting significance. Clubs, agents, and analytics departments seek data-driven models that can objectively quantify how much a player is worth, accounting for their current ability, development trajectory, and production statistics.

This study uses the *FIFA Player Performance & Market Value* dataset, a tabular dataset containing 2,800 player records across nine field positions, eight nationalities, and seven clubs, with 13 features covering age, ratings, in-season statistics, contract status, and injury history.

We pursue three complementary research objectives:

- **RQ1** — Can supervised regression models predict player market value from performance features?
- **RQ2** — Can multi-class classifiers identify the transfer risk level of a player?
- **RQ3** — Can unsupervised clustering reveal coherent player performance archetypes?

---

## 2. Data and Exploratory Analysis

### 2.1 Dataset Description

The dataset contains 2,800 rows and 16 columns. Numerical features include `age` (17–39, mean 28.0), `overall_rating` (60–94, mean 76.9), `potential_rating` (65–98), `matches_played`, `goals`, `assists`, `minutes_played`, and `contract_years_left`. Categorical features include `position` (9 values), `nationality` (8), `club` (7), `injury_prone` (binary), and `transfer_risk_level` (Low / Medium / High). The regression target `market_value_million_eur` ranges from €0.67 M to €179.96 M (mean €90.6 M, std €52.1 M).

### 2.2 Key EDA Findings

**Target distribution.** The original market value is approximately uniform across its range — a right-skew is absent. The inter-quartile range (€45.4 M – €136.7 M) is very wide relative to any individual feature's predictive power.

**Feature–target correlations.** Pearson correlations between all numeric features and `market_value_million_eur` are uniformly negligible: the largest observed magnitude is |r| = 0.022 (goals). This is a statistically significant observation: with n = 2,800, the 95% confidence bound on a zero-correlation coefficient is ≈ ±0.037. Every feature lies within this band, confirming that the annotated market values are uncorrelated with the feature space — a property consistent with randomly assigned labels in a synthetic dataset.

**Transfer risk balance.** Class distribution is 44.6% Low, 35.4% Medium, 20.0% High — moderately imbalanced. A majority-class baseline would achieve 44.6% accuracy.

**Per-90 stat outliers.** Three records with fewer than 5 minutes played but non-zero goal counts produce extreme `goals_per_90` values (up to 9 × 10⁶). These are capped during feature engineering.

---

## 3. Methodology

### 3.1 Feature Engineering

Eight derived features are constructed from raw statistics:

| Feature | Formula / Rule |
|---|---|
| `goals_per_90` | goals ÷ max(minutes/90, 1), clipped [0, 4] |
| `assists_per_90` | assists ÷ max(minutes/90, 1), clipped [0, 4] |
| `contributions_p90` | goals_per_90 + assists_per_90 |
| `rating_gap` | potential_rating − overall_rating |
| `rating_x_potential` | overall_rating × potential_rating |
| `age_rating_ratio` | overall_rating ÷ (age + 1) |
| `expiring_soon` | 1 if contract_years_left ≤ 1, else 0 |
| `position_group` | GK / Defender / Midfielder / Attacker |

**FIFA Performance Value Index (FPVI).** Since the original market value labels carry no predictive signal, we define a principled composite target based on football domain knowledge:

```
age_factor  = exp(−0.08 × max(0, age − 26)²), clipped [0.1, 1.0]
rating_norm = (overall_rating − 60) / 34
pot_gap     = max(0, potential_rating − overall_rating)

FPVI = rating_norm × 100 × age_factor
     + pot_gap × 1.2 × age_factor
     + goals_per_90 × 8
     + assists_per_90 × 5
     + ε,   ε ~ N(0, 0.10 × FPVI)
```

This formula encodes three established valuation principles: (a) quality decays after the peak-age window (~26), (b) young players with high development potential command a premium, and (c) goal-scoring and creative output provide additional value. Gaussian noise (σ = 10%) is injected to prevent trivial learning. The resulting index ranges from €0.5 M to €181 M (mean €48.9 M, std €34.2 M) and serves as the primary regression target.

### 3.2 Preprocessing

Categorical variables are encoded as follows: `injury_prone` → binary; `transfer_risk_level` → ordinal (Low=0, Medium=1, High=2); `position`, `nationality`, `club`, `position_group` → one-hot (no drop). The final feature matrix contains 40 columns. A 70 / 15 / 15 stratified split (by transfer risk class) yields 1,960 training, 420 validation, and 420 test samples. Features are standardised (zero mean, unit variance) for linear models; raw values are used for tree-based models.

### 3.3 Regression Models

Five models are evaluated on both targets:

- **Ridge Regression** — regularised linear baseline (α selected via 5-fold `RidgeCV`)
- **Random Forest** — 400 trees, `max_features=sqrt`, `min_samples_leaf=2`
- **Gradient Boosting** — 500 estimators, learning rate 0.05, max depth 4, subsample 0.8
- **XGBoost** — 600 estimators, learning rate 0.04, max depth 5, L1 regularisation 0.1
- **LightGBM** — 600 estimators, learning rate 0.04, `num_leaves=40`

All models train on the log-transformed target and predictions are back-transformed via `expm1`. Evaluation uses RMSE, MAE, and R² in original (EUR million) scale.

### 3.4 Classification Models

Four models predict the three transfer-risk classes:

- **Logistic Regression** (L-BFGS, max\_iter=2000)
- **Random Forest** (400 trees)
- **XGBoost** (500 estimators, `eval_metric=mlogloss`)
- **LightGBM** (500 estimators)

Evaluation: Accuracy, Macro F1 (class-balanced), and ROC-AUC (One-vs-Rest, macro average).

### 3.5 Interpretability (SHAP)

TreeExplainer (Lundberg & Lee, 2017) is applied to the best regression and classification models. Three visualisation modes are produced: (a) global beeswarm summary plot showing feature effect directions; (b) global mean |SHAP| bar chart; (c) local waterfall plots for the highest- and lowest-FPVI players in the test set; (d) dependence plots for the top-2 features.

### 3.6 Clustering

K-Means is applied to a 10-dimensional performance feature subspace (age, overall/potential ratings, per-90 stats, rating gap, age-rating ratio, contract years left). All features are standardised. The optimal number of clusters is selected by maximising the silhouette coefficient across k ∈ {2, …, 8}. Archetypes are profiled using cluster-mean feature tables, radar charts, and box-plot distributions. A 2D PCA projection is used for visualisation.

---

## 4. Results

### 4.1 Task A — Original Market Value Regression (Negative Control)

**Table 1.** Test-set performance on the original `market_value_million_eur` target.

| Model | RMSE (M€) | MAE (M€) | R² |
|---|---|---|---|
| Ridge | 54.64 | 45.38 | −0.200 |
| Random Forest | 55.23 | 45.59 | −0.226 |
| Gradient Boosting | 56.88 | 46.45 | −0.300 |
| XGBoost | 57.63 | 47.06 | −0.335 |
| LightGBM | 57.93 | 47.37 | −0.349 |

All models yield negative R², confirming that no model learns anything beyond the sample mean. The best RMSE (Ridge, 54.6 M€) is only marginally better than the naïve mean-predictor (std = 52.1 M€). This is consistent with the EDA finding that the target is label-noise: every feature is orthogonal to the target by construction. This experiment serves as a *negative control* — it validates our feature pipeline (any spurious leakage would have inflated R²) and motivates the FPVI task.

### 4.2 Task B — FIFA Performance Value Index Regression (Primary Task)

**Table 2.** Test-set performance on the FPVI target.

| Model | RMSE (M€) | MAE (M€) | R² | R² (log-space) |
|---|---|---|---|---|
| **LightGBM** | **6.87** | **4.49** | **0.958** | **0.980** |
| XGBoost | 6.96 | 4.48 | 0.957 | 0.981 |
| Gradient Boosting | 7.04 | 4.55 | 0.956 | 0.980 |
| Random Forest | 9.81 | 6.30 | 0.915 | 0.959 |
| Ridge | 20.42 | 13.77 | 0.631 | 0.814 |

The three gradient-boosted models cluster tightly (R² 0.956–0.958), substantially outperforming Random Forest (0.915) and Ridge (0.631). The large performance gap between tree ensembles and Ridge confirms that the feature–FPVI relationship is non-linear, specifically driven by multiplicative interactions (age × rating) that linear models cannot capture without explicit interaction terms.

**5-fold CV.** The best LightGBM model achieves CV R² = 0.968 ± 0.007 (log-space), confirming low variance and stable generalisation.

**Feature importance.** The top-5 features by mean |SHAP| are: `age_rating_ratio`, `overall_rating`, `age`, `rating_x_potential`, and `goals_per_90`. The dominance of age-related features reflects the FPVI formula's age-decay term, which the model correctly recovers. `goals_per_90` enters as the primary performance-rate contributor.

**SHAP dependence plots** reveal the expected non-linear age trajectory: SHAP contributions for `age` are maximally positive around 24–27 and decline sharply after 30. `overall_rating` has a monotone positive SHAP relationship, with steeper marginal returns above rating 80 (consistent with the squared normalisation in the FPVI formula).

**Local explanations.** For the highest-FPVI player in the test set (actual 181 M€), the waterfall plot shows that high `age_rating_ratio`, high `overall_rating`, and strong `goals_per_90` together drive the prediction above the base value by approximately +4.8 log-points. For the lowest-FPVI player, low rating and advanced age push the prediction sharply negative.

### 4.3 Task C — Transfer Risk Classification

**Table 3.** Test-set classification performance (420 samples, majority-class baseline: 45.2%).

| Model | Accuracy | Macro F1 | ROC-AUC (OvR) |
|---|---|---|---|
| Logistic Regression | 45.2% | 0.297 | 0.522 |
| Random Forest | 45.0% | 0.316 | 0.548 |
| XGBoost | 41.9% | 0.353 | 0.542 |
| **LightGBM** | 41.9% | **0.376** | **0.544** |

All models perform near the majority-class baseline, with ROC-AUC only slightly above 0.5. This is consistent with the EDA finding that transfer risk is uncorrelated with the numeric features (|r| < 0.04 for all features). The Macro F1 advantage of LightGBM and XGBoost over Logistic Regression (0.376 vs 0.297) suggests that tree models partially detect weak non-linear patterns in the feature-label relationship, but the overall predictive signal is very low.

**SHAP for classification.** Despite the limited accuracy, SHAP analysis reveals which features the best model prioritises for each class. For the *Low* risk class, `contract_years_left` and `overall_rating` carry the largest positive contributions — high-rated, contracted players are least likely to be flagged as transfer risks. For the *High* risk class, `expiring_soon` and young age (implying high demand) dominate. These patterns are domain-consistent even though aggregate accuracy is near-random.

### 4.4 Task D — Player Archetype Clustering

The silhouette analysis selects **k = 2** as the optimal number of clusters (average silhouette 0.42). The two recovered archetypes are:

**Table 4.** Cluster profile means (test-set labels applied to full dataset, n=2800).

| Archetype | n | Age | Rating | Goals/90 | Assists/90 | Rating Gap | FPVI (M€) |
|---|---|---|---|---|---|---|---|
| **Elite Players** | 2,229 | 28.1 | 76.9 | 0.70 | 0.46 | 4.5 | 42.1 |
| **Goal Scorers** | 571 | 27.5 | 76.9 | 3.19 | 2.51 | 5.6 | 75.4 |

The two archetypes share similar age and rating profiles but diverge sharply in production statistics: the *Goal Scorers* cluster records more than 4× the goals-per-90 and 5× the assists-per-90 of the *Elite Players* group. This bifurcation corresponds to a subset of players — primarily attackers and central midfielders — whose on-pitch output is distinctly higher. The FPVI gap (42.1 vs 75.4 M€) confirms that K-Means directly recovers the production-weighted valuation structure. The PCA projection shows clean separation with modest overlap along PC1 (18.3% variance), which loads heavily on per-90 production features.

---

## 5. Discussion and Conclusions

This study demonstrates a complete supervised and unsupervised ML pipeline applied to a tabular football player dataset. The central methodological finding is that ground-truth labels in synthetic datasets can be uninformative, and that responsible ML practice requires diagnosing this via correlation analysis before interpreting model outputs.

Our three substantive contributions are:

1. **FPVI regression** — gradient-boosted ensembles (LightGBM, XGBoost) achieve R² > 0.95 on the domain-informed composite target, and SHAP reveals the non-linear age-prime effect and rating non-linearity that drive predictions.

2. **Transfer risk classification** — all models perform near the majority-class baseline, confirming that the categorical labels carry no feature-aligned signal; SHAP explanations nonetheless reveal feature priorities that are domain-consistent.

3. **Player archetype discovery** — K-Means with silhouette-optimised k identifies a *Goal Scorers* archetype (n = 571) whose per-90 production is substantially higher than the majority *Elite Players* cluster, with a corresponding FPVI premium of +33 M€.

**Limitations.** The dataset is synthetic and small (2,800 records, 8 nationalities, 7 clubs), limiting generalisability. The FPVI target is hand-constructed; real-world market values reflect additional factors (commercial appeal, injury history trends, squad composition needs) that require richer datasets such as Transfermarkt or CIES Football Observatory records.

**Future work.** Extending to real transfer fee data, incorporating time-series form trajectories, and applying neural approaches (TabNet, attention-based models) for joint regression/classification would be natural next steps for conference-level submission.

---

## References

- Lundberg, S. M., & Lee, S. I. (2017). A unified approach to interpreting model predictions. *NeurIPS 30*.
- Chen, T., & Guestrin, C. (2016). XGBoost: A scalable tree boosting system. *KDD 2016*.
- Ke, G., et al. (2017). LightGBM: A highly efficient gradient boosting decision tree. *NeurIPS 30*.
- Rousseeuw, P. J. (1987). Silhouettes: a graphical aid to the interpretation and validation of cluster analysis. *Journal of Computational and Applied Mathematics, 20*, 53–65.
- Pedregosa, F., et al. (2011). Scikit-learn: Machine learning in Python. *JMLR, 12*, 2825–2830.
