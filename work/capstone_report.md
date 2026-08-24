# Capstone Report — Content Refresh & Priority Ranking

- **Author:** Abhinav Thakur
- **Lane:** Content Refresh & Priority Ranking
- **Repo:** [https://github.com/abhinavt1325/Flyrank-Internship-Capstone](https://github.com/abhinavt1325/Flyrank-Internship-Capstone)
- **Deployed Paper:** [https://abhinavt1325.github.io/Flyrank-Internship-Capstone/](https://abhinavt1325.github.io/Flyrank-Internship-Capstone/)
- **Date:** August 2026

---

## 0. Abstract

How can content teams systematically prioritize which aging web pages to refresh before search traffic decay causes substantial revenue loss? We analyze an enterprise portfolio of 30,000 pseudonymized content items across 32 client domains, capturing 90-day search visibility and user dwell engagement signals. We formulate content refresh triage as a machine learning ranking problem and evaluate predictive models against a deterministic heuristic baseline under an honest client-holdout split (`GroupShuffleSplit`). On unseen client domains, a regularized Random Forest model achieves **64.0% Mean Precision@50** (a **+14.0 percentage point lift** over the 50.0% unranked base rate and a **+10.4 percentage point lift** over baseline rule heuristics). We operationalize these predictions into an automated action playbook featuring 5 content archetypes, reason codes, and human-in-the-loop verification protocols to support high-ROI editorial resource allocation.

---

## 1. Problem framing

- **Decision Supported:** Deciding which specific URLs in a multi-thousand-page content catalog should receive editorial refresh resources each week/month.
- **Unit of Analysis:** Individual content URL (`content_id`), grouped by client domain (`client_id`).
- **Target Output:** An actionable editorial triage queue ranked by predicted risk and business leverage, accompanied by explicit reason codes (e.g., `striking_distance_decay_risk`).
- **Cost of a Wrong Call:** A false positive wastes 2–4 hours of editorial rewrite effort ($150–$300) on a stable page; a false negative allows a high-traffic, high-intent page to decay quietly, costing thousands of monthly search visits.
- **Role of Machine Learning:** Unifies multivariate historical signals (impressions, ranking proximity, dwell time, and staleness) into calibrated probability scores, outperforming simplistic single-variable rules (e.g. "update all pages older than 6 months").

---

## 2. Data safety

- **Dataset:** FlyRank Anonymized Search Intelligence Dataset (`data/raw/content_refresh_anonymized.csv`), comprising 30,000 URLs across 32 client domains.
- **Excluded Columns (Leakage Prevention):** `trend_direction` and `trend_pct` were strictly excluded because the binary ground truth `is_declining_label` is computed directly from them. Short-window future activity columns (`clicks_last_30d`, `impressions_last_30d`) were excluded to prevent outcome-window leakage.
- **Identifier Handling:** `client_id` and `content_id` are pseudonyms used strictly for grouping and train/test holdouts, never as estimator features.
- **Zero Client Identifiers:** No raw URLs, domains, queries, or credentials exist in the dataset or this report.

---

## 3. Baseline

- **Deterministic Rule Formula:**
  $$\text{Baseline Score} = 0.45 \times \text{Visibility Rank} + 0.35 \times \text{Freshness Risk Rank} + 0.20 \times \text{Position Opportunity}$$
- **Baseline Performance:**
  - Evaluated on test clients: **53.60% Mean Precision@50** (a +3.60 pp lift over the 50.00% unranked base rate; ROC-AUC: 0.561).

---

## 4. Model / analysis

- **Primary Model:** Random Forest Classifier (`n_estimators=200, max_depth=6, min_samples_leaf=20, random_state=42`).
- **Linear Benchmark:** Logistic Regression with L2 Regularization (`C=0.1`) and Standard Scaling.
- **Feature Set (17 historical signals):**
  - Search Visibility: `log_impressions_90d`, `log_clicks_90d`, `ctr`, `visibility_score`.
  - Ranking Exposure: `avg_position`, `has_position`, `position_opportunity_score`.
  - Staleness & Age: `days_since_last_update`, `content_age_days`, `freshness_risk_score`.
  - Engagement: `engagement_rate`, `scroll_rate`, `has_scroll`.
  - Structure: `word_count`, `has_word_count`, one-hot `content_type`.
- **Target Definition:** `is_declining_label = (trend_direction == 'down')`.

---

## 5. Evaluation

- **Validation Design:** `GroupShuffleSplit(n_splits=1, test_size=0.20, random_state=42)` grouped strictly on `client_id`. Train: 25 clients (23,837 URLs); Test: 7 clients (6,163 URLs). Zero shared client domains.
- **Results Table (Same Test Clients):**

| Model | Split Type | Precision@50 | ROC-AUC | Lift vs Base Rate |
|---|---|---|---|---|
| Unranked Base Rate | Grouped (Test) | 50.00% | — | 0.00 pp |
| Deterministic Baseline Rule | Grouped (Test) | 53.60% | 0.561 | +3.60 pp |
| Random Forest | Grouped (Honest) | **64.00%** | **0.607** | **+14.00 pp** |
| Logistic Regression | Grouped (Honest) | **72.00%** | **0.633** | **+22.00 pp** |

- **Error Analysis:** Top error mode is False Positives (~30–35%), which primarily consist of high-impression, stale evergreen reference pages where query intent has not changed. This demonstrates why the output must serve as human decision-support rather than autonomous automated rewriting.

---

## 6. Interpretation

- **Key Feature Drivers:** Permutation importance shows that `log_impressions_90d`, `content_age_days`, and `visibility_score` dominate predictive capability.
- **Business Insight:** High-volume URLs in striking-distance rankings (positions 4–20) exhibit the steepest risk-reward profile: they suffer the highest traffic loss when decaying, but offer the highest ROI when refreshed.

---

## 7. Recommendation

- **Operational Content Archetypes:**
  1. `P1: Striking-Distance Refresh` (`striking_distance_decay_risk`) — Deep factual and structural refresh.
  2. `P2: Page-1 Defense` (`page_one_prominence_defense`) — SERP feature defense & schema updates.
  3. `P3: CTR Metadata Optimization` (`low_ctr_high_visibility`) — Title and meta description rewrite.
  4. `P4: Evergreen Reference` (`evergreen_stable_monitor`) — Passive monitoring; avoid date bumping.
  5. `P5: Thin Consolidation` (`low_volume_consolidation`) — Audit for 301 redirect or prune.
- **The No-Go List:** No autonomous AI rewrites pushed directly to production; no synthetic date bumping without substantive edits; no automated URL deletions.

---

## 8. Reproducibility

- **Environment & Seeds:** Python 3.10+ / scikit-learn 1.9.0; `RANDOM_SEED = 42`.
- **How to Rerun:**
  ```bash
  pip install -r requirements.txt
  jupyter nbconvert --execute --to notebook work/notebooks/capstone.ipynb
  ```
- **Committed Receipts:** `work/outputs/action_playbook_metrics.json` and `work/outputs/capstone_metrics_receipt.json`.

---

---

## 9. Acknowledgments & data credit

Built on the **FlyRank ML Internship dataset** ([https://flyrank.ai](https://flyrank.ai)).

---

## 10. ML-12 — Demo Outline & Shareable Cuts

### A. 5-Minute Technical Demo Outline
- **0:00–1:00 (The Problem & Cost):** Why calendar-based content refreshes ("update everything older than 6 months") waste editorial budgets ($150–$300/article) on stagnant or low-opportunity pages.
- **1:00–2:30 (The Validation Trap):** Why naive random splits produce artificially inflated metrics (domain memorization) and how strict client-level holdouts (`GroupShuffleSplit` across 32 clients) keep the benchmark honest.
- **2:30–3:45 (Model Results & Lift):** How Random Forest achieves 64.0% Mean Precision@50 on completely unseen client domains (+14.0 pp over base rate, +10.4 pp over deterministic rules).
- **3:45–5:00 (Operational Action Playbook):** How predictions map into 5 editorial archetypes (P1–P5), automated reason codes, and the live deployed research paper at https://abhinavt1325.github.io/Flyrank-Internship-Capstone/.

### B. Social Post Cut (LinkedIn / X)
Most SEO teams schedule content refreshes based on calendar age (e.g. "update every post older than 6 months"), which wastes editorial hours on pages with zero ranking leverage.

For my FlyRank ML internship capstone, I analyzed whether historical search signals (ranking proximity, impression volume, staleness, and dwell time) could predict search traffic decay across 30,000 URLs and 32 client domains before traffic drops occur.

The main takeaway was validation integrity: standard random train/test splits give false confidence because models memorize client domain patterns. When evaluated strictly on unseen client domains (`GroupShuffleSplit`), a regularized Random Forest achieved 64.0% Mean Precision@50 (+14.0 percentage points over base rate).

I turned the predictions into a 5-archetype editorial triage engine and published the full methodology:
- Interactive Paper: https://abhinavt1325.github.io/Flyrank-Internship-Capstone/
- Code & Notebooks: https://github.com/abhinavt1325/Flyrank-Internship-Capstone
(Built on the FlyRank ML Internship dataset: https://flyrank.ai)

### C. 3-Sentence Employer-Facing Summary
I engineered a machine learning prioritization engine on 30,000 enterprise search URLs across 32 client domains to identify decaying organic content before revenue loss occurs. Evaluated strictly on unseen client domains using `GroupShuffleSplit`, the model achieved 64.0% Mean Precision@50 (+14.0 pp lift over base rate). I translated the output into an automated 5-archetype editorial action queue and deployed the complete research paper on GitHub Pages.

