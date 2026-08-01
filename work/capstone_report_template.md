📝 CAPSTONE REPORT — Fiza Aslam 🚀
1. PROBLEM FRAMING
What decision does this support?

Content teams at FlyRank need to prioritize which pages to refresh first. Manual review of thousands of pages is impossible. This model ranks pages by predicted decline risk, enabling editors to focus on high-priority pages.

Detail	Value
Unit of Analysis	One content page (content_hash_id)
Output	Ranked queue with decline scores (0-1) + action labels
Human Action	Review flagged pages → decide to refresh or keep
Cost of Wrong Call	Missing a declining page = traffic loss. Flagging a good page = wasted writer time.
Why ML helps: Hand-written rules achieved only 24% Precision@50. ML learns complex patterns from multiple signals (impressions, clicks, position) and achieves 90% Precision@50 — a 66% improvement.

2. DATA SAFETY
Data Used:

fact_content_daily_performance from FlyRank warehouse

Months: January-February 2026 (train), March 2026 (test)

Columns: content_hash_id, gsc_impressions, gsc_clicks, gsc_sum_position

Columns Deliberately Excluded:

ctr — Derived from clicks/impressions, used to create target (leakage risk)

trend_direction, trend_pct — Label-derived fields (direct leakage)

client_hash_id — Pseudonymous ID, used only for analysis, never as feature

All GA4 columns — Sparse data, not all clients have GA4

All AI traffic columns — Separate signal, not for refresh decisions

Leakage Risks Considered:

✅ CTR removed from features (target proxy leakage)

✅ Time-aware split (past → future, no temporal leakage)

✅ No client-identifying details anywhere in work/

✅ All claims use safe language (observed, measured, directional)

3. BASELINE
Baseline Rule:

text
Score = impressions × (1 - CTR)
Pages with high impressions but low CTR score higher — they're visible but underperforming.

Why It's Fair:

Uses same data, same split, same metric (Precision@50)

Simple, transparent, reproducible

Represents what a human would write without ML

Baseline Performance:

Metric	Value
Precision@50	0.24
Interpretation	Only 12 of top 50 flagged pages actually declining
4. MODEL / ANALYSIS
Method: Gradient Boosting Classifier

Why It Fits:

Handles non-linear relationships between features

Robust to class imbalance

Sequential learning captures complex patterns

Best performer among 4 models tested

Feature List (3 features):

impressions — Total GSC impressions (visibility signal)

clicks — Total GSC clicks (engagement signal)

avg_position — Average search position (ranking quality)

Intentionally Left Out:

ctr — Target proxy (leakage)

Demographics, client data — Not relevant for content refresh

Target Definition:

text
is_declining = (CTR percentile < 50%) AND (impressions > 5)
A page is "declining" if its CTR is below median relative to peers, but still has some visibility. 15% random noise added to simulate real-world uncertainty.

Models Tested (4):

Model	Precision@50	Accuracy
Baseline (Hand Rule)	0.24	—
Logistic Regression	0.48	71.4%
Decision Tree	0.84	84.7%
Random Forest	0.80	84.7%
Gradient Boosting	0.90	84.7%
5. EVALUATION
Split Design:

Type: Time-aware (Jan-Feb 2026 train, March 2026 test)

Why: Past → future reflects real-world use. Random split would leak future data.

No client grouping: Content pages are independent units.

Metrics (Gradient Boosting, threshold=0.35):

Metric	Not Declining	Declining	Overall
Precision	0.85	0.84	0.85
Recall	0.95	0.63	—
F1-Score	0.89	0.72	—
Accuracy	—	—	0.847
Error Analysis:

False Negatives (5,488): Declining pages missed — 37% of declining class

False Positives (1,636): Good pages flagged — 5% of non-declining class

Trade-off: Better to flag more (false positives) than miss declining pages (false negatives). Content teams can quickly dismiss false flags.

Feature Importance:

Feature	Importance
impressions	79.7%
clicks	20.2%
avg_position	0.2%
6. INTERPRETATION
What the Model Found:

Impressions dominate (80% importance): Whether a page gets seen at all is the strongest signal of its health. Low impressions + low CTR = likely declining.

Position alone is nearly useless (0.2%): Ranking position doesn't predict decline — surprising but valuable finding. Teams can deprioritize position-based alerts.

Clicks add context (20%): Engagement matters, but visibility matters more. A page with low impressions will decline regardless of click quality.

Surprises:

Position was expected to be more predictive — it wasn't. This saves teams from chasing ranking fluctuations.

63% recall means we catch ~2/3 of declining pages with 84% precision — practically useful for weekly review.

Negative Result (Valid):
Position's near-zero importance is a valid finding — it means ranking alone shouldn't trigger refresh decisions. This contradicts common intuition but is backed by data.

7. RECOMMENDATION
Ranked Actions:

Priority	Action	Threshold	Weekly Volume
🚨 URGENT_REFRESH	Immediate review	Score ≥ 0.70	~20 pages
🟡 REVIEW_NEEDED	Review this week	Score ≥ 0.35	~50 pages
🟢 MONITOR	Check next month	Score ≥ 0.20	~100 pages
✅ KEEP	No action	Score < 0.20	Remaining
How a FlyRank Editor Would Use This:

Monday morning: Open ranked queue

Start with URGENT_REFRESH (top 20)

Verify each manually — check content age, quality, business priority

Assign writers to refresh confirmed pages

Move to REVIEW_NEEDED if time permits

Sample 5-10 MONITOR pages for quality check

Confidence & Limits:

Confidence: Directional — 90% Precision@50, 63% recall on test month

Not production: Requires human review before any action

Time-bound: Trained on Jan-Feb 2026, tested on March 2026

Not causal: Model identifies correlation, not causation

Threshold adjustable: Teams can tune based on capacity

What Should NEVER Be Automated:

❌ Auto-refreshing content

❌ Auto-deleting pages

❌ Sending alerts to clients

❌ Using as the only input for content strategy

8. REPRODUCIBILITY
Commands to Re-run from Fresh Clone:

bash
git clone https://github.com/FizaAslam1/flyrank-ml-internship.git
cd flyrank-ml-internship
pip install -r requirements.txt
Notebook Sequence:

text
work/notebooks/w05_model.ipynb          → Model training + evaluation
work/notebooks/w06_validation_audit.ipynb → Validation + leakage audit
work/notebooks/w07_action_playbook.ipynb  → Action playbook + exports
Random Seeds:

python
random_state=42  # All sklearn models
np.random.seed(42)  # NumPy operations
Environment:

text
Python 3.12
Key packages: scikit-learn, pandas, numpy, duckdb, huggingface-hub, matplotlib
Full environment: pip freeze > requirements.txt (committed to repo)
Data Access:

Hugging Face token required (read access to FlyRank/internship-warehouse)

Token stored as Colab secret (HF_TOKEN) — never in code

Sample size: 50,000 rows/month (adjustable in USING SAMPLE clause)

CLAIMS CHECKLIST
Check	Status
Safe language used (observed, measured, directional, decision-support)	✅
No causal claims without experiment	✅
No "predicted Google's algorithm"	✅
No client-identifying details	✅
Numbers match fresh re-run	✅
Base rate reported (declining: 31.2% test)	✅
AUC/lift considered (Precision@50 vs baseline = 3.75x lift)	✅
