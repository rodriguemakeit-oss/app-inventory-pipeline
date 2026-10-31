# LeanIX Data Pipeline - Configuration Template

## Environment-Specific Configurations

### Development (Dev)
```yaml
# Workspace
workspace_host: https://workspace.example.invalid

# Data
catalog: leanix_dev
schema: default

# S3 Export
cdh_bucket: s3://demo-data-bucket/

# LeanIX API
leanix_subdomain: portfolio
leanix_api_token_scope: leanix-secrets-dev
leanix_api_token_key: api_token

# Scheduling
job_schedule_cron: "0 2 * * ?"           # Daily 2 AM UTC
timezone_id: Europe/Paris
pause_status: PAUSED                      # Paused by default in dev

# Volume
output_volume_path: /Volumes/leanix_dev/default/leanix_exports

# Permissions
run_as: null                              # Run as current user in dev
```

### Pre-Production (Preprod)
```yaml
# Workspace
workspace_host: https://workspace.example.invalid
workspace_root_path: /Workspace/Shared/.bundle/leanix-data-pipeline/preprod

# Data
catalog: leanix_preprod
schema: default

# S3 Export
cdh_bucket: s3://demo-data-bucket/

# LeanIX API
leanix_subdomain: portfolio
leanix_api_token_scope: leanix-secrets-preprod
leanix_api_token_key: api_token

# Scheduling
job_schedule_cron: "0 3 * * ?"           # Daily 3 AM UTC
timezone_id: Europe/Paris
pause_status: UNPAUSED                    # Active in preprod

# Volume
output_volume_path: /Volumes/leanix_preprod/default/leanix_exports

# Permissions
run_as: contributor@example.com     # Service principal
permissions:
  - service_principal_name: contributor@example.com
    level: CAN_MANAGE
  - group_name: leanix-admins-preprod
    level: CAN_VIEW
```

### Production (Prod)
```yaml
# Workspace
workspace_host: https://workspace.example.invalid
workspace_root_path: /Workspace/Shared/.bundle/leanix-data-pipeline/prod

# Data
catalog: leanix_prod
schema: default

# S3 Export
cdh_bucket: s3://demo-data-bucket/

# LeanIX API
leanix_subdomain: portfolio
leanix_api_token_scope: leanix-secrets-prod
leanix_api_token_key: api_token

# Scheduling
job_schedule_cron: "0 3 * * ?"           # Daily 3 AM UTC
timezone_id: Europe/Paris
pause_status: UNPAUSED                    # Active in prod

# Volume
output_volume_path: /Volumes/leanix_prod/default/leanix_exports

# Permissions
run_as: contributor@example.com        # Service principal
permissions:
  - service_principal_name: contributor@example.com
    level: CAN_MANAGE
  - group_name: leanix-admins-prod
    level: CAN_VIEW
  - user_name: contributor@example.com
    level: CAN_ATTACH_TO
```

## Environment Variables for Local Testing

Create `.env.local` for local development:

```bash
# Databricks
DATABRICKS_HOST=https://workspace.example.invalid
DATABRICKS_TOKEN=dapi1234567890...

# LeanIX
LEANIX_API_TOKEN=LXT_nEC7P6kKhJumJKhr98fgvNZwGCagCHRS6j7YAHAy
LEANIX_SUBDOMAIN=portfolio

# Paths
CATALOG=leanix_dev
SCHEMA=default
CDH_BUCKET=s3://demo-data-bucket/
OUTPUT_VOLUME=/Volumes/leanix_dev/default/leanix_exports

# Job Settings
JOB_SCHEDULE_CRON="0 2 * * ?"
TIMEZONE_ID=Europe/Paris
PAUSE_STATUS=PAUSED
```

## GitHub Actions Secrets

Configure these in GitHub repository settings:

### Preprod Deployment
```
DATABRICKS_PREPROD_HOST
DATABRICKS_PREPROD_TOKEN
```

### Production Deployment
```
DATABRICKS_PROD_HOST
DATABRICKS_PROD_TOKEN
```

### Optional: Notifications
```
SLACK_WEBHOOK_URL    # For deployment notifications
```

## Deployment Checklist

### Before Deploying to Dev
- [ ] Bundle validates: `databricks bundle validate`
- [ ] Credentials configured: `databricks workspace list`
- [ ] Secrets created in dev scope

### Before Deploying to Preprod
- [ ] Code reviewed and merged to `develop`
- [ ] Secrets created in preprod scope
- [ ] S3 bucket exists and accessible
- [ ] Service principal permissions set
- [ ] GitHub Actions secrets configured

### Before Deploying to Prod
- [ ] Release tag created: `git tag -a v1.0.0`
- [ ] Bundle validates for prod: `databricks bundle validate --target prod`
- [ ] Secrets created in prod scope
- [ ] Approval reviewers configured in GitHub
- [ ] Runbook and incident plan prepared

## Database Structure

After successful deployment, the following tables are created:

### Dimension Tables
- `dim_application` - Application master data
- `dim_<attribute>` - One table per business attribute (e.g., dim_business_criticality, dim_application_type)

### Fact Tables
- `fact_identity_governance` - Application governance data
- `fact_business_context_criticality` - Business criticality assessment
- `fact_architecture_hosting` - Infrastructure and hosting details
- `fact_data_privacy` - Privacy and data handling
- `fact_access_iam` - Access control and identity
- `fact_security_monitoring` - Security configuration
- `fact_risk_compliance` - Risk and compliance assessment

## Maintenance

### Regular Tasks

**Weekly:**
```bash
# Check job health
databricks jobs list-runs --job-name "leanix-data-pipeline-preprod" --limit 5
```

**Monthly:**
```bash
# Review cost in S3 and Databricks
aws s3 ls s3://demo-data-bucket/ --recursive --summarize
```

**Quarterly:**
```bash
# Update LeanIX token (expires yearly)
# Update service principal credentials
# Review and clean up old runs
```
