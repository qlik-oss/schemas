# AI Assistant Guide: Data Projects

## Overview

This guide provides comprehensive instructions for AI assistants to build data projects using YAML configuration files. All workflows, templates, and examples are consolidated here for efficient reference.

**Project Types:**
- **Replication Projects** (`DATA_MOVEMENT`) - For replication tasks
- **Data Pipeline Projects** (`DATA_PIPELINE`) - For pipeline tasks and Qlik Open Lakehouse tasks
  - Note: Qlik Open Lakehouse is a logical category but uses `DATA_PIPELINE` as the project type

**Note on Minimal Properties:** Task templates should follow schema requirements. Schemas are published on [SchemaStore](https://www.schemastore.org) and automatically applied by the YAML language server when files match the expected paths (see Schema Reference section below).

**No Comments in Generated Files:**
- ❌ **DO NOT** add comments (`#` in YAML, `//` or `/* */` in JSON) to any generated project files
- YAML and JSON files in this project must contain only data — no inline comments, no block comments, no explanatory annotations

---

## Unified Project Creation Workflow

### When User Asks to Create a New Project

**STEP 1: Ask for Project Name**

❓ **ASK:** "What would you like to name this project?"

✅ Once provided, create a root folder with that name. All project files will be created inside this folder.

**STEP 2: Ask for Task Type (Not Project Type)**

When asked to create a new project, present ALL available task types:

❓ **ASK:** "What type of task would you like to create?

**Replication Tasks:**
1. REPLICATION - Replicate data from supported data sources to any supported target
2. LAKE_LANDING - Land data to a data lake

**Data Pipeline Tasks:**
3. LANDING - Copy data from a data source to a landing area (supports CDC or scheduled reloads)
4. STORAGE - Create ready to consume datasets in a cloud data warehouse or Qlik Cloud
5. QVD_STORAGE - Create QVD datasets in Qlik Cloud
6. REGISTERED_DATA - Register data that already exists on the data platform
7. TRANSFORM - Create reusable data transformations based on rules and custom SQL
8. DATAMART - Create data marts from Storage or Transform tasks
9. KNOWLEDGE_MART - Create vector database knowledge marts for RAG applications
10. FILE_BASED_KNOWLEDGE_MART - Create file-based knowledge marts

**Qlik Open Lakehouse Tasks:**
11. LAKEHOUSE_STORAGE - Store data in Apache Iceberg format
12. LAKEHOUSE_MIRROR - Mirror tables to cloud data warehouse without data duplication
13. STREAMING_LAKE_LANDING - Stream data to a data lake
14. STREAMING_TRANSFORM - Create streaming data transformations

**STEP 3: Infer Project Type from Task Selection**

Based on the user's task selection:
- Tasks **1-2** → Create **Replication Project** (type: `DATA_MOVEMENT`)
- Tasks **3-10** → Create **Data Pipeline Project** (type: `DATA_PIPELINE`)
- Tasks **11-14** → Create **Qlik Open Lakehouse Project** (type: `DATA_PIPELINE`)

**STEP 4: Create Project with Appropriate Type**

✅ Create root folder: `<ProjectName>/`
✅ Create minimal `<ProjectName>/qtcp_project.yaml` with the correct project type
✅ Create minimal `<ProjectName>/bindingsTemplate.json` with empty variables: `{ "variables": {} }`
✅ Create empty subfolder `<ProjectName>/qtcp_tasks`

**STEP 5: Ask for Task Name**

❓ **ASK:** "What would you like to name this task?"

**STEP 6: Create the Task**

✅ Create the task directory: `<ProjectName>/qtcp_tasks/<TaskName>/`
✅ Create minimal `task.yaml` with required fields only

**Task ID Generation Rule (Applies to all task types):**
- Format: `<sanitized-task-name>-<NNNN>` where `NNNN` is exactly 4 digits
- Preferred default: generate `NNNN` as 4 random digits (for example: `4821`)
- Alternative when explicitly requested: use 4-digit sequential numbering (`0001`, `0002`, ...)

**STEP 7: For `DATA_PIPELINE` Projects, Ask for `platformType` After Creation**

❓ **ASK (after project + task files are created):** "Which `platformType` should I use?"

`platformType` is mandatory for `DATA_PIPELINE` and is an enum.
- Present only the enum values currently defined in the active schema for `properties.platformType`
- Update `<ProjectName>/qtcp_project.yaml` with the selected `properties.platformType`

❌ **DO NOT:** Delay project/task creation while waiting for `platformType`
❌ **DO NOT:** Ask for `platformType` as unrestricted free text when schema values are known

❌ **DO NOT:** Create `sourceSelection.yaml` unless user has specified data sources
❌ **DO NOT:** Create `schedule.yaml` automatically
❌ **DO NOT:** Create `transformationRules.yaml` automatically
❌ **DO NOT:** Create sample datasets automatically

**STEP 8: Ask About Additional Configuration**

💬 **After creation, ask:**
- For Replication tasks: "Would you like me to:
  - Add data sources to this task?
  - Add transformation rules to this task?"

- For Pipeline tasks: "Would you like me to:
  - Add data sources to this task?
  - Create a schedule for this task?
  - Add transformation rules to this task?"

---

## Project Type Reference

### Data Pipeline Projects and Qlik Open Lakehouse Projects (DATA_PIPELINE)

**Scheduling Rules by Task Type:**

| Task Type | Allowed Scheduling |
|---|---|
| LANDING, STORAGE, QVD_STORAGE, LAKEHOUSE_STORAGE | `TIME_BASED` only |
| REGISTERED_DATA | ❌ No scheduling — do not create `schedule.yaml` |
| All other DATA_PIPELINE tasks (TRANSFORM, DATAMART, KNOWLEDGE_MART, FILE_BASED_KNOWLEDGE_MART, LAKEHOUSE_MIRROR, STREAMING_TRANSFORM) | `TIME_BASED` or `EVENT_BASED` — ask the user which type |

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

**Minimal qtcp_project.yaml:**
```yaml
properties:
  type: DATA_PIPELINE
  platformType: <PLATFORM_TYPE>
```

**Minimal task.yaml:**
All Data Pipeline task types require `properties.name`, `properties.id`, and `properties.type` fields.

```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: <TASK_TYPE>
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

**RRULE Schedule Examples:**

Hourly:
```yaml
- 'RRULE:FREQ=HOURLY;INTERVAL=1'
```

Every 6 hours:
```yaml
- 'RRULE:FREQ=HOURLY;INTERVAL=6'
```

**Important Notes:**
- **Only ONE schedule** allowed per task — if `schedule.yaml` already exists, inform the user
- Always set `enabled: true` when creating new `schedule.yaml`
- Task directory names are customizable — task type is defined in `task.yaml`

---

### Replication Projects (DATA_MOVEMENT)

**Key Characteristics:**
- Task types: `REPLICATION`, `LAKE_LANDING`
- **Schedules NOT supported** - do not create `schedule.yaml`
- No `settings.artifactsNaming` in project.yaml

**Minimal qtcp_project.yaml:**
```yaml
properties:
  type: DATA_MOVEMENT
```

**Example minimal task.yaml (REPLICATION):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: REPLICATION

settings:
  targetEndpoint:
    targetConnection: '{{task.<task-id>.targetConnection}}'
    targetSchema: ''
    targetStorageConnection: '{{task.<task-id>.targetStorageConnection}}'
    targetControlTableSchema: ''
```

**Example minimal task.yaml (LAKE_LANDING):**
```yaml
properties:
  name: <TaskName>
  id: <task-id>
  type: LAKE_LANDING
```

---

## Dataset Creation Workflow

When asked to add datasets or data sources:

**STEP 1: Determine Target Task**
- ❓ **IF task not specified:** "Which task should this dataset be added to?" (List available tasks from `qtcp_tasks/` folder)
- ✅ **IF task is known:** Proceed to next step

**STEP 2: Identify task type and follow the correct workflow**

There are two distinct workflows depending on the task type:

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
includePatterns: []
excludePatterns: []
explicitlySelected:
  - name: <TableName>
    schema: '{{task.<task-id>:null$_$<schema>.schema}}'
    type: TABLE
```

#### Using Include/Exclude Patterns (Optional, Landing Tasks Only)

For bulk table selection, you can use patterns instead of explicitly listing each table:

Include tables starting with "dim_" or "fact_", exclude "temp_" and "archive_":
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
  - tablePattern: 'archive_%'
    schemaPattern: 'dbo'
    type: TABLE
explicitlySelected: []
```

**Pattern Syntax:**
- `%` - Wildcard matching any characters
- Exact match - No wildcards, match exact name
- Case sensitivity depends on source database

---

### Non-Landing Tasks (STORAGE, QVD_STORAGE, TRANSFORM, DATAMART, KNOWLEDGE_MART, FILE_BASED_KNOWLEDGE_MART, LAKEHOUSE_STORAGE, LAKEHOUSE_MIRROR, STREAMING_TRANSFORM, REGISTERED_DATA)

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

**Minimal sourceSelection.yaml (non-landing tasks, all tables via pattern):**
```yaml
includePatterns:
  - tablePattern: '%'
    sourceTask: 'storage1-3742'
    type: TABLE
excludePatterns: []
explicitlySelected: []

**When creating dataset YAML files** for non-landing tasks, use `properties.inputDatasets` to reference the upstream task and dataset.
- `taskId` MUST be the upstream task's `properties.id`
- `datasetId` MUST be the upstream dataset's `properties.id` from a dataset file under that same upstream task
- Refer to the `dataset` schema (`task.dataset.schema.json`) for the required properties per task type — the schema describes which fields are mandatory based on the dataset `type`
- ❌ **DO NOT** hardcode property lists — always consult the schema for current requirements

---

❌ **DO NOT:** Create separate dataset YAML files in `datasets/` folder unless explicitly requested
❌ **DO NOT:** Add example or sample tables

**Important:** Source selection is defined in the `sourceSelection.yaml` file's `explicitlySelected` array, not as separate files.

---

## Variable Naming Conventions

When working with `bindingsTemplate.json`, follow these conventions:

- `task.<task-id>.<property>` - Task-specific properties (e.g., `task.landing_cdc-0001.sourceConnection`)
- `task.<task-id>:null$_$<schema>.<property>` - Schema-specific properties (e.g., `task.landing_cdc-0001:null$_$dbo.schema`)
- `task-type.<type>.<property>` - Default values for task types (e.g., `task-type.landing.databaseName`)
- `project.current.<property>` - Project-level references (e.g., `project.current.projectId`)

**Binding variable rules:**
- Whenever a `{{...}}` binding variable is used in any project file, add it to `bindingsTemplate.json` `variables` with a blank value (`""`)
- If the variable refers to a **connection property** (variable name ends with `Connection`, e.g., `sourceConnection`, `targetConnection`, `targetStorageConnection`), also add it to the `connectionProperties` section with a `type` property indicating the connection type
  - If the connection type is not known, omit the `type` property for that variable — do not guess

**Example bindingsTemplate.json with variables and connectionProperties:**
```json
{
  "variables": {
    "platformConnection": "",
    "cloudStagingConnection": "",
    "projectName": "",
    "task-type.landing.databaseName": "",
    "task.landing_cdc-0001.sourceConnection": "",
    "task.landing_cdc-0001.taskSchema": "",
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

### Data Types Reference

**Common data types for ADD_COLUMN transformations:**
- INT4, INT8 - Integer types
- WSTRING, STRING - Text types
- NUMERIC - Decimal numbers
- DATE, TIME, DATETIME - Date/time types
- BLOB, CLOB, NCLOB - Large objects
- JSON - JSON documents

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
| Source selection | `**/qtcp_tasks/*/sourceselection.yaml` | [task.sourceselection.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.sourceselection.schema.json) |
| Transformation rules | `**/qtcp_tasks/*/transformationrules.yaml` | [task.transformation.rules.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.transformation.rules.schema.json) |
| Transformation data flow | `**/qtcp_tasks/*/transformationdataflow/*.yaml` | [task.transformationdataflow.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/task.transformationdataflow.schema.json) |
| New task defaults | `**/qtcp_tasks/newtaskdefaults.yaml` | [newtaskdefaults.schema.json](https://raw.githubusercontent.com/qlik-oss/schemas/refs/heads/main/qtcp/newtaskdefaults.schema.json) |

---

## Critical Rules Summary

1. **Follow the Unified Project Creation Workflow** — steps 1-8 above define the full creation flow
2. **Create minimal files** — only include required fields; consult schemas for what is required
3. **No automatic extras** — don't create files unless explicitly requested
4. **No comments in generated files** — YAML and JSON files must contain only data
5. **Dataset-level transformations** — use separate dataset files, not `transformationRules.yaml`
