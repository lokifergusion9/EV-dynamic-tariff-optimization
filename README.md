# ⚡ Agentic AI–Based Dynamic Tariff Optimization for EV Charging Networks

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![scikit-learn](https://img.shields.io/badge/scikit--learn-1.3%2B-orange)
![Status](https://img.shields.io/badge/pipeline-reproducible-brightgreen)

A self-improving pricing engine for EV charging stations. Three cooperating agents
**forecast demand**, **set a dynamic ₹/kWh tariff**, and **learn from simulated outcomes**.
The goal is to earn more revenue than a flat ₹15/kWh rate while easing peak-hour
congestion and moving demand into idle off-peak hours.

> Built on real charging sessions from the Caltech **ACN-Data** site and cross-checked
> against the Shenzhen **UrbanEV** network.

---

## 📊 Headline results

Learned dynamic policy compared with a fixed ₹15/kWh tariff:

| | Metric | Result |
|---|---|---|
| 💰 | Revenue gain | **+8.2%** |
| ⏱️ | Average peak wait-time reduction | **−22%** |
| 🌙 | Off-peak uplift | **+1,700 sessions** |
| 🏷️ | Effective price | **₹15.10 / kWh** (customers pay about the same on average) |

| Forecasting model | Score |
|---|---|
| Charging load (Gradient Boosting) | R² **0.90** · RMSE 0.95 · MAE 0.61 |
| Utilization (Random Forest) | R² **0.80** · RMSE 0.088 |
| Congestion classifier (Gradient Boosting) | Accuracy **0.98** · ROC-AUC 0.96 |

Over 30 learning episodes the policy score rises from **4.8 to 23.6**, which shows the
feedback loop improving its own decisions. The full numbers are in
[`outputs/RESULTS_SUMMARY.txt`](outputs/RESULTS_SUMMARY.txt).

<p align="center">
  <img src="outputs/figures/09_tariff_curve.png" width="48%" alt="Tariff curve">
  <img src="outputs/figures/08_learning_curve.png" width="48%" alt="Learning curve">
</p>

---

## 🧠 How it works

```
 ┌──────────────────┐    forecast     ┌──────────────────┐    tariffs     ┌──────────────────────┐
 │  Demand Agent    │ ──────────────▶ │  Tariff Agent    │ ─────────────▶ │  Monitoring Agent    │
 │  load, util,     │                 │  util → ₹/kWh    │                │  simulate response,  │
 │  P(congestion)   │                 │  surge / discount│ ◀───────────── │  hill-climb params   │
 └──────────────────┘                 └──────────────────┘  better params └──────────────────────┘
```

1. **Demand Prediction Agent** ([`demand_agent.py`](src/agents/demand_agent.py))
   predicts three things for each site-hour: expected sessions, charger utilization,
   and the probability of congestion. It uses calendar features plus lag and rolling
   features, and splits train/test **by time** so the model never sees the future.

2. **Tariff Pricing Agent** ([`tariff_agent.py`](src/agents/tariff_agent.py)) turns
   predicted utilization into a price multiplier on the ₹15/kWh baseline:

   | Predicted utilization | Action | Multiplier |
   |---|---|---|
   | > 80% | 🔺 **Surge** to spread out peak demand | up to 1.5× |
   | 30–80% | ➖ **Hold** | 1.0× |
   | < 30% | 🔻 **Discount** to attract off-peak sessions | down to 0.7× |

3. **Monitoring & Learning Agent** ([`monitoring_agent.py`](src/agents/monitoring_agent.py))
   replays history under each candidate tariff using a price-elasticity demand model.
   It scores every candidate on revenue, wait-time relief and off-peak uplift, and
   any policy that loses revenue is penalized. It then hill-climbs the surge cap,
   discount floor and thresholds, episode by episode.

---

## 📁 Datasets

| Dataset | Role | Coverage |
|---|---|---|
| **ACN-Data** ([`acndata_sessions.json.xlsx`](data/examples/acndata_sessions.json.xlsx)) | Primary | 14,999 clean sessions · Caltech · Apr–Dec 2018 |
| **UrbanEV / ST-EVCDP** ([source](https://github.com/IntelligentSystemsLab/ST-EVCDP)) | Cross-validation (optional) | 247 zones · 5-min resolution · Shenzhen · Jun 2022 |

**Data cleaning notes**
- 1,305 JSON pagination/meta rows in the raw ACN export were dropped because they have no `connectionTime`.
- `userInputs` is 100% null, so it was dropped.
- `kWhRequested` is about 78% missing, so `kWhDelivered` is used as the energy measure.

**Engineered features**

| Feature | Definition |
|---|---|
| Charger utilization rate | charging time ÷ connection time |
| Revenue per session | kWh delivered × tariff |
| Energy cost per kWh | time-of-use grid cost proxy (₹9 peak / ₹7 shoulder / ₹5 off-peak) |
| Queue length proxy | concurrent active sessions per site-hour |
| Occupancy density | active sessions ÷ stations (54 EVSEs) |
| Lags and rolling windows | `lag1`, `lag24`, `roll3` sessions, `lag1` utilization |

---

## 🚀 Quick start

```bash
git clone https://github.com/lokifergusion9/EV-Dynamic-Tariff-Optimization.git
cd EV-Dynamic-Tariff-Optimization
pip install -r requirements.txt

# Optional: UrbanEV cross-dataset comparison (enables figure 07, ~62 MB)
git clone --depth 1 https://github.com/IntelligentSystemsLab/ST-EVCDP.git data/raw/ST-EVCDP

cd src
python run_pipeline.py      # preprocess → EDA → 3 agents → evaluation (~1–2 min)
```

The ACN dataset ships with the repo in `data/examples/`, so no extra download is needed.
Results are deterministic (`RANDOM_STATE = 42`).

To read the narrative walkthrough, open
[`notebooks/OP26_Analytics.ipynb`](notebooks/OP26_Analytics.ipynb) (run it from `notebooks/`).

**Tuning:** every business constant (baseline tariff, surge/discount thresholds,
multiplier caps, elasticities, time-of-use costs) lives in [`src/config.py`](src/config.py).

---

## 🗂️ Project structure

```
├── src/
│   ├── config.py              # paths + all business/model constants
│   ├── preprocessing.py       # cleaning, feature engineering, hourly panel
│   ├── eda.py                 # 7 exploratory figures
│   ├── evaluation.py          # metrics, CSVs, result figures, summary
│   ├── run_pipeline.py        # end-to-end orchestrator
│   └── agents/
│       ├── demand_agent.py
│       ├── tariff_agent.py
│       └── monitoring_agent.py
├── notebooks/OP26_Analytics.ipynb
├── data/
│   ├── examples/              # ACN dataset + small UrbanEV samples
│   ├── processed/             # cleaned sessions, hourly panel, UrbanEV profile
│   └── raw/                   # (optional) UrbanEV clone, git-ignored
├── outputs/
│   ├── figures/               # 01–09 PNG charts
│   ├── csv/                   # metrics, tariff schedule, recommendations, episodes
│   ├── RESULTS_SUMMARY.txt
│   └── OP26_Project_Report.pdf
├── build_notebook.py          # regenerates the notebook
└── make_submission_zip.py     # packages the submission zip
```

### Outputs

| File | Contents |
|---|---|
| `outputs/csv/demand_metrics.csv` | RMSE / MAE / R² / accuracy / ROC-AUC for all three models |
| `outputs/csv/demand_feature_importance.csv` | What drives the load forecast (`roll3_sessions`, `hour` dominate) |
| `outputs/csv/tariff_schedule.csv` | Utilization → multiplier → ₹/kWh lookup table |
| `outputs/csv/tariff_recommendations.csv` | Hour-by-hour recommended tariff and action |
| `outputs/csv/monitoring_episodes.csv` | Parameters and KPIs for every learning episode |
| `outputs/csv/pricing_outcomes.csv` | Default vs learned policy KPIs |

---

## 🔍 Exploratory insights

<p align="center">
  <img src="outputs/figures/01_intraday_demand.png" width="48%" alt="Intraday demand">
  <img src="outputs/figures/03_utilization_heatmap.png" width="48%" alt="Utilization heatmap">
</p>

- Demand is **workplace-shaped**: arrivals spike at 07:00–10:00 (the 08:00 hour alone has about 19% of sessions), and a weekend day sees roughly half the sessions of a weekday.
- Station utilization is **genuinely low** (median 6% per site-hour, mean 15%). The biggest opportunity is filling idle hours, not handling over-capacity.
- Recent activity (`roll3_sessions`) and hour of day explain about 94% of the load model's feature importance.

---

## ⚠️ Assumptions & limitations

- **Currency:** ACN is US workplace charging. Prices use the brief's ₹15/kWh reference and are *not* a USD→INR conversion.
- **Demand response is assumed, not estimated.** Off-peak users are modeled as price-elastic (ε ≈ −1.45) and peak users as inelastic (ε ≈ −0.20). Revenue, wait-time and uplift figures are therefore **simulation-based counterfactuals**, not observed outcomes.
- **"Agentic" here means** an autonomous loop of ML forecast → rule-based pricing → feedback learning. It is deterministic and uses no external LLM or API.
- **Scope:** a single site over about 8 months. Validating on more sites, and on real price experiments, would be the next step.

---

## 🛠️ Tech stack

Python · pandas · NumPy · scikit-learn · matplotlib · seaborn · Jupyter
