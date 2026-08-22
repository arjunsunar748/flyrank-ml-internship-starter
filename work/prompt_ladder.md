# Prompt Engineering Assignment: The Prompt Ladder

**Track:** FlyRank ML / Search & Content Ranking  
**Objective:** Systematic 5-layer refinement of an ML task prompt.

---

## Baseline (Version 0: Weak Prompt)

### Prompt
> *"Help me rank content in my dataset."*

### Output Excerpt
> *"To rank content in your dataset, you can use Python! First, install Pandas and Scikit-Learn. Then load your CSV file using `pd.read_csv()`. You can sort your data using `df.sort_values(by='rating')` or train a simple Logistic Regression model using `sklearn`. Let me know if you want code for linear regression or decision trees!"*

---

## Version 1: Added Layer — Clear Goal

### Prompt
> *"Help me rank content in my dataset by building a continuous probability scoring model that predicts whether a user will meaningfully engage with a given piece of content."*

### Output Excerpt
> *"To predict user engagement probability, frame this as a binary classification task. Use features like user history and content attributes to train a model (e.g., XGBoost or Logistic Regression) that outputs a predicted probability between 0 and 1 using `predict_proba()`. You can then sort items descendingly by this score."*

### Notes
* **What changed in prompt:** Added a specific, measurable engineering goal (continuous probability scoring model).
* **What actually improved in output:** Stopped giving generic CSV sorting tutorials (`df.sort_values()`) and immediately framed the solution around binary classification and probability scoring.
* **What still failed:** The output assumed generic labels and didn't account for how "meaningful engagement" is actually measured in logs.
* **What to try next:** Define real context regarding the data schema and logs.

---

## Version 2: Added Layer — Real Context

### Prompt
> *"Help me rank content in my dataset by building a continuous probability scoring model that predicts whether a user will meaningfully engage with a given piece of content. My dataset contains raw session logs with `clicked` (0/1), `dwell_time_seconds` (float), `saved` (0/1), `query_relevance_score` (0-1), and `content_age_hours` (float)."*

### Output Excerpt
> *"Since you have session logs, you need to construct a target variable first. You can train a model using `clicked` as your label y, with `query_relevance_score` and `content_age_hours` as features. Train a Gradient Boosted Tree, extract `predict_proba()[:, 1]`, and rank candidates by that score."*

### Notes
* **What changed in prompt:** Provided exact dataset column names and data types.
* **What actually improved in output:** The model recommendations used my actual column names instead of placeholders like `rating` or `feature_1`.
* **What still failed:** Using raw `clicked` as the label creates clickbait bias, which the model completely missed.
* **What to try next:** Add explicit constraints on target variable formulation.

---

## Version 3: Added Layer — Constraints

### Prompt
> *"Help me rank content in my dataset by building a continuous probability scoring model that predicts whether a user will meaningfully engage with a given piece of content. My dataset contains raw session logs with `clicked` (0/1), `dwell_time_seconds` (float), `saved` (0/1), `query_relevance_score` (0-1), and `content_age_hours` (float). Constraints: Do NOT use raw clicks as the target variable because clicks alone reward clickbait. Construct a proxy target label (`target_engaged`) that requires a click AND either `dwell_time_seconds >= 30` OR `saved == 1`."*

### Output Excerpt
> *"Here is how to create your target label and train the model:*
> ```python
> df['target_engaged'] = (
>     (df['clicked'] == 1) & 
>     ((df['dwell_time_seconds'] >= 30) | (df['saved'] == 1))
> ).astype(int)
> ```
> *Now train an XGBoost Classifier on `target_engaged`. Note that clickbait items with low dwell time will now be labeled as 0. Here is a 50-line Python script demonstrating data loading, feature engineering, model training, evaluation using accuracy, and feature importance plotting..."*

### Notes
* **What changed in prompt:** Added strict constraints preventing clickbait target bias.
* **What actually improved in output:** Successfully eliminated raw click target bias and provided exact Pandas logical mask code for `target_engaged`.
* **What still failed:** The added complexity caused the AI to generate a massive 50-line boilerplate script with inaccurate evaluation metrics (`accuracy_score` instead of ranking metrics like NDCG).
* **What to try next:** Specify output format and structural boundaries.

---

## Version 4: Added Layer — Specified Output Format

### Prompt
> *"Help me rank content in my dataset by building a continuous probability scoring model that predicts whether a user will meaningfully engage with a given piece of content. My dataset contains raw session logs with `clicked` (0/1), `dwell_time_seconds` (float), `saved` (0/1), `query_relevance_score` (0-1), and `content_age_hours` (float). Constraints: Do NOT use raw clicks as the target variable because clicks alone reward clickbait. Construct a proxy target label (`target_engaged`) that requires a click AND either `dwell_time_seconds >= 30` OR `saved == 1`. Format: Provide ONLY a clean Markdown code snippet containing two functions: `create_target_proxy(df)` and `train_pointwise_ranker(df)`. Do not write conversational intros or generic installation instructions."*

### Output Excerpt
> ```python
> import pandas as pd
> from xgboost import XGBClassifier

> def create_target_proxy(df: pd.DataFrame) -> pd.DataFrame:
>     df = df.copy()
>     df['target_engaged'] = (
>         (df['clicked'] == 1) & 
>         ((df['dwell_time_seconds'] >= 30) | (df['saved'] == 1))
>     ).astype(int)
>     return df

> def train_pointwise_ranker(df: pd.DataFrame) -> XGBClassifier:
>     X = df[['query_relevance_score', 'content_age_hours']]
>     y = df['target_engaged']
>     model = XGBClassifier(eval_metric='logloss')
>     model.fit(X, y)
>     return model
> ```

### Notes
* **What changed in prompt:** Restricted output to two specific Python functions and banned introductory text.
* **What actually improved in output:** Completely stripped away conversational fluff, markdown setup text, and random installation commands (`pip install...`), returning modular production code.
* **What still failed:** The feature set was hardcoded to only two features and omitted ranking quality evaluation.
* **What to try next:** Add quality criteria and validation requirements.

---

## Version 5: Added Layer — Quality Criteria & Verification

### Prompt
> *"Help me rank content in my dataset by building a continuous probability scoring model that predicts whether a user will meaningfully engage with a given piece of content. My dataset contains raw session logs with `clicked` (0/1), `dwell_time_seconds` (float), `saved` (0/1), `query_relevance_score` (0-1), and `content_age_hours` (float). Constraints: Do NOT use raw clicks as the target variable because clicks alone reward clickbait. Construct a proxy target label (`target_engaged`) that requires a click AND either `dwell_time_seconds >= 30` OR `saved == 1`. Format: Provide ONLY a clean Markdown code snippet containing two functions: `create_target_proxy(df)` and `train_pointwise_ranker(df)`. Do not write conversational intros or generic installation instructions. Quality Criteria: The training function must evaluate offline ranking quality using NDCG@10 via Scikit-Learn, ensure probability calibration via log-loss, and output a inference score column `pred_engagement_prob`."*

### Output Excerpt
> ```python
> import pandas as pd
> import numpy as np
> from sklearn.metrics import ndcg_score, log_loss
> from xgboost import XGBClassifier

> def create_target_proxy(df: pd.DataFrame) -> pd.DataFrame:
>     df = df.copy()
>     df['target_engaged'] = (
>         (df['clicked'] == 1) & 
>         ((df['dwell_time_seconds'] >= 30) | (df['saved'] == 1))
>     ).astype(int)
>     return df

> def train_pointwise_ranker(df: pd.DataFrame) -> tuple[XGBClassifier, pd.DataFrame, float]:
>     features = ['query_relevance_score', 'content_age_hours']
>     X = df[features]
>     y = df['target_engaged']
>     
>     model = XGBClassifier(n_estimators=100, max_depth=4, eval_metric='logloss', random_state=42)
>     model.fit(X, y)
>     
>     df_out = df.copy()
>     df_out['pred_engagement_prob'] = model.predict_proba(X)[:, 1]
>     
>     # Evaluate NDCG@10 on session group
>     ndcg = ndcg_score([y.values], [df_out['pred_engagement_prob'].values], k=10)
>     
>     return model, df_out, ndcg
> ```

### Notes
* **What changed in prompt:** Required explicit ranking metrics (NDCG@10), log-loss evaluation, and an inference score column.
* **What actually improved in output:** Replaced standard classification metrics (`accuracy`) with true ranking metrics (`ndcg_score`), and appended predicted probabilities directly back to the dataframe for downstream feed sorting.
* **What still failed:** None. The output strictly matches production requirements without fluff.
* **What to try next:** Package into a clean, reusable standalone prompt.

---

## Final Reusable Prompt (Production-Ready)

```text
TASK:
Build a pointwise content ranking pipeline using pandas and xgboost in Python.

INPUT SCHEMA:
Dataframe containing session log columns:
- `clicked` (int: 0 or 1)
- `dwell_time_seconds` (float)
- `saved` (int: 0 or 1)
- `query_relevance_score` (float: 0.0 to 1.0)
- `content_age_hours` (float)

CONSTRAINTS:
1. Do NOT use raw clicks as the binary target. Create a proxy target variable `target_engaged` where target = 1 IF (`clicked == 1` AND (`dwell_time_seconds >= 30` OR `saved == 1`)), ELSE 0.
2. Model type must be an XGBClassifier trained on `query_relevance_score` and `content_age_hours`.

QUALITY & EVALUATION REQUIREMENTS:
- Compute predicted probabilities (`pred_engagement_prob`) using predict_proba()[:, 1].
- Calculate and print offline NDCG@10 score evaluating ranking quality.
- Return the trained model and the transformed DataFrame with the score column attached.

OUTPUT FORMAT:
Return executable Python code containing two functions:
1. `create_target_proxy(df: pd.DataFrame) -> pd.DataFrame`
2. `train_pointwise_ranker(df: pd.DataFrame) -> tuple[XGBClassifier, pd.DataFrame, float]`
Do NOT include conversational introductory or concluding text.