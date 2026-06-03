# AI Assistant Guide: Data Projects

This guide provides comprehensive instructions for AI assistants to build data projects using YAML configuration files. All workflows, templates, and examples are consolidated here for efficient reference.

**Project Types:**
- **Replication Projects** (`DATA_MOVEMENT`) - Use case: Replication
- **Data Pipeline Projects** (`DATA_PIPELINE`) - Use case: Data pipeline
  - Note: Qlik Open Lakehouse is a logical category but uses `DATA_PIPELINE` as the project type


## General

**Note on Minimal Properties:** Task templates should follow schema requirements. Schemas are published on [SchemaStore](https://www.schemastore.org) and automatically applied by the YAML language server when files match the expected paths (see Schema Reference section below).

**No Comments in Generated Files:**
- ❌ **DO NOT** add comments (`#` in YAML, `//` or `/* */` in JSON) to any generated project files
- YAML and JSON files in this project must contain only data — no inline comments, no block comments, no explanatory annotations

**Bindings Guidance:** Follow the canonical rules in the **Variable Naming Conventions** section below (single source of truth).

---

## Unified Project Creation Workflow

### When User Asks to Create a New Project

**STEP 1: Ask for Project Name**

❓ **ASK:** "What would you like to name this project?"

✅ Once provided, create a root folder with that name. All project files will be created inside this folder.

**STEP 2: Ask for Use Case**

❓ **ASK:** "What is the use case for this project?"

Present exactly these options:
1. Replication
2. Data pipeline

**STEP 3: Set Project Type from Use Case**

Based on the selected use case:
- **Replication** → Create project type `DATA_MOVEMENT`
- **Data pipeline** → Create project type `DATA_PIPELINE`

**STEP 4: For `DATA_PIPELINE` Projects, Determine `platformType`**

- **If the project includes (or the user is simultaneously requesting) a `LAKE_LANDING` task** → automatically set `platformType` to `SNOWFLAKE`. Do not ask the user.
- **Otherwise** → ❓ **ASK:** "Which `platformType` should I use?" and present only the enum values currently defined in the active schema for `properties.platformType`.

❌ **DO NOT:** Ask for `platformType` as unrestricted free text when schema values are known

**STEP 4b: For `SNOWFLAKE` platformType, Determine Landing Target**

When `platformType` is SNOWFLAKE:

- **If the project includes (or the user is simultaneously requesting) a `LAKE_LANDING` task** → automatically treat the landing target as `files in cloud storage`. Do not ask the user.
- **Otherwise** → ❓ **ASK:** "What is the landing target?"
  - `tables in Snowflake` — data lands as tables directly in Snowflake
  - `files in cloud storage` — data lands as files in cloud storage

If the landing target is **`files in cloud storage`** (whether chosen or inferred from a LAKE_LANDING task):
- Add `snowflakeStorageIntegration: '{{snowflakeStorageIntegration}}'` under `properties` in `qtcp_project.yaml`
- Ensure `snowflakeStorageIntegration` is added to `qtcp_bindings_definition.json` under `variables`

If the landing target is **`tables in Snowflake`**: no additional properties are needed.

❌ **DO NOT:** Create `sourceSelection.yaml` unless user has specified data sources
❌ **DO NOT:** Create `schedule.yaml` automatically
❌ **DO NOT:** Create `transformationRules.yaml` automatically
❌ **DO NOT:** Create sample datasets automatically

**STEP 5: Create Project with Appropriate Type**

✅ Create root folder: `<ProjectName>/`
✅ Create minimal `<ProjectName>/qtcp_project.yaml` with the correct project type
✅ Create minimal `<ProjectName>/qtcp_bindings_definition.json` with empty variables: `{ "variables": {} }`
✅ Create empty subfolder `<ProjectName>/qtcp_tasks`

**STEP 6: Suggest Creating a Task**

❓ **ASK:** "Would you like me to create a task now?"

If yes, ask for task type and only present allowed options for that project type:
- For `DATA_MOVEMENT` projects: `REPLICATION`, `LAKE_LANDING`
- For `DATA_PIPELINE` projects: all supported task types except `REPLICATION`
  - `LAKE_LANDING` is allowed in both project types

❌ **DO NOT create `DATAMART` or `KNOWLEDGE_MART` tasks from scratch.** If the user requests either, redirect them to create the task in the QTC UI and commit back.

**Source-matches-platform rule:** When the user wants to ingest data whose source is the **same platform as the project's `platformType`** (e.g., reading from Snowflake in a SNOWFLAKE project), use a `REGISTERED_DATA` task instead of a `LANDING` task. `LANDING` is for external/heterogeneous sources only.

**STEP 7: Ask for Task Name and Create the Task**

❓ **ASK:** "What would you like to name this task?"

✅ Create the task directory: `<ProjectName>/qtcp_tasks/<TaskName>/`
✅ Create minimal `task.yaml` with required fields only

**Task ID Generation Rule (Applies to all task types):**
- Format: `<sanitized-task-name>-<NNNN>` where `NNNN` is exactly 4 digits
- Preferred default: generate `NNNN` as 4 random digits (for example: `4821`)
- Alternative when explicitly requested: use 4-digit sequential numbering (`0001`, `0002`, ...)

**STEP 8: Ask About Additional Configuration**

💬 **After creation, ask:**
- "Would you like me to:
  - Add data sources to this task?
  - Add transformation rules to this task?
  - Add a schedule to this task?" (only if the task type supports scheduling)

---

## Task Type Availability by Project Type

| Project Type | Allowed Task Types |
|---|---|
| DATA_MOVEMENT | REPLICATION, LAKE_LANDING |
| DATA_PIPELINE | All supported task types except REPLICATION (LAKE_LANDING is allowed) |

---

## Project Type Reference

### Data Pipeline Projects and Qlik Open Lakehouse Projects (DATA_PIPELINE)

**Scheduling Rules by Task Type:**

| Task Type | Allowed Scheduling |
|---|---|
| STORAGE, QVD_STORAGE, LAKEHOUSE_STORAGE | `TIME_BASED` only |
| LANDING, LAKE_LANDING, STREAMING_LAKE_LANDING, REGISTERED_DATA, REPLICATION | ❌ No scheduling — do not create `schedule.yaml` |
| TRANSFORM, DATAMART, KNOWLEDGE_MART, FILE_BASED_KNOWLEDGE_MART, LAKEHOUSE_MIRROR, STREAMING_TRANSFORM | `TIME_BASED` or `EVENT_BASED` — ask the user which type |

**Schedule Creation Workflow:**

**STEP 1:** Check the task type against the table above.
- If scheduling is not allowed → inform the user and do not create `schedule.yaml`
- If only `TIME_BASED` is allowed → proceed directly to the time-based flow
- If both are allowed → ❓ **ASK:** "Should this task run on a time-based schedule or be triggered by another task completing (event-based)?"

**STEP 2a: Time-based schedule**
- Ask for free text, for example: "Describe the schedule in plain language (frequency, interval, and start time)."
- Parse and convert that description to RRULE with only `FREQ` and `INTERVAL`
- If required details are missing (for example frequency, interval, or start time), ask concise follow-up questions
- Write `timeBasedScheduling.schedule`, `timeBasedScheduling.startDateTime` (UTC), and `timeBasedScheduling.timezone: Etc/UTC`
- Do not mention RRULE conversion in the prompt to the user

**STEP 2b: Event-based schedule**
- ❓ **ASK:** "Should this task be triggered when any of its input tasks complete, or only when specific tasks complete?"
  - `ANY_INPUT_TASK` — triggered when any input task completes (no `triggerApps` needed)
  - `ANY_SELECTED_TASK` — triggered when specific selected tasks complete
- **IF `ANY_SELECTED_TASK`:** Read the current task's `sourceSelection.yaml` and collect the distinct source task names referenced there (via `sourceTask` in `explicitlySelected` or `includePatterns`). Deduplicate — if multiple entries refer to the same task, count it only once.
  - If there is exactly **one** distinct source task → suggest it to the user: "The only source task is `<task>` — shall I use it?"
  - If there are **multiple** distinct source tasks → ❓ **ASK:** "Which of these source tasks should trigger this schedule?" and list them
  - Populate `triggerApps` with the selected task(s).

**Schedule Modification Workflow:**

When the user asks to **change** an existing `schedule.yaml` (e.g., update timing, change scheduling type, add/remove trigger tasks):
After applying the change, check the `enabled` field on the scheduling entry that was modified:
- If `enabled` is missing or `false` → 💬 **INFORM AND ASK:** "The schedule is currently disabled. Would you like me to enable it as well?"
  - If yes → update `schedule.yaml` and set `enabled: true`
  - If no → leave `enabled` as-is (keep `false` if present, do not add it if absent)

---

**Minimal qtcp_project.yaml**
```yaml
properties:
  type: DATA_PIPELINE
  platformType: <PLATFORM_TYPE>
  platformConnection: '{{platformConnection}}'
  cloudStagingConnection: '{{cloudStagingConnection}}'
settings:
  artifactsNaming:
    prefixSchema: '{{project.current.prefixSchema}}'
```

**Minimal task.yaml per task type:**
All task types require `properties.name`, `properties.id`, and `properties.type`. Most task types also require `settings` with specific fields. Use the templates below as the starting point for each task type.

**Minimal task.yaml (LANDING):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: LANDING
settings:
  landingDwSettings:
    landingArtifactsLocation:
      dataAssetSchema: '{{task.<task-id>.taskSchema}}'
```

**Minimal task.yaml (STORAGE):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: STORAGE
settings:
  artifactsLocation:
    internalSchema: '{{task.<task-id>.internalSchema}}'
    taskSchema: '{{task.<task-id>.taskSchema}}'
    databaseName: '{{task.<task-id>.databaseName}}'
  taskRuntime:
    warehouseSelection:
      warehouseName: '{{task.<task-id>.warehouseName}}'
```

**Minimal task.yaml (TRANSFORM):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: TRANSFORM
settings:
  artifactsLocation:
    internalSchema: '{{task.<task-id>.internalSchema}}'
    taskSchema: '{{task.<task-id>.taskSchema}}'
    databaseName: '{{task.<task-id>.databaseName}}'
  taskRuntime:
    warehouseSelection:
      warehouseName: '{{task.<task-id>.warehouseName}}'
```

**DATAMART:** ❌ **DO NOT create from scratch.** If asked, inform the user: *"DATAMART tasks must be created in the QTC UI. Once created, use 'Commit Changes' to push the YAML to the repo, then pull."*

**Minimal task.yaml (REGISTERED_DATA):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: REGISTERED_DATA
settings:
  artifactsLocation:
    internalSchema: '{{task.<task-id>.internalSchema}}'
    taskSchema: '{{task.<task-id>.taskSchema}}'
    databaseName: '{{task.<task-id>.databaseName}}'
```

**Minimal task.yaml (LAKE_LANDING in DATA_PIPELINE):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: LAKE_LANDING
settings:
  landingDwSettings:
    landingArtifactsLocation:
      dataAssetSchema: '{{task.<task-id>.taskSchema}}'
```

**Minimal task.yaml (LAKEHOUSE_STORAGE):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: LAKEHOUSE_STORAGE
settings:
  artifactsLocation:
    internalSchema: '{{task.<task-id>.internalSchema}}'
    taskSchema: '{{task.<task-id>.taskSchema}}'
  taskRuntime:
    lakehouseCluster: '{{task.<task-id>.lakehouseCluster}}'
```

**Minimal task.yaml (LAKEHOUSE_MIRROR):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: LAKEHOUSE_MIRROR
settings:
  artifactsLocation:
    internalSchema: '{{task.<task-id>.internalSchema}}'
    taskSchema: '{{task.<task-id>.taskSchema}}'
    databaseName: '{{task.<task-id>.databaseName}}'
  platformConfig:
    connection: '{{task.<task-id>.targetConnection}}'
  taskRuntime:
    warehouseSelection:
      warehouseName: '{{task.<task-id>.warehouseName}}'
```

**Minimal task.yaml (STREAMING_LAKE_LANDING):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: STREAMING_LAKE_LANDING
settings:
  taskRuntime:
    lakehouseClusterId: '{{task.<task-id>.lakehouseCluster}}'
```

**Minimal task.yaml (STREAMING_TRANSFORM):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: STREAMING_TRANSFORM
settings:
  generalSettings:
    artifactsLocation:
      internalSchema: '{{task.<task-id>.internalSchema}}'
      taskSchema: '{{task.<task-id>.taskSchema}}'
  runtimeSettings:
    lakehouseClusterId: '{{task.<task-id>.lakehouseCluster}}'
```

**KNOWLEDGE_MART:** ❌ **DO NOT create from scratch.** If asked, inform the user: *"KNOWLEDGE_MART tasks must be created in the QTC UI. Once created, use 'Commit Changes' to push the YAML to the repo, then pull."*

**Minimal task.yaml (FILE_BASED_KNOWLEDGE_MART):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: FILE_BASED_KNOWLEDGE_MART
settings:
  taskRuntime:
    warehouseSelection:
      warehouseName: '{{task.<task-id>.warehouseName}}'
```

**Minimal schedule.yaml (Time-based):**
```yaml
scheduling:
  - schedulingType: TIME_BASED
    enabled: true
    timeBasedScheduling:
      schedule:
        - 'RRULE:FREQ=DAILY;INTERVAL=1'
      startDateTime: '2026-03-18T15:30:44.7460000Z'
      timezone: 'Etc/UTC'
```

**Minimal schedule.yaml (Event-based, ANY_INPUT_TASK):**
```yaml
scheduling:
  - schedulingType: EVENT_BASED
    enabled: true
    eventBasedScheduling:
      eventSchedulingType: ANY_INPUT_TASK
```

**Minimal schedule.yaml (Event-based, ANY_SELECTED_TASK):**
```yaml
scheduling:
  - schedulingType: EVENT_BASED
    enabled: true
    eventBasedScheduling:
      eventSchedulingType: ANY_SELECTED_TASK
      triggerApps:
        - projectId: '{{ref(project.current.projectId)}}'
          dataAppId: <source-task-id>
```

**Important Notes:**
- **Only ONE schedule** allowed per task — if `schedule.yaml` already exists, inform the user
- Always set `enabled: true` when creating new `schedule.yaml`
- Task directory names are customizable — task type is defined in `task.yaml`
- RRULE format: `RRULE:FREQ=<DAILY|HOURLY|WEEKLY|MONTHLY>;INTERVAL=<N>` — only `FREQ` and `INTERVAL` are used

---

### Replication Projects (DATA_MOVEMENT)

**Key Characteristics:**
- Task types: `REPLICATION`, `LAKE_LANDING`
- **Schedules NOT supported** - do not create `schedule.yaml`
- No `settings.artifactsNaming` in project.yaml
- For `REPLICATION` tasks, always set both `fullLoad: true` and `applyChanges: true` under `settings.taskSettings` by default
- Only deviate from this default if the user explicitly specifies different modes; the valid modes are `fullLoad`, `applyChanges`, and `storeChanges`
- Write these mode flags only under `settings.taskSettings` (never directly under `settings`)
- Only write the selected mode flag(s) as `true`; do not add unselected mode flags with `false`

**Minimal qtcp_project.yaml:**
```yaml
properties:
  type: DATA_MOVEMENT
```

**Minimal task.yaml (REPLICATION):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: REPLICATION

settings:
  taskSettings:
    fullLoad: true
    applyChanges: true
  targetEndpoint:
    targetConnection: '{{task.<task-id>.targetConnection}}'
    targetSchema: '{{task.<task-id>.targetSchema}}'
    targetStorageConnection: '{{task.<task-id>.targetStorageConnection}}'
```

**Minimal task.yaml (LAKE_LANDING in DATA_MOVEMENT):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: LAKE_LANDING

settings:
  targetEndpoint:
    targetConnection: '{{task.<task-id>.targetConnection}}'
```

- For `LAKE_LANDING` tasks in a `DATA_MOVEMENT` project, always include `settings.targetEndpoint.targetConnection`
- ❌ **DO NOT** add `settings.targetEndpoint.targetConnection` to `LAKE_LANDING` tasks in a `DATA_PIPELINE` project
- For `LAKE_LANDING` tasks in a `DATA_PIPELINE` project (open lake house platform), use `settings.landingDwSettings.landingArtifactsLocation.dataAssetSchema` — see the template in the Data Pipeline section above

---

## Source Selection Workflow

When the user asks to **"add source"**, **"add table"**, **"add view"**, **"add dataset"**, or provides database/schema/name/type values → **update or create `sourceSelection.yaml` only**.

❌ **DO NOT** create a dataset file (`datasets/*.yaml`) unless the user explicitly asks for dataset-level transformations, **except** when an explicit source is added to a Non-Landing Task — in that case, always create a dataset file for each explicitly selected source.

**Intent Resolution (highest priority, apply before any file creation):**
- If user says: "add source", "add table", "add view", or gives `database/schema/name/type` values: create or update `sourceSelection.yaml`.
- If user says only "add dataset": treat it as source-selection intent by default and update `sourceSelection.yaml`.
- Create `datasets/*.yaml` only when user explicitly says "create dataset file", "create dataset yaml", or asks for dataset-level transformation modeling.
- If ambiguity remains, ask one concise clarification and do not create files until clarified.

**STEP 1: Determine Target Task**
- ❓ **IF task not specified:** "Which task should this source be added to?" (List available tasks from `qtcp_tasks/` folder)
- ✅ **IF task is known:** Proceed to next step

**STEP 2: Identify task type and follow the correct workflow**
- **Mandatory task-type gate (MUST):** Read the target task's `task.yaml` and branch strictly by `properties.type` before creating or updating `sourceSelection.yaml`.
- **Do not cross-apply templates (MUST NOT):** Never apply Landing or Non-Landing sourceSelection templates to `REGISTERED_DATA` tasks.

There are three distinct workflows depending on the task type:

---

### Landing Tasks (LANDING, LAKE_LANDING, STREAMING_LAKE_LANDING, REPLICATION)

Landing tasks read **directly from a data source connection**. The user must provide:
1. **Data source** (the connection)
2. **Schema** and **table names**

**Prompting:**
- ❓ **ASK:** "Please provide the table names and schemas you want to include"
- ❓ **IF schema not specified:** "What is the schema for this table?"

**Add entries to the task's `sourceSelection.yaml`.**
- Refer to the `sourceSelection` schema for required and optional properties in `explicitlySelected` items
- Landing tasks use `sourceConnection` at the top level

**Minimal sourceSelection.yaml (landing tasks):**
```yaml
sourceConnection: '{{task.<task-id>.sourceConnection}}'
explicitlySelected:
  - name: <TableName>
    schema: '{{task.<task-id>:<database>$_$<schema>.schema}}'
    type: TABLE
```

#### Using Include/Exclude Patterns (Optional, Landing Tasks Only)

For bulk table selection, you can use patterns instead of explicitly listing each table.
Pattern syntax: `%` matches any characters; exact strings match literally. Case sensitivity depends on the source database.

```yaml
sourceConnection: '{{task.<task-id>.sourceConnection}}'
includePatterns:
  - tablePattern: 'dim_%'
    schemaPattern: 'dbo'
    type: TABLE
  - tablePattern: 'fact_%'
    schemaPattern: 'dbo'
    type: TABLE
excludePatterns:
  - tablePattern: 'temp_%'
    schemaPattern: 'dbo'
    type: TABLE
```

---

### Non-Landing Tasks (STORAGE, QVD_STORAGE, TRANSFORM, DATAMART, KNOWLEDGE_MART, FILE_BASED_KNOWLEDGE_MART, LAKEHOUSE_STORAGE, LAKEHOUSE_MIRROR, STREAMING_TRANSFORM)

Non-landing tasks read **from an upstream task**, not directly from a data source connection. The user must provide:
1. **Source task** (the upstream task that provides the data)
2. **Which datasets/tables** from that task to include

**Prompting:**
- ❓ **ASK:** "Which task should this read from?" (List available tasks from `qtcp_tasks/` folder)
- ❓ **ASK:** "Which tables/datasets from that task do you want to include?"

**Add entries to the task's `sourceSelection.yaml`.**
- Non-landing tasks use `sourceTask` on each `explicitlySelected` item to reference the upstream task
- ❌ **DO NOT** include `sourceConnection` — non-landing tasks do not use a direct data source connection
- Refer to the `sourceSelection` schema for required and optional properties in `explicitlySelected` items

**`sourceTask` value format:**
- Always use the plain task ID string (e.g., `storage1-3742`)
- ❌ DO NOT use `{{ref(...)}}` or any template variable syntax for `sourceTask`

**Minimal sourceSelection.yaml (non-landing tasks, explicitly selected tables):**
```yaml
explicitlySelected:
  - name: <upstream-dataset-name>
    sourceTask: <upstream-task-id>
    type: TABLE
    sourceTableId: <upstream-dataset-id>
```

**Minimal sourceSelection.yaml (non-landing tasks, all tables via pattern):**
```yaml
includePatterns:
  - tablePattern: '%'
    sourceTask: <upstream-task-id>
    type: TABLE
```

**When creating dataset YAML files** for non-landing tasks, use `properties.inputDatasets` to reference the upstream task and dataset.
- `taskId` MUST be the upstream task's `properties.id`
- `datasetId` MUST be the upstream dataset's `properties.id` from a dataset file under that same upstream task — **if no dataset file exists for the upstream task (e.g., a LANDING task), use the dataset name as the `datasetId` instead. Never reference a `datasetId` that does not exist.**
- `name` MUST be provided as the reference name for the input dataset within this file
- Refer to the `dataset` schema (`task.dataset.schema.json`) for the required properties per task type — the schema describes which fields are mandatory based on the dataset `type`
- ❌ **DO NOT** hardcode property lists — always consult the schema for current requirements

**TRANSFORM tasks require an explicit `datasets/<TableName>.yaml` for every output table.** There are three patterns:

**Pattern 1: Passthrough (simple copy)**
```yaml
properties:
  name: <DatasetName>
  id: <dataset-id>
  inputDatasets:
    - taskId: <upstream-task-id>
      datasetId: <upstream-dataset-id>
      name: <upstream-dataset-name>
mappings:
  mappings: []
```

**Pattern 2: Custom SQL** — use for JOINs, aggregations, or calculated columns. Requires an `alias` array mapping SQL placeholders to dataset references.
```yaml
properties:
  id: customerorders-ab12
  name: customerOrders
  inputDatasets:
    - datasetId: customers--4w3
      name: customers
      taskId: onboarding_storage--4w0
    - datasetId: orders--4w6
      name: orders
      taskId: onboarding_storage--4w0
   customDatasetSettings:
    customSql:
      expressionStatement: "SELECT c.[customer_id], ...\nFROM ${customers} AS c\nINNER JOIN ${orders} AS o ON c.[customer_id] = o.[customer_id]"
      alias:
        - name: customers
          value: '{{ref(project.current.projectId)}}$_$onboarding_storage--4w0$_$customers--4w3'
        - name: orders
          value: '{{ref(project.current.projectId)}}$_$onboarding_storage--4w0$_$orders--4w6'
      incremental: false
```
Key rules:
- SQL uses `${alias}` placeholders — alias must match `inputDatasets[].name` AND `alias[].name`
- `alias[].value` format: `{{ref(project.current.projectId)}}$_$<taskId>$_$<datasetId>`

**Pattern 3: Data Flow** — visual canvas JOINs. ❌ **DO NOT generate or edit Pattern 3 files.** The graph JSON contains UUID cross-references that are error-prone to hand-author. If the user asks for a data flow dataset, inform them: *"Data flow datasets must be created in the QTC UI."*

> **Recommendation for TRANSFORM datasets generally:** Configure output datasets in QTC UI, use "Commit Changes" to push generated YAML back, then pull. This avoids ID mismatches — QTC generates IDs like `customers-xah-` that must be referenced exactly in downstream tasks.

**Minimal dataset.yaml (non-landing, non-TRANSFORM tasks):**
```yaml
properties:
  name: <DatasetName>
  id: <dataset-id>
  inputDatasets:
    - taskId: <upstream-task-id>
      datasetId: <upstream-dataset-id>
      name: <upstream-dataset-name>
```

---

### Registered Data Tasks (REGISTERED_DATA)

`REGISTERED_DATA` tasks register **pre-existing data assets** by specifying their exact location (database, schema, name). They do not read from a live connection or from an upstream task. The user must provide:
1. **Database name**
2. **Schema**
3. **Table name**

**Prompting:**
- ❓ **ASK:** "Please provide the database, schema, and table names you want to register"
- ❓ **IF database not specified:** "What is the database for these tables?"
- ❓ **IF schema not specified:** "What is the schema for this table?"
- ❓ **IF table/view name not specified:** "What are the table or view names to register?"
- ❓ **IF type not specified:** "What is the source type: table or view?"
- **If any required location value is missing (`database`, `schema`, `name`, `type`), ask follow-up questions and do not write `sourceSelection.yaml` yet.**

**Add entries to the task's `sourceSelection.yaml`.**
- ❌ **DO NOT** include `sourceConnection` — `REGISTERED_DATA` tasks do not use a data source connection
- ❌ **DO NOT** include `sourceTask` — `REGISTERED_DATA` tasks do not read from an upstream task
- ❌ **DO NOT** use `includePatterns` / wildcard patterns as a fallback for `REGISTERED_DATA` requests such as "add source"
- Each `explicitlySelected` item must specify `database`, `schema`, `name`, and `type`
- If the request uses the word "dataset" without explicitly saying "dataset file"/"dataset yaml", still treat it as source-selection intent and update `sourceSelection.yaml`.

**Completion check (REGISTERED_DATA, MUST pass before final response):**
1. `sourceSelection.yaml` contains no `sourceConnection`.
2. `sourceSelection.yaml` contains no `sourceTask`.
3. Every `explicitlySelected` item includes `database`, `schema`, `name`, and `type`.
4. No wildcard fallback was used to bypass missing required values.

**Minimal sourceSelection.yaml (REGISTERED_DATA tasks):**
```yaml
explicitlySelected:
  - database: <DatabaseName>
    schema: <SchemaName>
    name: <TableName>
    type: TABLE
```

---

## Variable Naming Conventions

When working with `qtcp_bindings_definition.json`, follow these conventions:

- `task.<task-id>.<property>` - Task-specific properties (e.g., `task.landing_cdc-0001.sourceConnection`)
- `task.<task-id>:<database>$_$<schema>.<property>` - Schema-specific properties, where `<database>` is the database name or `null` if not specified (e.g., `task.landing_cdc-0001:null$_$dbo.schema`)
- `task-type.<type>.<property>` - Default values for task types (e.g., `task-type.landing.databaseName`)
- `project.current.<property>` - Project-level references (e.g., `project.current.projectId`)

**Task-type-defaulted properties:**

The following properties use a two-level binding when added to a task:

- `warehouseName`
- `databaseName`
- `databricksVectorSearchEndpoint`
- `indexDatabase`
- `vectorDbTargetType`
- `vectorDbConnection`
- `llmConnection`
- `databaseSelectionMethod`
- `warehouseSelectionMethod`
- `indexDatabaseSelectionMethod`
- `lakehouseCluster`
- `folder`
- `snowflakeExternalVolume`
- `snowflakeOpenCatalog`

For these properties, instead of adding a blank value in `qtcp_bindings_definition.json`, set the task-level variable's value to a `task-type` reference, and also add the corresponding `task-type` variable with a blank value.

**Task type name mapping** (for the `task-type` key):

| Task type (in task.yaml) | task-type key segment |
|---|---|
| LANDING | `landing` |
| STORAGE | `storage` |
| TRANSFORM | `transform` |
| REGISTERED_DATA | `registered` |
| LAKE_LANDING | `lakeLanding` |
| LAKEHOUSE_STORAGE | `icebergStorage` |
| LAKEHOUSE_MIRROR | `mirror` |
| STREAMING_LAKE_LANDING | `streamingLanding` |
| STREAMING_TRANSFORM | `icebergTransform` |
| DATAMART | `datamart` |
| KNOWLEDGE_MART | `knowledgeMart` |
| FILE_BASED_KNOWLEDGE_MART | `fileBasedKnowledgeMart` |
| REPLICATION | `replication` |

**Example:** For a `LANDING` task with id `landing-1234` and property `databaseName`, add to `qtcp_bindings_definition.json`:
```json
{
  "task.landing-1234.databaseName": "{{task-type.landing.databaseName}}",
  "task-type.landing.databaseName": ""
}
```
If `task-type.landing.databaseName` already exists in `variables` (from a previous task of the same type), do not add a duplicate — leave the existing entry as-is.

**Ordering:** All `task-type.*` variables must appear at the top of the `variables` object, before any `task.*` or other variables. When adding a new `task-type.*` variable, insert it at the top of the list. When adding a `task.*` variable whose value references a `task-type.*` variable, place it after all `task-type.*` entries.

**Mandatory Binding variable rules (MUST):**
- Whenever a `{{...}}` binding variable is used in any project file, add it to `qtcp_bindings_definition.json` `variables` with a blank value (`""`) — **except** for task-type-defaulted properties (see above), which follow the two-level binding rule instead
- If the variable refers to a **connection property** (variable name ends with `Connection`, e.g., `sourceConnection`, `targetConnection`, `targetStorageConnection`), add it to the `connectionProperties` section **only if** the `type` or `kindId` are known — do not guess or add an empty entry
- **Mandatory synchronization gate (MUST):** Before finalizing any response, extract every `{{...}}` variable from all changed project files and ensure each variable exists in `qtcp_bindings_definition.json` under `variables`.
- **Completion criteria:** Do not state task completion until binding synchronization is performed and verified.

**Example qtcp_bindings_definition.json with variables and connectionProperties:**
```json
{
  "variables": {
    "platformConnection": "",
    "cloudStagingConnection": "",
    "projectName": "",
    "task-type.landing.databaseName": "",
    "task.landing_cdc-0001.sourceConnection": "",
    "task.landing_cdc-0001.databaseName": "{{task-type.landing.databaseName}}",
    "task.landing_cdc-0001:null$_$dbo.schema": ""
  },
  "connectionProperties": {
    "task.landing_cdc-0001.sourceConnection": {
      "type": "repsrc_mssql"
    }
  }
}
```

---

## Transformation Rules Workflow (Both Project Types)

⚠️ **CRITICAL: Always Create Truly Minimal YAML Files**

When creating transformation rules, include **ONLY required properties**. Do NOT add optional properties like `ordinal`, `enabled`, `id`, `description`, `nullable`, `originalTypeFull`, etc.

**Schema Requirements:**
- Only `name` and `actionType` are required in transformation rules
- Only `type` is required in `datatype` objects
- All other fields are optional

### When asked to add transformation rules:

**STEP 1: Determine Transformation Scope**
- ❓ **ASK:** "Should this transformation apply to:
  1. All tables in the task (table-level)
  2. A specific dataset only (dataset-level)"

**STEP 2: For Table-Level Transformations**
- ✅ **CREATE/UPDATE:** `transformationRules.yaml` in the task directory
- ✅ **ADD:** Rule to the `rules` array with minimal properties only
- ❓ **ASK:** "What transformation would you like to apply?"
- **Location:** `qtcp_tasks/<TaskName>/transformationRules.yaml`

**Minimal transformationRules.yaml:**
```yaml
rules:
  - name: Rule_1
    actionType: RENAME_TABLE
    scope:
      whereTableName: '%'
    action:
      renameType: TO_UPPER
```

**Available `renameType` values for `RENAME_TABLE` and `RENAME_COLUMN`:**
- `RENAME` - Direct rename to specific value
- `ADD_PREFIX` - Add prefix to names
- `REMOVE_SUFFIX` - Remove suffix from names
- `REPLACE_PREFIX` - Replace existing prefix (requires `oldValue`)
- `TO_LOWER` - Convert to lowercase
- `TO_UPPER` - Convert to uppercase
- `EXPRESSION` - Use expression for transformation
- `NAME_MAPPING` - Map specific names (requires `valueMapping`)

**Other action types:**
- `ADD_COLUMN` - Add new column
- `DROP_COLUMN` - Remove column
- `CHANGE_COLUMN_DATA_TYPE` - Change data type
- `REPLACE_COLUMN_VALUE` - Transform column values

**STEP 3: For Dataset-Level Transformations**

⚠️ **IMPORTANT:** Dataset-level transformations are added to separate dataset files, NOT to `transformationRules.yaml`.

- ❓ **ASK:** "Which dataset/table should this transformation apply to?"
- ❓ **ASK:** "What transformation would you like to apply?"
  - ADD - Add a new column
  - DROP - Remove a column
  - Rename column - Rename an existing column (uses KEEP action with newColumnName)
  - Add value expression - Transform column values (uses KEEP action with expression)
- ✅ **CREATE:** `qtcp_tasks/<TaskName>/datasets/` folder if it doesn't exist
- ✅ **CREATE:** `qtcp_tasks/<TaskName>/datasets/<DatasetName>.yaml` file if it doesn't exist
- ✅ **ADD:** Minimal `properties` section with mandatory `name`, `id` and `inputDatasets` (with `taskId` referencing the source task)
- ✅ **ADD:** `transformations` section with the transformation details

**Allowed Actions for Dataset-Level Transformations:**
- `ADD`: Add a new column (requires `columnName` and `newDataType`)
- `DROP`: Remove a column (requires only `columnName`)
- `KEEP` with rename: Rename existing column (requires `columnName` and `newColumnName`)
- `KEEP` with expression: Add value transformation (requires `columnName` and `expression`)

**Dataset Transformation Examples:**

**Example 1: ADD a new column**
```yaml
properties:
  name: <DatasetName>
  id: <dataset-id>
  inputDatasets:
    - taskId: <task-id>

transformations:
  columnTransformations:
    - action: ADD
      columnName: new_column
      newDataType:
        type: STRING
```

**Example 2: DROP a column**
```yaml
properties:
  name: <DatasetName>
  id: <dataset-id>
  inputDatasets:
    - taskId: <task-id>

transformations:
  columnTransformations:
    - action: DROP
      columnName: old_column
```

**Example 3: KEEP with rename**
```yaml
properties:
  name: <DatasetName>
  id: <dataset-id>
  inputDatasets:
    - taskId: <task-id>

transformations:
  columnTransformations:
    - action: KEEP
      columnName: old_name
      newColumnName: new_name
```

**Example 4: KEEP with expression**
```yaml
properties:
  name: <DatasetName>
  id: <dataset-id>
  inputDatasets:
    - taskId: <task-id>

transformations:
  columnTransformations:
    - action: KEEP
      columnName: price
      expression:
        expressionStatement: ${price} * 1.1
```

❌ **DO NOT:** Modify `transformationRules.yaml` for dataset-level transformations
❌ **DO NOT:** Create transformation rules automatically
❌ **DO NOT:** Use action types other than ADD, DROP, or KEEP for dataset-level transformations

---

## Schema Reference

All YAML files in a QTCP project are validated against JSON schemas published on SchemaStore.

| File | Path Pattern | Schema URL |
|------|-------------|------------|
| Project definition | `**/qtcp_project.yaml` | [project.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/project.schema.json) |
| Task definition | `**/qtcp_tasks/*/task.yaml` | [task.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.schema.json) |
| Dataset definition | `**/qtcp_tasks/*/datasets/*.yaml` | [task.dataset.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.dataset.schema.json) |
| Schedule | `**/qtcp_tasks/*/schedule.yaml` | [task.schedule.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.schedule.schema.json) |
| Data model | `**/qtcp_tasks/*/model.yaml` | [task.model.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.model.schema.json) |
| Source selection | `**/qtcp_tasks/*/sourceSelection.yaml` | [task.sourceselection.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.sourceselection.schema.json) |
| Transformation rules | `**/qtcp_tasks/*/transformationRules.yaml` | [task.transformation.rules.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.transformation.rules.schema.json) |
| Transformation data flow | `**/qtcp_tasks/*/transformationDataFlows/*.yaml` | [task.transformationdataflow.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.transformationdataflow.schema.json) |
| New task defaults | `**/qtcp_tasks/newTaskDefaults.yaml` | [newtaskdefaults.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/newtaskdefaults.schema.json) |

---

## Critical Rules Summary

1. **Follow the Unified Project Creation Workflow** — steps 1-8 above define the full creation flow
2. **Create minimal files** — only include required fields; consult schemas for what is required
3. **No automatic extras** — don't create files unless explicitly requested
4. **No comments in generated files** — YAML and JSON files must contain only data
5. **Dataset-level transformations** — use separate dataset files, not `transformationRules.yaml`