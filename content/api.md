---
title: "API Reference"
description: "Complete API reference for Teal's Go packages, interfaces, and functions."
---

## Overview

This page provides a comprehensive API reference for Teal's core packages and interfaces. Use this reference when extending Teal with custom functionality or integrating it into your Go applications.

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

## Debug UI REST API

When running with the Debug UI binary, a REST API is available for DAG control and monitoring.

### Base URL

```
http://localhost:8080
```

### Endpoints

#### GET /api/dag

Returns the current DAG structure and status.

**Response:**

```json
{
  "nodes": [
    {
      "name": "staging.model1",
      "upstreams": [],
      "downstreams": ["dds.fact1"],
      "status": "completed"
    }
  ],
  "edges": [
    {
      "from": "staging.model1",
      "to": "dds.fact1"
    }
  ]
}
```

#### POST /api/run

Triggers DAG execution.

**Request Body:**

```json
{
  "taskName": "manual_run_001",
  "inputData": {
    "source": "api",
    "date": "2024-01-01"
  }
}
```

**Response:**

```json
{
  "status": "running",
  "taskName": "manual_run_001",
  "taskUUID": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### GET /api/status

Returns current execution status.

**Response:**

```json
{
  "status": "SUCCESS",
  "taskName": "manual_run_001",
  "startTime": "2024-01-01T10:00:00Z",
  "endTime": "2024-01-01T10:05:00Z",
  "duration": 300
}
```

#### GET /api/tasks

Returns history of task executions.

**Response:**

```json
{
  "tasks": [
    {
      "taskName": "manual_run_001",
      "taskUUID": "550e8400-e29b-41d4-a716-446655440000",
      "status": "SUCCESS",
      "startTime": "2024-01-01T10:00:00Z",
      "endTime": "2024-01-01T10:05:00Z"
    }
  ]
}
```

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

Enable `is_data_framed` only when necessary to minimize memory usage:

```yaml
{{ define "profile.yaml" }}
    is_data_framed: true  # Only if downstream needs DataFrame
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
- [GitHub Repository](https://github.com/go-teal/teal) - Source code and examples
