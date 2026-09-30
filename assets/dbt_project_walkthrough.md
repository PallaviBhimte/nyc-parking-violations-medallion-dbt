# Apply Medallion Architecture on NYC parking violations data 
## (using dbt core)

### Overview:

In this project we transform raw NYC parking violations data into the Medallion Architecture for a company's data lakehouse. We use dbt Core to implement it, which brings software engineering best practices to SQL transformations.

By the end, the pipeline and assets roughly match the medallion architecture.

### Data:

This project utilizes two public government datasets sourced from NYC Open Data:

**1. [NYC Parking Violations Issued - Fiscal Year 2023](https://data.cityofnewyork.us/City-Government/Parking-Violations-Issued-Fiscal-Year-2023/pvqr-7yc4)**  
> Parking Violations Issuance datasets contain violations issued during the respective fiscal year. The Issuance datasets are not updated to reflect violation status, the information only represents the violation(s) at the time they are issued. Since appearing on an issuance dataset, a violation may have been paid, dismissed via a hearing, statutorily expired, or had other changes to its status. To see the current status of outstanding parking violations, please look at the Open Parking & Camera Violations dataset.

This dataset is large, so we use a sample of 100K records.

**2. [NYC Department of Finance Parking Violation Codes](https://data.cityofnewyork.us/Transportation/DOF-Parking-Violation-Codes/ncbg-6agr)**  
> This dataset defines the parking violation codes in New York City and lists the fines. Each fine amount includes a $15 New York State Criminal Justice surcharge.

# Project

## Step 1: Download dbt Core

### Install the dbt Core via pip:

The package `dbt-core` is what every connector and plugin in the dbt ecosystem builds on, so it goes in first.


```bash
❯ pip install dbt-core==1.6.1
```

Documentation: https://docs.getdbt.com/docs/core/pip-install

```bash
❯ pip show dbt-core

Name: dbt-core
Version: 1.6.1
Summary: With dbt, data analysts and engineers can build analytics the way engineers build applications.
Home-page: https://github.com/dbt-labs/dbt-core
Author: dbt Labs
Author-email: info@dbtlabs.com
```

```bash
❯ pip install dbt-duckdb==1.6.0
```

### Install the dbt connector to DuckDB:

DuckDB is a lightweight analytical database, well suited to local and small data such as this project.

Documentation: https://docs.getdbt.com/docs/core/connect-data-platform/duckdb-setup

```bash
❯ pip show dbt-duckdb

Name: dbt-duckdb
Version: 1.6.0
Summary: The duckdb adapter plugin for dbt (data build tool)
Home-page: https://github.com/jwills/dbt-duckdb
Author: Josh Wills
Author-email: joshwills+dbt@gmail.com
```

### Create a requirements.txt file:

Create a `requirements.txt` file in the repo with the following:
```bash
dbt-core==1.6.1
dbt-duckdb==1.6.0
```

## Step 2: Download duckDB

We use DuckDB because it is simple and runs entirely in the local environment. There is no need to set up an external cloud account such as GCP BigQuery.

Documentation: https://duckdb.org/docs/installation

### Install DuckDB via pip:

```bash
❯ pip install duckdb==0.9.0
```

```bash
❯ pip show duckdb
Name: duckdb
Version: 0.8.1
Summary: DuckDB embedded database
Home-page: https://www.duckdb.org
```

### Update the requirements.txt file:

Update `requirements.txt` to the following:
```bash
dbt-core==1.6.1
dbt-duckdb==1.6.0
duckdb==0.9.0
```

## Step 3: Prepare the Database Environment

In most cases dbt Core connects to an existing database. Here we use DuckDB so that everything runs locally and inside a Jupyter notebook, which means setting up the database ourselves in a few commands.

### Create the database file:

The database doesn't exist yet, so DuckDB autogenerates `nyc_parking_violations.db` on the first SQL command. Run the following code in the notebook `run_sql_queries_here.ipynb`:

```python
sql_query = '''
show tables
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())
```

The database file now appears in the `data` folder.

### Import CSV data into the database:

With the database in place, insert the CSV data using the following SQL commands in the notebook:

```python
sql_query_import_1 = '''
CREATE OR REPLACE TABLE parking_violation_codes AS
SELECT *
FROM read_csv_auto(
  'data/dof_parking_violation_codes.csv',
  normalize_names=True
  )
'''

sql_query_import_2 = '''
CREATE OR REPLACE TABLE parking_violations_2023 AS
SELECT *
FROM read_csv_auto(
  'data/parking_violations_issued_fiscal_year_2023_sample.csv',
  normalize_names=True
  )
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
  con.sql(sql_query_import_1)
  con.sql(sql_query_import_2)
```
Running `show tables` again shows the new tables in DuckDB.

```python
sql_query = '''
show tables
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())

# output
# ┌─────────────────────────┐
# │          name           │
# │         varchar         │
# ├─────────────────────────┤
# │ parking_violation_codes │
# │ parking_violations_2023 │
# └─────────────────────────┘
```

We can also query the data directly:
```python
sql_query = '''
SELECT * FROM parking_violation_codes LIMIT 5
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())
```

Output:

| code | definition | manhattan_96th_st_below | all_other_areas |
|---:|---|---:|---:|
| *int64* | *varchar* | *int64* | *int64* |
| 1 | FAILURE TO DISPLAY BUS PERMIT | 515 | 515 |
| 2 | NO OPERATOR NAM/ADD/PH DISPLAY | 515 | 515 |
| 3 | UNAUTHORIZED PASSENGER PICK-UP | 515 | 515 |
| 4 | BUS PARKING IN LOWER MANHATTAN | 115 | 115 |
| 5 | BUS LANE VIOLATION | 250 | 250 |


## Step 4: Create a dbt Project

dbt has a builtin command, `dbt init`, that generates a new project with every file needed to get started and organizes the directory correctly.

```bash
❯ dbt init

Running with dbt=1.6.1
Creating dbt configuration folder at 
Enter a name for your project (letters, digits, underscore):
```

`dbt init` asks for a project name. Enter `nyc_parking_violations`.

```bash
❯ Enter a name for your project (letters, digits, underscore): nyc_parking_violations

Your new dbt project "nyc_parking_violations" was created!
```

The `logs` and `nyc_parking_violations` folders and their files are now autogenerated.

## Step 5: Prepare the dbt Environment

Before using dbt Core, we set up the configuration and connect dbt to the database (DuckDB for this project).

### The project YAML file:

One of the autogenerated files in `nyc_parking_violations` is `dbt_project.yml`. It controls every setting for the project and tells dbt Core where to look and what to act on.

Documentation: https://docs.getdbt.com/docs/build/projects#project-configuration

The default settings suffice for now. The key components of `dbt_project.yml` are:

The name entered earlier is passed on to the rest of the project's directories and configuration files, and dbt uses it to know which project to reference. It makes little difference at this scale, but a production dbt project can have multiple profiles to manage.
```yaml
profile: 'nyc_parking_violations'
```

dbt also needs to know where to look for specific files such as SQL models or tests. dbt references numerous paths; only `model-paths` and `test-paths` matter for this project.
```yaml
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]
```

Every run creates "assets" that are essentially logs of the actions dbt conducted. These settings tell dbt Core where to clean them up when they are no longer wanted.
```yaml
clean-targets:
  - "target"
  - "dbt_packages"
```

Finally, the `models` configuration is where most of the work happens. It tells the project how the output of the SQL models should be treated. The autogenerated default materializes output as a view, which can be changed to tables, incremental builds, or ephemeral (analogous to a temp table).
```yaml
models:
  nyc_parking_violations:
    example:
      +materialized: view
```

### The profiles YAML file:

Running `dbt init` produced this:
```bash
Setting up your profile.
Which database would you like to use?
[1] duckdb

Enter a number: 1
No sample profile found for duckdb.
```

DuckDB is a relatively new open-source database and has no sample profile in the `dbt init` workflow, as noted by "No sample profile found for duckdb." That means we write our own `profiles.yml` file. This file is a key difference between dbt Core and dbt Cloud, which manages it automatically.

It doesn't need to be written from scratch: the `dbt-duckdb` integration documents a template, which we implement in the next section.

Documentation: https://github.com/jwills/dbt-duckdb#configuring-your-profile

### Creating the profiles YAML file:

1. In the `nyc_parking_violations` dbt project, create a new file called `profiles.yml`.

```bash
❯ cd nyc_parking_violations
❯ touch profiles.yml
```

The `profiles.yml` file now appears in the `nyc_parking_violations` directory.

2. Add the following lines to `profiles.yml`, based on the `dbt-duckdb` instructions.

```yaml
default:
  outputs:
   dev:
     type: duckdb
  target: dev
```

This is a `dev` profile. A later step updates it to also allow a production workflow.

### Connecting the profiles and project YAML files:

From the dbt documentation on how the `dbt_project.yml` and  `profiles.yml` files interact:

> *"When you run dbt from the CLI, it reads your dbt_project.yml file to find the profile name, and then looks for a profile with the same name in your profiles.yml file. This profile contains all the information dbt needs to connect to your data platform."*

The CLI command `dbt debug` checks whether `dbt_project.yml` and `profiles.yml` are connected. With the current configuration it returns an error that's easy to fix:

```bash
❯ dbt debug
Running with dbt=1.6.1
dbt version: 1.6.1
python version: 3.9.15
.
.
.
Configuration:
  profiles.yml file [ERROR invalid]
  dbt_project.yml file [OK found and valid]
Required dependencies:
 - git [OK found]

Connection test skipped since no profile was found
1 check failed:
Profile loading failed for the following reason:
Runtime Error
  Could not find profile named 'nyc_parking_violations'
```

The fix is to make the line `profile: 'nyc_parking_violations'` in `dbt_project.yml` match the header line in `profiles.yml`. Update `profiles.yml` from `default:` to `nyc_parking_violations:` and add the path to the DuckDB database. The final `profiles.yml` looks like this:

```yaml
nyc_parking_violations:
  outputs:
   dev:
     type: duckdb
     path: '../data/nyc_parking_violations.db'
  target: dev
```

After saving, `dbt debug` passes all checks:

```bash
❯ dbt debug
Running with dbt=1.6.1
dbt version: 1.6.1
python version: 3.9.15
.
.
.
adapter type: duckdb
adapter version: 1.6.0
Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]
Required dependencies:
 - git [OK found]

Connection:
  database: nyc_parking_violations
  schema: main
  path: ../data/nyc_parking_violations.db
.
.
.
All checks passed!
```

We can now start building dbt models.

## Step 6: Creating a dbt Model

`dbt init` created two example models in `nyc_parking_violations/models/example`. We delete them and write our own.

### Create a dbt model file:

dbt models are `.sql` files with a few extra tools available inside them (covered in Step 8). Create the first model with the `touch` command:
```bash
touch models/example/first_model.sql
```

The model uses the following SQL:
```sql
SELECT * FROM parking_violation_codes
```

## Step 7: Using dbt CLI Commands

Three dbt CLI commands cover most of the work:  
1. `dbt debug`: Checks whether the configuration files are set up correctly and recognized by dbt Core.
2. `dbt compile`: Runs all the dbt models end-to-end, but doesn't execute the models' SQL code nor materialize the tables; which is useful for quickly checking the models for errors.
3. `dbt run`: Runs all the dbt models end-to-end, executes the SQL code, and materialize the tables based on the profile configurations.

*Note: a common mistake is running dbt commands outside the dbt project directory, which results in an error.*

### `dbt debug`:

`dbt debug` is worth running first when developing a project, as it catches simple errors early.
```bash
❯ dbt debug
Running with dbt=1.6.1
dbt version: 1.6.1
python version: 3.9.15
.
.
.
adapter type: duckdb
adapter version: 1.6.0
Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]
Required dependencies:
 - git [OK found]

Connection:
  database: nyc_parking_violations
  schema: main
  path: ../data/nyc_parking_violations.db
  config_options: None
  extensions: None
  settings: None
  external_root: .
  use_credential_provider: None
  attach: None
  filesystems: None
  remote: None
  plugins: None
  disable_transactions: False
Registered adapter: duckdb=1.6.0
  Connection test: [OK connection ok]

All checks passed!
```

### `dbt compile`:

This command is optional, but running it before `dbt run` catches model errors early. That matters little on a project this small; on projects processing large amounts of data it saves considerable time by avoiding a late breaking error.

```bash
❯ dbt compile
Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Found 1 model, 0 sources, 0 exposures, 0 metrics, 348 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')
```

### `dbt run`:

This is the most important dbt Core CLI command. It runs the entire project and carries out the data transformations in the database.

```bash
❯ dbt run
Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Found 1 model, 0 sources, 0 exposures, 0 metrics, 348 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')

1 of 1 START sql view model main.first_model ................................... [RUN]
1 of 1 OK created sql view model main.first_model .............................. [OK in 0.05s]

Finished running 1 view model in 0 hours 0 minutes and 0.12 seconds (0.12s).

Completed successfully

Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

`dbt run` adds a new table to the DuckDB database. Open the notebook `run_sql_queries_here.ipynb` and run `show tables` to confirm.

```python
sql_query = '''
show tables
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())

# output
# ┌─────────────────────────┐
# │          name           │
# │         varchar         │
# ├─────────────────────────┤
# │ first_model             │
# │ parking_violation_codes │
# │ parking_violations_2023 │
# └─────────────────────────┘
```

Note how the table name in the database matches the filename of the model. This is why every dbt model name needs to be unique.

## Step 8: The dbt `ref` Function

dbt adds "jinja syntax" to `.sql` files to extend what SQL can do. The most important of these are `ref` statements, written as `{{ref('your_dbt_model_name')}}`. They let dbt build lineage for the data transformations, which is what drives orchestration, dependencies, and documentation.

### Create a dbt model utilizing `ref`:

Create the second model with `touch` and name it `ref_model.sql`.

```bash
touch models/example/ref_model.sql
```

The second model uses the following SQL and `ref` command:
```sql
SELECT
    COUNT(*)
FROM
    {{ref('first_model')}}
```

### Run the dbt models with the `ref` syntax:

With the second model in place, the `ref` syntax shows what dbt does for data transformations.

```bash
❯ dbt run
Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Found 2 models, 0 sources, 0 exposures, 0 metrics, 348 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')

1 of 2 START sql view model main.first_model ................................... [RUN]
1 of 2 OK created sql view model main.first_model .............................. [OK in 0.05s]
2 of 2 START sql view model main.ref_model ..................................... [RUN]
2 of 2 OK created sql view model main.ref_model ................................ [OK in 0.02s]

Finished running 2 view models in 0 hours 0 minutes and 0.14 seconds (0.14s).

Completed successfully

Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

dbt knew to run `first_model.sql` first and `ref_model.sql` second.

The output of `ref_model.sql` is visible in the database with the following query in `run_sql_queries_here.ipynb`.

```python
sql_query = '''
SELECT * FROM ref_model
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())

# output
# ┌──────────────┐
# │ count_star() │
# │    int64     │
# ├──────────────┤
# │           97 │
# └──────────────┘
```

### View the dbt project data lineage:

Step 14 covers dbt docs in detail. This much is enough to visualize the project.

The CLI command `dbt docs generate` autogenerates the docs files in the `nyc_parking_violations/target` folder.

```bash
❯ dbt docs generate
Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Found 2 models, 0 sources, 0 exposures, 0 metrics, 348 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')

Building catalog
Catalog written to nyc_parking_violations/target/catalog.json
```

The command `dbt docs serve` then launches the docs as a local website.

```bash
❯ dbt docs serve
05:42:00  Running with dbt=1.6.1
Serving docs at 8080
To access from your browser, navigate to: http://localhost:8080
```

The generated docs show the two models and the lineage between them.

## Step 9: Implementing Medallion Architecture Planning

With the dbt Core basics covered, we can return to the goal:

> In this project we transform raw NYC parking violations data into the Medallion Architecture for a company's data lakehouse. We use dbt Core to implement it, which brings software engineering best practices to SQL transformations.

The goal is a data pipeline and assets that roughly match the medallion architecture.

### Project breakdown:

The project breaks into the following components:
- **Bronze Data**
  - Goal: Raw data with minimal cleaning and transformations.
  - Tables:
    - bronze_parking_violation_codes
    - bronze_parking_violations
- **Silver Data**
  - Goal: Cleaned data with applied business logic, and ultimately in an established data model.
  - Tables:
    - silver_parking_violation_codes
    - silver_parking_violations
    - silver_violation_tickets
    - silver_violation_vehicles
- **Gold Data**
  - Goal: Metrics built on top of silver data that are often served to the business via dashboards.
  - Tables:
    - gold_ticket_metrics
    - gold_vehicles_metrics

## Step 10: Medallion Architecture - Bronze Data:

Bronze data stays in a *mostly* raw state, with minor transformations that make it easier to manage in the analytical database. Here we take a subset of the parking violations columns.

When this step is done, the database holds:  
**Bronze Data**
- bronze_parking_violation_codes
- bronze_parking_violations

Create a directory within the `nyc_parking_violations/models` folder called `bronze` and create two files called `bronze_parking_violation_codes.sql` and 
`bronze_parking_violations.sql`.

```bash
❯ mkdir models/bronze
❯ touch models/bronze/bronze_parking_violation_codes.sql
❯ touch models/bronze/bronze_parking_violations.sql
```

### Create `bronze_parking_violation_codes.sql` table:

Add the following SQL to `bronze_parking_violation_codes.sql`. It's a small table, so we bring in all the columns. The one change is renaming `code` to `violation_code` so it aligns with the parking violations dataset.

```sql
SELECT
    code AS violation_code,
    definition,
    manhattan_96th_st_below,
    all_other_areas
FROM
    parking_violation_codes
```

### Create `bronze_parking_violations.sql` table:

Add the following SQL to `bronze_parking_violations.sql`. This table has many columns, but we only need a subset.

```sql
SELECT
    summons_number,
    registration_state,
    plate_type,
    issue_date,
    violation_code,
    vehicle_body_type,
    vehicle_make,
    issuing_agency,
    vehicle_expiration_date,
    violation_location,
    violation_precinct,
    issuer_precinct,
    issuer_code,
    issuer_command,
    issuer_squad,
    violation_time,
    violation_county,
    violation_legal_code,
    vehicle_color,
    vehicle_year,
FROM
    parking_violations_2023
```

### Build bronze tables via dbt:

With the bronze models in place, run the dbt commands from Step 7 to update the database.

```bash
❯ dbt debug
❯ dbt compile
❯ dbt run

Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Found 4 models, 0 sources, 0 exposures, 0 metrics, 348 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')

1 of 4 START sql view model main.bronze_parking_violation_codes ................ [RUN]
1 of 4 OK created sql view model main.bronze_parking_violation_codes ........... [OK in 0.06s]
2 of 4 START sql view model main.bronze_parking_violations ..................... [RUN]
2 of 4 OK created sql view model main.bronze_parking_violations ................ [OK in 0.02s]
3 of 4 START sql view model main.first_model ................................... [RUN]
3 of 4 OK created sql view model main.first_model .............................. [OK in 0.02s]
4 of 4 START sql view model main.ref_model ..................................... [RUN]
4 of 4 OK created sql view model main.ref_model ................................ [OK in 0.02s]

Finished running 4 view models in 0 hours 0 minutes and 0.19 seconds (0.19s).

Completed successfully

Done. PASS=4 WARN=0 ERROR=0 SKIP=0 TOTAL=4
```

### View bronze tables in database:

The changes are visible in the database:

```python
sql_query = '''
show tables
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())

# output
# ┌────────────────────────────────┐
# │              name              │
# │            varchar             │
# ├────────────────────────────────┤
# │ bronze_parking_violation_codes │
# │ bronze_parking_violations      │
# │ first_model                    │
# │ parking_violation_codes        │
# │ parking_violations_2023        │
# │ ref_model                      │
# └────────────────────────────────┘
```
```python
sql_query = '''
SELECT * FROM bronze_parking_violations LIMIT 3
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())
```

Output:

| summons_number | plate_id | … | violation_post_code | violation_descript… |
|---:|---|:-:|---|---|
| *int64* | *varchar* | | *varchar* | *varchar* |
| 9010912681 | MN686640 | … | 08 | 17-No Stand (exc a… |
| 4858762841 | LBG4461 | … | | PHTO SCHOOL ZN SPE… |
| 4854645684 | 0982AY | … | | PHTO SCHOOL ZN SPE… |

*3 rows, 27 columns (4 shown)*


## Step 11: Medallion Architecture - Silver Data

Silver data aligns with the established data model for the analytical database. Data modeling is extremely important in data engineering, but it isn't the goal here. Instead we focus on four outcomes for the silver data:  
1. Expand the parking violation code table, `parking_violation_codes`, so that each row is unique and is either a `manhattan_96th_st_below` or a `all_other_areas` fee.
2. Classify each row in `parking_violations` as either occurring in `manhattan_96th_st_below` or `all_other_areas`.
3. Creating a new table by merging the cleaned `parking_violation_codes` and `parking_violations` tables and extracting only the violations data to determine the actual fee for each violation.
4. Creating a new table by extracting the vehicle data from the cleaned `parking_violations` table.

When this step is done, the database holds:  
**Silver Data**
- silver_parking_violation_codes
- silver_parking_violations
- silver_violation_tickets
- silver_violation_vehicles

Create a directory within the `nyc_parking_violations/models` folder called `silver` and create four files called `silver_parking_violation_codes.sql`, 
`silver_parking_violations.sql`, `silver_violation_tickets.sql`, and `silver_violation_vehicles.sql`.

```bash
❯ mkdir models/silver
❯ touch models/silver/silver_parking_violation_codes.sql
❯ touch models/silver/silver_parking_violations.sql
❯ touch models/silver/silver_violation_tickets.sql
❯ touch models/silver/silver_violation_vehicles.sql
```

### Create `silver_parking_violation_codes.sql` table:

Add the following SQL to `silver_parking_violation_codes.sql`. The source table, `bronze_parking_violation_codes`, holds fee values in two separate columns, `manhattan_96th_st_below` and `all_other_areas`, and we need them under a single column so the table merges cleanly in a later step.

```sql
WITH manhattan_violation_codes AS (
    SELECT
        violation_code,
        definition,
        TRUE AS is_manhattan_96th_st_below,
        manhattan_96th_st_below AS fee_usd,
    FROM
        {{ref('bronze_parking_violation_codes')}}
),

all_other_violation_codes AS (
    SELECT
        violation_code,
        definition,
        FALSE AS is_manhattan_96th_st_below,
        all_other_areas AS fee_usd,
    FROM
        {{ref('bronze_parking_violation_codes')}}
)

SELECT * FROM manhattan_violation_codes
UNION ALL
SELECT * FROM all_other_violation_codes
ORDER BY violation_code ASC
```

Note that `{{ref()}}` is only used for tables in the database, not for tables created during the query such as `manhattan_violation_codes`.

### Create `silver_parking_violations.sql` table:

Add the following SQL to `silver_parking_violations.sql`. This step classifies whether a ticket was issued below Manhattan 96th Street. As a simplification, `violation_county` equal to `MN` is treated as `is_manhattan_96th_st_below` being `TRUE`.

That is a heavy simplification. Resolving business logic nuances like this is a core part of data engineering work, and on a real project it takes substantial time to confirm that the data semantics match the business use case.

```sql
SELECT
    summons_number,
    registration_state,
    plate_type,
    issue_date,
    violation_code,
    vehicle_body_type,
    vehicle_make,
    issuing_agency,
    vehicle_expiration_date,
    violation_location,
    violation_precinct,
    issuer_precinct,
    issuer_code,
    issuer_command,
    issuer_squad,
    violation_time,
    violation_county,
    violation_legal_code,
    vehicle_color,
    vehicle_year,
    CASE WHEN
        violation_county == 'MN'
        THEN TRUE
        ELSE FALSE
        END AS is_manhattan_96th_st_below
FROM
    {{ref('bronze_parking_violations')}}
```

### Create `silver_violation_tickets.sql` table:

Add the following SQL to `silver_violation_tickets.sql`. This step merges the two silver tables created above, so `{{ref()}}` appears again. The table drops the columns related to vehicle information and determines the `fee_usd` value for each violation, merging `silver_parking_violations` and `silver_parking_violation_codes` on `violation_code` and `is_manhattan_96th_st_below`.

```sql
SELECT
    violations.summons_number,
    violations.issue_date,
    violations.violation_code,
    violations.is_manhattan_96th_st_below,
    violations.issuing_agency,
    violations.violation_location,
    violations.violation_precinct,
    violations.issuer_precinct,
    violations.issuer_code,
    violations.issuer_command,
    violations.issuer_squad,
    violations.violation_time,
    violations.violation_county,
    violations.violation_legal_code,
    codes.fee_usd
FROM
    {{ref('silver_parking_violations')}} AS violations
LEFT JOIN
    {{ref('silver_parking_violation_codes')}} AS codes ON
    violations.violation_code = codes.violation_code AND
    violations.is_manhattan_96th_st_below = codes.is_manhattan_96th_st_below
```

### Create `silver_violation_vehicles.sql` table:

The last silver table is `silver_violation_vehicles`. It keeps only the columns related to vehicle information, plus the primary key `summons_number`.

```sql
SELECT
    summons_number,
    registration_state,
    plate_type,
    vehicle_body_type,
    vehicle_make,
    vehicle_expiration_date,
    vehicle_color,
    vehicle_year
FROM
    {{ref('silver_parking_violations')}}
```

### Build silver tables via dbt:

With the silver models in place, run the dbt commands to update the database.
```bash
❯ dbt debug
❯ dbt compile
❯ dbt run

Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Found 8 models, 0 sources, 0 exposures, 0 metrics, 348 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')

1 of 8 START sql view model main.bronze_parking_violation_codes ................ [RUN]
1 of 8 OK created sql view model main.bronze_parking_violation_codes ........... [OK in 0.05s]
2 of 8 START sql view model main.bronze_parking_violations ..................... [RUN]
2 of 8 OK created sql view model main.bronze_parking_violations ................ [OK in 0.02s]
3 of 8 START sql view model main.first_model ................................... [RUN]
3 of 8 OK created sql view model main.first_model .............................. [OK in 0.02s]
4 of 8 START sql view model main.silver_parking_violation_codes ................ [RUN]
4 of 8 OK created sql view model main.silver_parking_violation_codes ........... [OK in 0.02s]
5 of 8 START sql view model main.silver_parking_violations ..................... [RUN]
5 of 8 OK created sql view model main.silver_parking_violations ................ [OK in 0.02s]
6 of 8 START sql view model main.ref_model ..................................... [RUN]
6 of 8 OK created sql view model main.ref_model ................................ [OK in 0.05s]
7 of 8 START sql view model main.silver_violation_tickets ...................... [RUN]
7 of 8 OK created sql view model main.silver_violation_tickets ................. [OK in 0.02s]
8 of 8 START sql view model main.silver_violation_vehicles ..................... [RUN]
8 of 8 OK created sql view model main.silver_violation_vehicles ................ [OK in 0.02s]

Finished running 8 view models in 0 hours 0 minutes and 0.34 seconds (0.34s).

Completed successfully

Done. PASS=8 WARN=0 ERROR=0 SKIP=0 TOTAL=8
```

### View silver tables in database:

The changes are visible in the database:

```python
sql_query = '''
show tables
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())

# output
# ┌────────────────────────────────┐
# │              name              │
# │            varchar             │
# ├────────────────────────────────┤
# │ bronze_parking_violation_codes │
# │ bronze_parking_violations      │
# │ first_model                    │
# │ parking_violation_codes        │
# │ parking_violations_2023        │
# │ ref_model                      │
# │ silver_parking_violation_codes │
# │ silver_parking_violations      │
# │ silver_violation_tickets       │
# │ silver_violation_vehicles      │
# └────────────────────────────────┘
```

```python
sql_query = '''
SELECT * FROM silver_violation_vehicles LIMIT 3
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())
```

Output:

| summons_number | registration_state | plate_type | … | vehicle_year |
|---:|---|---|:-:|---:|
| *int64* | *varchar* | *varchar* | | *int64* |
| 9010912681 | CA | PAS | … | 0 |
| 4858762841 | NY | PAS | … | 2003 |
| 4854645684 | FL | PAS | … | 2022 |

*3 rows, 8 columns (4 shown)*


## Step 12: Medallion Architecture - Gold Data

Gold data holds the metrics and aggregates that downstream consumers use in reports and dashboards.

When this step is done, the database holds:  
**Gold Data**
- gold_ticket_metrics
- gold_vehicles_metrics

Create a directory within the `nyc_parking_violations/models` folder called `gold` and create two files called `gold_ticket_metrics.sql` and `gold_vehicles_metrics.sql`.

```bash
❯ mkdir models/gold
❯ touch models/gold/gold_ticket_metrics.sql
❯ touch models/gold/gold_vehicles_metrics.sql
```

### Create `gold_ticket_metrics.sql` table:

Add the following SQL to `gold_ticket_metrics.sql`. This table shows how many tickets were issued and the total fee revenue the city made for each violation code.

```sql
SELECT
    violation_code,
    COUNT(summons_number) AS ticket_count,
    SUM(fee_usd) AS total_revenue_usd
FROM
    {{ref('silver_violation_tickets')}}
GROUP BY
    violation_code
ORDER BY
    total_revenue_usd DESC
```

### Create `gold_vehicles_metrics.sql` table:

The final model is `gold_vehicles_metrics.sql`. It determines which vehicles get the most violation tickets.

```sql
SELECT
    registration_state,
    COUNT(summons_number) AS ticket_count,
FROM
    {{ref('silver_violation_vehicles')}}
WHERE
    registration_state != 'NY'
GROUP BY
    registration_state
ORDER BY
    ticket_count DESC
```
### Build gold tables via dbt:

With the gold models in place, run the dbt commands to update the database.

```bash
❯ dbt debug
❯ dbt compile
❯ dbt run

Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Found 10 models, 0 sources, 0 exposures, 0 metrics, 348 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')

1 of 10 START sql view model main.bronze_parking_violation_codes ............... [RUN]
1 of 10 OK created sql view model main.bronze_parking_violation_codes .......... [OK in 0.06s]
2 of 10 START sql view model main.bronze_parking_violations .................... [RUN]
2 of 10 OK created sql view model main.bronze_parking_violations ............... [OK in 0.02s]
3 of 10 START sql view model main.first_model .................................. [RUN]
3 of 10 OK created sql view model main.first_model ............................. [OK in 0.02s]
4 of 10 START sql view model main.silver_parking_violation_codes ............... [RUN]
4 of 10 OK created sql view model main.silver_parking_violation_codes .......... [OK in 0.02s]
5 of 10 START sql view model main.silver_parking_violations .................... [RUN]
5 of 10 OK created sql view model main.silver_parking_violations ............... [OK in 0.02s]
6 of 10 START sql view model main.ref_model .................................... [RUN]
6 of 10 OK created sql view model main.ref_model ............................... [OK in 0.05s]
7 of 10 START sql view model main.silver_violation_tickets ..................... [RUN]
7 of 10 OK created sql view model main.silver_violation_tickets ................ [OK in 0.02s]
8 of 10 START sql view model main.silver_violation_vehicles .................... [RUN]
8 of 10 OK created sql view model main.silver_violation_vehicles ............... [OK in 0.02s]
9 of 10 START sql view model main.gold_ticket_metrics .......................... [RUN]
9 of 10 OK created sql view model main.gold_ticket_metrics ..................... [OK in 0.02s]
10 of 10 START sql view model main.gold_vehicles_metrics ....................... [RUN]
10 of 10 OK created sql view model main.gold_vehicles_metrics .................. [OK in 0.02s]

Finished running 10 view models in 0 hours 0 minutes and 0.36 seconds (0.36s).

Completed successfully

Done. PASS=10 WARN=0 ERROR=0 SKIP=0 TOTAL=10
```
### View gold tables in database:

The changes are visible in the database:

```python
sql_query = '''
show tables
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())

# output
# ┌────────────────────────────────┐
# │              name              │
# │            varchar             │
# ├────────────────────────────────┤
# │ bronze_parking_violation_codes │
# │ bronze_parking_violations      │
# │ first_model                    │
# │ gold_ticket_metrics            │
# │ gold_vehicles_metrics          │
# │ parking_violation_codes        │
# │ parking_violations_2023        │
# │ ref_model                      │
# │ silver_parking_violation_codes │
# │ silver_parking_violations      │
# │ silver_violation_tickets       │
# │ silver_violation_vehicles      │
# └────────────────────────────────┘
```

```python
sql_query = '''
SELECT * FROM gold_vehicles_metrics LIMIT 3
'''

with duckdb.connect('data/nyc_parking_violations.db') as con:
    display(con.sql(sql_query).df())

# output
# ┌────────────────────┬──────────────┐
# │ registration_state │ ticket_count │
# │      varchar       │    int64     │
# ├────────────────────┼──────────────┤
# │ NJ                 │         9258 │
# │ PA                 │         3514 │
# │ FL                 │         2414 │
# └────────────────────┴──────────────┘
```

Note that `BLANKPLATE` appears as a `plate_id`, and at a substantially higher count than the rest. On a real project this needs a decision: does the business want these values present, or is it a data quality issue?

## Step 13: Materialization of dbt Models

With all the models in place, the next question is how they are materialized for data consumers in the database. dbt offers the following materialization methods:
- table
- view
- incremental
- ephemeral
- materialized view

The [materialization documentation](https://docs.getdbt.com/docs/build/materializations) from dbt is worth reading in full.

### Materialization in our dbt project:
This project uses only the `table`, `view`, and `ephemeral` methods. The tradeoffs between them come down to scalability, navigation of the database, and security.

The `bronze` tables are `view` materializations: they aren't referenced often, so users can wait for the underlying query. The `gold` tables are `table` materializations, since dashboards and reports need the data ready without waiting for queries to run. The `silver` tables are more nuanced.

Only `silver_violation_tickets` and `silver_violation_vehicles` are meant for data consumers, since they hold the final business logic, so they are materialized as a `view`. `silver_parking_violation_codes` and `silver_parking_violations` are intermediary steps that consumers shouldn't touch, so they get the `ephemeral` materialization, which runs the model during `dbt run` without loading the result into the database.

The `example` models also become `ephemeral`, as they are no longer needed.

### Implementing materialization in our `dbt_project.yml` file:

This is configured in `dbt_project.yml`. The lines to change currently look like this:

```yaml
models:
  nyc_parking_violations:
    example:
      +materialized: view
```

They need to become:

```yaml
models:
  nyc_parking_violations:
    # Config indicated by + and applies to all files under models/example/
    example:
      +materialized: ephemeral
    bronze:
      +materialized: view
    silver:
      silver_parking_violation_codes:
        +materialized: ephemeral
      silver_parking_violations:
        +materialized: ephemeral
      silver_violation_tickets:
        +materialized: view
      silver_violation_vehicles:
        +materialized: view
    gold:
      +materialized: table
```

When an entire directory shares one materialization, the `+materialized:` notation applies to every file under it. The `silver` models need each materialization listed explicitly.

`dbt run` now outputs:

```bash
❯ dbt run
Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Unable to do partial parsing because a project config has changed
Found 10 models, 0 sources, 0 exposures, 0 metrics, 348 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')

1 of 6 START sql view model main.bronze_parking_violation_codes ................ [RUN]
1 of 6 OK created sql view model main.bronze_parking_violation_codes ........... [OK in 0.05s]
2 of 6 START sql view model main.bronze_parking_violations ..................... [RUN]
2 of 6 OK created sql view model main.bronze_parking_violations ................ [OK in 0.02s]
3 of 6 START sql view model main.silver_violation_tickets ...................... [RUN]
3 of 6 OK created sql view model main.silver_violation_tickets ................. [OK in 0.03s]
4 of 6 START sql view model main.silver_violation_vehicles ..................... [RUN]
4 of 6 OK created sql view model main.silver_violation_vehicles ................ [OK in 0.02s]
5 of 6 START sql table model main.gold_ticket_metrics .......................... [RUN]
5 of 6 OK created sql table model main.gold_ticket_metrics ..................... [OK in 0.04s]
6 of 6 START sql table model main.gold_vehicles_metrics ........................ [RUN]
6 of 6 OK created sql table model main.gold_vehicles_metrics ................... [OK in 0.06s]

Finished running 4 view models, 2 table models in 0 hours 0 minutes and 0.31 seconds (0.31s).

Completed successfully

Done. PASS=6 WARN=0 ERROR=0 SKIP=0 TOTAL=6
```

In step 12 the `dbt run` output showed 10 models; with the updated materialization it shows 6, because the `ephemeral` models never land in the database.

## Step 14: Documentation as Code via dbt

Step 8 used the dbt docs commands to generate a documentation website.

Running them again now gives:

```bash
❯ dbt docs generate
❯ dbt docs serve
```

The project now matches the medallion architecture it aimed for.

The output is a directed acyclic graph (DAG), which traces the lineage of the data models. This is what `{{ref()}}` buys: the documentation is generated automatically, and dbt handles the orchestration behind it. It matters little for a project this small, but it is a major differentiator for production projects with hundreds of models and complex relationships.

### Further documentation via dbt:
The DAG is useful, but a lot of information about the data inside the models is still missing.

For the project to be production ready, we add documentation as code through the `schema.yml` and `docs_blocks.md` files in the models folder.

### The `schema.yml` file:

`schema.yml` is the configuration file that establishes documentation and tests for the project. It needs to follow the format below for dbt to pick it up and generate documentation.

```yaml
models:
  # model 1
  - name: <name of dbt model>
    description: <description of dbt model>
    columns:
      # column 1
      - name: <name of the column within the parent dbt model>
        description: <description of the column>
      # column 2
      - name: <name of the column within the parent dbt model>
        description: <description of the column>
      # column n
      - name: <name of the column within the parent dbt model>
        description: <description of the column>
  # model 2
  - name: <name of dbt model>
    description: <description of dbt model>
    columns:
      - name: <name of the column within the parent dbt model>
        description: <description of the column>
  # model n
  - name: <name of dbt model>
    description: <description of dbt model>
    columns:
      - name: <name of the column within the parent dbt model>
        description: <description of the column>
```

Create a folder called `/models/docs`, add the `schema.yml` file to it, and document the `bronze_parking_violation_codes` model.

```bash
❯ mkdir models/docs
❯ touch models/docs/schema.yml
```

```yaml
# models/docs/schema.yml
models:
  - name: bronze_parking_violation_codes
    description: Raw data representing the violation codes and their fees.
    columns:
      - name: violation_code
        description: The standardized code of the violation.
      - name: definition
        description: Description of the violation for a respective code.
      - name: manhattan_96th_st_below
        description: The fee in $USD for a violation on or below Manhattan 96th Street.
      - name: all_other_areas
        description: The fee in $USD for a violation not on or below Manhattan 96th Street.
```

Save `schema.yml` and generate the docs again to see that model's documentation.

```bash
❯ dbt docs generate
❯ dbt docs serve
```

### The `docs_blocks.md` file:

Filling in `schema.yml` by hand for every model and column creates a problem: the same columns, such as `violation_code`, appear across multiple models. Documenting each one manually violates the DRY principle (don't repeat yourself) and leaves multiple sources of information that will diverge over time.

The fix is another "jinja" function called `Docs Blocks`. It needs a markdown file (`.md`) using `{% docs <docs blocks name> %} {% enddocs %}`. The file can be called anything; `docs_blocks.md` is the explicit choice. `Docs Blocks` are written like this:

```md
{% docs <docs blocks name> %}
<documentation>
{% enddocs %}
```

This is similar to assigning a string to a variable that the project can access:

```txt
# docs blocks
{% docs example_name %}
This is example text.
{% enddocs %}

# python
example_name = 'This is example text.'
```

To call that "variable", use `'{{ doc("<docs blocks name>") }}'` in `schema.yml`.

The documentation is now reusable and stays in line with the DRY principle.

### Implementing `Docs Blocks`:

This step covers three things:
1. Create the `docs_blocks.md` file within the `/models/docs` folder.
2. Create a `Docs Block` named `violation_code` within `docs_blocks.md`.
3. Assign the `Docs Block` to the `violation_code` column within our current `schema.yml` file.

**Create the `docs_blocks.md` file within the `/models/docs` folder.**

```bash
❯ touch models/docs/docs_blocks.md
```

**Create a `Docs Block` named `violation_code` within `docs_blocks.md`.**

```txt
# models/docs/docs_blocks.md

{% docs violation_code %}
The standardized code of the violation.
{% enddocs %}
```

**Assign the `Docs Block` to the `violation_code` column within our current `schema.yml` file.**

```yaml
# models/docs/schema.yml
models:
  - name: bronze_parking_violation_codes
    description: Raw data representing the violation codes and their fees.
    columns:
      - name: violation_code
        description: '{{ doc("violation_code") }}'
      - name: definition
        description: Description of the violation for a respective code.
      - name: manhattan_96th_st_below
        description: The fee in $USD for a violation on or below Manhattan 96th Street.
      - name: all_other_areas
        description: The fee in $USD for a violation not on or below Manhattan 96th Street.
```

Generating the docs again shows the same documentation as before, now backed by `Docs Blocks`.

```bash
❯ dbt docs generate
❯ dbt docs serve
```

### Complete dbt documentation:

Documenting every model and column by hand is tedious. The complete `schema.yml` and `docs_blocks.md` files live in `/models/docs` and are worth reading in full.

Generating the docs again shows an entirely documented dbt project.

```bash
❯ dbt docs generate
❯ dbt docs serve
```

## Step 15: Implementing Tests Within the dbt Project

dbt implements tests that run as part of the pipeline to ensure data quality. There are three types:
1. Singular tests
2. Generic tests
3. Out-of-box generic tests

This project uses a singular test, the `unique` and `not_null` out-of-box tests, and a custom generic test.

### Creating custom singular tests

Singular tests are SQL statements that fail if any rows are returned. Create a file called `violation_codes_revenue.sql`.

```bash
❯ touch test/violation_codes_revenue.sql
```

Add the following SQL to `violation_codes_revenue.sql`. It identifies any violation code where the total fees are less than or equal to $1.

The configuration at the top, `{{ config(severity = 'warn') }}`, makes a failure warn rather than error.

```sql
-- test/violation_codes_revenue.sql
-- Every violation code should have a total fee amount
-- greater than or equal to $1.
{{ config(severity = 'warn') }}

SELECT
    violation_code,
    SUM(fee_usd) AS total_revenue_usd
FROM
    {{ref('silver_parking_violation_codes')}}
GROUP BY
    violation_code
HAVING
    NOT(total_revenue_usd >= 1)
```

### Creating custom generic tests

Generic tests are worth creating once several singular tests start to look alike. Create a new folder within `test` called `generic` and add the file `generic_not_null.sql`.

As with `Docs Blocks`, these are jinja functions passed into `schema.yml`. [dbt's documentation on the topic](https://docs.getdbt.com/guides/best-practices/writing-custom-generic-tests#generic-tests-with-default-config-values) covers more.

This one duplicates the built-in `not_null` test, kept minimal on purpose.

```bash
❯ touch test/generic
❯ touch test/generic/generic_not_null.sql
```

```sql
-- test/generic/generic_not_null.sql
-- source: https://docs.getdbt.com/guides/best-practices/writing-custom-generic-tests#generic-tests-with-default-config-values
{% test generic_not_null(model, column_name) %}

    select *
    from {{ model }}
    where {{ column_name }} is null

{% endtest %}
```

### Implementing tests within the `schema.yml` file

Tests are applied to specific models and columns in the `schema.yml` file for that model. dbt ships with several built-in generic tests, including `unique`, `not_null`, `accepted_values`, and `relationships`. We use `unique` and `not_null`, and add the custom `generic_not_null`. [dbt's documentation](https://docs.getdbt.com/docs/build/tests) covers tests in more depth.

Here is how the tests are defined for the `bronze_parking_violations` model:

```yaml
  - name: bronze_parking_violations 
    description: Raw data related to parking violations in 2023, encompassing various details about each violation.
    columns:
      - name: summons_number
        description: '{{ doc("summons_number") }}'
        tests:
          - unique
          - not_null
          - generic_not_null
```

### Storing test failures

dbt can also store test failures in the database for further analysis, configured in `dbt_project.yml` with the `store_failures` option. Set to `true`, dbt creates a separate table for each failing test, holding the rows that caused the failure.

```yaml
# dbt_project.yml
tests:
  +store_failures: true
```

### Using the `dbt test` CLI command

The `dbt test` command runs every test in the project and prints a summary of the results.

```bash
❯ dbt test

Running with dbt=1.6.1
Registered adapter: duckdb=1.6.0
Unable to do partial parsing because config vars, config profile, or config target have changed
Unable to do partial parsing because profile has changed
Found 10 models, 4 tests, 0 sources, 0 exposures, 0 metrics, 349 macros, 0 groups, 0 semantic models

Concurrency: 1 threads (target='dev')

1 of 4 START test generic_not_null_bronze_parking_violations_summons_number .... [RUN]
1 of 4 PASS generic_not_null_bronze_parking_violations_summons_number .......... [PASS in 0.14s]
2 of 4 START test not_null_bronze_parking_violations_summons_number ............ [RUN]
2 of 4 PASS not_null_bronze_parking_violations_summons_number .................. [PASS in 0.10s]
3 of 4 START test unique_bronze_parking_violations_summons_number .............. [RUN]
3 of 4 PASS unique_bronze_parking_violations_summons_number .................... [PASS in 0.12s]
4 of 4 START test violation_codes_revenue ...................................... [RUN]
4 of 4 WARN 1 violation_codes_revenue .......................................... [WARN 1 in 0.13s]

Finished running 4 tests in 0 hours 0 minutes and 0.71 seconds (0.71s).

Completed with 1 warning:

Warning in test violation_codes_revenue (tests/violation_codes_revenue.sql)
Got 1 result, configured to warn if != 0

  compiled Code at target/compiled/nyc_parking_violations/tests/violation_codes_revenue.sql

  See test failures:
  ---------------------------------------------------------------------------------------
  select * from "nyc_parking_violations"."main_dbt_test__audit"."violation_codes_revenue"
  ---------------------------------------------------------------------------------------

Done. PASS=3 WARN=1 ERROR=0 SKIP=0 TOTAL=4
```

## Step 16: Deploying the dbt Project to Prod

How a dbt project deploys to production depends heavily on the underlying infrastructure. This project keeps it simple and uses GitHub Actions, which takes three steps:
1. Create a prod DuckDB database.
2. Create a prod dbt profile within `profiles.yml`.
3. Create the configuration file to run our GitHub Actions workflow.

The point of all this is protecting the data. Experimenting with new transformations is good, but the production database needs to stay stable and vetted. Separating `dev` and `prod` is best practice and keeps quality data available in the long run.

### Creating a prod database

Create a new database with the code from the earlier step. The only change is the name, `prod_nyc_parking_violations.db`.

```python
sql_query_import_1 = '''
CREATE OR REPLACE TABLE parking_violation_codes AS
SELECT *
FROM read_csv_auto(
  'data/dof_parking_violation_codes.csv',
  normalize_names=True
  )
'''

sql_query_import_2 = '''
CREATE OR REPLACE TABLE parking_violations_2023 AS
SELECT *
FROM read_csv_auto(
  'data/parking_violations_issued_fiscal_year_2023_sample.csv',
  normalize_names=True
  )
'''

with duckdb.connect('data/prod_nyc_parking_violations.db') as con:
  con.sql(sql_query_import_1)
  con.sql(sql_query_import_2)
```

### Creating a prod dbt profile

The dbt configuration now needs to support multiple profiles. The default stays `dev`, and `dbt run --target prod` updates the new production database.

```yaml
# profiles.yml
nyc_parking_violations:
  outputs:
   dev:
     type: duckdb
     path: '../data/nyc_parking_violations.db'
   prod:
     type: duckdb
     # note that path is slightly different as GitHub actions
     # start in the root directory and not in the
     # nyc_parking_violations directory
     path: './data/prod_nyc_parking_violations.db'  
  target: dev
```

### Deploying with GitHub Actions

GitHub provides templates for Python actions to build from. This workflow has the following steps:
1. On branch push or pull, run the workflow (runs can also be scheduled via cron jobs).
2. Create the environment variables `DBT_PROFILES_DIR` and `DBT_PROJECT_DIR`.
3. Build the environment via ubuntu (linux) to run bash.
4. Setup python environment.
5. Install pip and install dependencies based on `requirements.txt` file.
6. Run dbt debug, compile, and run prod
7. Run dbt test in prod

The [GitHub Actions documentation](https://docs.github.com/en/actions) covers the rest.

```yaml
# .github/workflows/run-dbt-prod.yml
name: run_dbt_prod

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  # schedule:
  #   - cron: '0 8 * * *'

env:
  DBT_PROFILES_DIR: ./nyc_parking_violations
  DBT_PROJECT_DIR: ./nyc_parking_violations

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up Python 3.10
      uses: actions/setup-python@v3
      with:
        python-version: "3.10"
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
    - name: Run dbt Prod
      run: |
        dbt debug
        dbt compile --target prod
        dbt run --target prod
    - name: Test dbt Prod
      run: |
        dbt test --target prod
```

Every update pushed to the repo in GitHub triggers the workflow.

# Result
The dbt project is now built, documented, tested, and deployed to production.
