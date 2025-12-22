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

```

# CI/CD Model Overview (GitHub Actions → GitHub Packages → Octopus Deploy)

This project uses a CI/CD model designed for environments where direct API calls from GitHub Actions to Octopus Deploy are restricted by company policy.

### ✅ CI (Continuous Integration) — GitHub Actions
GitHub Actions is responsible for:
- Checking out the repository
- Installing Python dependencies
- Packaging the `src` folder into a versioned ZIP artifact
- Publishing the artifact to **GitHub Packages**

No deployment or Octopus API calls occur in GitHub Actions.

### ✅ CD (Continuous Deployment) — Octopus Deploy
Octopus Deploy handles:
- Automatically detecting new packages in GitHub Packages
- Automatically creating a release when a new package arrives
- Automatically deploying to the **DEV** environment
- Allowing controlled promotion to **QA** and **PROD** through the Octopus lifecycle

This separation ensures:
- Compliance with company policy
- Full auditability in Octopus
- Clean, maintainable CI workflows in GitHub
- Controlled, gated deployments across environments

### ✅ Artifact Flow
1. GitHub Actions builds and publishes the ZIP package.
2. Octopus polls GitHub Packages as an external feed.
3. When a new version is detected:
   - Octopus creates a release
   - Octopus deploys to DEV
   - QA and PROD deployments follow lifecycle rules

``` yaml
name: Build Python Project

on:
  push:
    branches:
      - feature/cicd   # test branch
      - main           # protected branch with reviews

jobs:
  build:
    runs-on: windows-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Package src folder
        run: |
          mkdir artifacts
          powershell Compress-Archive -Path src -DestinationPath artifacts/crmdsl-reconciliation-app.${{ github.run_number }}.zip

      - name: Upload artifact to GitHub Packages
        uses: actions/upload-artifact@v4
        with:
          name: crmdsl-reconciliation-app
          path: artifacts/crmdsl-reconciliation-app.${{ github.run_number }}.zip


```
## Instructions — Configure Octopus to Auto‑Import Packages


## 🔐 Configuring Octopus Deploy to Access GitHub Packages

This project uses **GitHub Packages** as the artifact feed for Octopus Deploy.  
Because GitHub Packages requires authentication, Octopus must be configured with a **GitHub Personal Access Token (PAT)** that has permission to read packages.

Follow the steps below to configure secure access.

---

## ✅ Step 1 — Create a GitHub Personal Access Token (PAT)

1. Go to **GitHub → Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens**  
   (Classic tokens also work if your organisation allows them.)
2. Create a new token with the following scopes:

### Required scopes
- **`read:packages`**  
  Allows Octopus to read packages from GitHub Packages.
- **`repo`** *(only required if the repository is private)*  
  Allows access to private package metadata.

3. Copy the PAT — you will need it when configuring Octopus.

> ⚠️ Treat the PAT like a password.  
> It should only be stored inside Octopus as a secure credential.

---

## ✅ Step 2 — Store the PAT in Octopus Deploy

1. In Octopus, navigate to:  
   **Library → External Feeds**
2. Click **Add Feed** and choose **GitHub Packages**.
3. Configure the feed:

| Field | Value |
|-------|--------|
| **Feed Name** | GitHubPackages |
| **Feed URL** | `https://nuget.pkg.github.com/<your-org-or-username>/index.json` |
| **Username** | Your GitHub username (or GitHub service account username) |
| **Password** | The PAT created in Step 1 |

4. Save the feed.

Octopus can now authenticate to GitHub Packages using the PAT.

---

## ✅ Step 3 — Assign the Feed to Your Octopus Project

1. Go to your Octopus project:  
   **Projects → crmdsl-reconciliation → Process**
2. Edit the **Deploy a Package** step.
3. Under **Package Source**, select the feed you created (e.g., `GitHubPackages`).
4. Set the **Package ID** to match the artifact published by GitHub Actions:
5. Save the step.

Octopus will now pull the package directly from GitHub Packages during deployments.

---

## ✅ Result

With these steps complete:

- GitHub Actions builds and publishes the package.
- GitHub Packages stores the artifact securely.
- Octopus Deploy authenticates using the PAT.
- Octopus automatically retrieves the package during deployments.

This model keeps CI and CD cleanly separated and fully compliant with environments where GitHub → Octopus API calls are restricted.





## 🚀 Full CI/CD Flow

This project uses a CI/CD model designed for environments where direct API calls from GitHub Actions to Octopus Deploy are restricted by company policy.  
The pipeline cleanly separates **Continuous Integration (CI)** and **Continuous Deployment (CD)** responsibilities while maintaining full automation and auditability.

---

## ✅ Continuous Integration (CI) — GitHub Actions

GitHub Actions is responsible for building and packaging the application.

### CI Responsibilities
- Checkout the repository
- Install Python dependencies
- Package the `src` folder into a versioned ZIP artifact
- Publish the artifact to **GitHub Packages**

### Why GitHub Actions only handles CI
Company policy prevents GitHub Actions from:
- Pushing packages directly to Octopus Deploy
- Creating releases in Octopus
- Triggering deployments via Octopus API

By limiting GitHub Actions to CI tasks only, the workflow remains compliant and secure.

---

## ✅ Artifact Storage — GitHub Packages

GitHub Packages acts as the central artifact repository.

### Why GitHub Packages
- Secure, authenticated package storage
- Native integration with GitHub Actions
- Octopus Deploy can authenticate using a GitHub PAT
- Supports versioned ZIP artifacts for Python projects

Each CI run publishes a new package version, which Octopus can detect automatically.

---

## ✅ Continuous Deployment (CD) — Octopus Deploy

Octopus Deploy handles all deployment automation and environment promotion.

### CD Responsibilities
- Poll GitHub Packages for new versions
- Automatically import new packages
- Automatically create a release when a new package arrives
- Automatically deploy to **DEV**
- Allow controlled promotion to **QA** and **PROD** via lifecycle rules

### Why Octopus handles CD
- Full audit trail of deployments and approvals
- Built‑in lifecycle management (DEV → QA → PROD)
- Environment‑specific configuration and variables
- Manual intervention steps for QA/PROD if required
- Zero dependency on GitHub Actions for deployment logic

---

## ✅ End‑to‑End Flow Summary

1. **Developer pushes code** to `feature/cicd` or `main`.
2. **GitHub Actions CI** runs:
   - Installs dependencies
   - Packages the `src` folder
   - Publishes the ZIP artifact to GitHub Packages
3. **Octopus External Feed** (GitHub Packages) detects the new package.
4. **Octopus Auto‑Release Creation**:
   - Creates a new release using the package version
   - Deploys automatically to **DEV**
5. **Lifecycle Promotion**:
   - QA deployment becomes available after DEV succeeds
   - PROD deployment becomes available after QA succeeds
   - Approvals or manual intervention steps can be added as needed

---

## ✅ Benefits of This CI/CD Model

### 🔒 Policy‑Compliant
- No GitHub → Octopus API calls
- All deployments controlled within Octopus

### 🧩 Clean Separation of Responsibilities
- GitHub Actions = Build  
- GitHub Packages = Artifact storage  
- Octopus Deploy = Release + Deploy + Promote  

### 🧾 Fully Auditable
- Octopus logs every deployment, promotion, and approval
- GitHub logs every build and artifact version

### 🔁 Reproducible and Consistent
- Build once → deploy the same artifact across all environments
- No environment drift

---
### nupkg

src/
    *.py
    requirements.txt
packages/
    <all wheels downloaded from JFrog>


### post deploy script 

Write-Host "Starting offline Python dependency installation..."

$requirements = ".\src\requirements.txt"
$packageDir   = ".\packages"

if (-Not (Test-Path $requirements)) {
    Write-Error "requirements.txt not found at $requirements"
    exit 1
}

if (-Not (Test-Path $packageDir)) {
    Write-Error "Offline package directory not found at $packageDir"
    exit 1
}

Write-Host "Installing dependencies from offline wheel cache..."
pip install `
    --no-index `
    --find-links $packageDir `
    -r $requirements

Write-Host "Offline dependency installation complete."



## ✅ Visual Overviews

```mermaid
flowchart LR

A[Developer Pushes Code] --> B[GitHub Actions CI]
B --> C[Build + Package Python Project]
C --> D[Publish ZIP to GitHub Packages]

D --> E[Octopus External Feed Polling]
E --> F[Octopus Detects New Package]
F --> G[Auto Create Release]
G --> H[Deploy to DEV]

H --> I[Promote to QA]
I --> J[Promote to PROD]


flowchart TD

    A[Developer Pushes Code<br/>main / feature / hotfix] --> B[GitHub Actions CI]

    subgraph CI[GitHub Actions Build Pipeline]
        B1[Validate Python Syntax]
        B2[Download Wheels from JFrog<br/>pip download → packages/]
        B3[Bundle src/ + packages/<br/>into .nupkg]
        B4[Publish .nupkg to GitHub Packages]
        B5[Create Octopus Release]

        B --> B1 --> B2 --> B3 --> B4 --> B5
    end

    B5 --> C[Octopus Deploy]

    subgraph CD[Octopus Deployment Process]
        C1[Fetch .nupkg from GitHub Packages]
        C2[Extract Package on Windows VM]
        C3[Post‑Deploy Script:<br/>pip install --no-index --find-links packages<br/>-r src/requirements.txt]
        C --> C1 --> C2 --> C3
    end

    C3 --> D[Offline Windows VM<br/>Fully Isolated Deployment]

sequenceDiagram
    autonumber

    participant Dev as Developer
    participant GH as GitHub Actions (CI)
    participant JF as JFrog Artifactory
    participant GP as GitHub Packages (NuGet Feed)
    participant OD as Octopus Deploy (CD)
    participant VM as Offline Windows VM

    Dev->>GH: Push code (main/feature/hotfix)

    GH->>GH: Validate Python syntax
    GH->>JF: pip download -r src/requirements.txt<br/>--index-url JFrog<br/>--trusted-host
    JF-->>GH: Return wheels (.whl) + tarballs

    GH->>GH: Bundle src/ + packages/ into .nupkg
    GH->>GP: Publish .nupkg to GitHub Packages
    GP-->>GH: Package stored

    GH->>OD: Create Octopus Release

    OD->>GP: Fetch .nupkg
    GP-->>OD: Return package

    OD->>VM: Deploy package to offline VM
    VM->>VM: Extract src/ and packages/

    VM->>VM: pip install --no-index<br/>--find-links packages<br/>-r src/requirements.txt

    VM-->>OD: Deployment complete

'''






