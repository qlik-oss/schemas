# Qlik Data Integration Schemas

JSON Schema definitions for Qlik data integration projects and tasks. This repository is the versioned, auditable reference for the schemas used to configure and build data integration pipelines programmatically with Qlik.

Schemas are distributed via CDN for direct consumption by tools and pipelines.

## Purpose

These schemas define the structure and validation rules for **QTCP** (Qlik Transformation and Connectivity Platform) project and task configurations. They are intended as a reference for:

- **Customers and partners** building or automating data integration projects via API or tooling
- **Tool authors** who need schemas for validation, autocompletion, or code generation
- **Anyone auditing** how schema definitions change over time

For the full API and configuration specifications for Qlik data integration, see the [Qlik Developer portal](https://qlik.dev).

## Schema Overview

All schemas live in [`qtcp/`](qtcp/) and conform to [JSON Schema draft-07](https://json-schema.org/draft-07/json-schema-release-notes.html).

### Project and Task Configuration

| Schema | Description |
|--------|-------------|
| `project.schema.json` | Top-level project configuration (Data Movement or Data Pipeline) |
| `task.schema.json` | Task configuration with type-discriminated routing to settings schemas |
| `task.schedule.schema.json` | Task scheduling — time-based (RFC-5545 RRULE), event-based, data-event-based |
| `task.dataset.schema.json` | Dataset definitions within tasks, including cross-project references |
| `task.model.schema.json` | Data model entity relationships |
| `task.sourceselection.schema.json` | Source table and view pattern selection |
| `task.transformationrules.schema.json` | Declarative transformation rules (rename, add/drop columns, type changes, value replacement) |
| `task.transformationdataflow.schema.json` | Transformation flow definitions |

### Task Settings

Each task type has a corresponding `task.settings.{type}.schema.json`. Shared definitions reused across types are consolidated in `task.settings.common.schema.json`.

| Task Type | Description |
|-----------|-------------|
| `landing` | Land raw data from source systems |
| `lakelanding` | Land data into a data lake |
| `streaminglakelanding` | Streaming data lake landing |
| `storage` | Store processed data |
| `qvdstorage` | QVD-format data storage |
| `lakehousestorage` | Lakehouse storage (Iceberg) |
| `lakehousemirror` | Lakehouse mirroring |
| `replication` | Data replication |
| `transform` | Batch data transformation |
| `streamingtransform` | Streaming data transformation |
| `datamart` | Data mart creation |
| `knowledgemart` | Knowledge mart for AI/LLM workloads |
| `filebasedknowledgemart` | File-based knowledge mart |
| `registereddata` | Registered data assets |

### New Task Defaults

`newtaskdefaults.schema.json` and its type-specific companions (`newtaskdefaults.settings.{type}.schema.json`) define the default values applied when a new task of each type is created.

### Supported Target Platforms

Schemas support configuration targeting:

- Qlik Open Lakehouse
- Snowflake
- BigQuery
- Azure Synapse / Microsoft Fabric
- Databricks
- Amazon Redshift
- SQL Server
- QVD

## License

[MIT](LICENSE)
