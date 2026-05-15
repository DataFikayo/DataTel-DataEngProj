# DataTel Automated Telecom Data Pipeline

## Project Overview

This project builds an end-to-end data pipeline for DataTel Communications,
a mid-sized telecom operator in Nigeria. The company collects millions of 
records daily across three operational systems — billing, network sessions, 
and customer records. These systems store data independently with no 
standardisation, resulting in duplicates, missing values, inconsistent 
formatting, and timestamps stored as plain text.

The goal of this pipeline is to consolidate all three sources into a single 
clean data warehouse that powers three business use cases:

- Customer analytics — identifying the most valuable users
- Churn risk detection — flagging customers becoming inactive
- Revenue operations — surfacing mismatches between data consumed and revenue generated

---

## Tools Used

- **BigQuery** — all staging, transformation, and warehouse tables
- **Python / Google Colab** — data generation
- **SQL** — all pipeline logic
- **GitHub** — version control and submission

---

## Data Sources

All three source tables were loaded into BigQuery under the `datatel` dataset:

| Table | Description | Rows |
|-------|-------------|------|
| `transactions` | One row per billing event | 1,530,000 |
| `sessions` | One row per network session | 3,060,000 |
| `customer_table` | One row per registered customer | 101,000 |

Known issues in source data:
- `transactions` contains ~30,000 duplicate transaction_ids from billing retries
- `sessions` contains duplicate session_ids from logging retries
- `amount` in transactions can be NULL for failed transactions
- `data_used_mb` in sessions can be NULL for interrupted sessions
- `end_time` in sessions can precede `start_time` due to clock sync errors
- `country` in customers can be NULL for older migrated records
- `name` and `email` have inconsistent capitalisation

---

## Pipeline Architecture

transactions (raw)     sessions (raw)     customer_table (raw)
│                     │                     │
▼                     ▼                     ▼
Stage 1: Data Quality Checks (nulls + duplicates)
│
▼
Stage 2: Staging Layer (clean, deduplicate, cast types)
stg_billing        stg_sessions        stg_customers
│
▼
Stage 3: Transformation Layer (aggregations)
agg_user_revenue       agg_user_usage
agg_monthly_revenue    agg_arpu
agg_session_distribution
│
▼
Stage 4: Data Warehouse
dw_user_analytics
│
▼
Stage 5: Analytical Queries
Top customers · Segmentation · Churn risk · Revenue mismatch
│
▼
Stage 6: Incremental Load
Append only new records to stg_billing


---

## Stage 1 — Data Quality Checks

**What it does:** Runs null and duplicate checks against the raw source 
tables before any data is moved or cleaned. No data is modified at this stage.

**Findings:**

- Null checks on `transaction_id`, `customer_id`, and `session_id` returned 
zero rows. No missing primary identifiers were found.

- The `transactions` table contained 30,000 duplicate `transaction_id` values, 
each appearing exactly twice. These were caused by retry events in the billing 
system. If left uncleaned, they would double-count revenue — a customer who 
paid ₦5,000 once would appear to have paid ₦10,000.

- The `sessions` table also contained duplicate `session_id` values appearing 
twice each, caused by logging retries. If left uncleaned, session counts and 
data usage totals would be inflated, making customers appear more active 
than they actually are.

**File:** `stage1`

---

## Stage 2 — Staging Layer

**What it does:** Produces a cleaned, typed, and deduplicated copy of each 
source table. No business metrics are calculated here. The goal is simply 
to make the data safe and consistent before aggregation begins.

**Tables produced:**
- `stg_billing` — deduplicated transactions, NULLs replaced, dates cast to TIMESTAMP
- `stg_sessions` — dates cast, NULLs replaced, session duration calculated in seconds
- `stg_customers` — names standardised, emails lowercased, NULL countries filled

**Key decisions:**

ROW_NUMBER() was used for deduplication in stg_billing rather than 
SELECT DISTINCT. DISTINCT removes rows that are completely identical across 
all columns. In this dataset, duplicate transactions share the same 
transaction_id but may differ in other columns. ROW_NUMBER() partitions 
by transaction_id and orders by date descending, so only the most recent 
version of each duplicate is kept. This is the standard SQL approach to 
controlled deduplication.

COALESCE was used to replace NULL amounts and NULL data_used_mb with zero, 
ensuring downstream aggregations never produce NULL totals.

CASE WHEN was used in stg_sessions to set session_duration_sec to zero 
for any record where end_time precedes start_time, guarding against 
negative durations caused by clock synchronisation errors.

**File:** `stage2`

---

## Stage 3 — Transformation Layer

**What it does:** Builds five aggregation tables from the clean staging 
tables. Each table answers one specific analytical question.

**Tables produced:**

| Table | What it contains |
|-------|-----------------|
| `agg_user_revenue` | Total revenue and transaction count per customer |
| `agg_user_usage` | Total data used, average session duration, session count per customer |
| `agg_monthly_revenue` | Revenue broken down by calendar month per customer |
| `agg_arpu` | Average Revenue Per User calculated per active month |
| `agg_session_distribution` | Count of short, medium, and long sessions per customer |

**Key decisions:**

NULLIF was used in agg_arpu to prevent a divide-by-zero error. ARPU is 
calculated as total revenue divided by the number of distinct active months. 
If a customer has no transactions at all, the denominator would be zero. 
NULLIF(count, 0) returns NULL when the count is zero, and dividing by NULL 
produces NULL rather than a runtime error. This keeps the table safe without 
requiring special-case handling in every downstream query.

Session bucketing was separated into two steps — session_buckets first labels 
every session as short, medium, or long, then agg_session_distribution counts 
them per customer. Separating the bucketing from the aggregation keeps each 
query simple and independently testable. If the bucket thresholds change, 
only one query needs updating.

**File:** `stage3`

---

## Stage 4 — Data Warehouse Table

**What it does:** Joins all staging and aggregation tables into one wide, 
denormalised table called `dw_user_analytics`. This is the single table that 
analysts, dashboards, and downstream models query.

**Columns:**

| Column | Source |
|--------|--------|
| customer_id, customer_name, email, country | stg_customers |
| total_revenue, total_transactions | agg_user_revenue |
| total_data_used_mb, avg_session_duration_sec, total_sessions | agg_user_usage |
| arpu | agg_arpu |
| short_sessions, medium_sessions, long_sessions | agg_session_distribution |
| avg_data_per_session_mb | derived |

**Join strategy:**

stg_customers is the anchor table. LEFT JOINs are used to attach all 
aggregation tables. This ensures every customer appears in the output 
regardless of whether they have made a transaction or started a session yet. 
An INNER JOIN would silently drop any customer with no billing or session 
records — for example a customer who registered yesterday. This would make 
the customer count wrong and cause newly registered users to disappear from 
all analytics until their first transaction.

After joining, COALESCE wraps every metric column to convert NULLs to zero. 
This keeps the table safe for arithmetic without requiring callers to handle 
NULLs themselves.

**File:** `stage4`

---

## Stage 5 — Analytical Queries

**What it does:** Answers four business questions directly from 
dw_user_analytics.

**Query 1 — Top 10 customers by revenue:**
Returns the ten customers who have generated the most lifetime revenue, 
ranked highest first.

**Query 2 — Customer segmentation:**
Labels every customer as High Value (above ₦5,000,000), Mid Value 
(above ₦1,000,000), or Low Value (all others) based on total revenue.

**Query 3 — Churn risk detection:**
Flags customers with fewer than 5 total sessions AND less than ₦1,000 
in total revenue as High Risk. All others are labelled Active.

Limitation: A brand new customer who registered yesterday would also 
have 0 sessions and 0 revenue and would be incorrectly flagged as High Risk. 
To improve this rule, an account age condition should be added — only flag 
customers whose account is older than 30 days but still shows low activity. 
This targets genuinely inactive customers rather than new ones.

**Query 4 — Revenue vs usage mismatch:**
Returns customers who have consumed more than 10,000 MB of data but 
generated less than ₦500 in revenue. These users may be on outdated or 
under-priced legacy plans and represent a revenue recovery opportunity.

**File:** `stage5`

---

## Stage 6 — Incremental Loading

**What it does:** Instead of rebuilding stg_billing from scratch on every 
pipeline run, the incremental load appends only rows that are newer than 
the latest timestamp already present in the table.

**How it works:**
A scalar subquery dynamically fetches MAX(transaction_ts) from stg_billing. 
The INSERT then filters the source table to only rows with a transaction_date 
greater than that value. No date is ever hardcoded, so the query works 
correctly on every run regardless of when it last executed.

A freshness check query returns MAX(transaction_ts) from stg_billing after 
each run, giving operators a quick way to confirm the pipeline ran 
successfully and data is not stale.

**Extending to stg_sessions:**
The same pattern applies directly to stg_sessions. The INSERT would filter 
src_network_sessions where CAST(start_time AS TIMESTAMP) is greater than 
SELECT MAX(start_time) FROM stg_sessions. Only the table name and timestamp 
column change — the logic is identical.

**File:** `stage6`
