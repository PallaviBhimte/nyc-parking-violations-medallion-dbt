# NYC Parking Violations — Medallion Architecture with dbt

A dbt Core project that transforms raw NYC parking violations data into a bronze/silver/gold
medallion architecture on DuckDB, with documentation, data tests, and a CI pipeline that runs
the models against a production database on every push.

| Layer | Model | Materialization | Purpose |
|---|---|---|---|
| Bronze | `bronze_parking_violations` | view | 20 of the 34 raw columns |
| Bronze | `bronze_parking_violation_codes` | view | Violation codes and their fees |
| Silver | `silver_parking_violations` | ephemeral | Flags whether a ticket was issued below Manhattan 96th St |
| Silver | `silver_parking_violation_codes` | ephemeral | Unpivots the two fee columns into one `fee_usd` column |
| Silver | `silver_violation_tickets` | view | Tickets joined to their actual fee |
| Silver | `silver_violation_vehicles` | view | Vehicle attributes keyed by summons number |
| Gold | `gold_ticket_metrics` | table | Ticket count and total revenue per violation code |
| Gold | `gold_vehicles_metrics` | table | Ticket count by out-of-state registration |

## Data

Two public datasets from [NYC Open Data](https://opendata.cityofnewyork.us/):

- **[Parking Violations Issued — Fiscal Year 2023](https://data.cityofnewyork.us/City-Government/Parking-Violations-Issued-Fiscal-Year-2023/pvqr-7yc4)** — a 100,000-row sample
- **[DOF Parking Violation Codes](https://data.cityofnewyork.us/Transportation/DOF-Parking-Violation-Codes/ncbg-6agr)** — 97 codes with their fees

Both CSVs are committed under `data/`. The DuckDB database is built from them, not stored in git.

## Stack

| | |
|---|---|
| Transformation | dbt-core 1.6.1 |
| Warehouse | DuckDB 0.9.0 (via dbt-duckdb 1.6.0) |
| Runtime | Python 3.11 |
| CI | GitHub Actions |


## Setup

```bash
# 1. Create the environment
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 2. Load the CSVs into DuckDB
#    Open run_sql_queries_here.ipynb, select .venv as the kernel,
#    and run the CREATE TABLE cells. This builds data/nyc_parking_violations.db.
```


## Running

dbt must run from inside the project directory, because `profiles.yml` uses a path relative to the
working directory.

```bash
cd nyc_parking_violations
export DBT_PROFILES_DIR=.

dbt debug      # verify the connection
dbt run        # build all models
dbt test       # run data tests
dbt build      # run + test in dependency order

dbt docs generate && dbt docs serve   # lineage graph at localhost:8080
```

`DBT_PROFILES_DIR=.` is required because `profiles.yml` lives in the project rather than `~/.dbt/`.

## Tests

`dbt test` runs four tests. Failing rows are written to the database
(`store_failures: true`) so they can be inspected with SQL.

| Test | Type | Target |
|---|---|---|
| `unique` | built-in generic | `bronze_parking_violations.summons_number` |
| `not_null` | built-in generic | `bronze_parking_violations.summons_number` |
| `generic_not_null` | custom generic | `bronze_parking_violations.summons_number` |
| `violation_codes_revenue` | singular | Every violation code should total at least $1 in fees |

`violation_codes_revenue` is configured with `severity: warn` and currently reports one row —
a code with no fee attached, kept as a deliberate data-quality signal rather than a build failure.

## Documentation

Column descriptions live in `models/docs/` and are split in two:

- **`docs_blocks.md`** — 27 named text blocks, one per column
- **`schema.yml`** — wires each model's columns to a block via `{{ doc("...") }}`, and attaches tests

Columns such as `violation_code` appear in several models, so defining each description once keeps
them from drifting apart.

## Deployment

[`.github/workflows/run-dbt-prod.yml`](.github/workflows/run-dbt-prod.yml) runs `dbt debug`,
`compile`, `run`, and `test` against the `prod` target on every push and pull request to `main`.

The `prod` target points at `data/prod_nyc_parking_violations.db`, which **is** committed —
unlike the dev database. CI never runs the notebook, so the raw tables the bronze models read
have to already exist in that file.

## Project structure

```
├── data/                          # source CSVs + DuckDB databases
├── nyc_parking_violations/        # the dbt project
│   ├── models/
│   │   ├── bronze/                # raw, lightly shaped
│   │   ├── silver/                # cleaned, business logic applied
│   │   ├── gold/                  # metrics for dashboards
│   │   └── docs/                  # schema.yml + docs_blocks.md
│   ├── tests/                     # singular and custom generic tests
│   ├── dbt_project.yml            # materializations per layer
│   └── profiles.yml               # dev + prod DuckDB connections
├── run_sql_queries_here.ipynb     # loads the CSVs into DuckDB
└── requirements.txt
```