# E2E Testing Playbook

Complete guide for testing Tapworks connectors end-to-end on the shared demo workspace.

## Workspace

- **URL**: https://e2-demo-field-eng.cloud.databricks.com
- **CLI profile**: `e2-demo-field-eng`
- **Catalog**: `tapworks` (owned by yas.mokri)
- **Demo connections doc**: https://docs.google.com/document/d/1H7Kfy0YJBQta3rdDpVjkNcPVRn10x8tytTB6DJ6WPmE

## General Workflow

Every E2E test follows the same 5 steps. Connector-specific details are in the sections below.

### 1. Create target schema

```bash
databricks schemas create <schema_name> tapworks --profile e2-demo-field-eng
```

### 2. Create `pipeline_config.csv`

Create `e2e/<connector>/pipeline_config.csv`. Always include `pause_status` column set to `PAUSED`.

### 3. Generate DAB YAML

```bash
cd /Users/yas.mokri/sources/lakeflow-tapworks
PYTHONPATH=src .venv/bin/python3 -c "
from tapworks.core.runner import run_pipeline_generation
result = run_pipeline_generation(
    connector_name='<connector>',
    input_source='e2e/<connector>/pipeline_config.csv',
    output_dir='e2e/<connector>/deployment',
    targets={'dev': {'workspace_host': 'https://e2-demo-field-eng.cloud.databricks.com'}},
)
print(result.to_string())
"
```

### 4. Deploy and run

```bash
cd e2e/<connector>/deployment/<project_name>
databricks bundle deploy --profile e2-demo-field-eng --target dev
databricks bundle run <job_resource_name> --profile e2-demo-field-eng --target dev
```

### 5. Cleanup

```bash
cd e2e/<connector>/deployment/<project_name>
databricks bundle destroy --profile e2-demo-field-eng --target dev --auto-approve
```

Then manually drop the target schema if no longer needed:
```bash
databricks schemas delete tapworks.<schema_name> --profile e2-demo-field-eng
```

---

## Connector Guides

### SQL Server

**Type**: Database (gateway + pipeline + job)

#### Source

| Field | Value |
|---|---|
| Connection | `lfcddemo-azure-sqlserver` (ACTIVE, read_only, owned by robert.lee) |
| Host | `choo9chu-sq.database.windows.net:1433` |
| Database | `vaevee3u` |
| Schema | `lfcddemo` |
| Tables | `intpk` (PK, Change Tracking), `dtix` (no PK, CDC) |
| Secret scope | `lfcddemo`, key `choo9chu-sq` |

#### Source prerequisites

- Change Tracking enabled on `intpk`
- CDC enabled on `dtix`
- Both are pre-configured on the shared demo server

#### Target

- Schema: `tapworks.tapworks`

#### CSV columns

```
project_name,source_database,source_schema,source_table_name,target_catalog,target_schema,target_table_name,prefix,subgroup,gateway_catalog,gateway_schema,connection_name,schedule,pause_status
```

#### Post-generation steps

None — Tapworks generates correct YAML for SQL Server.

---

### PostgreSQL

**Type**: Database (gateway + pipeline + job)

#### Source

| Field | Value |
|---|---|
| Connection | `lfcddemo-azure-pg` (ACTIVE, read_only, owned by robert.lee) |
| Host | `jouzerai9oon4moh-azure-pg.postgres.database.azure.com:5432` |
| Database | `oigix9su` |
| Schema | `lfcddemo` |
| Tables | `intpk` (PK), `dtix` (no PK), `machine`, `cycle` |
| Secret scope | `lfcddemo`, key `jouzerai9oon4moh-azure-pg_json` (base64 JSON) |

#### Source prerequisites

1. `wal_level = logical` (already set)
2. A **publication** covering the tables you want to ingest (already exists: `databricks_publication` covers all 4 tables)
3. A **replication slot** per pipeline — must be created before first run:
   ```sql
   -- Connect as user with REPLICATION privilege
   SELECT pg_create_logical_replication_slot('<slot_name>', 'pgoutput');
   ```
4. When deleting a pipeline, **manually drop the slot**:
   ```sql
   SELECT pg_drop_replication_slot('<slot_name>');
   ```

**psql connection** (using libpq on macOS):
```bash
PGPASSWORD='<password>' /opt/homebrew/opt/libpq/bin/psql \
  -h jouzerai9oon4moh-azure-pg.postgres.database.azure.com \
  -p 5432 -U <user> -d oigix9su
```

Credentials are in the secret (base64 decode the `_json` key):
```bash
databricks secrets get-secret lfcddemo "jouzerai9oon4moh-azure-pg_json" --profile e2-demo-field-eng
# Decode: echo "<value>" | base64 -d
```

#### Target

- Schema: `tapworks.postgresql`

#### CSV columns

Same as SQL Server, plus `slot_name` and `publication_name`:
```
project_name,source_database,source_schema,source_table_name,target_catalog,target_schema,target_table_name,prefix,subgroup,gateway_catalog,gateway_schema,connection_name,schedule,pause_status,slot_name,publication_name
```

#### Post-generation steps

None — Tapworks now generates `source_configurations` with `slot_config` automatically when `slot_name` is provided in the CSV. Include `slot_name` and optionally `publication_name` (defaults to `databricks_publication`) columns in your CSV.

---

### Workday Reports

**Type**: SaaS (pipeline + job, no gateway)

#### Source

| Field | Value |
|---|---|
| Connection | `workday_demo` |
| Report 1 | All Active Employees — PK: `Employee_ID` |
| Report 2 | Employee Review Ratings — PK: `Employee` |
| Report 3 | Time Off Balances by Country — PK: `Worker` |

Report URLs:
- `https://wd2-impl-services1.workday.com/ccx/service/customreport2/databricks_gms2/lmcneil/All_Active_Employees_Data?format=json`
- `https://wd2-impl-services1.workday.com/ccx/service/customreport2/databricks_gms2/lmcneil/Employee_Review_Ratings_Detail?format=json`
- `https://wd2-impl-services1.workday.com/ccx/service/customreport2/databricks_gms2/lmcneil/Time_Off_Balances_by_Country?Location_Address_-_Country!WID=bc33aa3152ec42d4995f4791a106ed09&format=json`

#### Source prerequisites

None — connection is pre-configured.

#### Target

- Schema: `tapworks.workday`

#### CSV columns

```
project_name,source_url,target_catalog,target_schema,target_table_name,prefix,subgroup,connection_name,schedule,primary_keys,pause_status
```

Note: SaaS connectors use `source_url` and `primary_keys` instead of `source_database`/`source_schema`/`source_table_name`.

#### Post-generation steps

None — Tapworks generates correct YAML for Workday Reports.

---

### Google Analytics 4

**Type**: SaaS (pipeline + job, no gateway)

#### Source

| Field | Value |
|---|---|
| Connection | `andresz-ga4` (OAuth U2M, owned by andres.zuniga) |
| GCP Project | `gcp-sandbox-field-eng` |
| GA4 Property | `analytics_538839022` |
| Tables | `events`, `events_intraday`, `users` |

**How the source details were found**: The connection itself only stores OAuth credentials, not the GCP project/property. These were discovered by inspecting an existing pipeline (`test-ga4`, pipeline `48325101`) that uses the same connection.

#### Source prerequisites

- OAuth token must be valid — this is a U2M connection, so the owner (`andres.zuniga`) must re-authorize if the token expires
- No server-side setup needed (GA4 is read-only via BigQuery export)

#### Target

- Schema: `tapworks.ga4`

#### CSV columns

```
project_name,source_catalog,source_schema,tables,target_catalog,target_schema,connection_name,prefix,subgroup,schedule,pause_status
```

Note: GA4 uses `source_catalog` (GCP project), `source_schema` (GA4 property ID), and `tables` (comma-separated list) instead of individual table rows.

#### Post-generation steps

None — Tapworks generates correct YAML for GA4.

#### Known issues

- `andresz-ga4` connection failed (April 2026) with `SAAS_CONNECTOR_SOURCE_API_ERROR` / `UNKNOWN`. Other pipelines using this connection also fail. Likely an expired/revoked OAuth token that only the owner can fix.

---

### Salesforce

**Type**: SaaS (pipeline + job, no gateway)

#### Source

| Field | Value |
|---|---|
| Connection | Create your own — do NOT use Databricks' SFDC org |

#### Source prerequisites

- Sign up for a Salesforce Developer Edition: https://developer.salesforce.com/signup
- Create a Connected App for OAuth
- Create a Databricks UC connection of type SALESFORCE

#### Status: Not tested yet

---

### ServiceNow

**Type**: SaaS (pipeline + job, no gateway)

#### Source

| Field | Value |
|---|---|
| Connection | Create your own — dev instances sleep after 10 days |

#### Source prerequisites

- Sign up for a ServiceNow Developer Instance: https://developer.servicenow.com
- Note: instances hibernate after 10 days of inactivity

#### Status: Not tested yet

---

## Retrieving Source Credentials

All demo source credentials are stored in Databricks secret scope `lfcddemo`. To decode:

```bash
# List available secrets
databricks secrets list-secrets lfcddemo --profile e2-demo-field-eng

# Get a secret (JSON keys are base64-encoded)
databricks secrets get-secret lfcddemo "<key>_json" --profile e2-demo-field-eng

# Decode the value
echo "<base64_value>" | base64 -d | python3 -m json.tool
```

## Rules

- **Always deploy jobs PAUSED** — set `pause_status` column to `PAUSED` in every CSV
- **Track all deployed resources** in `e2e/TODO.md` under "Deployed Resources"
- **PostgreSQL**: always drop replication slots when cleaning up pipelines
- **`deployment/` dirs are gitignored** — only CSV configs and docs are committed
