# App Inventory Pipeline

A Databricks Asset Bundle example for ingesting an application inventory, transforming it into dimensional tables, and exporting curated data.

## What it demonstrates

- Configuration-driven Databricks jobs and environments
- A notebook-based ingestion and transformation flow
- Dimensional modelling and data export patterns
- Optional scheduling and failure-alert configuration

## Repository map

- `src/notebooks/` — transformation notebook
- `resources/` — job definitions
- `databricks.yaml` — bundle targets and example settings

## Run locally

See [GUIDE.md](GUIDE.md). Running the pipeline requires a Databricks workspace, an inventory source, and storage configured by you. Sample configuration uses placeholders and includes no real inventory.

## Scope

This is a portfolio adaptation, not a ready-to-deploy client integration. Review the example settings and validate permissions, schedules, and destinations before use. See [VALIDATION.md](VALIDATION.md).