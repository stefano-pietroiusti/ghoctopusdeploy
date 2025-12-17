# CI/CD Pipeline with GitHub Enterprise Server and Octopus Deploy

This repository demonstrates how to set up a CI/CD pipeline for deploying Python scripts to a VM using **GitHub Enterprise Server (GHES)** and **Octopus Deploy**.  
The pipeline automatically deploys to **Development** when changes are pushed to `main`, and then promotes the same release to **QA** and **Production**.

---

## 🔧 Prerequisites

- GitHub Enterprise Server repository with Actions enabled.
- Octopus Deploy server accessible from GHES runners.
- Octopus API key (stored securely as a GitHub secret).
- Deployment targets (VMs) registered in Octopus under environments:
  - `Development`
  - `QA`
  - `Production`

## 🐙 Octopus Deploy Configuration (InstallDir)

### Installation Directory
All environments (Development, QA, Production) use the same installation path on their VMs:


### Steps
1. In Octopus Project → **Variables**, define:
   - `InstallDir = D:\MyRepo\src`
2. In the **Deploy a Package** step:
   - Enable **Custom Installation Directory**.
   - Set the directory to `#{InstallDir}`.
3. Octopus will extract the package into `D:\MyRepo\src` on every VM.

### Promotion Flow
- **Development** → Deploys to `D:\MyRepo\src`
- **QA** → Promotes same release to `D:\MyRepo\src`
- **Production** → Promotes same release to `D:\MyRepo\src`

This ensures consistency across environments and simplifies auditability.


---

## 🔑 Secrets Configuration

In your GHES repository, go to:

**Settings → Secrets and variables → Actions → New repository secret**

Add the following secrets:

- `OCTOPUS_API_KEY` → Your Octopus API key
- `OCTOPUS_SERVER_URL` → Your Octopus server URL (e.g., `https://octopus.example.com`)

The Octopus Server URL is the base address of your Octopus Deploy instance.

If you’re self‑hosting: it’s the URL you use in the browser to log into Octopus (e.g. https://octopus.mycompany.local or https://octopus.mycompany.com).

If you’re using Octopus Cloud: it’s the cloud instance URL (e.g. https://yourcompany.octopus.app).

OCTOPUS_API_KEY → paste the API key you were given.
OCTOPUS_SERVER_URL → paste the Octopus URL you found.
OCTOPUS_SERVICE_ACCOUNT → the publishing service account name (optional, only if you need to reference it in scripts or workflows).

---

## ⚙️ Workflow File

Create `.github/workflows/deploy.yml` with the following content:

```yaml
name: CI-CD Python Scripts

on:
  push:
    branches:
      - main

jobs:
  deploy-to-dev:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Package Python scripts
        run: |
          mkdir -p artifacts
          zip -r artifacts/python-scripts.${{ github.run_number }}.zip ./src

      - name: Push Package to Octopus
        uses: OctopusDeploy/push-package-action@v3
        with:
          api_key: ${{ secrets.OCTOPUS_API_KEY }}
          server: ${{ secrets.OCTOPUS_SERVER_URL }}
          space: 'Default'
          packages: 'artifacts/python-scripts.${{ github.run_number }}.zip'

      - name: Create and Deploy Release to Dev
        uses: OctopusDeploy/create-release-action@v3
        with:
          api_key: ${{ secrets.OCTOPUS_API_KEY }}
          server: ${{ secrets.OCTOPUS_SERVER_URL }}
          space: 'Default'
          project: 'PythonScriptsProject'
          release_number: ${{ github.run_number }}
          deploy_to: 'Development'

  promote-to-qa:
    runs-on: ubuntu-latest
    needs: deploy-to-dev
    steps:
      - name: Promote Release to QA
        uses: OctopusDeploy/deploy-release-action@v3
        with:
          api_key: ${{ secrets.OCTOPUS_API_KEY }}
          server: ${{ secrets.OCTOPUS_SERVER_URL }}
          space: 'Default'
          project: 'PythonScriptsProject'
          release_number: ${{ github.run_number }}
          deploy_to: 'QA'

  promote-to-prod:
    runs-on: ubuntu-latest
    needs: promote-to-qa
    steps:
      - name: Promote Release to Prod
        uses: OctopusDeploy/deploy-release-action@v3
        with:
          api_key: ${{ secrets.OCTOPUS_API_KEY }}
          server: ${{ secrets.OCTOPUS_SERVER_URL }}
          space: 'Default'
          project: 'PythonScriptsProject'
          release_number: ${{ github.run_number }}
          deploy_to: 'Production'
