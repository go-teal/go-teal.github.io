---
title: "Documentation"
description: "Complete documentation for Teal ETL tool, including configuration, materializations, template functions, and advanced features."
---

## Overview

Teal is a high-performance, scalable open-source ETL tool built on Go, designed to streamline data transformation and orchestration. It combines the best features of tools like [dbt](https://www.getdbt.com/), [Dagster](https://dagster.io/), and [Airflow](https://airflow.apache.org/), while solving common problems found in traditional Python-based solutions.

### Understanding DAGs in Teal

At the core of Teal's execution model is the **Directed Acyclic Graph (DAG)**, a fundamental concept in data pipeline orchestration. A DAG represents your data transformation workflow where:

- **Nodes** are your SQL models (assets) - each representing a data transformation
- **Edges** are dependencies between models - automatically created when one model references another using the `{{ Ref("stage.model_name") }}` function
- **Directed** means dependencies flow in one direction (from source data → staging → transformations → analytics)
- **Acyclic** means no circular dependencies - preventing infinite loops in your pipeline

When you run Teal, it analyzes all your SQL models, builds the dependency graph, and executes them in the correct topological order, ensuring that upstream models complete before downstream models that depend on them.

**Go Concurrency & Performance:**

Teal leverages **Go's concurrency primitives** (goroutines and channels) to maximize parallel execution:
- Each independent asset executes in its own **goroutine** for true parallelism
- **Channels** coordinate dependencies and synchronize execution flow
- Assets at the same DAG level run **concurrently** when dependencies allow
- Optimized for multi-core CPUs to minimize total pipeline execution time

### Assets: The Building Blocks

In Teal, an **asset** is a unit of data transformation or computation. There are two types:

**1. SQL Model Assets**

SQL files that transform data using SELECT statements. Each SQL model automatically becomes a node in the DAG. For example:

```sql
-- This model depends on staging.stg_airports
select
    sha256(airport_code::varchar) as airport_key,
    airport_code,
    airport_name,
    city
from {{ Ref("staging.stg_airports") }}
```

The `{{ Ref() }}` function serves two purposes:
- Declares a dependency (creates an edge in the DAG)
- Gets replaced with the actual table/view name during code generation

**2. Raw Assets**

Custom Go functions for complex operations beyond SQL capabilities (API calls, file processing, custom algorithms). These integrate seamlessly into the DAG alongside SQL models.

### Stage Architecture: Organizing Your Pipeline

Teal allows you to organize your data pipeline into **stages** - logical groupings of models that represent phases in your data transformation workflow. You can define **any number of stages** that fit your architecture in the `profile.yaml` file.

A common pattern is the **three-tier medallion architecture**, but you're free to use as many stages as needed (e.g., `raw`, `staging`, `intermediate`, `dds`, `mart`, `reporting`):

```mermaid
graph LR
    A[Raw Data Sources] --> B[Staging Layer]
    B --> C[DDS Layer]
    C --> D[Mart Layer]

    style B fill:#dcfce7,stroke:#15803d
    style C fill:#dbeafe,stroke:#1e40af
    style D fill:#fef3c7,stroke:#d97706
```

**Example: Three-Tier Pattern**

**Staging Layer** (`staging/`)
- **Purpose**: Raw data ingestion and initial cleaning
- **Operations**: Load CSV files, database tables via connections (e.g., DuckDB's `postgres_scan` or `attach` patterns), API responses, or mount tables to external databases
- **Characteristics**: Minimal transformations, 1:1 with source systems
- **Materialization**: Usually `table` (see [Materializations](#materializations))
- **Note**: Database engines like DuckDB support reading from external databases using `db_link` patterns (e.g., [postgres extension](https://duckdb.org/docs/stable/core_extensions/postgres)) or mounting tables. These patterns require installation of database extensions (see [DuckDB extensions](#duckdb) configuration). To mount a table to an external database, use a `custom` or `raw` SQL asset

**DDS Layer** (`dds/` - Data Distribution Service)
- **Purpose**: Dimensional modeling and business logic
- **Operations**: Create dimensions and facts, apply business rules, add surrogate keys
- **Characteristics**: Normalized structures, referential integrity, warehouse audit columns
- **Materialization**: Usually `table` with indexes and primary keys (see [Materializations](#materializations))

**Mart Layer** (`mart/`)
- **Purpose**: Aggregated analytics and reporting
- **Operations**: Multi-table joins, aggregations, KPI calculations
- **Characteristics**: Denormalized for query performance, business-friendly naming
- **Materialization**: Usually `view` for real-time data or `table` for performance (see [Materializations](#materializations))

Stages are purely organizational - they help you structure your codebase and visualize your pipeline, but Teal's DAG execution is determined solely by `{{ Ref() }}` dependencies, not stage names.

#### Folder Structure and Configuration

To create stages in your Teal project, follow this structure:

```txt
your-project/
├── profile.yaml          # Define your stages here
├── config.yaml           # Database connections
├── assets/
│   ├── models/           # SQL model assets
│   │   ├── staging/      # Stage folder (must match profile.yaml)
│   │   │   ├── stg_flights.sql
│   │   │   └── stg_airports.sql
│   │   ├── dds/          # Stage folder
│   │   │   ├── dim_airports.sql
│   │   │   └── fact_flights.sql
│   │   └── mart/         # Stage folder
│   │       └── mart_flight_performance.sql
│   └── tests/            # Test assets
│       ├── test_data_integrity.sql
│       └── dds/
│           └── test_dim_airports_unique.sql
└── store/                # Data files (CSV, DuckDB, etc.)
    ├── flights.csv
    └── test.duckdb
```

**How to configure stages:**

1. **Define stages in `profile.yaml`**:

```yaml
version: '1.0.0'
name: 'my-project'
connection: 'default'
models:
  stages:
    - name: staging
    - name: dds
    - name: mart
```

2. **Create corresponding folders** under `assets/models/`:
   - Each stage name in `profile.yaml` must have a matching folder
   - Folder names must exactly match the stage names
   - Place your `.sql` files inside these folders

3. **Create SQL model files**:
   - File name becomes the model name (without `.sql` extension)
   - Reference models using `{{ Ref("stage_name.model_name") }}`
   - Example: `{{ Ref("staging.stg_flights") }}` refers to `assets/models/staging/stg_flights.sql`

4. **Tests** (optional):
   - Place test files in `assets/tests/`
   - Can organize tests by stage in subfolders (e.g., `assets/tests/dds/`)
   - Reference in model profiles using `tests:` parameter

### Building a SQL DAG: Complete Example

Let's see how a complete three-tier pipeline works using a flight analytics example:

#### Stage 1: Staging - Data Ingestion

```sql
-- File: assets/models/staging/stg_flights.sql
{{ define "profile.yaml" }}
    materialization: 'table'
    description: 'Flight operations staging - raw CSV ingestion'
{{ end }}

select
    flight_id,
    flight_number,
    route_id,
    aircraft_type,
    scheduled_departure,
    scheduled_arrival,
    actual_departure,
    actual_arrival,
    status
from read_csv('store/flights.csv',
    delim = ',',
    header = true,
    columns = {
        'flight_id': 'INT',
        'flight_number': 'VARCHAR',
        'route_id': 'INT',
        'aircraft_type': 'VARCHAR',
        'scheduled_departure': 'TIMESTAMP',
        'scheduled_arrival': 'TIMESTAMP',
        'actual_departure': 'TIMESTAMP',
        'actual_arrival': 'TIMESTAMP',
        'status': 'VARCHAR'
    }
)
```

This staging model:
- Reads raw CSV data using DuckDB's `read_csv` function (see [Databases](#databases))
- Uses [`materialization: 'table'`](#materializations) to create a persistent table (see [Materializations](#materializations))
- No dependencies yet - it's a source node in the DAG
- Defines typed columns for data quality and schema enforcement

#### Stage 2: DDS - Dimensional Modeling with Incremental Loading

```sql
-- File: assets/models/dds/fact_flights.sql
{{ define "profile.yaml" }}
    materialization: 'incremental'
    is_data_framed: True
    description: 'Flight operations fact table with incremental loading'
    primary_key_fields: ['flight_key']
    indexes:
      - name: 'flight_id_idx'
        unique: true
        fields: ['flight_id']
      - name: 'flight_date_idx'
        fields: ['flight_date']
{{ end }}

with flight_staging as (
    select
        f.*,
        r.route_key,
        r.origin_airport_key,
        r.destination_airport_key,
        r.average_duration_minutes as route_avg_duration
    from {{ Ref("staging.stg_flights") }} f
    inner join {{ Ref("dds.dim_routes") }} r
        on f.route_id = r.route_id
    where f.status = 'COMPLETED'
)
select
    sha256(origin_airport_key || '-' || destination_airport_key || '-' || flight_number) as flight_key,
    flight_id,
    flight_number,
    route_key,
    origin_airport_key,
    destination_airport_key,
    scheduled_departure,
    actual_departure,
    actual_arrival,
    date(scheduled_departure) as flight_date,
    extract(year from scheduled_departure) as flight_year,
    -- Calculate delays in minutes
    extract(epoch from (actual_departure - scheduled_departure)) / 60 as departure_delay_minutes,
    extract(epoch from (actual_arrival - scheduled_arrival)) / 60 as arrival_delay_minutes,
    -- Performance indicators
    case
        when extract(epoch from (actual_arrival - scheduled_arrival)) / 60 <= 15 then true
        else false
    end as on_time_arrival,
    current_timestamp as dw_created_at
from flight_staging
{% if IsIncremental() %}
where actual_arrival > (select coalesce(max(actual_arrival), '1900-01-01'::timestamp) from {{ this() }})
{% endif %}
```

This fact table model demonstrates **incremental loading**:
- **Depends on** `staging.stg_flights` and `dds.dim_routes` via [`{{ Ref() }}`](#template-functions) - creates edges in the DAG
- Uses [`materialization: 'incremental'`](#materializations) to append only new data instead of full refresh (see [Materializations](#materializations))
- **IsIncremental() pattern**: The [`{% if IsIncremental() %}`](#template-functions) block adds a filter on subsequent runs (see [Template Functions](#template-functions)):
  - **First run**: No filter, loads all historical data
  - **Subsequent runs**: Only loads flights with `actual_arrival` newer than the max value already in the table
  - Uses [`{{ this() }}`](#template-functions) to reference the current table for checking the max timestamp
- Adds **surrogate key** using SHA256 hashing for composite business keys
- Calculates **derived metrics** (delays, on-time performance) at load time
- Defines **indexes** on flight_id (unique) and flight_date for query performance
- Uses `is_data_framed: True` for cross-database operations (see [Cross-Database References](#cross-database-references))

**Why Incremental Loading?**
- **Performance**: Only processes new records, not entire dataset every time
- **Efficiency**: Reduces compute time and costs for large fact tables
- **Speed**: Faster DAG execution as data volume grows
- **Use Case**: Perfect for append-only fact tables with timestamp-based filtering

**Testing Fact Tables:**

Data quality is critical for fact tables. Here's an example test for `fact_flights` that validates business rules:

```sql
-- File: assets/tests/test_flight_delays.sql
{{ define "profile.yaml" }}
    connection: 'default'
    description: 'Flight delay anomaly detection - validates realistic operational data'
{{ end }}

-- Test passes if no flights have unrealistic delays
-- Returns rows only when data quality issues are found
select
    flight_id,
    flight_number,
    departure_delay_minutes,
    arrival_delay_minutes,
    case
        when departure_delay_minutes > 1440 then 'Excessive departure delay (>24h)'
        when arrival_delay_minutes > 1440 then 'Excessive arrival delay (>24h)'
        when departure_delay_minutes < -120 then 'Departed too early (>2h)'
        when arrival_delay_minutes < -120 then 'Arrived too early (>2h)'
    end as issue_type
from {{ Ref("dds.fact_flights") }}
where
    departure_delay_minutes > 1440
    or arrival_delay_minutes > 1440
    or departure_delay_minutes < -120
    or arrival_delay_minutes < -120
```

This test:
- **Validates business rules**: Checks that delays fall within realistic operational ranges
- **Uses `{{ Ref() }}`**: Creates a dependency on `fact_flights` in the DAG (test runs after the fact table is built)
- **Returns anomalies**: Query returns rows only when issues are detected (empty result = test passes)
- **Catches data quality issues**: Identifies timezone errors, date rollover problems, or system integration failures
- **Prevents bad analytics**: Stops unrealistic data from skewing KPIs and forecasts

See [Data Testing](#data-testing) for more details on writing and organizing tests.

#### Stage 3: Mart - Analytics & Reporting

```sql
-- File: assets/models/mart/mart_flight_performance.sql
{{ define "profile.yaml" }}
    materialization: 'view'
    is_data_framed: True
    description: 'Route-level operational performance analytics'
{{ end }}

select
    -- Route information
    r.origin_airport,
    origin.airport_name as origin_airport_name,
    r.destination_airport,
    dest.airport_name as destination_airport_name,

    -- Flight metrics
    count(distinct f.flight_id) as total_flights,

    -- Delay metrics
    avg(f.departure_delay_minutes) as avg_departure_delay_minutes,
    avg(f.arrival_delay_minutes) as avg_arrival_delay_minutes,

    -- On-time performance (within 15 minutes)
    sum(case when f.arrival_delay_minutes <= 15 then 1 else 0 end)::float
        / count(*) * 100 as on_time_arrival_pct,

    f.flight_year,
    f.flight_month

from {{ Ref("dds.fact_flights") }} f
join {{ Ref("dds.dim_routes") }} r on f.route_key = r.route_key
join {{ Ref("dds.dim_airports") }} origin on f.origin_airport_key = origin.airport_key
join {{ Ref("dds.dim_airports") }} dest on f.destination_airport_key = dest.airport_key
group by
    r.origin_airport,
    origin.airport_name,
    r.destination_airport,
    dest.airport_name,
    f.flight_year,
    f.flight_month
```

This mart model:
- **Depends on** three DDS models via [`{{ Ref() }}`](#template-functions) - creates multiple edges in the DAG (see [Template Functions](#template-functions))
- Performs **multi-table joins** across dimensions and facts
- Calculates **aggregated KPIs** (on-time percentage, average delays)
- Uses [`materialization: 'view'`](#materializations) for real-time analytics (see [Materializations](#materializations))
- Won't execute until all upstream dependencies complete

### DAG Execution Flow

When you run `teal`, here's what happens:

```mermaid
graph TB
    subgraph "Staging Layer"
        S1[stg_flights]
        S2[stg_airports]
        S3[stg_routes]
    end

    subgraph "DDS Layer"
        D1[dim_airports]
        D2[dim_routes]
        D3[fact_flights]
    end

    subgraph "Mart Layer"
        M1[mart_flight_performance]
    end

    S2 --> D1
    S3 --> D2
    S1 --> D3

    D1 --> M1
    D2 --> M1
    D3 --> M1

    style S1 fill:#dcfce7
    style S2 fill:#dcfce7
    style S3 fill:#dcfce7
    style D1 fill:#dbeafe
    style D2 fill:#dbeafe
    style D3 fill:#dbeafe
    style M1 fill:#fef3c7
```

1. **Staging models** execute first (no dependencies) - can run in parallel
2. **DDS models** execute after their staging dependencies complete - some parallelization possible
3. **Mart models** execute last after all DDS dependencies complete

Teal's DAG engine automatically determines the optimal execution order and maximizes parallelization where possible using Go's concurrency primitives.

## Configuration

### config.yaml

The `config.yaml` file defines your project module and database connections:

```yaml
version: '1.0.0'
module: github.com/my_user/my_test_project
connections:
  - name: default
    type: duckdb
    config:
      path: ./store/test.duckdb
      extensions:
        - postgres
        - httpfs
```

**Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| version | String constant | `1.0.0` |
| module | String | Generated Go module name |
| connections | Array of objects | Array of database connections |
| connections.name | String | Name of the connection used in the model profile |
| connections.type | String | Driver name: `duckdb`, `postgres` |

Teal supports multiple connections. See [Databases](#databases) section for specific configuration parameters.

### profile.yaml

The `profile.yaml` file defines your project structure and model stages:

```yaml
version: '1.0.0'
name: 'my-test-project'
connection: 'default'
models:
  stages:
    - name: staging
      models:
        - name: model1
          tests:
            - name: "root.test_model1_unique"
    - name: dds
    - name: mart
      models:
        - name: custom_asset
          materialization: 'raw'
          connection: 'default'
          raw_upstreams:
            - "dds.model1"
            - "dds.model2"
```

**Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| version | String constant | `1.0.0` |
| name | String | Base name for generated binaries (creates both production and UI versions) |
| connection | String | Default connection from `config.yaml` |
| models.stages | Array | List of stages; folder `assets/models/<stage name>` must exist |

#### Model Profile

Asset profiles can be specified via `profile.yaml` or via a Go template in your SQL model file:

```sql
{{ define "profile.yaml" }}
    connection: 'default'
    description: 'Staging addresses from CSV file'
    materialization: 'table'
    is_data_framed: true
    primary_key_fields:
      - "id"
    indexes:
      - name: "wallet"
        unique: false
        fields:
          - "wallet_id"
{{ end }}

select
    id,
    wallet_id,
    wallet_address,
    currency
from read_csv('store/addresses.csv',
    delim = ',',
    header = true,
    columns = {
        'id': 'INT',
        'wallet_id': 'VARCHAR',
        'wallet_address': 'VARCHAR',
        'currency': 'VARCHAR'}
)
```

**Model Profile Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| name | String | filename | Must match file name (without .sql extension) |
| description | String | | Optional description of the model's purpose |
| connection | String | profile.connection | Connection name from `config.yaml` |
| materialization | String | table | See [Materializations](#materializations) |
| is_data_framed | boolean | false | See [Cross-database references](#cross-database-references) |
| persist_inputs | boolean | false | See [Cross-database references](#cross-database-references) |
| primary_key_fields | Array of string | | List of fields for primary unique index |
| indexes | Array | | List of indexes (table and incremental only) |

## Materializations

Teal supports several materialization types for your SQL models:

| Materialization | Description |
|----------------|-------------|
| **table** | Result stored in table matching model name. If table exists, it's truncated. If not, it's created. |
| **incremental** | Result appended to existing table. If table doesn't exist, it's created. |
| **view** | SQL query saved as a view. |
| **custom** | Custom SQL query executed; no tables or views created. |
| **raw** | Custom Go function executed. |

## Template Functions

Teal uses the **[pongo2](https://github.com/flosch/pongo2) template engine** (v6), which is **Django-compatible**. This means you can use familiar Django/Jinja2 template syntax in your SQL models.

### Static and Dynamic Functions

Teal distinguishes between generation-time and runtime evaluation:

**Static functions** (evaluated during `teal gen` - **double braces** `{{ }}`):

```sql
{{ Ref("staging.model") }}  -- Replaced with actual table name during code generation
```

**Dynamic variables and functions** (evaluated at runtime - **single braces** `{ }`):

```sql
'{ TaskID }' as task_id
'{ TaskUUID }' as task_uuid
'{ InstanceName }' as instance
'{ InstanceUUID }' as instance_uuid
'{ ENV("SOURCE_SCHEMA", "public") }' as schema
```

**Dynamic control structures** (evaluated at runtime - Django/Jinja2 syntax):

```sql
{% if IsIncremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this() }})
{% endif %}
```

### List of Functions

| Name | Input Parameters | Output | When Evaluated | Description | Example |
|------|-----------------|--------|----------------|-------------|---------|
| Ref | `"<stage>.<model>"` | string | Static | Main function for DAG dependencies. Replaced with actual table name during `teal gen`. | `{{ Ref("staging.customers") }}` |
| this | None | string | Both | Returns the name of the current table. | `{{ this() }}` or `{ this() }` |
| ENV | `envName`, `defaultValue` | string | Dynamic | Gets environment variable value at runtime. | `{ ENV("DB_SCHEMA", "public") }` |
| IsIncremental | None | boolean | Dynamic | Returns true if model is in incremental mode. | `{% if IsIncremental() %}...{% endif %}` |
| TaskID | (variable) | string | Dynamic | The task identifier from the Push method. | `{ TaskID }` |
| TaskUUID | (variable) | string | Dynamic | The unique UUID assigned for task tracking. | `{ TaskUUID }` |
| InstanceName | (variable) | string | Dynamic | The DAG instance name. | `{ InstanceName }` |
| InstanceUUID | (variable) | string | Dynamic | The unique UUID assigned to the DAG instance. | `{ InstanceUUID }` |

### Complete Example

```sql
{{ define "profile.yaml" }}
    materialization: 'incremental'
    is_data_framed: true
{{ end }}

SELECT
    order_id,
    customer_id,
    order_date,
    total_amount,
    '{ TaskID }' as etl_task_id,
    '{ TaskUUID }' as etl_run_id,
    current_timestamp as processed_at
FROM {{ Ref("staging.raw_orders") }}
{% if IsIncremental() %}
    WHERE order_date > (SELECT COALESCE(MAX(order_date), '1900-01-01') FROM {{ this() }})
{% endif %}
```

## Databases

### DuckDB

**Configuration Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| connections.type | String | `duckdb` |
| extensions | Array of strings | List of [DuckDB extensions](https://duckdb.org/docs/extensions/overview.html) |
| path | String | Path to the DuckDB database file |
| path_env | String | Environment variable containing the path (overrides `path`) |
| extraParams | Object | Name-value pairs for [DuckDB configuration](https://duckdb.org/docs/configuration/overview.html) |

### PostgreSQL

**Configuration Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| connections.type | String | `postgres` |
| host | String | Hostname or IP address of PostgreSQL server |
| host_env | String | Environment variable name for host |
| port | String | Port number (default: 5432) |
| port_env | String | Environment variable name for port |
| database | String | Database name |
| database_env | String | Environment variable name for database |
| user | String | Username for authentication |
| user_env | String | Environment variable name for user |
| password | String | Password for authentication |
| password_env | String | Environment variable name for password |
| db_root_cert | String | Path to root certificate file for SSL |
| db_root_cert_env | String | Environment variable name for root cert path |
| db_cert | String | Path to client certificate file for SSL |
| db_cert_env | String | Environment variable name for client cert path |
| db_key | String | Path to client key file for SSL |
| db_key_env | String | Environment variable name for client key path |
| db_sslnmode | String | SSL mode: `disable`, `require`, `verify-ca`, `verify-full` |
| db_sslnmode_env | String | Environment variable name for SSL mode |

## Cross-Database References

Cross-database references allow seamless queries across different databases, even with different database drivers.

**Key Parameters:**

- **is_data_framed**: When `true`, query results are saved to a [gota.DataFrame](https://github.com/go-gota/) structure and passed to the next DAG node.
- **persist_inputs**: When `true`, all incoming DataFrames are saved to a temporary table in the database connection configured in the model profile.

**Example Workflow:**

```mermaid
flowchart TB
    subgraph gen["Generation Time - Stage: example"]
        direction LR
        subgraph db1gen["database1.example"]
            model1gen["example.model1.sql"]
        end
        subgraph db2gen["database2.example"]
            model2gen["example.model2.sql"]
        end
        model2gen -.->|"Ref 'example.model1.sql'"| model1gen
    end

    gen ==>|"On Runtime"| runtime

    subgraph runtime["Runtime - Stage: example"]
        direction LR
        subgraph db1run["database1.example"]
            model1run["example.model1.sql"]
        end

        df["gota.DataFrame"]

        subgraph db2run["database2.example"]
            model2run["example.model2.sql"]
            tmp["tmp_example_model1<br/>table"]
        end

        model1run --> df
        df --> tmp
        tmp -.->|"Ref 'tmp_example_model1'"| model2run
    end
```

## Raw Assets

Raw assets are custom functions written in Go that can accept and return dataframes with custom logic.

Raw assets must implement:

```go
type ExecutorFunc func(ctx *TaskContext, modelProfile *configs.ModelProfile) (interface{}, error)
```

**TaskContext provides:**
- `TaskID`: Task identifier
- `TaskUUID`: Unique UUID for tracking
- `InstanceName`: DAG instance name
- `InstanceUUID`: DAG instance UUID
- `Input`: Map of upstream asset results

**Retrieving upstream dataframes:**

```go
df := ctx.Input["dds.model1"].(*dataframe.DataFrame)
```

### Registration and Declaration

Register raw assets in the main function:

```go
processing.GetExecutors().Executors["<stage>.<asset name>"] = yourPackage.YourRawAssetFunction
```

Set upstream dependencies via `raw_upstreams` in the model profile.

## Data Testing

### Simple Model Testing

Tests verify data integrity by executing SQL queries that return row counts. If the count is zero, the test passes.

Tests should be placed in:
- `assets/tests/` - Root tests (stage: `root`)
- `assets/tests/<stage>/` - Stage-specific tests

**Test Naming:** `<stage>.<test_name>`

**Example:**

```sql
{{- define "profile.yaml" }}
    connection: 'default'
{{- end }}

select pk_id, count(pk_id) as c
from {{ Ref "dds.fact_transactions" }}
group by pk_id
having c > 1
```

Root tests are automatically executed after all DAG tasks complete when running with `--with-tests` flag.

#### Test Profile

```yaml
{{ define "profile.yaml" }}
    connection: 'default'
    description: 'Test that ensures airport keys are unique'
{{ end }}
```

**Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| name | String | `<stage>.<filename>` | Test name pattern |
| description | String | | Optional description of what the test validates |
| connection | String | profile.connection | Connection name from `config.yaml` |

## General Architecture

```mermaid
classDiagram
    class Asset {
        <<interface>>
        +Execute(ctx) any, error
        +GetUpstreams() []string
        +GetDownstreams() []string
        +GetName() string
    }

    class SQLModelAsset {
        <<class>>
    }

    class RawAsset {
        <<class>>
    }

    class DBDriver {
        <<interface>>
        +Connect() error
        +Begin() any, error
        +Commit(tx any) error
        +Rollback(tx any) error
        +Close() error
        +Exec(tx any, sql string) error
        +ToDataFrame(sql string) DataFrame, error
        +PersistDataFrame(tx any, name string, df DataFrame) error
        +SimpleTest(sql string) string, error
    }

    class DuckDB {
        <<class>>
    }

    class PostgreSQL {
        <<class>>
    }

    class DAG {
        <<interface>>
        +Run() WaitGroup
        +Push(...)
        +Stop()
    }

    class ChannelDAG {
        <<class>>
    }

    class Executor {
        <<interface>>
        +func(ctx, modelProfile) any, error
    }

    class Routine {
        <<class>>
    }

    Asset <|.. SQLModelAsset : implements
    Asset <|.. RawAsset : implements
    SQLModelAsset o-- DBDriver : uses
    RawAsset o-- Executor : uses
    DBDriver <|.. DuckDB : implements
    DBDriver <|.. PostgreSQL : implements
    DAG <|.. ChannelDAG : implements
    ChannelDAG *-- Routine : contains
    Routine o-- Asset : uses
```

## Understanding the Generated Main Files

Teal generates two entry points for different use cases:

### Production Binary (my-test-project.go)
- Uses **Channel DAG** for high-performance concurrent execution
- Generates unique task names with timestamps
- Optimized for production deployments
- No UI server or debugging overhead

### Debug UI Binary (my-test-project-ui.go)
- Uses **Debug DAG** for visualization and monitoring
- Provides REST API endpoints for DAG control and status
- Includes execution tracking and task history
- Ideal for development and debugging

**How It Works:**

1. `dag.Run()` builds a DAG based on Ref from your .sql models, where each node is an asset and each edge is a Go channel.
2. `dag.Push()` triggers the execution with a unique task name for tracking.
3. `dag.Stop()` sends the deactivation command.

