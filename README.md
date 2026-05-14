# Qlik Data Integration Schemas

JSON Schema definitions for Qlik data integration projects and tasks. This repository is the versioned, auditable reference for the schemas used to configure and build data integration pipelines programmatically with Qlik.

> [!NOTE]
> Qlik may add other resource schemas in future. If you're interested only in Qlik Data Integration schemas, load only from the `qtcp` directory.

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

## License

[MIT](LICENSE)
