<div align="center">

<h1>ExperimentOS</h1>

<h3>Bayesian A/B testing, CUPED variance reduction, and metric governance in one product analytics dashboard.</h3>

<p>
ExperimentOS turns noisy product experiments into clear decisions. It tells a team
<b>"Variant B has a 100% probability of beating Variant A"</b>
instead of handing them a p-value and leaving the interpretation to a meeting.
</p>

<br>

<a href="https://github.com/yaswanthakkireddy/ExperimentOS/actions">
  <img alt="CI" src="https://img.shields.io/github/actions/workflow/status/yaswanthakkireddy/ExperimentOS/ci.yml?style=for-the-badge&label=Tests&logo=github&logoColor=white">
</a>
<a href="https://python.org">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white">
</a>

<br><br>

<table>
  <tr>
    <td align="center"><b>Bayesian A/B</b><br>Posterior probability</td>
    <td align="center"><b>CUPED</b><br>25% sample reduction</td>
    <td align="center"><b>Funnel + Retention</b><br>Realistic drop-off</td>
    <td align="center"><b>Metric Governance</b><br>dbt-style YAML registry</td>
    <td align="center"><b>AI Memo</b><br>Stakeholder-ready decision</td>
  </tr>
</table>

<br>

<img src="visuals/01-ab-test-engine.png" alt="ExperimentOS dashboard preview" width="95%">

</div>

ExperimentOS is built on the UCI Online Retail II dataset: real UK e-commerce transaction data with real users, countries, invoices, products, revenue, and uneven behavior. The app loads the raw Excel file into SQLite, simulates realistic browsing and cart events, runs statistical analysis, and presents the result as a working Streamlit dashboard.

The app can be run locally from this repository; hosted-demo references are omitted while the public Space is unavailable.

---

## Why This Exists

Product teams run A/B tests all the time, but the decision process is often messy. One person reads a p-value as "probability the feature works." Another person changes the revenue definition halfway through the analysis. The experiment runs too long because the metric is noisy. The decision memo is written manually, inconsistently, or not at all.

ExperimentOS fixes that workflow with a small, end-to-end experimentation system:

- Bayesian A/B testing for conversion rates
- CUPED variance reduction to reduce required sample size
- Cohort retention analysis
- View to cart to purchase funnel analysis
- dbt-style metric registry
- GitHub Actions gate for metric definition changes
- Gemini-powered decision memo generation
- Streamlit dashboard for local or hosted deployment

The project uses the UCI Online Retail II dataset: real UK e-commerce transactions from 2009 to 2011. The data has real users, real countries, real revenue, seasonal patterns, outliers, and uneven behavior. That makes the analysis more useful than a synthetic demo.

---

## Dashboard Preview

### Bayesian A/B Test Engine

The A/B engine compares Control A and Treatment B using a Beta-Binomial Bayesian model. In the current run, Variant B has higher simulated purchase conversion, a clear posterior separation, and a shipping recommendation.

![A/B Test Engine](visuals/01-ab-test-engine.png)

### CUPED Variance Reduction

CUPED adjusts the post-experiment metric using pre-experiment behavior. That reduces variance and lowers the sample size needed for the same decision confidence.

![CUPED Variance Reduction](visuals/02-cuped-variance-reduction.png)

### Cohort Retention

Retention is shown by weekly acquisition cohort. The chart is limited to the latest cohorts and the first eight weeks so the pattern is readable.

![Cohort Retention](visuals/03-cohort-retention.png)

### Conversion Funnel

The funnel shows realistic drop-off from view to add-to-cart to purchase, with a separate A/B comparison.

![Conversion Funnel](visuals/04-conversion-funnel-overall.png)

![A/B Funnel Comparison](visuals/05-conversion-funnel-ab-comparison.png)

### Metric Registry

Metric definitions live in YAML and are version-controlled. A GitHub Actions workflow blocks pull requests that change canonical metrics without approval.

![Metric Registry](visuals/06-metric-registry.png)

![Metric Registry YAML](visuals/07-metric-registry-yaml.png)

### Decision Memo

The dashboard can generate a product-facing decision memo from the experiment results. A generated example is included in [`decision_memo.txt`](decision_memo.txt).

![Decision Memo](visuals/08-decision-memo-overview.png)

---

## Current Result Snapshot

| Metric | Control A | Treatment B |
|---|---:|---:|
| Users | 9,166 | 7,750 |
| Purchase conversion | 32.0% | 38.0% |
| Revenue | GBP 8.63M | GBP 9.11M |
| Observed lift | - | +6.0 percentage points |
| P(B > A) | - | 100.0% |
| Recommendation | - | Ship B - Strong evidence |

Funnel snapshot:

| Stage | Rate |
|---|---:|
| View to cart | 51.9% |
| Cart to purchase | 67.0% |
| Overall conversion | 34.7% |

CUPED snapshot:

| Metric | Value |
|---|---:|
| Variance reduction | 25.0% |
| Original sample size | 4,718 |
| CUPED sample size | 3,536 |
| Sample saved | 25.1% |

---

## Sample Decision Memo

The memo generator turns experiment statistics into language a product team can use.

From [`decision_memo.txt`](decision_memo.txt):

> We ran a controlled experiment across 16,916 users split between Control A (9,166 users) and Treatment B (7,750 users). The primary success metric was purchase conversion rate - the percentage of users who completed at least one purchase during the experiment window.

> Control A achieved a conversion rate of 32.00%. Treatment B achieved 38.00%. The observed difference is +6.00 percentage points in favour of Treatment B. Our Bayesian model puts the probability that B is genuinely better at 100.0%.

> Based on 100.0% probability that Treatment B outperforms Control A, we have sufficient evidence to ship. The observed +6.00pp improvement in conversion rate represents a meaningful business impact that justifies full rollout.

The full memo is tracked as a project artifact so reviewers can see what the AI output looks like without running the app.

---

## How It Works

### 1. Data Loader

`data/loader.py` reads `data/online_retail_II.xlsx`, combines both Excel sheets, cleans invalid rows, engineers user and revenue columns, assigns users deterministically to A or B, simulates funnel events, and writes three SQLite tables:

- `events`
- `users`
- `invoices`

The generated database is `data/events.db`. It is ignored by Git because it can be regenerated.

### 2. Bayesian A/B Test

`experiments/bayesian_ab.py` uses a Beta(1,1) prior and updates it with observed conversions:

```python
Posterior A = Beta(1 + conversions_A, 1 + users_A - conversions_A)
Posterior B = Beta(1 + conversions_B, 1 + users_B - conversions_B)
P(B > A) = mean(samples_B > samples_A)
```

The output is directly interpretable: probability B is better, expected lift, credible interval, expected loss, and recommendation.

### 3. CUPED

`experiments/cuped.py` uses pre-experiment behavior as a covariate:

```python
theta = Cov(post_metric, pre_metric) / Var(pre_metric)
Y_cuped = post_metric - theta * (pre_metric - mean(pre_metric))
```

This reduces variance and lowers the sample size needed for the same statistical power.

### 4. Analytics

`analytics/funnel_analysis.py` computes unique users at each stage of the purchase journey.  
`analytics/cohort_retention.py` computes weekly retention by acquisition cohort.

### 5. Governance

`dbt_project/models/metrics/metric_definitions.yml` stores canonical metric definitions.  
`.github/workflows/metric_gate.yml` blocks pull requests that modify those definitions without review.

---

## Run Locally

```bash
git clone https://github.com/yaswanthakkireddy/ExperimentOS.git
cd ExperimentOS

pip install -r requirements.txt
python data/loader.py
streamlit run dashboard/app.py
```

The app opens at:

```text
http://localhost:8501
```

---

## Run Tests

```bash
pytest tests/ -v --tb=short
```

Current local suite:

```text
30 passed
```

---

## Project Structure

```text
ExperimentOS/
├── app.py
├── README.md
├── requirements.txt
├── verify.py
├── decision_memo.txt
├── visuals/
│   ├── 01-ab-test-engine.png
│   ├── 02-cuped-variance-reduction.png
│   ├── 03-cohort-retention.png
│   ├── 04-conversion-funnel-overall.png
│   ├── 05-conversion-funnel-ab-comparison.png
│   ├── 06-metric-registry.png
│   ├── 07-metric-registry-yaml.png
│   ├── 08-decision-memo-overview.png
│   └── 09-decision-memo-detail.png
├── data/
│   ├── online_retail_II.xlsx
│   └── loader.py
├── experiments/
│   ├── bayesian_ab.py
│   ├── cuped.py
│   └── frequentist_ab.py
├── analytics/
│   ├── cohort_retention.py
│   └── funnel_analysis.py
├── memo/
│   └── decision_memo.py
├── dashboard/
│   └── app.py
├── dbt_project/
│   └── models/
├── tests/
├── docs/
└── .github/workflows/
```

---

## Stack

| Layer | Tools |
|---|---|
| App | Streamlit, Plotly |
| Data | pandas, SQLite, openpyxl |
| Statistics | numpy, scipy, PyMC, statsmodels |
| Experiment methods | Bayesian Beta-Binomial, CUPED, chi-square baseline |
| Metric governance | dbt project structure, GitHub Actions |
| Memo generation | Google Gemini |
| Hosting | Hugging Face Spaces |

---

## Documentation

The technical notes are in [`docs/`](docs/):

- [`docs/problem_statement.md`](docs/problem_statement.md)
- [`docs/scope.md`](docs/scope.md)
- [`docs/algorithms.md`](docs/algorithms.md)
- [`docs/results.md`](docs/results.md)
- [`docs/future_work.md`](docs/future_work.md)

---

## What This Shows

ExperimentOS is not just a dashboard. It is a compact version of a real experimentation workflow:

1. Load messy transaction data.
2. Convert it into event, user, and invoice tables.
3. Run an interpretable Bayesian A/B test.
4. Reduce experiment duration with CUPED.
5. Analyze retention and funnel drop-off.
6. Govern metric definitions in version control.
7. Generate a decision memo stakeholders can read.
8. Deploy the whole workflow as a public app.

That is the core story: better experiment decisions, faster reads, fewer metric arguments, and a clear record of why the decision was made.

