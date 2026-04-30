# E2E Test: SQL Server Connector on e2-demo-field-eng

## Status: Deployed, ran successfully

## Key Facts (from Demo Guide + secrets + repo)

### Source SQL Server
- **Host**: `choo9chu-sq.database.windows.net:1433` (Azure SQL Server, serverless, 24x7)
- **Database (catalog)**: `vaevee3u` (decoded from Databricks secret scope=`lfcddemo`, key=`choo9chu-sq`)
- **Schema**: `lfcddemo`
- **Tables**:
  - `intpk` — has primary key, uses **Change Tracking** (CT)
  - `dtix` — no primary key, uses **Change Data Capture** (CDC)
- **Credentials** (from secret): USER=`ahng3phoongohph4`, DBA=`vae9aij7shae4ain`

### Connections
- Demo guide recommends `lfcddemo-sq` — but this **does NOT exist** on e2-demo-field-eng
- `lfcddemo-azure-sqlserver` **does exist** — same host, same secret, owned by robert.lee, read_only
- If `lfcddemo-azure-sqlserver` doesn't work, we can create `lfcddemo-sq` using the secret credentials

### Target (Databricks)
- **Workspace**: `https://e2-demo-field-eng.cloud.databricks.com` (profile: `e2-demo-field-eng`)
- **Catalog**: `tapworks` (created, owned by yas.mokri)
- **Schema**: `tapworks.tapworks` (created)

### Architecture (from demo guide)
- **Gateway pipeline**: classic compute, continuous mode, connects to SQL Server
- **Ingestion pipeline**: serverless, trigger or continuous mode, creates streaming tables
- **Connection**: UC connection with credentials to source
- Gateway does snapshot + CDC/CT at the same time
- CT preferred for PK tables (2% load), CDC for non-PK tables (high load: 3x IO, 30% CPU, 25% RAM)
- Expected pipeline start time on e2-demo-field-eng: < 5 min

## What's Done

### 1. Prerequisites verified
- CLI profile works: `databricks workspace list / --profile e2-demo-field-eng`
- Connection `lfcddemo-azure-sqlserver` exists and is ACTIVE
- Target catalog `tapworks` + schema `tapworks.tapworks` created

### 2. Input CSV created
File: `e2e/sql_server/pipeline_config.csv`
```csv
project_name,source_database,source_schema,source_table_name,target_catalog,target_schema,target_table_name,prefix,subgroup,gateway_catalog,gateway_schema,connection_name,schedule
tapworks_test,vaevee3u,lfcddemo,intpk,tapworks,tapworks,intpk,test,01,tapworks,tapworks,lfcddemo-azure-sqlserver,*/15 * * * *
tapworks_test,vaevee3u,lfcddemo,dtix,tapworks,tapworks,dtix,test,01,tapworks,tapworks,lfcddemo-azure-sqlserver,*/15 * * * *
```

### 3. Reference material
- Robert Lee's repo cloned to `/tmp/lakeflow_connect`
- Key notebook: `/tmp/lakeflow_connect/sqlserver/03_lakeflow_connect_demo.ipynb`
- Demo guide: `/Users/yas.mokri/sources/S8_ Lakeflow Connect Database Demo Guide.txt`

## Steps

See [../README.md](../README.md) for the general generate/deploy/run/cleanup workflow.

## Risks / Open Questions
1. **Connection name mismatch**: Demo guide says `lfcddemo-sq`, we have `lfcddemo-azure-sqlserver`. Should work since it points to the same server, but if not, create `lfcddemo-sq` using the secret credentials.
2. **Schema/tables existence**: The `lfcddemo` schema with `intpk`/`dtix` should exist (shared demo, always-up), but hasn't been verified via SQL query yet.
3. **CDC/CT already enabled?**: LakeFlow Connect requires CT on `intpk` and CDC on `dtix`. The shared demo should have these pre-configured.
