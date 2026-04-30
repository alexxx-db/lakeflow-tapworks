# E2E Testing

End-to-end tests for Tapworks connectors deployed on the **e2-demo-field-eng** workspace.

## Workspace

- **URL**: https://e2-demo-field-eng.cloud.databricks.com
- **CLI profile**: `e2-demo-field-eng`
- **Catalog**: `tapworks` (owned by yas.mokri)

## Tested Connectors

| Connector | Folder | Status |
|---|---|---|
| SQL Server | `sql_server/` | Deployed, ran successfully |
| Workday Reports | `workday_reports/` | Deployed, ran successfully, data landed |
| PostgreSQL | `postgresql/` | Deployed, ran successfully |
| Google Analytics 4 | `google_analytics/` | Deployed, ran successfully |
| Salesforce | `salesforce/` | Deployed, ran successfully |
| ServiceNow | `servicenow/` | Deployed, ran successfully |

## Available Demo Connections

Source: https://docs.google.com/document/d/1H7Kfy0YJBQta3rdDpVjkNcPVRn10x8tytTB6DJ6WPmE

### SaaS Connectors

| Connector | Connection Name | Status |
|---|---|---|
| Workday Reports | `workday_demo` | Ready |
| Workday HCM | `workday_hcm_demo` | Beta |
| Dynamics 365 | `d365_demo` | Ready |
| Confluence | `confluence_demo` | Ready |
| HubSpot | `hubspot_demo` | Beta |
| Google Ads | `google_ads_demo` | Beta |
| Salesforce | *(create own dev instance)* | Don't use Databricks' SFDC |
| ServiceNow | *(create own dev instance)* | Dev instances sleep after 10 days |

### Database Connectors

| Connector | Connection Name | Notes |
|---|---|---|
| SQL Server | `lfcddemo-azure-sqlserver` | Always-on Azure SQL |
| PostgreSQL | `lfcddemo-azure-pg` | XS 2-core, always running |

## How to Run an E2E Test

### 1. Generate DAB YAML

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

### 2. Deploy

```bash
cd e2e/<connector>/deployment/<project_name>
databricks bundle deploy --profile e2-demo-field-eng --target dev
```

### 3. Run (manual trigger)

```bash
databricks bundle run <job_resource_name> --profile e2-demo-field-eng --target dev
```

### 4. Cleanup

```bash
cd e2e/<connector>/deployment/<project_name>
databricks bundle destroy --profile e2-demo-field-eng --target dev --auto-approve
```

## Notes

- `deployment/` directories are gitignored -- only CSV configs and docs are tracked
- **Always deploy jobs in PAUSED state** -- set `pause_status` column to `PAUSED` in every CSV config. This avoids unexpected runs and costs on the shared e2-demo-field-eng workspace. Trigger runs manually via `databricks bundle run` when ready.
- PyPI proxy required: `https://pypi-proxy.dev.databricks.com/simple` (configured in `~/.config/pip/pip.conf`)
