---
title: "API Reference"
description: "Complete API reference for Teal's Go packages, interfaces, and functions."
---

## Overview

This page provides a comprehensive API reference for Teal's core packages and interfaces. Use this reference when extending Teal with custom functionality or integrating it into your Go applications.

## Template Functions

Teal uses the **[pongo2](https://github.com/flosch/pongo2) template engine** (v6), which is **Django-compatible**. This means you can use familiar Django/Jinja2 template syntax in your SQL models.

### Template Engine Features

- **Django-compatible syntax**: If you know Django templates or Jinja2, you already know pongo2
- **Control structures**: `{% if condition %}...{% endif %}`, `{% for item in items %}...{% endfor %}`
- **Variables and filters**: `{{ variable }}`, `{{ variable|upper }}`, `{{ variable|safe }}`
- **Template inheritance**: `{% extends %}` and `{% block %}` (for advanced use cases)
- **Comments**: `{# This is a comment #}`

### Static and Dynamic Evaluation

Teal template functions are evaluated at different stages:

**Generation-time evaluation** (during `teal gen`):
- `{{ Ref("staging.model") }}` - Replaced with actual table name and establishes DAG dependencies
- `{{ this() }}` - Replaced with current model's table name

**Runtime evaluation** (during DAG execution):
- `{{ TaskID }}` - Current task identifier
- `{{ TaskUUID }}` - Unique task UUID
- `{{ InstanceName }}` - DAG instance name
- `{{ InstanceUUID }}` - DAG instance UUID
- `{{ ENV("VAR_NAME", "default") }}` - Environment variable value
- `{% if IsIncremental() %}...{% endif %}` - Control structures

**Example showing both:**
```sql
-- Generation time: Ref() resolves to "staging.raw_orders"
-- Runtime: TaskID gets actual value during execution
SELECT *
FROM {{ Ref("staging.raw_orders") }}
WHERE task_id = '{{ TaskID }}'
```

**Processing Flow:**
1. **During `teal gen`**: `{{ Ref(...) }}` and `{{ this() }}` are evaluated and replaced with actual table names. All other template syntax is preserved in the generated Go code.
2. **At runtime**: `{{ TaskID }}`, `{{ ENV(...) }}`, and control structures `{% %}` are evaluated when SQL executes during DAG execution.

### List of Functions

| Name | Input Parameters | Output | When Evaluated | Description | Example |
|------|-----------------|--------|----------------|-------------|---------|
| Ref | `"<stage>.<model>"` | string | Generation-time | Main function for DAG dependencies. Replaced with actual table name during `teal gen`. | `{{ Ref("staging.customers") }}` |
| this | None | string | Generation-time | Returns the name of the current table. | `{{ this() }}` |
| ENV | `envName`, `defaultValue` | string | Runtime | Gets environment variable value at runtime. | `{{ ENV("DB_SCHEMA", "public") }}` |
| IsIncremental | None | boolean | Runtime | Returns true if model is in incremental mode. Use in control structures. | `{% if IsIncremental() %}...{% endif %}` |
| TaskID | (variable) | string | Runtime | The task identifier from the Push method. | `{{ TaskID }}` |
| TaskUUID | (variable) | string | Runtime | The unique UUID assigned for task tracking. | `{{ TaskUUID }}` |
| InstanceName | (variable) | string | Runtime | The DAG instance name. | `{{ InstanceName }}` |
| InstanceUUID | (variable) | string | Runtime | The unique UUID assigned to the DAG instance. | `{{ InstanceUUID }}` |

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
    '{{ TaskID }}' as etl_task_id,
    '{{ TaskUUID }}' as etl_run_id,
    current_timestamp as processed_at
FROM {{ Ref("staging.raw_orders") }}  -- Resolved at generation-time
{% if IsIncremental() %}
    WHERE order_date > (SELECT COALESCE(MAX(order_date), '1900-01-01') FROM {{ this() }})
{% endif %}
```

## Core Packages

### pkg/core

The core package provides the singleton instance that manages configuration and global state.

#### GetInstance()

Returns the singleton core instance.

```go
import "github.com/go-teal/teal/pkg/core"

core := core.GetInstance()
```

#### Init(configPath, projectPath string)

Initializes the core instance with configuration.

```go
core.GetInstance().Init("config.yaml", ".")
```

**Parameters:**
- `configPath` - Path to config.yaml file
- `projectPath` - Root path of the project

### pkg/dags

The dags package provides DAG implementations for orchestrating asset execution.

#### DAG Interface

```go
type DAG interface {
    Run() *sync.WaitGroup
    Push(taskName string, inputData map[string]interface{}, done chan map[string]interface{}) chan map[string]interface{}
    Stop()
}
```

**Methods:**

##### Run() *sync.WaitGroup

Builds and starts the DAG execution. Returns a WaitGroup that can be used to wait for completion.

```go
wg := dag.Run()
```

##### Push(taskName, inputData, done) chan

Triggers DAG execution with a specific task name and input data.

```go
result := <-dag.Push("task_123", inputDataMap, make(chan map[string]interface{}))
```

**Parameters:**
- `taskName` - Unique identifier for this execution
- `inputData` - Map of input data to pass to the DAG
- `done` - Channel to signal completion

**Returns:** Channel that will receive the final result

##### Stop()

Signals the DAG to stop execution.

```go
dag.Stop()
```

#### InitChannelDag

Creates a production Channel DAG instance.

```go
func InitChannelDag(
    dagConfig map[string][]string,
    projectAssets map[string]processing.Asset,
    config *configs.Config,
    instanceName string
) DAG
```

**Parameters:**
- `dagConfig` - Map of asset names to their dependencies
- `projectAssets` - Map of asset names to Asset implementations
- `config` - Configuration object
- `instanceName` - Name for this DAG instance

**Example:**

```go
import (
    "github.com/go-teal/teal/pkg/dags"
    "github.com/my_user/my_project/internal/assets"
)

dag := dags.InitChannelDag(
    assets.DAG,
    assets.ProjectAssets,
    config,
    "my-pipeline-instance"
)
```

#### InitChannelDagWithTests

Creates a Channel DAG with integrated testing.

```go
func InitChannelDagWithTests(
    dagConfig map[string][]string,
    projectAssets map[string]processing.Asset,
    projectTests map[string]processing.Asset,
    config *configs.Config,
    instanceName string
) DAG
```

**Parameters:**
- Same as `InitChannelDag`, plus:
- `projectTests` - Map of test names to test Asset implementations

#### InitDebugDag

Creates a Debug DAG instance with REST API and visualization support.

```go
func InitDebugDag(
    dagConfig map[string][]string,
    projectAssets map[string]processing.Asset,
    config *configs.Config,
    instanceName string
) *DebugDag
```

### pkg/processing

The processing package defines interfaces and types for asset execution.

#### Asset Interface

```go
type Asset interface {
    Execute(ctx *TaskContext) (interface{}, error)
    GetUpstreams() []string
    GetDownstreams() []string
    GetName() string
}
```

**Methods:**

##### Execute(ctx *TaskContext) (interface{}, error)

Executes the asset with the given task context.

**Parameters:**
- `ctx` - Task context containing runtime information and input data

**Returns:**
- Result of execution (can be nil, DataFrame, or custom type)
- Error if execution failed

##### GetUpstreams() []string

Returns list of upstream dependency names.

```go
upstreams := asset.GetUpstreams()
// Returns: []string{"staging.model1", "staging.model2"}
```

##### GetDownstreams() []string

Returns list of downstream dependent names.

```go
downstreams := asset.GetDownstreams()
```

##### GetName() string

Returns the asset name.

```go
name := asset.GetName()
// Returns: "dds.fact_transactions"
```

#### TaskContext

Provides runtime information for asset execution.

```go
type TaskContext struct {
    TaskID       string
    TaskUUID     string
    InstanceName string
    InstanceUUID string
    Input        map[string]interface{}
}
```

**Fields:**
- `TaskID` - Task identifier from Push method
- `TaskUUID` - Unique UUID for this task execution
- `InstanceName` - Name of the DAG instance
- `InstanceUUID` - UUID of the DAG instance
- `Input` - Map of upstream results (key: asset name, value: result data)

**Example:**

```go
func MyRawAsset(ctx *processing.TaskContext, modelProfile *configs.ModelProfile) (interface{}, error) {
    // Access task information
    log.Info().Str("taskID", ctx.TaskID).Msg("Executing")

    // Access upstream data
    if upstream, ok := ctx.Input["dds.model1"]; ok {
        df := upstream.(*dataframe.DataFrame)
        // Process dataframe
    }

    return result, nil
}
```

#### ExecutorFunc

Type definition for raw asset functions.

```go
type ExecutorFunc func(ctx *TaskContext, modelProfile *configs.ModelProfile) (interface{}, error)
```

#### GetExecutors()

Returns the global executors registry for raw assets.

```go
import "github.com/go-teal/teal/pkg/processing"

processing.GetExecutors().Executors["staging.my_raw_asset"] = MyRawAssetFunc
```

### pkg/configs

The configs package defines configuration structures.

#### Config

Main configuration structure.

```go
type Config struct {
    Version     string
    Module      string
    Connections []ConnectionConfig
}
```

#### ConnectionConfig

Database connection configuration.

```go
type ConnectionConfig struct {
    Name   string
    Type   string
    Config map[string]interface{}
}
```

#### ModelProfile

Model-specific configuration.

```go
type ModelProfile struct {
    Name              string
    Description       string
    Connection        string
    Materialization   string
    IsDataFramed      bool
    PersistInputs     bool
    PrimaryKeyFields  []string
    Indexes           []Index
}
```

#### Index

Index configuration for models.

```go
type Index struct {
    Name   string
    Unique bool
    Fields []string
}
```

### pkg/drivers

The drivers package provides database driver interfaces and implementations.

#### DBDriver Interface

```go
type DBDriver interface {
    Connect() error
    Begin() (interface{}, error)
    Commit(tx interface{}) error
    Rollback(tx interface{}) error
    Close() error
    Exec(tx interface{}, sql string) error
    GetListOfFields(tx interface{}, tableName string) ([]string, error)
    CheckTableExists(tx interface{}, tableName string) bool
    CheckSchemaExists(tx interface{}, schemaName string) bool
    ToDataFrame(sql string) (*dataframe.DataFrame, error)
    PersistDataFrame(tx interface{}, name string, df *dataframe.DataFrame) error
    SimpleTest(sql string) (string, error)
    GetRawConnection() interface{}
    ConcurrencyLock()
    ConcurrencyUnlock()
}
```

## Command-Line Interface

### teal init

Initializes a new Teal project in the current directory.

```bash
teal init
```

Creates:
- `config.yaml` - Database connections configuration
- `profile.yaml` - Project profile and stages
- `assets/` - Directory for SQL models
- `docs/` - Directory for generated documentation

### teal gen

Generates Go code from SQL models and configuration.

```bash
teal gen [flags]
```

**Flags:**
- `--project-path` - Path to project directory (default: current directory)
- `--config-file` - Path to config.yaml (default: ./config.yaml)

**Generated Files:**
- `cmd/<project-name>/` - Production binary
- `cmd/<project-name>-ui/` - Debug UI binary
- `internal/assets/` - Generated asset implementations
- `internal/model_tests/` - Generated test implementations
- `go.mod` - Go module file
- `Makefile` - Build and run commands
- `docs/README.md` - Project documentation
- `docs/graph.mmd` - Mermaid DAG visualization

### teal ui

Starts the Debug UI server with hot-reload for development.

```bash
teal ui [flags]
```

**Flags:**
- `--port` - Port for API server (default: `8080`). UI Dashboard runs on port + 1 (default: `8081`)
- `--log-output` - Log output format: `json` or `raw` (default: `raw`)
- `--log-level` - Log level: `panic`, `fatal`, `error`, `warn`, `info`, `debug`, `trace` (default: `info`)

**Example:**
```bash
# Start with default ports (API: 8080, UI Dashboard: 8081)
teal ui

# Start with custom ports (API: 9090, UI Dashboard: 9091)
teal ui --port 9090 --log-level debug
```

**Features:**
- Automatic file watching for `assets/`, `profile.yaml`, and `config.yaml`
- Hot-reload on changes (regenerates code and restarts API server)
- Launches both API server and UI Dashboard web application
- Graceful shutdown handling
- Built-in debouncing to prevent excessive regenerations

**Access:**
- **API Server**: `http://localhost:8080` (or custom port)
- **UI Dashboard**: `http://localhost:8081` (API port + 1)

## Custom Asset Development

### Creating a SQL Model Asset

SQL model assets are automatically generated from `.sql` files in `assets/models/<stage>/`.

**Example: assets/models/staging/customers.sql**

```sql
{{ define "profile.yaml" }}
    connection: 'default'
    materialization: 'table'
    description: 'Staging table for customer data'
{{ end }}

SELECT
    id,
    name,
    email,
    created_at
FROM read_csv('data/customers.csv')
```

### Creating a Raw Asset

Raw assets are custom Go functions for complex transformations.

**Step 1: Create the function**

```go
package custom

import (
    "github.com/go-teal/teal/pkg/processing"
    "github.com/go-teal/teal/pkg/configs"
    "github.com/go-gota/gota/dataframe"
)

func ProcessCustomers(ctx *processing.TaskContext, profile *configs.ModelProfile) (interface{}, error) {
    // Get upstream data
    rawData := ctx.Input["staging.raw_customers"].(*dataframe.DataFrame)

    // Process data
    processed := rawData.Filter(
        dataframe.F{Colname: "active", Comparator: "==", Comparando: true},
    )

    // Return dataframe
    return processed, nil
}
```

**Step 2: Register in main.go**

```go
import "github.com/my_user/my_project/pkg/custom"

processing.GetExecutors().Executors["dds.customers_processed"] = custom.ProcessCustomers
```

**Step 3: Configure in profile.yaml**

```yaml
models:
  stages:
    - name: dds
      models:
        - name: customers_processed
          materialization: 'raw'
          raw_upstreams:
            - "staging.raw_customers"
```

## Error Handling

### Asset Execution Errors

When an asset execution fails, the error is propagated through the DAG and logged.

```go
func MyAsset(ctx *processing.TaskContext, profile *configs.ModelProfile) (interface{}, error) {
    if err := validateInput(ctx.Input); err != nil {
        return nil, fmt.Errorf("input validation failed: %w", err)
    }

    // Process...

    return result, nil
}
```

### Test Failures

When tests fail (return non-zero row count), the DAG execution status is set to `TESTS_FAILED`.

## Performance Optimization

### Concurrency Control

Database drivers implement concurrency locks for thread-safe operations:

```go
driver.ConcurrencyLock()
defer driver.ConcurrencyUnlock()

// Perform thread-safe operation
```

### DataFrame Caching

Enable `is_data_framed` to allow downstream models to query the asset's data using SQL. When enabled, the asset's output is cached as a DataFrame, enabling seamless database queries.

```yaml
{{ define "profile.yaml" }}
    is_data_framed: true  # Enable SQL queries on this asset's data
{{ end }}
```

Use `persist_inputs: true` to create temporary tables in the destination database for upstream DataFrames, enabling cross-database queries:

```yaml
{{ define "profile.yaml" }}
    persist_inputs: true  # Creates temporary tables for upstream DataFrames
{{ end }}
```

### Incremental Processing

Use incremental materialization for large datasets:

```sql
{{ define "profile.yaml" }}
    materialization: 'incremental'
{{ end }}

SELECT *
FROM source_table
{% if IsIncremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this() }})
{% endif %}
```

## Best Practices

1. **Use Stages Wisely**: Organize models into logical stages (staging, dds, mart)
2. **Test Everything**: Write tests for data quality checks
3. **Document Models**: Add descriptions to model profiles
4. **Environment Variables**: Use `_env` suffixes for sensitive configuration
5. **Incremental When Possible**: Use incremental materialization for large, append-only datasets
6. **Monitor Performance**: Use Debug UI to visualize and monitor DAG execution
7. **Version Control**: Commit `config.yaml`, `profile.yaml`, and SQL models to git
8. **CI/CD Integration**: Run `teal gen` in CI pipeline to validate models

## Further Reading

- [Quick Start Guide](/quickstart/) - Get started with Teal
- [Full Documentation](/docs/) - Complete feature documentation
- <a href="https://github.com/go-teal/teal" target="_blank" rel="noopener noreferrer">GitHub Repository</a> - Source code and examples
