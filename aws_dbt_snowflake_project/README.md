# 🏠 Airbnb End-to-End Data Engineering Pipeline

> **Built following the project tutorial by [Ansh Lamba] — one of the clearest voices in the data engineering space. His walkthroughs on dbt + Snowflake are genuinely worth your time if you're learning the modern data stack.**

---

## What This Is

A production-style data pipeline for Airbnb data — built end-to-end using Snowflake, dbt, and AWS S3. The project walks through every layer of the modern data stack: raw ingestion, cleaning, transformation, and analytics-ready output, all organized through a Medallion architecture (Bronze → Silver → Gold).

This isn't just a tutorial clone. Along the way I implemented incremental loading, SCD Type 2 snapshots, custom Jinja macros, data quality tests, and a denormalized One Big Table for downstream analytics. The kind of thing you'd actually see in a real data team's repo.

---

## Architecture

```
CSV Source Files
      │
      ▼
  AWS S3 Bucket
      │
      ▼
Snowflake Staging
      │
      ├──► 🥉 Bronze Layer  →  Raw tables, minimal transforms
      │
      ├──► 🥈 Silver Layer  →  Cleaned, validated, standardized
      │
      └──► 🥇 Gold Layer    →  Analytics-ready: OBT + Fact table
```

**Stack:** Snowflake · dbt Core · AWS S3 · Python 3.12 · Git

---

## Data Model

### 🥉 Bronze — Raw Ingestion
Lands data from staging with no business logic applied. Source of truth for everything downstream.

| Table | Description |
|---|---|
| `bronze_bookings` | Raw booking transactions |
| `bronze_hosts` | Raw host records |
| `bronze_listings` | Raw property listings |

### 🥈 Silver — Cleaned & Standardized
Type casting, null handling, deduplication, and price categorization happen here.

| Table | Description |
|---|---|
| `silver_bookings` | Validated booking records |
| `silver_hosts` | Enhanced host profiles with quality metrics |
| `silver_listings` | Standardized listings with price tier tagging |

### 🥇 Gold — Analytics-Ready
Business logic lives here. Optimized for consumption by BI tools or analysts.

| Table | Description |
|---|---|
| `obt` | One Big Table — denormalized join of bookings, listings, hosts |
| `fact` | Fact table for dimensional modeling |
| Ephemeral models | Intermediate transforms; never materialized |

### Snapshots — SCD Type 2
Historical tracking across all three domains. Every change to a host, listing, or booking gets a `valid_from` / `valid_to` record — no data is ever overwritten.

| Snapshot | Tracks |
|---|---|
| `dim_bookings` | Booking record changes over time |
| `dim_hosts` | Host profile evolution |
| `dim_listings` | Listing updates and price changes |

---

## Project Structure

```
AWS_DBT_Snowflake/
├── main.py                              # Entry point
├── pyproject.toml                       # Python dependencies
├── SourceData/
│   ├── bookings.csv
│   ├── hosts.csv
│   └── listings.csv
├── DDL/
│   ├── ddl.sql                          # Table creation scripts
│   └── resources.sql
└── aws_dbt_snowflake_project/
    ├── dbt_project.yml
    ├── models/
    │   ├── sources/sources.yml
    │   ├── bronze/                      # Raw layer
    │   ├── silver/                      # Cleaned layer
    │   └── gold/                        # Analytics layer
    │       └── ephemeral/
    ├── macros/
    │   ├── tag.sql                      # Price categorization
    │   ├── trimmer.sql                  # String utilities
    │   ├── multiply.sql
    │   └── generate_schema_name.sql
    ├── snapshots/                       # SCD Type 2 configs
    ├── tests/                           # Data quality tests
    ├── analyses/                        # Ad-hoc exploration
    └── seeds/                           # Static reference data
```

---

## Getting Started

### Prerequisites
- Python 3.12+
- A Snowflake account
- An AWS account (for S3)

### 1. Clone & set up environment

```bash
git clone <repository-url>
cd AWS_DBT_Snowflake

python -m venv .venv
source .venv/bin/activate        # Mac/Linux
.venv\Scripts\Activate.ps1       # Windows PowerShell

pip install -e .
```

Core dependencies: `dbt-core>=1.11.2`, `dbt-snowflake>=1.11.0`, `sqlfmt`

### 2. Configure Snowflake connection

Create `~/.dbt/profiles.yml`:

```yaml
aws_dbt_snowflake_project:
  outputs:
    dev:
      type: snowflake
      account: <your-account-identifier>
      user: <your-username>
      password: <your-password>
      role: ACCOUNTADMIN
      database: AIRBNB
      warehouse: COMPUTE_WH
      schema: dbt_schema
      threads: 4
  target: dev
```

> ⚠️ Never commit `profiles.yml` to Git. Use environment variables for sensitive values.

### 3. Set up Snowflake schema

Run `DDL/ddl.sql` in your Snowflake worksheet to create the staging tables.

### 4. Load source data

Upload the CSVs from `SourceData/` to Snowflake staging:

```
bookings.csv  →  AIRBNB.STAGING.BOOKINGS
hosts.csv     →  AIRBNB.STAGING.HOSTS
listings.csv  →  AIRBNB.STAGING.LISTINGS
```

---

## Running the Pipeline

```bash
cd aws_dbt_snowflake_project

# Verify connection
dbt debug

# Install packages
dbt deps

# Run everything
dbt build

# Or layer by layer
dbt run --select bronze.*
dbt run --select silver.*
dbt run --select gold.*

# Run snapshots (SCD Type 2)
dbt snapshot

# Run data quality tests
dbt test

# Generate + serve documentation
dbt docs generate && dbt docs serve
```

To fully rebuild from scratch (ignores incremental state):
```bash
dbt run --full-refresh
```

---

## Key Implementation Details

### Incremental Loading
Bronze and silver models process only new records on each run, using a high-water mark on `CREATED_AT`. Full refresh available when needed.

```sql
{{ config(materialized='incremental') }}

{% if is_incremental() %}
  WHERE CREATED_AT > (SELECT COALESCE(MAX(CREATED_AT), '1900-01-01') FROM {{ this }})
{% endif %}
```

### Custom `tag()` Macro
Categorizes listing prices into `low`, `medium`, `high` tiers — reusable across any model.

```sql
{{ tag('CAST(PRICE_PER_NIGHT AS INT)') }} AS PRICE_PER_NIGHT_TAG
```

### Dynamic OBT with Jinja Loops
The Gold OBT model generates its joins programmatically, so adding a new dimension is a config change, not a rewrite.

### Schema Isolation by Layer
A custom `generate_schema_name` macro ensures each layer lands in its own Snowflake schema automatically — no manual schema management needed.

---

## Data Quality

Tests run across all layers:
- Unique key constraints on primary keys
- Not null checks on critical fields
- Referential integrity between bookings, hosts, and listings
- Custom business rule assertions

Data lineage is tracked automatically by dbt — you can see exactly which source feeds which model feeds which output.

---

## What I'd Add Next

- [ ] CI/CD pipeline (GitHub Actions running `dbt build` on PR)
- [ ] Data quality dashboard surfacing test results over time
- [ ] PII masking for host contact fields
- [ ] Alerting on source data anomalies (row count drops, null spikes)
- [ ] BI layer — Power BI or Looker connecting directly to the Gold schema

---

## Credit

This project was built alongside **[Ansh Lamba's](https://www.youtube.com/@anshlamba)** data engineering tutorial series. If you're trying to learn dbt and Snowflake from scratch, his explanations of medallion architecture, incremental models, and SCD snapshots are among the best I've found — practical, no fluff, genuinely teaches you to think in layers rather than just copy SQL.
