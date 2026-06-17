# PostgreSQL MCP Server Product Plan

## 1. Purpose

Create an MCP server that allows AI clients to safely retrieve data from approved PostgreSQL databases through curated commands.

The server does not expose arbitrary database access. Instead, it discovers YAML-defined commands from a filesystem catalog, exposes those commands as MCP tools, executes approved SQL scripts or PostgreSQL functions/procedures, and returns results to the AI client.

## 2. Goals

* Expose approved PostgreSQL-backed commands as MCP tools.
* Support multiple PostgreSQL servers and versions.
* Support shared-service operation for multiple users and databases.
* Support username/password, `.pgpass`, service files, SSL/TLS, and cloud-managed PostgreSQL connections.
* Keep secrets outside YAML command/config files.
* Allow commands to be organized in nested folders.
* Support SQL scripts and PostgreSQL function/procedure invocation.
* Support positional and named argument substitution.
* Audit every command execution.
* Support automatic catalog reload through file watching.
* Surface invalid catalog entries through explicit status and failed-tool behavior.
* Keep the implementation language/platform open.

## 3. Non-Goals for MVP

* Arbitrary SQL execution by the AI.
* Full RBAC/authorization model.
* Complex PostgreSQL type mapping.
* Database schema discovery.
* Query builder functionality.
* Result pagination or streaming.

## 4. Catalog Layout

```text
postgres-mcp-catalog/
  servers/
    ERP/
      server.yaml
      commands/
        inventory_counts/
          config.yaml
          inventory_counts.sql

        purchasing/
          open_purchase_orders/
            config.yaml
            open_purchase_orders.sql

    MyProduct/
      server.yaml
      commands/
        product_search/
          config.yaml
          product_search.sql

    DW/
      server.yaml
      commands/
        sales_summary/
          config.yaml
          sales_summary.sql

    DBAI/
      server.yaml
      commands/
        top_products/
          config.yaml
          top_products.sql
```

Discovery rule:

```text
Any folder under a server's commands/ directory that contains config.yaml is treated as a command.
```

## 5. Catalog Root

```yaml
catalog:
  root: /etc/postgres-mcp/servers

  watch:
    enabled: true
    debounceMs: 500

runtime:
  timeout: 1m
  maxRows: null

pool:
  # Implementation profile defines exact supported fields.
  # These values are global defaults for each PostgreSQL server.
```

`runtime.timeout` is a duration value such as `30s`, `1m`, or `5m`. The built-in default is `1m`, and the MCP server's global configuration may override that default through its normal configuration system, such as environment variables or a config file.

`runtime.maxRows` is optional and defaults to unlimited. When set, it applies to the aggregate row count across the entire command response.

`pool` defines global PostgreSQL connection pool defaults. The exact pool fields are implementation-profile-specific because PostgreSQL drivers expose different pool settings. Each implementation must document and validate its supported pool fields.

MVP supports one catalog root. That root contains one folder per configured PostgreSQL server.

Global configuration loading should follow this precedence, highest to lowest:

```text
Command-line arguments, if supported by the implementation
Environment variables
Global config file
Built-in defaults
```

Recommended global config file locations:

```text
POSTGRES_MCP_CONFIG, if set
/etc/postgres-mcp/config.yaml
~/.config/postgres-mcp/config.yaml
./config.yaml, for local development
```

Recommended environment variable naming convention:

```text
POSTGRES_MCP_CATALOG_ROOT
POSTGRES_MCP_CATALOG_WATCH_ENABLED
POSTGRES_MCP_CATALOG_WATCH_DEBOUNCE_MS
POSTGRES_MCP_RUNTIME_TIMEOUT
POSTGRES_MCP_RUNTIME_MAX_ROWS
POSTGRES_MCP_AUDIT_ENABLED
POSTGRES_MCP_AUDIT_SINK
```

Implementation-profile-specific values, such as pool settings, should use the same prefix and document their exact names.

## 6. Server Config Contract

File:

```text
servers/ERP/server.yaml
```

Example:

```yaml
name: ERP System
alias: ERP
description: ERP System running PostgreSQL 16

postgres:
  version: 16

  connect:
    type: usernamePassword
    host: erp-db.example.com
    port: 5432
    database: erp_prod
    ssl:
      mode: require

secrets:
  provider: env
  usernameRef: POSTGRES_ERP_USERNAME
  passwordRef: POSTGRES_ERP_PASSWORD

runtime:
  maxRows: 1000
  timeout: 30s

pool:
  # Implementation profile defines exact supported fields.
```

Connection string example:

```yaml
name: Data Warehouse
alias: DW
description: Legacy reporting warehouse

postgres:
  version: 14

  connect:
    type: connectionString
    connectionStringRef: POSTGRES_DW_URL

secrets:
  provider: env

runtime:
  maxRows: 5000
  timeout: 1m
```

`.pgpass` / service file example:

```yaml
name: MyProduct
alias: MyProduct
description: Custom product database

postgres:
  version: 15

  connect:
    type: service
    serviceName: myproduct-prod
    serviceFile: /etc/postgresql/pg_service.conf
    passFile: /run/secrets/pgpass

secrets:
  provider: none

runtime:
  maxRows: 1000
  timeout: 30s
```

SSL/TLS configuration uses a structured `ssl` object:

```yaml
ssl:
  mode: require
  ca: /path/to/ca.crt
  cert: /path/to/client.crt
  key: /path/to/client.key
  rejectUnauthorized: true
```

Supported `ssl.mode` values:

```text
disable
allow
prefer
require
verify-ca
verify-full
```

Certificate and key paths are optional and depend on the selected PostgreSQL deployment and implementation driver.

For `usernamePassword` connections, SSL/TLS is configured directly with the `ssl` object. For `connectionString` connections, SSL/TLS options should be supplied through the connection string when supported by the selected driver. For `service` connections, SSL/TLS options should be supplied through the PostgreSQL service file.

Server-level `runtime` values override MCP server global runtime defaults. Server-level `pool` values override MCP server global pool defaults for that PostgreSQL server.

Each configured PostgreSQL server uses the credential configured for that server. MCP caller identity is not used for PostgreSQL authentication in MVP; it is used for audit and future authorization context.

## 7. Command Config Contract

File:

```text
servers/ERP/commands/inventory_counts/config.yaml
```

SQL script example:

```yaml
name: Inventory Counts
alias: InventoryCounts
description: Returns a list of items and the number of each we have on hand

script: inventory_counts.sql

runtime:
  timeout: 45s
  maxRows: 1000

args:
  - name: filter
    type: string
    optional: true
    description: Optional item filter
```

Function example:

```yaml
name: Inventory Counts
alias: InventoryCounts
description: Returns inventory counts from a PostgreSQL function

function: inventory_counts

args:
  - name: filter
    type: string
    optional: true
    description: Optional item filter
```

Procedure example:

```yaml
name: Refresh Sales Summary
alias: RefreshSalesSummary
description: Refreshes the sales summary materialized data

procedure: refresh_sales_summary

args:
  - name: salesDate
    type: date
    optional: false
    description: Sales date to refresh
```

Rules:

```text
script, function, and procedure are mutually exclusive.
script resolves relative to the command folder.
alias is optional.
If alias is omitted, it is derived from name by removing spaces.
```

Command-level `runtime` values override PostgreSQL server runtime values, which override MCP server global runtime values. Command configs do not define pool settings.

## 8. MCP Tool Naming

Tool names use:

```text
<serverAlias>.<commandAlias>
```

Examples:

```text
ERP.InventoryCounts
DW.SalesSummary
MyProduct.ProductSearch
DBAI.TopProducts
```

Aliases should be matched case-insensitively where the MCP framework allows it.

Alias derivation:

```text
"Inventory Counts" -> "InventoryCounts"
"Sales Summary" -> "SalesSummary"
```

## 9. Argument Model

Supported MVP argument types:

```text
string
integer
number
boolean
date
timestamp
json
```

Example:

```yaml
args:
  - name: filter
    type: string
    optional: true
    description: Optional item filter

  - name: maxRows
    type: integer
    optional: true
    default: 100
    sensitive: false
    description: Maximum rows to return
```

Arguments are passed to scripts in YAML order.

If an optional argument is omitted and defines a `default`, the MCP server injects the default before validation and binding. If an optional argument is omitted and has no default, the MCP server binds `null`.

`sensitive` defaults to `false`. When `true`, the MCP server redacts that argument value from server-generated observability output, including audit events, logs, status diagnostics, and error context. `sensitive` does not alter execution behavior and does not redact database result rows. Database result rows are returned as produced by PostgreSQL.

Argument type mapping:

```text
string -> JSON string -> PostgreSQL text-compatible value
integer -> JSON integer -> PostgreSQL integer-compatible value
number -> JSON number -> PostgreSQL numeric-compatible value
boolean -> JSON boolean -> PostgreSQL boolean-compatible value
date -> JSON string, format date -> PostgreSQL date-compatible value
timestamp -> JSON string, format date-time -> PostgreSQL timestamp/timestamptz-compatible value
json -> any JSON value -> PostgreSQL json/jsonb-compatible value
```

MVP validation should require dates to use `YYYY-MM-DD` format and timestamps to include an explicit timezone offset or `Z`. Implementations must document any language-specific numeric limits, such as integer precision limits.

## 10. SQL Script Parameter Rules

Support positional parameters:

```sql
SELECT *
FROM inventory
WHERE item_name ILIKE '%' || $1 || '%';
```

Support named parameters before driver binding:

```sql
SELECT *
FROM inventory
WHERE item_name ILIKE '%' || :filter || '%';
```

MVP rules:

```text
$1 maps to the first arg.
$2 maps to the second arg.
:argName maps to an arg by name.
Argument names must be alphanumeric.
The implementation must use prepared statements or driver-level parameter binding.
No arbitrary AI-generated SQL is allowed.
SQL scripts may contain multiple statements.
$1 / :argName parameters are available consistently across all statements in the script.
A script must use either positional parameters or named parameters, not both.
Named parameters must be transformed using SQL-aware lexing or tokenization so placeholders inside string literals, quoted identifiers, comments, and dollar-quoted strings are not replaced.
Multi-statement scripts must be split using SQL-aware lexing or tokenization. Implementations must not split SQL scripts with ad hoc semicolon string splitting.
```

The required lexer/tokenizer only needs to identify executable SQL text versus single-quoted string literals, double-quoted identifiers, dollar-quoted strings, line comments, and block comments. A semicolon outside those contexts is a statement boundary. A `:argName` outside those contexts is a named parameter.

Recommended PostgreSQL script style:

```sql
SELECT
  item_name,
  count_on_hand
FROM
  inventory
WHERE
  $1 IS NULL
  OR item_name ILIKE '%' || $1 || '%'
ORDER BY
  item_name;
```

## 11. Function and Procedure MVP Support

Function execution:

```sql
SELECT * FROM inventory_counts($1);
```

Procedure execution:

```sql
CALL refresh_sales_summary($1);
```

Supported MVP behavior:

```text
Function name or procedure name.
IN parameters only.
Arguments passed in YAML order.
Functions may return scalar values, rows, tables, JSON, or void.
Procedures may return status and rows affected where available.
Functions and procedures execute inside the MCP-managed outer transaction.
```

PostgreSQL procedures that perform transaction control may fail when called inside the MCP-managed outer transaction. The MCP server does not inspect procedure or function bodies to prevent this. If PostgreSQL rejects the call, the server returns the PostgreSQL error in structured form with secrets redacted.

Procedures primarily return execution status. Procedures with OUT parameters or result rows are implementation-dependent and may be treated as future support unless the selected implementation profile explicitly supports them.

Advanced support can be added later for:

```text
INOUT parameters
REFCURSOR handling
Composite type mapping
Array type mapping
Explicit schema qualification controls
```

## 12. Result Handling

The MCP server should return PostgreSQL results as directly as practical.

For multi-statement SQL scripts, all result sets are returned in execution order.

Single-statement scripts use the single-statement result format. Scripts with more than one parsed statement use the multi-statement result format.

For query-like results:

```json
{
  "rows": [
    {
      "item_name": "Widget A",
      "count_on_hand": 42
    }
  ],
  "rowCount": 1
}
```

For non-query execution:

```json
{
  "status": "success",
  "rowsAffected": 3
}
```

For procedure execution:

```json
{
  "status": "success",
  "command": "CALL",
  "rowsAffected": null,
  "message": "Procedure executed successfully"
}
```

For multi-statement execution:

```json
{
  "results": [
    {
      "statementIndex": 0,
      "rowsAffected": 3
    },
    {
      "statementIndex": 1,
      "rows": [
        {
          "item_name": "Widget A",
          "count_on_hand": 42
        }
      ],
      "rowCount": 1
    }
  ],
  "totalRowCount": 1
}
```

For complex PostgreSQL output, including `json`, `jsonb`, arrays, and composite values, the server should preserve structure where possible and allow the consuming AI to interpret it.

The MCP server does not redact database result rows. Approved scripts, functions, and procedures are responsible for returning only data that the MCP client should receive.

If `maxRows` is configured, it applies to the aggregate row count across the full command response. A command that exceeds `maxRows` fails with `RESULT_LIMIT_EXCEEDED`, rolls back the outer transaction, writes a failed audit event, and returns an error that includes the actual and configured row counts.

The server may include minimal metadata:

```json
{
  "server": "ERP",
  "command": "InventoryCounts",
  "durationMs": 125,
  "result": {}
}
```

## 13. Secret Management

Passwords must not be stored directly in server YAML.

Supported secret providers should include:

### Environment Variables

```yaml
secrets:
  provider: env
  usernameRef: POSTGRES_ERP_USERNAME
  passwordRef: POSTGRES_ERP_PASSWORD
```

### Mounted Files

```yaml
secrets:
  provider: file
  usernamePath: /run/secrets/postgres-erp-username
  passwordPath: /run/secrets/postgres-erp-password
```

### Connection String Reference

```yaml
postgres:
  connect:
    type: connectionString
    connectionStringRef: POSTGRES_ERP_URL
```

### PostgreSQL Native Files

```yaml
postgres:
  connect:
    type: service
    serviceName: erp-prod
    serviceFile: /etc/postgresql/pg_service.conf
    passFile: /run/secrets/pgpass
```

Future providers:

```text
HashiCorp Vault
AWS Secrets Manager
Azure Key Vault
GCP Secret Manager
Kubernetes Secrets
```

## 14. Security Model

MVP security posture:

```text
Permissive command visibility.
No arbitrary SQL execution.
Only configured commands are exposed.
Each PostgreSQL server uses configured shared service credentials.
Secrets are externalized.
Secrets are redacted from logs.
Prepared statements or driver-level parameter binding are required.
Every execution is audited.
```

MVP safety depends on curated catalog entries and least-privileged PostgreSQL credentials. The MCP server does not invent SQL, accept arbitrary SQL from AI clients, or classify SQL as read-only/write-capable before execution.

Future authorization model:

```yaml
security:
  roles:
    - inventory-read
    - erp-readonly
```

## 15. Auditing

Auditing is core functionality.

Every command execution should record:

```text
timestamp
requestId
mcpClient
mcpUser, if available
serverAlias
commandAlias
commandPath
args, redacted where needed
durationMs
status
rowCount, if available
errorCode, if failed
errorMessage, if failed
```

Audit events record full command arguments by default. Arguments marked `sensitive: true` are redacted. Resolved infrastructure secrets, passwords, keys, and credential-bearing connection strings are never recorded.

Example audit event:

```json
{
  "timestamp": "2026-06-16T15:22:00Z",
  "requestId": "01JZ...",
  "mcpClient": "Claude Desktop",
  "serverAlias": "ERP",
  "commandAlias": "InventoryCounts",
  "durationMs": 184,
  "status": "success",
  "rowCount": 12
}
```

Audit config:

```yaml
audit:
  enabled: true
  sink: stdout
```

Future sinks:

```text
file
syslog
database
OpenTelemetry
HTTP webhook
```

## 16. File Watching and Reload

Automatic file watching is required.

Config:

```yaml
catalog:
  watch:
    enabled: true
    debounceMs: 500
```

Reload behavior:

```text
Detect changed server.yaml files.
Detect changed command config.yaml files.
Detect changed SQL scripts.
Revalidate affected server or command.
Update MCP tool catalog.
Emit reload errors to logs and health status.
```

The current filesystem catalog is the source of truth. The server does not execute a last-known-good version when the current config is invalid.

If an invalid server or command still has a resolvable alias, the MCP server exposes the corresponding MCP tool as a failed tool. Calling the tool returns a detailed structured configuration error, the path to the failed config or script, and an explicit instruction that the agent should not retry the tool until catalog status reports it healthy.

If a server or command name/alias is invalid and no identity can be resolved, no MCP tool is exposed for that entry. The failure is still reported by status diagnostics with the config path.

## 17. Validation Rules

Startup and reload validation should check:

```text
Each server folder has server.yaml.
Each server has name and alias or derivable alias.
Each command folder has config.yaml.
Each command has name and alias or derivable alias.
Each command has exactly one of script, function, or procedure.
Script files exist.
Argument names are alphanumeric.
Tool names are unique after alias normalization.
Secret references are present when required.
Connection config is valid enough to attempt connection.
Unknown fields are rejected according to the base schema and selected implementation profile.
```

Validation failures are scoped to the affected config entry whenever possible. A single misconfigured PostgreSQL server or command must not prevent the MCP process from starting or prevent unrelated valid servers and commands from working.

Implementation profiles must define and validate their concrete driver-specific pool schema. Unknown fields in implementation-specific sections such as `pool` are rejected by that profile.

## 18. Runtime Behavior

Startup flow:

```text
Load global config.
Discover server folders.
Load each server.yaml.
Discover command folders.
Load each command config.yaml.
Validate catalog.
Create PostgreSQL connection providers.
Expose executable tools and resolvable failed tools.
Expose core status tools.
Start file watcher.
```

Tool execution flow:

```text
Receive MCP tool call.
Resolve <serverAlias>.<commandAlias>.
Validate arguments.
Resolve secrets.
Acquire PostgreSQL connection from the configured server pool.
Begin MCP-managed outer transaction.
Execute script, function, or procedure using bound parameters.
Collect result.
Commit transaction if execution succeeds and constraints are satisfied.
Roll back transaction on execution error, timeout, or result limit violation.
Write audit event.
Return result to MCP client.
```

`runtime.timeout` applies to the overall command execution, including pool acquisition, transaction handling, all statements, result collection, and commit or rollback handling. Implementations should use database-enforced timeout controls where available, with an application-level deadline covering any work the database timeout cannot cover. On timeout, the MCP server attempts to cancel the running PostgreSQL work, rolls back the transaction, and returns the connection to the pool only if it is confirmed clean. Otherwise, the connection is discarded.

MVP uses the PostgreSQL default transaction isolation level, normally `READ COMMITTED`. Configurable transaction isolation is a future enhancement.

Connection pooling is required for MVP. Pool settings are resolved from MCP server global defaults and per-PostgreSQL-server overrides.

## 19. Status Tools

Status tools are core MCP server tools and must be exposed whenever the MCP process can start, even if the catalog fails completely.

Required status tools:

```text
PostgresMCP.Status
PostgresMCP.CatalogStatus
```

`PostgresMCP.Status` returns compact service health for humans and agents deciding whether the MCP server is usable:

```json
{
  "status": "degraded",
  "startedAt": "2026-06-17T15:00:00Z",
  "lastReloadAt": "2026-06-17T15:35:00Z",
  "catalog": {
    "loadedServers": 2,
    "failedServers": 1,
    "loadedCommands": 12,
    "failedCommands": 3
  },
  "tools": {
    "executable": 12,
    "failedTools": 3,
    "statusTools": 2
  },
  "detailsTool": "PostgresMCP.CatalogStatus"
}
```

`status` is one of `healthy`, `degraded`, or `unhealthy`. `healthy` means the MCP process is running and all configured catalog entries loaded successfully. `degraded` means the process is running and at least one catalog entry failed. `unhealthy` means the process is running but core service behavior is impaired.

Examples of `unhealthy` include an inaccessible catalog root at startup, inability to evaluate catalog state, or failure of the status tools' own dependencies.

`PostgresMCP.Status` does not include individual failure details or active PostgreSQL connection health. Use `PostgresMCP.CatalogStatus` for catalog diagnostics.

`PostgresMCP.CatalogStatus` returns detailed catalog diagnostics, including the catalog root, servers, commands, failed config paths, validation errors, failed script paths, secret reference names, and resolved tool names. It may include environment variable names, secret reference names, and filesystem paths, but never resolved secret values.

Catalog object status uses `healthy`, `degraded`, `failed`, or `not_found`. A command with `status: "healthy"` is executable. A command with `status: "failed"` and a `toolName` is exposed as an error-returning tool. If a command name or alias cannot be resolved, it appears in diagnostics as an unresolved config and has no tool.

`PostgresMCP.CatalogStatus` supports optional drill-down arguments:

```text
PostgresMCP.CatalogStatus()
PostgresMCP.CatalogStatus({ "serverAlias": "ERP" })
PostgresMCP.CatalogStatus({ "serverAlias": "ERP", "commandAlias": "InventoryCounts" })
```

With no arguments, `PostgresMCP.CatalogStatus` returns all server summaries for the catalog root.

```json
{
  "status": "degraded",
  "catalogRoot": "/etc/postgres-mcp/servers",
  "servers": [
    {
      "alias": "ERP",
      "name": "ERP System",
      "status": "degraded",
      "toolPrefix": "ERP",
      "loadedCommands": 8,
      "failedCommands": 1,
      "details": {
        "serverAlias": "ERP"
      }
    }
  ],
  "unresolvedConfigs": [
    {
      "code": "SERVER_CONFIG_INVALID",
      "path": "/etc/postgres-mcp/servers/Broken/server.yaml"
    }
  ]
}
```

With `serverAlias`, it returns all command summaries for that server.

```json
{
  "status": "degraded",
  "catalogRoot": "/etc/postgres-mcp/servers",
  "server": {
    "alias": "ERP",
    "name": "ERP System",
    "path": "/etc/postgres-mcp/servers/ERP/server.yaml",
    "status": "healthy"
  },
  "commands": [
    {
      "alias": "InventoryCounts",
      "name": "Inventory Counts",
      "toolName": "ERP.InventoryCounts",
      "path": "/etc/postgres-mcp/servers/ERP/commands/inventory_counts/config.yaml",
      "status": "healthy"
    },
    {
      "alias": "OpenPurchaseOrders",
      "name": "Open Purchase Orders",
      "toolName": "ERP.OpenPurchaseOrders",
      "path": "/etc/postgres-mcp/servers/ERP/commands/purchasing/open_purchase_orders/config.yaml",
      "status": "failed",
      "code": "COMMAND_CONFIG_INVALID"
    }
  ],
  "unresolvedConfigs": [
    {
      "code": "COMMAND_CONFIG_INVALID",
      "path": "/etc/postgres-mcp/servers/ERP/commands/broken_alias/config.yaml"
    }
  ]
}
```

With `serverAlias` and `commandAlias`, it returns full command detail.

Example failed command diagnostic:

```json
{
  "serverAlias": "ERP",
  "commandAlias": "InventoryCounts",
  "toolName": "ERP.InventoryCounts",
  "status": "failed",
  "path": "/etc/postgres-mcp/servers/ERP/commands/inventory_counts/config.yaml",
  "code": "COMMAND_CONFIG_INVALID",
  "message": "Unknown field: runtmie"
}
```

If the requested server or command alias is not found, `PostgresMCP.CatalogStatus` returns a normal status payload with `status: "not_found"` instead of throwing an execution error.

```json
{
  "status": "not_found",
  "serverAlias": "ERP",
  "commandAlias": "MissingCommand",
  "message": "No catalog command found for alias: ERP.MissingCommand"
}
```

## 20. Error Handling

Errors should be structured:

```json
{
  "status": "error",
  "error": {
    "code": "POSTGRES_EXECUTION_FAILED",
    "message": "Command execution failed.",
    "details": "PostgreSQL error detail with secrets redacted"
  }
}
```

Configuration and PostgreSQL execution errors should include detailed developer-facing diagnostics. Raw PostgreSQL error detail, hint, position, SQLSTATE, and related fields should be included where available. Secrets and credential-bearing connection strings must still be redacted.

Error redaction rules:

```text
Always include PostgreSQL SQLSTATE code when available.
Include PostgreSQL message, detail, hint, position, schema, table, column, dataType, and constraint when available.
Redact resolved passwords, keys, tokens, and credential-bearing connection strings from every error field.
Redact values for args marked sensitive.
Do not include stack traces in MCP tool responses.
Do not include raw connection configuration in MCP tool responses.
Include SQL text only if the implementation can guarantee it does not contain resolved secrets or sensitive argument values.
```

Failed tools return structured errors and an explicit agent instruction:

```json
{
  "status": "error",
  "error": {
    "code": "COMMAND_CONFIG_INVALID",
    "message": "Tool is unavailable because its catalog entry is invalid.",
    "path": "/etc/postgres-mcp/servers/ERP/commands/inventory_counts/config.yaml",
    "details": "script file not found: inventory_counts.sql",
    "agentInstruction": "Do not retry this tool until PostgresMCP.CatalogStatus reports it healthy."
  }
}
```

Error classes:

```text
CATALOG_LOAD_FAILED
SERVER_CONFIG_INVALID
COMMAND_CONFIG_INVALID
SECRET_RESOLUTION_FAILED
POSTGRES_CONNECTION_FAILED
POSTGRES_EXECUTION_FAILED
COMMAND_TIMEOUT
RESULT_LIMIT_EXCEEDED
```

## 21. MCP Tool and Config Schemas

MVP should include portable schemas for MCP status tools and YAML config files. These schemas define the base contract. Implementation profiles fill in language and driver-specific details, especially PostgreSQL pool configuration.

### Status Tool Input Schemas

`PostgresMCP.Status` accepts no input:

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {}
}
```

`PostgresMCP.CatalogStatus` accepts optional drill-down arguments:

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "serverAlias": {
      "type": "string",
      "description": "Optional server alias to inspect."
    },
    "commandAlias": {
      "type": "string",
      "description": "Optional command alias to inspect. Requires serverAlias."
    }
  },
  "allOf": [
    {
      "if": {
        "required": ["commandAlias"]
      },
      "then": {
        "required": ["serverAlias"]
      }
    }
  ]
}
```

### Dynamic Command Tool Input Schemas

Each catalog command produces an MCP tool input schema from its `args` list. Unknown input fields are rejected.

Example command config:

```yaml
args:
  - name: filter
    type: string
    optional: true
    description: Optional item filter

  - name: maxAgeDays
    type: integer
    optional: false
    description: Maximum age in days
```

Generated MCP input schema:

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "filter": {
      "type": "string",
      "description": "Optional item filter"
    },
    "maxAgeDays": {
      "type": "integer",
      "description": "Maximum age in days"
    }
  },
  "required": ["maxAgeDays"]
}
```

Argument type mapping:

```text
string -> JSON string
integer -> JSON integer
number -> JSON number
boolean -> JSON boolean
date -> JSON string with format date
timestamp -> JSON string with format date-time
json -> any JSON value
```

### YAML Schema Files

MVP should provide schema files for editor support and validation. The base schema files should reject unknown fields. Implementation profiles may extend placeholders marked below, but must still validate their concrete fields.

Schema validators must load referenced schema files before compiling schemas that use cross-document `$ref` values. The `$id` values and validator registration keys must be consistent so references such as `global-config.schema.yaml#/$defs/runtime` and `implementation-profile.schema.yaml#/$defs/pool` resolve reliably.

Recommended schema files:

```text
schemas/
  global-config.schema.yaml
  server.schema.yaml
  command.schema.yaml
  implementation-profile.schema.yaml
```

`global-config.schema.yaml` shape:

```yaml
$schema: https://json-schema.org/draft/2020-12/schema
$id: https://example.com/postgres-mcp/global-config.schema.yaml
type: object
additionalProperties: false
required:
  - catalog
properties:
  catalog:
    type: object
    additionalProperties: false
    required:
      - root
    properties:
      root:
        type: string
      watch:
        type: object
        additionalProperties: false
        properties:
          enabled:
            type: boolean
          debounceMs:
            type: integer
            minimum: 0
  runtime:
    $ref: "#/$defs/runtime"
  pool:
    $ref: "implementation-profile.schema.yaml#/$defs/pool"
  audit:
    type: object
    additionalProperties: false
    properties:
      enabled:
        type: boolean
      sink:
        type: string
$defs:
  runtime:
    type: object
    additionalProperties: false
    properties:
      timeout:
        type: string
        pattern: "^[0-9]+(ms|s|m|h)$"
      maxRows:
        anyOf:
          - type: integer
            minimum: 0
          - type: "null"
```

`server.schema.yaml` shape:

```yaml
$schema: https://json-schema.org/draft/2020-12/schema
$id: https://example.com/postgres-mcp/server.schema.yaml
type: object
additionalProperties: false
required:
  - name
  - postgres
  - secrets
properties:
  name:
    type: string
  alias:
    type: string
  description:
    type: string
  postgres:
    type: object
    additionalProperties: false
    required:
      - connect
    properties:
      version:
        type: integer
      connect:
        oneOf:
          - $ref: "#/$defs/usernamePasswordConnect"
          - $ref: "#/$defs/connectionStringConnect"
          - $ref: "#/$defs/serviceConnect"
  secrets:
    oneOf:
      - $ref: "#/$defs/envSecrets"
      - $ref: "#/$defs/fileSecrets"
      - $ref: "#/$defs/noSecrets"
  runtime:
    $ref: "global-config.schema.yaml#/$defs/runtime"
  pool:
    $ref: "implementation-profile.schema.yaml#/$defs/pool"
$defs:
  usernamePasswordConnect:
    type: object
    additionalProperties: false
    required: [type, host, port, database]
    properties:
      type:
        const: usernamePassword
      host:
        type: string
      port:
        type: integer
      database:
        type: string
      ssl:
        $ref: "#/$defs/ssl"
  connectionStringConnect:
    type: object
    additionalProperties: false
    required: [type, connectionStringRef]
    properties:
      type:
        const: connectionString
      connectionStringRef:
        type: string
  serviceConnect:
    type: object
    additionalProperties: false
    required: [type, serviceName]
    properties:
      type:
        const: service
      serviceName:
        type: string
      serviceFile:
        type: string
      passFile:
        type: string
  envSecrets:
    type: object
    additionalProperties: false
    required: [provider]
    properties:
      provider:
        const: env
      usernameRef:
        type: string
      passwordRef:
        type: string
  fileSecrets:
    type: object
    additionalProperties: false
    required: [provider]
    properties:
      provider:
        const: file
      usernamePath:
        type: string
      passwordPath:
        type: string
  noSecrets:
    type: object
    additionalProperties: false
    required: [provider]
    properties:
      provider:
        const: none
  ssl:
    type: object
    additionalProperties: false
    required: [mode]
    properties:
      mode:
        enum: [disable, allow, prefer, require, verify-ca, verify-full]
      ca:
        type: string
      cert:
        type: string
      key:
        type: string
      rejectUnauthorized:
        type: boolean
```

`command.schema.yaml` shape:

```yaml
$schema: https://json-schema.org/draft/2020-12/schema
$id: https://example.com/postgres-mcp/command.schema.yaml
type: object
additionalProperties: false
required:
  - name
properties:
  name:
    type: string
  alias:
    type: string
  description:
    type: string
  script:
    type: string
  function:
    type: string
  procedure:
    type: string
  runtime:
    $ref: "global-config.schema.yaml#/$defs/runtime"
  args:
    type: array
    items:
      type: object
      additionalProperties: false
      required:
        - name
        - type
      properties:
        name:
          type: string
          pattern: "^[A-Za-z0-9]+$"
        type:
          enum: [string, integer, number, boolean, date, timestamp, json]
        optional:
          type: boolean
        default:
          description: Default argument value.
        sensitive:
          type: boolean
        description:
          type: string
oneOf:
  - required: [script]
  - required: [function]
  - required: [procedure]
```

`implementation-profile.schema.yaml` shape:

```yaml
$schema: https://json-schema.org/draft/2020-12/schema
$id: https://example.com/postgres-mcp/implementation-profile.schema.yaml
$comment: >
  This file is completed by the selected implementation. It must name the
  implementation language, MCP framework, PostgreSQL driver, and concrete pool
  configuration fields supported by that driver.
type: object
additionalProperties: false
properties:
  language:
    type: string
  mcpFramework:
    type: string
  postgresDriver:
    type: string
$defs:
  pool:
    type: object
    additionalProperties: false
    properties:
      # Implementation profile fills in driver-specific pool fields here.
      # Example for one implementation might include maxConnections,
      # idleTimeout, connectionLifetime, or acquisitionTimeout.
```

Each implementation profile must fill in the driver-specific contract without changing the base product behavior:

```text
Language and runtime version.
MCP framework or library.
PostgreSQL driver and supported driver version range.
Concrete pool configuration fields and defaults.
SQL lexer or tokenizer strategy for multi-statement scripts and named parameter transformation.
Timeout strategy, including database-enforced timeout support where available.
File watching library or mechanism.
YAML parsing and schema validation libraries.
Request ID generation strategy for audit correlation.
PostgreSQL service file and pass file support strategy.
```

## 22. Deployment Model

The product should be implementation-language agnostic.

Runtime requirements:

```text
PostgreSQL driver available for the selected implementation language
MCP server framework or library available for the selected implementation language
Filesystem access to the catalog root
Network access to PostgreSQL databases
Secret provider access
```

The base design defines portable behavior and configuration concepts. Once a customer selects an implementation language and PostgreSQL driver, that implementation must provide a concrete implementation profile that fills in driver-specific configuration contracts, especially connection pool fields.

## 23. MVP Scope

MVP must include:

```text
MCP server startup
Single catalog root loading
Per-server server.yaml
Nested command folders
Command config.yaml
SQL script execution
Simple PostgreSQL function execution
Simple PostgreSQL procedure execution
MCP-managed outer transactions for scripts, functions, and procedures
Username/password connections
Connection string connections
.pgpass/service file support
Environment variable secrets
File-based secrets
Connection pooling with global defaults and per-server overrides
Tool naming as <serverAlias>.<commandAlias>
Argument validation
Sensitive argument redaction for server-generated observability output
$1 / $2 positional parameters
:argName named parameters
Prepared statements or driver-level binding
Result return
Aggregate optional maxRows enforcement
Overall command timeout using duration values
Audit logging
File watching and reload
Failed tools for resolvable invalid catalog entries
PostgresMCP.Status
PostgresMCP.CatalogStatus
MCP status tool input schemas
Dynamic command tool input schema generation
YAML schema files for global config, server config, command config, and implementation profile
Structured errors
```

## 24. Future Enhancements

```text
RBAC and command filtering
OAuth/OIDC identity mapping
Multiple catalog roots
Advanced PostgreSQL type support
REFCURSOR handling
OpenTelemetry tracing
Database audit sink
Result pagination
Result streaming
CSV/JSON/table formatting options
Command tags/categories
Command deprecation lifecycle
Catalog signing or integrity checks
Admin API
Dry-run validation mode
CLI catalog validator
Read-only transaction enforcement
Configurable transaction isolation
```

## 25. Example End-to-End Command

Directory:

```text
servers/ERP/
  server.yaml
  commands/
    inventory_counts/
      config.yaml
      inventory_counts.sql
```

`server.yaml`:

```yaml
name: ERP System
alias: ERP
description: ERP System running PostgreSQL 16

postgres:
  version: 16

  connect:
    type: usernamePassword
    host: erp-db.example.com
    port: 5432
    database: erp_prod
    ssl:
      mode: require

secrets:
  provider: env
  usernameRef: POSTGRES_ERP_USERNAME
  passwordRef: POSTGRES_ERP_PASSWORD

runtime:
  maxRows: 1000
  timeout: 30s
```

`config.yaml`:

```yaml
name: Inventory Counts
alias: InventoryCounts
description: Returns a list of items and the number of each we have on hand

script: inventory_counts.sql

args:
  - name: filter
    type: string
    optional: true
    description: Optional filter for items
```

`inventory_counts.sql`:

```sql
SELECT
  item_name,
  count_on_hand
FROM
  inventory
WHERE
  $1 IS NULL
  OR item_name ILIKE '%' || $1 || '%'
ORDER BY
  item_name;
```

Exposed MCP tool:

```text
ERP.InventoryCounts
```

Example tool input:

```json
{
  "filter": "widget"
}
```

Example result:

```json
{
  "server": "ERP",
  "command": "InventoryCounts",
  "durationMs": 91,
  "result": {
    "rows": [
      {
        "item_name": "Widget A",
        "count_on_hand": 42
      }
    ],
    "rowCount": 1
  }
}
```
