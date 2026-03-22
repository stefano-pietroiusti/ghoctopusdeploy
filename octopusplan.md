# Octopus Deploy Configuration Guide
Multi‑Environment CI/CD for Scripts + dbt

This document describes the required Octopus Deploy configuration to support full CI/CD across Dev, Test, UAT, and Prod using GitHub Actions and Octopus Deploy.

------------------------------------------------------------
1. Overview
------------------------------------------------------------
The deployment pipeline consists of:

- Scripts artifact (PowerShell, Python, or application scripts)
- dbt Core artifact (models, macros, seeds, profiles template)

A single Octopus Deploy project contains two deployment steps:
1. Deploy Scripts
2. Deploy dbt

All environment‑specific values are injected by Octopus at deploy time.

------------------------------------------------------------
2. Required Octopus Variables
------------------------------------------------------------
Create the following variables and scope them per environment:

DbServer                Database server hostname
DbName                  Database name
DbUser                  Username
DbPassword (sensitive)  Password
DbSchema                Schema for dbt models
AppConfigValue_*        Script‑specific config values
DBT_TARGET              dbt target name (dev/test/uat/prod)

Scope variables to:
Environment → Deployment Step (optional) → Target (optional)

------------------------------------------------------------
3. Scripts Deployment Step
------------------------------------------------------------

3.1 Package Deployment
Deploy the scripts artifact to the target folder on the application server.

3.2 Config Transformation
Enable one of the following:
- Structured Configuration Variables
- Substitute Variables in Files

Transform:
config.template.json → config.json

3.3 Execution
If scripts require execution:
- Add a script step (PowerShell, Python, etc.)
- Ensure the transformed config file is used

------------------------------------------------------------
4. dbt Deployment Step
------------------------------------------------------------

4.1 Package Deployment
Deploy the dbt artifact to the application server.

4.2 profiles.yml Transformation
Use variable substitution on:
profiles.yml.template → profiles.yml

Variables injected:
DB_HOST
DB_USER
DB_PASSWORD
DB_NAME
DB_SCHEMA
DBT_TARGET

4.3 dbt Execution
Run:

dbt deps
dbt seed   (optional)
dbt run
dbt test

Ensure environment variables are set:

DBT_TARGET=#{DBT_TARGET}
DB_HOST=#{DbServer}
DB_USER=#{DbUser}
DB_PASSWORD=#{DbPassword}
DB_NAME=#{DbName}
DB_SCHEMA=#{DbSchema}

------------------------------------------------------------
5. Lifecycle Configuration
------------------------------------------------------------

Dev
- Auto‑deploy on release creation

Test / UAT / Prod
- Manual promotion
- Optional approvals and gates

------------------------------------------------------------
6. GitHub Actions Integration
------------------------------------------------------------
GitHub Actions must:

- Build scripts artifact
- Build dbt artifact
- Push both to Octopus
- Create a release
- Auto‑deploy to Dev

Octopus handles all higher‑environment deployments.

------------------------------------------------------------
7. Validation Checklist
------------------------------------------------------------

[ ] No environment‑specific values in either repo  
[ ] All configs are tokenised templates  
[ ] profiles.yml uses env vars only  
[ ] Octopus variables scoped correctly  
[ ] Dev auto‑deploy works  
[ ] Test/UAT/Prod deploy using same artifacts  
