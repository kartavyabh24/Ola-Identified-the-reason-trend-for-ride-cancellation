# Ola — Identifying Reasons & Trends for Ride Cancellations

> Analyze multi-source trip data, uncover cancellation drivers, and ship a Power BI dashboard + SQL transformations.
> **Outcome:** Implemented strategies that **reduced cancellations by ~10%**.

![status-badge](https://img.shields.io/badge/status-active-brightgreen) ![powerbi](https://img.shields.io/badge/Power%20BI-Model%20%26%20Dashboard-blue) ![sql](https://img.shields.io/badge/SQL-Transformations-purple)

---

## 🔗 Repository

**Repo:** `Ola-Identified-the-reason-trend-for-ride-cancellation`
**Author:** Kartavya Bhardwaj | [LinkedIn](https://www.linkedin.com/in/kartavya24) • [GitHub](https://github.com/kartavyabh24)

---

## 🎯 Problem Statement

Ride cancellations hurt customer trust, driver productivity, and platform economics. This project pinpoints **who cancels**, **when**, and **why**, and designs **actions** to reduce the rate.

---

## 📦 Data Sources (illustrative)

* `rides.csv` — request, accept, pickup, cancel timestamps, city, surge, ETA
* `drivers.csv` — driver_id, experience, rating, cancellation streaks
* `customers.csv` — user_id, tenure, prior cancels, payment method
* `support_tickets.csv` — ticket tags (ETA wrong, driver no-show, app issues)
* `geo_zones.csv` — zone_id, locality, traffic tier

> Feel free to rename to your actual files; column names below assume typical ride-hailing schemas.

---

## 🧰 Tech Stack

* **SQL** (ANSI; runnable on SQLite/Postgres/MySQL with minor tweaks)
* **Power BI** (data model, DAX, visuals)
* **Python (optional)** for quick data checks / preprocessing (Pandas)

---

## 📁 Repository Structure (suggested)

```
.
├─ /data/                 # raw & cleaned (git-ignored if needed)
├─ /sql/
│   ├─ 01_cleaning.sql
│   ├─ 02_model_build.sql
│   └─ 03_kpi_views.sql
├─ /powerbi/
│   └─ ola_cancellations.pbix
├─ /img/
│   ├─ dashboard-overview.png
│   └─ cancellation-by-hour.png
├─ README.md
└─ requirements.txt       # if you use Python
```

---

## 🔑 KPI Definitions

* **Cancellation Rate (CR)**
  [
  CR = \frac{\text{# Cancelled Rides}}{\text{# Requested Rides}} \times 100
  ]
* **Driver-Initiated Cancellation Rate (DCR)**
  **Customer-Initiated Cancellation Rate (CCR)**
* **On-Time Arrival % (OTA)** = arrivals within X minutes of promised ETA
* **Peak Window** = locale-specific hours (e.g., 8–10 AM, 6–9 PM)

---

## 🧹 Data Cleaning & Processing (Process Summary)

1. **Standardize timestamps** to UTC; derive `request_hr`, `dow`, `is_peak`.
2. **Normalize enums** (status, cancel_reason).
3. **De-duplicate** multiple events per `ride_id` (keep latest terminal state).
4. **Outlier handling** (extreme ETAs, impossible distances).
5. **Join** drivers/customers/geo to rides; **impute** small gaps where safe.
6. **Create Model Tables**:

   * `f_rides` (facts)
   * `d_time`, `d_city_zone`, `d_driver`, `d_customer` (dimensions)
7. **Expose KPI Views** for BI.

---

## 🧾 Sample SQL (portable)

### A) Canonical Cancellation Rate View

```sql
-- sql/03_kpi_views.sql
CREATE VIEW v_cancellation_rate AS
SELECT
  r.city,
  DATE(r.request_time) AS ride_date,
  COUNT(*) AS total_requests,
  SUM(CASE WHEN r.status = 'cancelled' THEN 1 ELSE 0 END) AS total_cancellations,
  ROUND(100.0 * SUM(CASE WHEN r.status = 'cancelled' THEN 1 ELSE 0 END) / COUNT(*), 2) AS cancellation_rate_pct
FROM f_rides r
GROUP BY r.city, DATE(r.request_time);
```

### B) Who Cancels & Why

```sql
CREATE VIEW v_cancel_reason_breakdown AS
SELECT
  COALESCE(r.cancel_initiator, 'unknown') AS initiator,
  COALESCE(LOWER(r.cancel_reason), 'unspecified') AS reason,
  COUNT(*) AS cancels,
  ROUND(100.0 * COUNT(*) / NULLIF(SUM(COUNT(*)) OVER (), 0), 2) AS share_pct
FROM f_rides r
WHERE r.status = 'cancelled'
GROUP BY COALESCE(r.cancel_initiator, 'unknown'), COALESCE(LOWER(r.cancel_reason), 'unspecified')
ORDER BY cancels DESC;
```

### C) Time-of-Day Trend

```sql
CREATE VIEW v_cancel_by_hour AS
SELECT
  EXTRACT(HOUR FROM r.request_time) AS request_hr,
  COUNT(*) AS requests,
  SUM(CASE WHEN r.status='cancelled' THEN 1 ELSE 0 END) AS cancels,
  ROUND(100.0 * SUM(CASE WHEN r.status='cancelled')::int / NULLIF(COUNT(*),0), 2) AS cr_pct
FROM f_rides r
GROUP BY EXTRACT(HOUR FROM r.request_time)
ORDER BY request_hr;
```

> For MySQL: replace `EXTRACT(HOUR FROM ...)` with `HOUR(...)` and cast syntax accordingly.

---

## 📊 Power BI Model & Measures

**Relationships**
`f_rides` (many) → `d_time`, `d_city_zone`, `d_driver`, `d_customer` (one)

**Core DAX**

```DAX
Total Requests = COUNTROWS(f_rides)

Total Cancels = CALCULATE(
  COUNTROWS(f_rides),
  f_rides[status] = "cancelled"
)

Cancellation Rate % =
DIVIDE([Total Cancels], [Total Requests])

Driver Cancels =
CALCULATE([Total Cancels], f_rides[cancel_initiator] = "driver")

Customer Cancels =
CALCULATE([Total Cancels], f_rides[cancel_initiator] = "customer")

Peak Hours CR % =
VAR PeakHours =
    TREATAS( {8,9,18,19,20,21}, 'd_time'[hour] )
RETURN
CALCULATE( [Cancellation Rate %], PeakHours )

ETA Miss Flag =
IF( f_rides[actual_arrival_min] - f_rides[promised_eta_min] > 5, 1, 0 )

CR When ETA Missed % =
CALCULATE( [Cancellation Rate %], f_rides[ETA Miss Flag] = 1 )
```

**Recommended Visuals**

* KPI cards: `Cancellation Rate %`, `Driver Cancels`, `Customer Cancels`
* Heatmap: `hour` × `dow` with `CR%`
* Bar: `cancel_reason` by `share_pct`
* Map: `city_zone` with `CR%`
* Line: `date` vs `CR%` (annotate intervention date)

Screenshots (place in `/img` and reference below):

```md
![Dashboard Overview](./img/dashboard-overview.png)
![Cancellation by Hour](./img/cancellation-by-hour.png)
```

---

## 🔍 Key Insights (What Drove the 10% Reduction)

1. **ETA accuracy** issues correlated with high cancellations during **peak hours**.
2. **Driver-initiated cancels** spiked in **dense zones** with poor parking/approach.
3. **Cash-only profiles** and **high surge** increased early cancels.
4. **New drivers (<60 days)** showed a higher DCR than experienced cohorts.

### Actions Implemented

* **ETA model guardrails:** widened confidence band display in peak windows.
* **Driver incentives** in high-cancel zones during peak to complete pickups.
* **Upfront clarity** for users (parking instructions, pick-up points).
* **Payment nudges** (save card, wallet cashback) to reduce cash-related cancels.
* **New-driver mentorship** + temporary radius throttling in complex zones.

> **Result:** Post-intervention average **CR dropped by ~10%** (like-for-like weeks).

---

## ⚙️ How to Reproduce

### 1) Clone

```bash
git clone https://github.com/kartavyabh24/Ola-Identified-the-reason-trend-for-ride-cancellation.git
cd Ola-Identified-the-reason-trend-for-ride-cancellation
```

### 2) (Optional) Python Env

```bash
python -m venv .venv
source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3) Run SQL

* Create your DB (e.g., `sqlite3 ola.db` / Postgres DB)
* Execute scripts in `/sql` in order:
  `01_cleaning.sql` → `02_model_build.sql` → `03_kpi_views.sql`

### 4) Open Power BI

* Open `powerbi/ola_cancellations.pbix`
* Set data source to your DB / CSV folder
* Refresh → validate visuals match

---

## 📜 Data Dictionary (core fields)

| Field                  | Type     | Description                            |
| ---------------------- | -------- | -------------------------------------- |
| ride_id                | string   | unique trip id                         |
| request_time           | datetime | user requested timestamp               |
| accept_time            | datetime | driver accepted timestamp              |
| cancel_time            | datetime | cancellation timestamp (if any)        |
| status                 | enum     | requested/accepted/completed/cancelled |
| cancel_initiator       | enum     | driver/customer/system                 |
| cancel_reason          | text     | standardized reason                    |
| city, zone_id          | text     | geography                              |
| surge_multiplier       | float    | pricing factor                         |
| promised_eta_min       | int      | ETA shown to user                      |
| actual_arrival_min     | int      | observed arrival time                  |
| driver_id, customer_id | string   | keys                                   |

---

## ✅ Assumptions & Limitations

* Sample schemas; adjust field names to match your real exports.
* Mobile app logs/support tags may be noisy → cleaned by controlled dictionary.
* Seasonality/holidays not fully modeled; could be added as features.
* Post-intervention measurement uses like-for-like windows to reduce bias.

---

## 🗺️ Roadmap

* Add **A/B test** flags in facts table for robust causal lift.
* Train a **propensity-to-cancel** model; target high-risk segments.
* **What-if** analysis page in PBIX (simulating surge/ETA changes).
* Integrate **real-time alerts** when CR exceeds thresholds.

---

## 📄 License

MIT (feel free to modify for your org). Add a `LICENSE` file if distributing.

---

## 👤 Contact

**Kartavya Bhardwaj**
Email: [kartavyabhardwajwork@gmail.com](mailto:kartavyabhardwajwork@gmail.com) • Phone: +1 201-526-5479
LinkedIn: [https://www.linkedin.com/in/kartavya24](https://www.linkedin.com/in/kartavya24) • GitHub: [https://github.com/kartavyabh24](https://github.com/kartavyabh24)

 
* I can also include a small `sample.db` (SQLite) + prebuilt views to make the PBIX refresh plug-and-play.
* Want me to push this README as a commit to your repo and add placeholder screenshots?
