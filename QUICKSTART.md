# Quick Start Guide - LeanIX Data Pipeline

## Prerequisites Installation

### 1. Install Databricks Bundle CLI

```bash
# macOS with Homebrew
brew install databricks

# Or with pip
pip install databricks-cli databricks

# Verify installation
databricks --version
```

### 2. Configure Databricks Authentication

```bash
# Interactive setup
databricks configure --token

# Or environment variables
export DATABRICKS_HOST=https://your-workspace.cloud.databricks.com
export DATABRICKS_TOKEN=dapi1234567890...
```

### 3. Create Secret Scopes

```bash
# Dev
databricks secrets create-scope --scope leanix-secrets-dev
databricks secrets put --scope leanix-secrets-dev --key api_token --string-value "your_leanix_token"

# Preprod
databricks secrets create-scope --scope leanix-secrets-preprod
databricks secrets put --scope leanix-secrets-preprod --key api_token --string-value "your_leanix_token"

# Prod
databricks secrets create-scope --scope leanix-secrets-prod
databricks secrets put --scope leanix-secrets-prod --key api_token --string-value "your_leanix_token"
```

## Deployment Commands

### Dev Environment (Local Testing)

```bash
# Deploy
databricks bundle deploy

# Show resources
databricks bundle show

# Run job manually
databricks bundle run leanix_data_pipeline

# Watch logs in real-time
databricks runs watch <run_id>
```

### Preprod Environment

```bash
# Deploy
databricks bundle deploy --target preprod

# Run job
databricks bundle run --target preprod leanix_data_pipeline

# List recent runs
databricks jobs list-runs --job-name leanix-data-pipeline-preprod
```

### Production Environment

```bash
# Validate first
databricks bundle validate --target prod

# Deploy (requires proper permissions)
databricks bundle deploy --target prod

# Verify deployment
databricks bundle show --target prod
```

## Configuration Customization

Edit `databricks.yaml` to customize:

1. **Workspace URLs** (hosts)
2. **Catalog/Schema names** (database targets)
3. **S3 CDH bucket paths** (export destinations)
4. **Scheduling cron expressions** (job frequency)
5. **Service principals** (run-as identities)

## GitHub Actions Setup

### 1. Create GitHub Secrets

In your GitHub repository settings, add:

```
DATABRICKS_PREPROD_HOST     = https://workspace.example.invalid
DATABRICKS_PREPROD_TOKEN    = dapi...
DATABRICKS_PROD_HOST        = https://workspace.example.invalid
DATABRICKS_PROD_TOKEN       = dapi...
SLACK_WEBHOOK_URL           = https://hooks.slack.com/services/... (optional)
```

### 2. Deployment Triggers

**Preprod:** Push to `develop` branch
```bash
git push origin feature/new-feature
# Create PR and merge to develop
```

**Prod:** Create a release tag
```bash
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0
# Approve deployment in GitHub Actions UI
```

## Troubleshooting

### Token Not Working
```bash
# Verify token is valid
databricks workspace list

# Regenerate if needed
# 1. Go to Databricks workspace
# 2. User settings → Access tokens → Generate
```

### Bundle Validation Errors
```bash
# Check syntax
databricks bundle validate

# Show detailed output
databricks bundle validate --target preprod -v
```

### Job Not Running
```bash
# Check job exists
databricks jobs list --name-regex leanix

# View job details
databricks jobs get --job-name "leanix-data-pipeline-dev"

# Check recent failures
databricks jobs list-runs --job-name "leanix-data-pipeline-dev" --limit 10
```

## Data Outputs

### Databricks Tables
Tables are created in the format:
```
<catalog>.<schema>.dim_<section>
<catalog>.<schema>.fact_<section>
```

Example paths:
- `leanix_dev.default.dim_application`
- `leanix_dev.default.fact_identity_governance`

### S3 CDH Export
Files are exported to:
```
s3://demo-data-bucket<env>-568535/<table_name>
```

## Support

For issues:
1. Check logs: `databricks runs get-output --run-id <run_id>`
2. Validate config: `databricks bundle validate --target <env>`
3. Contact: contributor@example.com
