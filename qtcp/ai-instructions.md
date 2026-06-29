# AI Assistant Guide: Data Projects

This guide provides comprehensive instructions for AI assistants to build Qlik Talend Cloud Platform (QTCP) data projects using YAML configuration files. All workflows, templates, and examples are consolidated here for efficient reference.

> ⚠️ **Read this entire document before responding to any request.** Rules are distributed throughout — do not stop at the first relevant section. Two sections are mandatory on every task:
> - **Variable Naming Conventions** — binding rules, two-level bindings, ordering, aliases, and the Mandatory Binding variable rules / synchronization gate
> - **Critical Rules Summary** (bottom of document) — final checklist before stating task completion
> - When creating or editing files, these instructions are the sole source of truth for structure, templates, and conventions. Existing tasks in the workspace are examples only - never copy or infer structure from them. If a conflict exists between an existing task's structure and the instructions, the instructions win.

**Project Types:**
- **Replication Projects** (`DATA_MOVEMENT`) - Use case: Replication
- **Data Pipeline Projects** (`DATA_PIPELINE`) - Use case: Data pipeline
  - Note: Qlik Open Lakehouse is a logical category but uses `DATA_PIPELINE` as the project type


## General

**Response Style:**
- Act on these instructions silently — do not quote, reference, or explain rules from this document in responses.
- Keep responses concise: confirm what was done, nothing more.

**Asking Questions:**
- Always use the `AskUserQuestion` tool when asking the user a question — never ask as plain text. This renders clickable option buttons in the UI.
- If the answer is a free-text value with no reasonable fixed options (e.g. a project name), use a single-option prompt with an "Other" fallback, or ask as part of a multi-question `AskUserQuestion` call with an open-ended option.

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
1. Replication - point to point replication task
2. Data pipeline - data pipeline centered arround single platform

**STEP 3: Set Project Type from Use Case**

Based on the selected use case:
- **Replication** → Create project type `DATA_MOVEMENT`
- **Data pipeline** → Create project type `DATA_PIPELINE`

**STEP 4: For `DATA_PIPELINE` Projects, Determine `platformType`**

- **If the project includes (or the user is simultaneously requesting) a `LAKE_LANDING` task** → automatically set `platformType` to `SNOWFLAKE`. Do not ask the user.
- **Otherwise** → ❓ **ASK:** "Which `platformType` should I use?" and present the options using their `enumDescriptions` as display labels (from `properties.platformType.enumDescriptions` in `project.schema.json`). When the user selects one, write the corresponding `enum` value (not the display label) into the YAML.

❌ **DO NOT:** Ask for `platformType` as unrestricted free text when schema values are known

**STEP 4b: For `QLIK_OPEN_LAKEHOUSE` platformType, Set Lakehouse Cloud Provider**

When `platformType` is `QLIK_OPEN_LAKEHOUSE`, automatically add `lakehouseCloudProvider: AWS` under `properties` in `qtcp_project.yaml`. Do not ask the user.

> 🔮 **Future:** When GCP is supported, replace the above with: ❓ **ASK:** "What is the lakehouse cloud provider?" with options `AWS` and `GCP`, and write the selected literal value.

**STEP 4c: For `SNOWFLAKE` platformType, Determine Landing Target**

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
✅ Create `<ProjectName>/qtcp_project.yaml` using the template for the correct project type.
✅ Create `<ProjectName>/qtcp_bindings_definition.json` — apply the **Mandatory Binding variable rules** to the project yaml just created: extract every `{{...}}` variable and add it to `variables` with a blank value `""`.
> 🔗 Before writing any binding variable, read the **Variable Naming Conventions** section below — it defines two-level bindings, ordering requirements, property name aliases, and the mandatory synchronization gate.
✅ Create empty subfolder `<ProjectName>/qtcp_tasks`

**STEP 6: Suggest Creating a Task**

❓ **ASK:** "Would you like me to create a task now?"

If yes, ask for task type and only present allowed options based on project type **and** platform type:

- For `DATA_MOVEMENT` projects: `REPLICATION`, `LAKE_LANDING`
- For `DATA_PIPELINE` projects, allowed task types depend on `platformType` (read from `qtcp_project.yaml`):
  - `SYNAPSE`, `FABRIC`, `MSSQL`, `REDSHIFT`, `BIGQUERY`, `DATABRICKS`, or `SNOWFLAKE` with landing target = tables in Snowflake: `LANDING`, `STORAGE`, `REGISTERED_DATA`, `TRANSFORM`
  - `SNOWFLAKE` with landing target = files in cloud storage: `LAKE_LANDING`, `STORAGE`, `REGISTERED_DATA`, `TRANSFORM`
  - `QLIK_OPEN_LAKEHOUSE`: `LAKE_LANDING`, `LAKEHOUSE_STORAGE`, `STREAMING_LAKE_LANDING`, `STREAMING_TRANSFORM`, `LAKEHOUSE_MIRROR`, `REPLICATE_LANDING`
  - `QVD`: `LANDING`, `QVD_STORAGE`

❌ Do **not** include `DATAMART`, `KNOWLEDGE_MART`, or `FILE_BASED_KNOWLEDGE_MART` in the presented list — if the user requests any of these, redirect them to create it in the QTC UI and commit back

**Source-matches-platform rule:** When the user wants to ingest data whose source is the **same platform as the project's `platformType`** (e.g., reading from Snowflake in a SNOWFLAKE project), use a `REGISTERED_DATA` task instead of a `LANDING` task. `LANDING` is for external/heterogeneous sources only.

**STEP 7: Ask for Task Name and Create the Task**

❓ **ASK:** "What would you like to name this task?"

✅ Create the task directory: `<ProjectName>/qtcp_tasks/<TaskName>/`
✅ Create minimal `task.yaml` with required fields only
✅ When creating dataset YAML files for this task, follow the **Dataset YAML Files** rules in the Dataset section below

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

| Project Type | Platform Type | Allowed Task Types |
|---|---|---|
| DATA_MOVEMENT | (any) | REPLICATION, LAKE_LANDING |
| DATA_PIPELINE | SYNAPSE, FABRIC, MSSQL, REDSHIFT, BIGQUERY, DATABRICKS, SNOWFLAKE (landing → tables in Snowflake) | LANDING, STORAGE, REGISTERED_DATA, TRANSFORM |
| DATA_PIPELINE | SNOWFLAKE (landing → files in cloud storage) | LAKE_LANDING, STORAGE, REGISTERED_DATA, TRANSFORM |
| DATA_PIPELINE | QLIK_OPEN_LAKEHOUSE | LAKE_LANDING, LAKEHOUSE_STORAGE, STREAMING_LAKE_LANDING, STREAMING_TRANSFORM, LAKEHOUSE_MIRROR, REPLICATE_LANDING |
| DATA_PIPELINE | QVD | LANDING, QVD_STORAGE |

---

## Project Type Reference

### Data Pipeline Projects and Qlik Open Lakehouse Projects (DATA_PIPELINE)

**Scheduling Rules by Task Type:**

| Task Type | Allowed Scheduling |
|---|---|
| STORAGE | `TIME_BASED` always allowed; `EVENT_BASED` conditionally allowed — see STORAGE check below |
| QVD_STORAGE, LANDING, LAKE_LANDING, STREAMING_LAKE_LANDING, REGISTERED_DATA, REPLICATION, REPLICATE_LANDING | ❌ No scheduling — do not create `schedule.yaml` |
| TRANSFORM, DATAMART, KNOWLEDGE_MART, FILE_BASED_KNOWLEDGE_MART, LAKEHOUSE_MIRROR, STREAMING_TRANSFORM, LAKEHOUSE_STORAGE | `TIME_BASED` or `EVENT_BASED` — ask the user which type |

**STORAGE event-based eligibility check:**

Before offering `EVENT_BASED` to a STORAGE task, check whether its source landing task is full-load-only:
1. Read the current task's `sourceSelection.yaml` and find the `sourceTask` value.
2. Search for a task whose `id` property matches that value.
3. In that task's `task.yaml`, look under `settings` for a `fullLoadOnly` property.
4. If `fullLoadOnly: true` → both `TIME_BASED` and `EVENT_BASED` are allowed.
5. If `fullLoadOnly` is absent or `false` → only `TIME_BASED` is allowed.

**Schedule Creation Workflow:**

**STEP 1:** Check the task type against the table above.
- If scheduling is not allowed → inform the user and do not create `schedule.yaml`
- If the task is `STORAGE` → perform the eligibility check above to determine whether `EVENT_BASED` is also available
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
  exportVersion: '1.0'
  name: '{{projectName}}'
  space: 'ref{project.current.spaceId}'
  type: DATA_PIPELINE
  platformType: <PLATFORM_TYPE>
  platformConnection: '{{platformConnection}}'
  cloudStagingConnection: '{{cloudStagingConnection}}'
settings:
  artifactsNaming:
    prefixSchema: '{{project.current.prefixSchema}}'
```

**Minimal task.yaml per task type:**

All task types require `properties.name`, `properties.id`, and `properties.type`. The table below lists the required `settings` fields for each type. Use `{{task.<task-id>.<property>}}` binding variables for all values unless noted otherwise.
> Properties marked **(two-level)** use `task-type.*` references instead of blank values — see **Variable Naming Conventions** section below. Exception: `LAKEHOUSE_MIRROR` tasks never use two-level binding.

| Task Type | Required `settings` fields |
|---|---|
| LANDING | `landingDwSettings.landingArtifactsLocation.dataAssetSchema` → `taskSchema` |
| LAKE_LANDING *(DATA_PIPELINE)* | `landingDwSettings.landingArtifactsLocation.dataAssetSchema` → `taskSchema` |
| STORAGE | `artifactsLocation.internalSchema`, `artifactsLocation.taskSchema`, `artifactsLocation.databaseName` *(two-level)*, `taskRuntime.warehouseSelection.warehouseName` *(two-level)* |
| TRANSFORM | `artifactsLocation.internalSchema`, `artifactsLocation.taskSchema`, `artifactsLocation.databaseName` *(two-level)*, `taskRuntime.warehouseSelection.warehouseName` *(two-level)* |
| REGISTERED_DATA | `artifactsLocation.internalSchema`, `artifactsLocation.taskSchema`, `artifactsLocation.databaseName` *(two-level)* |
| LAKEHOUSE_STORAGE | `artifactsLocation.internalSchema`, `artifactsLocation.taskSchema`, `taskRuntime.lakehouseCluster` *(two-level)* |
| LAKEHOUSE_MIRROR | `artifactsLocation.internalSchema`, `artifactsLocation.taskSchema`, `artifactsLocation.databaseName`, `platformConfig.platformType`, `platformConfig.connection` → `targetConnection` — see LAKEHOUSE_MIRROR platform notes below. Add `taskRuntime.warehouseSelection.warehouseName` only when `platformConfig.platformType` is `SNOWFLAKE`. ⚠️ No two-level binding for any field — all variables get a blank value directly in `qtcp_bindings_definition.json`. |
| STREAMING_LAKE_LANDING | `taskRuntime.lakehouseClusterId` → `lakehouseCluster` *(two-level)* |
| STREAMING_TRANSFORM | `generalSettings.artifactsLocation.internalSchema`, `generalSettings.artifactsLocation.taskSchema`, `runtimeSettings.lakehouseClusterId` → `lakehouseCluster` *(two-level)* |
| FILE_BASED_KNOWLEDGE_MART | `taskRuntime.warehouseSelection.warehouseName` *(two-level)* |
| REPLICATION | `taskSettings.fullLoad: true`, `taskSettings.applyChanges: true`, `targetEndpoint.targetConnection`, `targetEndpoint.targetSchema`, `targetEndpoint.targetStorageConnection` |
| LAKE_LANDING *(DATA_MOVEMENT)* | `targetEndpoint.targetConnection` |
| DATAMART | ❌ DO NOT create — must be created in QTC UI then committed back |
| KNOWLEDGE_MART | ❌ DO NOT create — must be created in QTC UI then committed back |
| REPLICATE_LANDING | `taskRuntime.lakehouseClusterId` → `lakehouseCluster` *(two-level)* |

**LAKEHOUSE_MIRROR: Data Warehouse Platform**

When creating a `LAKEHOUSE_MIRROR` task, after asking for the task name also ask:

❓ **ASK:** "What is the Data warehouse platform?" and present the enum values from `settings.platformConfig.platformType` in the schema.

Set `platformType` under `settings.platformConfig` in `task.yaml`.

Based on the chosen `platformType`, add the corresponding settings block under `settings.platformConfig`, and conditionally include `taskRuntime.warehouseSelection.warehouseName`:

- **`SNOWFLAKE`:** add `snowflakeIcebergSettings` with properties `snowflakeExternalVolume` and `snowflakeCatalogIntegration`; add `taskRuntime.warehouseSelection.warehouseName`
- **`REDSHIFT`:** add `redshiftIcebergSettings` with property `redshiftExternalSchema`; do **not** add `taskRuntime.warehouseSelection.warehouseName`
- **`DATABRICKS`:** add `databricksIcebergSettings` with property `databricksForeignCatalog`; do **not** add `taskRuntime.warehouseSelection.warehouseName`

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
  exportVersion: '1.0'
  name: '{{projectName}}'
  space: 'ref{project.current.spaceId}'
  type: DATA_MOVEMENT
```

Required settings fields for each task type are listed in the table above.

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

### Landing Tasks (LANDING, LAKE_LANDING, STREAMING_LAKE_LANDING, REPLICATION, REPLICATE_LANDING)

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

**`REPLICATE_LANDING` tasks** follow the same pattern but also require `rootDirectoryPath` as an additional root-level property:

**Minimal sourceSelection.yaml (REPLICATE_LANDING tasks):**
```yaml
sourceConnection: '{{task.<task-id>.sourceConnection}}'
rootDirectoryPath: ''
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

For `REPLICATE_LANDING` tasks, also include `rootDirectoryPath` at the root level alongside `sourceConnection`.

---

### Non-Landing Tasks (STORAGE, QVD_STORAGE, TRANSFORM, DATAMART, KNOWLEDGE_MART, FILE_BASED_KNOWLEDGE_MART, LAKEHOUSE_STORAGE, LAKEHOUSE_MIRROR, STREAMING_TRANSFORM)

Non-landing tasks read **from an upstream task**, not directly from a data source connection. The user must provide:
1. **Source task** (the upstream task that provides the data)
2. **Which datasets/tables** from that task to include

**Prompting:**
- ❓ **ASK:** "Which task should this read from?" (List available tasks from `qtcp_tasks/` folder)
  - ⚠️ **Landing source restriction:** Only `STORAGE` tasks may read from landing task types (`LANDING`, `LAKE_LANDING`, `STREAMING_LAKE_LANDING`, `REPLICATION`, `REPLICATE_LANDING`). All other non-landing task types **cannot** read from landing tasks — do not present landing tasks as source options for them.
- ❓ **ASK:** "Which tables/datasets from that task do you want to include?"

**Add entries to the task's `sourceSelection.yaml`.**
- Non-landing tasks use `sourceTask` on each `explicitlySelected` item to reference the upstream task
- ❌ **DO NOT** include `sourceConnection` or `rootDirectoryPath` — non-landing tasks do not use a direct data source connection
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
> 🔗 Any `{{...}}` variable introduced here must be added to `qtcp_bindings_definition.json` — see **Variable Naming Conventions: Mandatory Binding variable rules** below.
- `taskId` MUST be the upstream task's `properties.id`
- `datasetId` MUST be the upstream dataset's `properties.id` from a dataset file under that same upstream task — **if no dataset file exists for the upstream task (e.g., a LANDING task), convert the dataset name to a `datasetId` using these rules: letters and digits are kept as-is and lowercased; separators (any separator constant, `_`, or whitespace) are replaced with `_`; any other special character is encoded as `$HH$` where `HH` is the character's uppercase hex code (e.g., `@` → `$40$`, `!` → `$21$`). Examples: `"My App!"` → `my_app$21$`, `"hello world"` → `hello_world`, `"foo@bar"` → `foo$40$bar`. Never reference a `datasetId` that does not exist.**
- `name` MUST be provided as the reference name for the input dataset within this file
- Refer to the `dataset` schema (`task.dataset.schema.json`) for the required properties per task type — the schema describes which fields are mandatory based on the dataset `type`
- ❌ **DO NOT** hardcode property lists — always consult the schema for current requirements

❌ **MANDATORY — TRANSFORM tasks: you MUST create `datasets/<TableName>.yaml` for every output table. Adding or updating `sourceSelection.yaml` alone is never sufficient. Do not consider the task complete until both files exist.**

**When the user adds a wildcard pattern (e.g. `tablePattern: '%'`) or asks to add all datasets to a TRANSFORM task:**
1. Update `sourceSelection.yaml` as requested.
2. Then inspect the referenced upstream task for known dataset names — check its own `sourceSelection.yaml` (look at `explicitlySelected[].name`) and its `datasets/` folder for existing `.yaml` files.
3. For every dataset name that matches the pattern, create a corresponding `datasets/<TableName>.yaml` in the current TRANSFORM task (Pattern 1 passthrough unless the user specifies otherwise). Skip any table that already has a dataset file.
4. If no upstream dataset names can be determined (e.g. the upstream task uses a wildcard with no resolved names), note this explicitly and remind the user to add the dataset files once the table names are known.

There are three patterns for TRANSFORM dataset files:

**Pattern 1: Passthrough (simple copy)**
```yaml
properties:
  name: <DatasetName>
  id: <dataset-id>
  inputDatasets:
    - taskId: <upstream-task-id>
      datasetId: <upstream-dataset-id>
      name: <upstream-dataset-name>
  tableDef:
    columns: []
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
- ❌ **DO NOT** include `sourceConnection` or `rootDirectoryPath` — `REGISTERED_DATA` tasks do not use a data source connection
- ❌ **DO NOT** include `sourceTask` — `REGISTERED_DATA` tasks do not read from an upstream task
- ❌ **DO NOT** use `includePatterns` / wildcard patterns as a fallback for `REGISTERED_DATA` requests such as "add source"
- Each `explicitlySelected` item must specify `database`, `schema`, `name`, and `type`
- If the request uses the word "dataset" without explicitly saying "dataset file"/"dataset yaml", still treat it as source-selection intent and update `sourceSelection.yaml`.

**Completion check (REGISTERED_DATA, MUST pass before final response):**
1. `sourceSelection.yaml` contains no `sourceConnection` or `rootDirectoryPath`.
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

For these properties, instead of adding a blank value in `qtcp_bindings_definition.json`, set the task-level variable's value to a `task-type` reference, and also add the corresponding `task-type` variable with a blank value.

⚠️ **Exception — `LAKEHOUSE_MIRROR` tasks:** Two-level binding does **not** apply. Even for properties in the list above, always add a blank value directly in `qtcp_bindings_definition.json` (no `task-type` reference).

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
| REPLICATE_LANDING | `replicateLanding` |

**Example:** For a `LANDING` task with id `landing-1234` and property `databaseName`, add to `qtcp_bindings_definition.json`:
```json
{
  "task.landing-1234.databaseName": "{{task-type.landing.databaseName}}",
  "task-type.landing.databaseName": ""
}
```
If `task-type.landing.databaseName` already exists in `variables` (from a previous task of the same type), do not add a duplicate — leave the existing entry as-is.

**Ordering:** All `task-type.*` variables must appear at the top of the `variables` object, before any `task.*` or other variables. When adding a new `task-type.*` variable, insert it at the top of the list. When adding a `task.*` variable whose value references a `task-type.*` variable, place it after all `task-type.*` entries.

**Property name aliases:**

Some properties must bind to a variable whose name differs from the property name itself. The task-id prefix follows the normal pattern, but the trailing property name is remapped. The current alias mappings are:

| Property name (in task.yaml) | Variable name suffix to use |
|---|---|
| `snowflakeCatalogIntegration` | `snowflakeOpenCatalog` |

**Example:** For a task with id `mi1-5284`, the property `snowflakeCatalogIntegration` should be written as:
```yaml
snowflakeCatalogIntegration: '{{task.mi1-5284.snowflakeOpenCatalog}}'
```
and `task.mi1-5284.snowflakeOpenCatalog` added to `qtcp_bindings_definition.json` with a blank value.

**Mandatory Binding variable rules (MUST):**
- Whenever a `{{...}}` binding variable is used in any project file, add it to `qtcp_bindings_definition.json` `variables` with a blank value (`""`) — **except** for task-type-defaulted properties (see above), which follow the two-level binding rule instead
- If the variable refers to a **connection property** (variable name ends with `Connection`, e.g., `sourceConnection`, `targetConnection`, `targetStorageConnection`), add it to the `connectionProperties` section **only if** the `type` or `kindId` are known — do not guess or add an empty entry
- **Mandatory synchronization gate (MUST):** Before finalizing any response, extract every `{{...}}` variable from all changed project files and ensure each variable exists in `qtcp_bindings_definition.json` under `variables`.
- **Two-level binding completeness check (MUST):** For every `task.*` variable in `qtcp_bindings_definition.json` whose value is a `task-type.*` reference (e.g. `"{{task-type.replicateLanding.lakehouseCluster}}"`), verify that the corresponding `task-type.*` variable also exists in `qtcp_bindings_definition.json` with a blank value `""`. If it is missing, add it at the top of `variables`.
- **Completion criteria:** Do not state task completion until both synchronization checks above are performed and verified.

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

> ⚠️ Verify every item below before stating task completion.

1. **Follow the Unified Project Creation Workflow** — steps 1-8 above define the full creation flow
2. **Create minimal files** — only include required fields; consult schemas for what is required
3. **No automatic extras** — don't create files unless explicitly requested
4. **No comments in generated files** — YAML and JSON files must contain only data
5. **Dataset-level transformations** — use separate dataset files, not `transformationRules.yaml`
6. **Binding synchronization (MUST)** — extract every `{{...}}` variable from all changed files and verify each exists in `qtcp_bindings_definition.json` under `variables` before responding (→ **Variable Naming Conventions: Mandatory Binding variable rules**)
7. **Two-level bindings** — task-type-defaulted properties (e.g. `warehouseName`, `databaseName`, `lakehouseCluster`) use a `task-type.*` reference, not a blank value — except `LAKEHOUSE_MIRROR` tasks (→ **Variable Naming Conventions** section). After adding any two-level binding, verify that the corresponding `task-type.*` variable with a blank value also exists in `qtcp_bindings_definition.json` — the synchronization gate does **not** catch missing `task-type.*` entries automatically.
8. **Use `AskUserQuestion` tool** — never ask questions as plain text; always render clickable options