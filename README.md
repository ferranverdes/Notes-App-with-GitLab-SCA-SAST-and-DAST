# 🗒️ Notes App with GitLab SCA, SAST and DAST

## ⚡ TLDR

* Minimal Notes REST API (create/list notes) built with Express and Prisma.  
* Deployed to Google Cloud Run via Pulumi, orchestrated end-to-end by GitLab CI/CD.  
* Three fully isolated environments (**dev**, **stage**, **prod**) using separate GCP projects, credentials, and Pulumi stacks:

  ![Architecture overview][1]

* Data seed strategy per environment: **dev** is purged and reseeded on every deploy, **stage** is seeded while preserving existing data, and **prod** is never seeded.  
* Access control enforced with GCP IAM: **dev** is kept private, **stage** is only reachable by the GitLab DAST service account, and **prod** is publicly accessible.  
* Security integrated into the pipeline with GitLab **SCA, SAST, IaC scanning, Secret Detection, and DAST**, with **DAST running only on `main`** so findings appear in the Vulnerability Report dashboard.  
* Developer-friendly workflow: local development on PostgreSQL, isolated tests on SQLite, and a simple promotion model `feature/* → dev → main → prod`.

## 🧩 Project Context

A minimal REST API built to demonstrate secure, automated deployments on Google Cloud using Pulumi, GitLab CI/CD, and **multi-environment pipelines**.

This project is designed for learning purposes, showcasing how to integrate **GitLab's built-in SCA, SAST**, and **DAST** — helping developers understand how security automation fits into a real DevSecOps workflow.

The app exposes two endpoints:

* **POST `/notes`** → Creates a new note and returns `{ title, description }`.  
* **GET `/notes`** → Retrieves a list of all existing notes.  

It is intentionally simple: no validation, no authentication, and no dependencies beyond what is strictly required, with certain vulnerabilities deliberately included to be reported by automated scans.

## ☁️ Cloud Deployment (GitLab + Pulumi + GCP)

This section provides instructions for setting up credentials and initiating the automated CI/CD pipeline in GitLab.

### 1️⃣ Fork the repository

Fork this repository into your own GitLab workspace.  
You'll gain full control over the CI/CD pipelines, environment variables, and Pulumi configuration.

### 2️⃣ Create a Pulumi access token

Pulumi uses an access token to authenticate and manage your infrastructure state.

1. Go to your Pulumi account → **Personal access tokens**.  
2. Click **Create token** and copy the generated value.  
3. In your GitLab fork, go to **Settings → CI/CD → Variables** and add:
  
    * **Key:** `PULUMI_ACCESS_TOKEN`.
    * **Value:** (paste your Pulumi token).
    * **Environment:** "All (default)"
    * **Visibility:** ✅ **Masked**  
    * **Flags:** 🚫 **Unset the "Protect variable"** checkbox for demonstration purposes.

This allows Pulumi to manage infrastructure for all environments securely during CI/CD runs.

### 3️⃣ Create three Google Cloud projects

You'll need one Google Cloud project per environment.

| Environment | Example project ID |
|--------------|-------------------|
| Development  | `notes-dev-191823` |
| Staging      | `notes-stage-298734` |
| Production   | `notes-prod-716232` |

Create them in **Google Cloud Console → Manage Resources → Create Project**.

### 4️⃣ Create a service account for each project

Each environment needs a service account with permissions to deploy via Pulumi.

1. Go to **IAM & Admin → Service Accounts → Create service account**.  
2. Name it `gitlab-build-sa`.  
3. Assign the following role:

    * `Owner` (for demonstration purposes only — in real-world scenarios, grant the minimum roles required).  

4. Generate a **JSON key** and download it to your local machine.

#### 🔐 Additional service account for DAST (staging only)

In the **staging** project, create an extra service account named `gitlab-dast-sa`.  
This account will be used by GitLab DAST to authenticate and scan your deployed service.

1. Go to **IAM & Admin → Service Accounts → Create service account**.  
2. Name it `gitlab-dast-sa`.  
3. Do not assign any role.
4. Generate a **JSON key** and download it to your local machine.

### 5️⃣ Store credentials in GitLab (Base64 encoded)

Each environment (`dev`, `stage`, `prod`) needs its own encoded service account key stored in GitLab as a CI/CD variable.
The staging environment additionally uses a separate key for DAST scans.

First, encode your JSON keys locally:

  ```bash
  base64 ~/Downloads/notes-dev-191823-015df9e2cf47.json
  ```

Then, in **GitLab → Settings → CI/CD → Variables**, create the following variables. Each one must be scoped to its corresponding environment (`dev`, `stage`, or `prod`).

Define the variable for each environment as follows:

| Key | Environment Scope | Visibility | Flags |
|-----|--------------------|-------------|--------|
| `GOOGLE_CREDENTIALS_B64` | dev | ✅ Masked | 🚫 Unset "Protect variable" |
| `GOOGLE_CREDENTIALS_B64` | stage | ✅ Masked | 🚫 Unset "Protect variable" |
| `GOOGLE_CREDENTIALS_DAST_B64` | stage | ✅ Masked | 🚫 Unset "Protect variable" |
| `GOOGLE_CREDENTIALS_B64` | prod | ✅ Masked | 🚫 Unset "Protect variable" |

This setup allows each pipeline environment to access the correct credentials while keeping them hidden from logs. All variables are left unprotected for demonstration purposes only.

### 6️⃣ Update Pulumi configuration files

Each environment has its own Pulumi configuration file under both folders:

```text
environments/build
environments/deploy
```

Update the following files with your **project's ID**:

* `Pulumi.dev.yaml`
* `Pulumi.main.yaml`, intended for the configuration of the staging environment.
* `Pulumi.prod.yaml`

These files define which GCP project and resources are used during each environment's deployment.

#### 🔐 Environment Accessibility (IAM Controls)

Access to each deployed Cloud Run service is enforced through **GCP IAM–based access control**:

* **dev** → Not accessible to anyone (intended to later grant access to a group of developers).
* **stage** → Accessible **only** by the GitLab DAST service account. In real-world scenarios, a developer group can also be granted access.
* **prod** → Publicly accessible.

### 7️⃣ Understand branch behaviour

| Branch | Purpose | Pipeline Behaviour |
|---------|----------|--------------------|
| `main` | Default branch mapped to staging environment | Runs `pulumi up --yes` for stage |
| `dev` | Mapped to development environment | Runs `pulumi up --yes` for dev |
| `prod` | Mapped to production environment | Runs `pulumi up --yes` for prod |

**Important:**  
Only the **main** branch runs **DAST scans**, because GitLab only aggregates findings from the **default branch** into the **Vulnerability Report Dashboard**.

### 8️⃣ Typical workflow pattern

This project suggests a simple and clear workflow where code moves progressively from feature branches to development, then staging, and finally production.
The **main** branch serves as the central, stable source of truth for the project.

```text
feature/*  →  dev  →  main  →  prod
```

1. Developers merge feature branches into `dev`.
2. When validated: merge `dev` → `main`, which deploys to staging and runs DAST.
3. When approved for release: merge `main` → `prod`, which deploys to production.

Each merge triggers the pipeline for that environment automatically.

### (Optional) Manually trigger the first deployment

This step is **optional** but recommended to verify your setup before pushing any code to GitLab.

You can simulate the pipeline locally to confirm that Pulumi and your credentials are correctly configured.  
To do this, you can temporarily copy your **development environment** JSON key to the following path:

```bash
cp ~/Downloads/notes-dev-191823-015df9e2cf47.json environments/credentials/service-account-key.json
```

Then, run the following command to trigger the deployment locally. It will execute the `build`, `deploy`, and `seed` stages, deploying your stack to Google Cloud:

```bash
npx -y gitlab-ci-local run build deploy seed --privileged \
  --variable "PULUMI_ACCESS_TOKEN=<your-pulumi-token>" \
  --variable "GOOGLE_CREDENTIALS_B64=$(base64 environments/credentials/service-account-key.json)"
```

### (Optional) Manually query the staging environment with the DAST service account

Once the staging environment is deployed, you can manually call the Cloud Run service using the `gitlab-dast-sa` service account (the same identity used by GitLab DAST) it has access.

1. Set the Cloud Run URL from the Pulumi output (replace with your actual URL):

    ```bash
    export CLOUD_RUN_URL="https://notes-service-XXXX-ew.a.run.app"
    ```

2. Activate the `gitlab-dast-sa` service account locally using its JSON key:

    ```bash
    gcloud auth activate-service-account \
      --key-file=~/Downloads/notes-stage-298734-e6485c9ac1ea.json
    ```

3. Generate an identity token for the Cloud Run service:

    ```bash
    export ACCESS_TOKEN=$(gcloud auth print-identity-token \
      --audiences="$CLOUD_RUN_URL")
    ```

4. Call the staging endpoint with `curl`:

   * List notes:

   ```bash
   curl -H "Authorization: Bearer $ACCESS_TOKEN" \
        -H "Content-Type: application/json" \
        "$CLOUD_RUN_URL/notes"
   ```

   * Create a new note:

   ```bash
   curl -X POST "$CLOUD_RUN_URL/notes" \
     -H "Authorization: Bearer $ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"title":"DAST test", "description":"Testing staging access via gitlab-dast-sa"}'
   ```

If everything is configured correctly, these requests will succeed even though the Cloud Run service is **not publicly accessible**, because they are authenticated as `gitlab-dast-sa@<staging-project>.iam.gserviceaccount.com`.

## 💻 Local Development Setup

You can also run the Notes API locally to test new changes before pushing them to GitLab.

### 1️⃣ Clone the repository

```bash
git clone https://gitlab.com/<your-username>/notes-app.git
cd notes-app
```

### 2️⃣ Install dependencies

```bash
cd app
npm install
```

This installs:

* **express** → HTTP server  
* **dotenv** → Environment management  
* **@prisma/client** & **prisma** → Database ORM and schema tools  

Prisma automatically generates its client via the `postinstall` script.

### 3️⃣ Configure local environment variables

Duplicate `.env.sample` as `.env` and fill it with your PostgreSQL details:

```env
DB_USER=postgres
DB_PASSWORD=yourpassword
DB_HOST=localhost
DB_PORT=5432
DB_NAME=notes_local

DATABASE_URL="postgresql://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}?schema=public"

PORT=3000
```

💡 **Tip:** Start PostgreSQL and create your database if it doesn't exist:

```bash
createdb notes_local
```

### 4️⃣ Apply Prisma migrations

```bash
npm run migrate
```

This command will:

* Create the `Note` table in PostgreSQL.  
* Generate the Prisma client under `node_modules/@prisma/client`.

If needed, you can regenerate the client manually:

```bash
npm run generate
```

### 5️⃣ Run the server

```bash
npm run dev
```

Expected output:

```text
🚀 Notes API running on http://localhost:3000
```

### 6️⃣ Check the endpoints

#### ➕ Create a new note

```bash
curl -X POST http://localhost:3000/notes \
  -H "Content-Type: application/json" \
  -d '{"title":"Meeting Notes", "description":"Discuss Q4 roadmap"}'
```

✅ **Expected response:**

```json
{
  "title": "Meeting Notes", 
  "description": "Discuss Q4 roadmap"
}
```

#### 📋 List all notes

```bash
curl http://localhost:3000/notes
```

✅ **Expected response:**

```json
[
  {

    "id": 1,
    "title": "Meeting Notes",
    "description": "Discuss Q4 roadmap",
    "createdAt": "2025-10-23T09:00:00.000Z"

  }, 
  {

    "id": 2,
    "title": "Brainstorm",
    "description": "Ideas for next sprint",
    "createdAt": "2025-10-23T09:05:00.000Z"

  }
]
```

## 🧪 Running Tests Locally

To avoid touching your PostgreSQL database during testing, this setup uses an isolated SQLite instance.

### 1️⃣ Run the tests

```bash
cd app
npm test
```

When you run the tests:

1. A temporary `test.db` is created.  
2. A separate Prisma test client (`@prisma/test/client`) is generated.  
3. All API requests run using SQLite.  
4. The file is deleted after completion.

✅ **Result:** fast, repeatable, and safe testing for local development.

## 🛠️ What the `.gitlab-ci.yml` File Contains

The `.gitlab-ci.yml` file in this repository orchestrates the complete CI/CD and security workflow.

### 🏗️ Pipeline Stages

* **`test`** → Runs the application test suite (against SQLite) and multiple static code scans.
* **`build`** → Uses Pulumi to build the application container image and push it to Artifact Registry.
* **`deploy`** → Uses Pulumi to deploy infrastructure and the Notes API to the target environment (dev, stage, prod).
* **`dast`** → Runs GitLab DAST scans against the staging environment (only on `main` branch).

![Pipeline jobs][2]

### 🔍 Security Scanning Jobs

The pipeline reuses GitLab's built-in security templates to provide:

* **Dependency Scanning (SCA)** — Finds vulnerabilities in third-party libraries.  
* **Secret Detection** — Detects hard-coded secrets in the repository.  
* **SAST (Semgrep)** — Analyses application source code for vulnerabilities.  
* **IaC SAST (KICS)** — Scans Pulumi and IaC artefacts for misconfigurations.  
* **API Security / DAST** — Runs dynamic tests against the running staging service.

These jobs run automatically on relevant branches and upload results back to GitLab.

### 🌐 Environment-Specific Deployments

Based on the branch:

* `dev` → Deploys to the **development** environment using `Pulumi.dev.yaml`.  
* `main` → Deploys to the **staging** environment using `Pulumi.main.yaml` and triggers **DAST**.  
* `prod` → Deploys to the **production** environment using `Pulumi.prod.yaml`.  

Each environment uses its own `GOOGLE_CREDENTIALS_B64` (and `GOOGLE_CREDENTIALS_DAST_B64` only for staging).

### 🌱 Database Seeding Logic

The deployment jobs also control seeding:

* **dev** → Purges existing data and re-seeds from scratch on every deploy.  
* **stage** → Seeds new records without removing existing staging data.  
* **prod** → Never seeds, to protect real production data.

In summary, `.gitlab-ci.yml` ties together testing, building, deployment, and security scanning into a single automated workflow.

## 🛡️ GitLab Vulnerability Report Dashboard

The **Vulnerability Report Dashboard** in GitLab aggregates security findings produced by the pipeline.

![Vulnerability report dashboard][3]

GitLab aggregates findings into the dashboard **only from the default branch** (`main` in this project).

## 🔐 Security Integrations

All previously described security features are already integrated:

* **🔍 SCA (Software Composition Analysis)** — Automatically detects vulnerable dependencies.  
* **🧠 SAST (Static Application Security Testing)** — Scans your code for vulnerabilities before deployment.  
* **🧪 DAST (Dynamic Application Security Testing)** — Tests your live staging endpoints for real-world attack patterns.  
* **🔑 Secret Detection** — Identifies hard-coded credentials and secrets.  
* **📦 IaC SAST (KICS)** — Scans your Pulumi and other IaC artefacts for misconfigurations.  

GitLab pipelines run these scans automatically for each relevant branch, allowing you to **see and fix security findings directly within your CI/CD workflow and the Vulnerability Report Dashboard**.

## 🧩 Tech Stack Overview

| Layer | Technology | Purpose |
|--------|-------------|----------|
| Backend | Express.js | Minimal REST API |
| ORM | Prisma | Database schema and data access |
| Database | PostgreSQL (Cloud), SQLite (Test) | Data persistence |
| IaC | Pulumi | Cloud infrastructure automation |
| CI/CD | GitLab | Pipeline orchestration |
| Platform | Google Cloud Run | Serverless application hosting |

## 🎯 Learning Objectives

This project demonstrates how to:

* Build and deploy cloud-native applications using **Pulumi and GitLab CI/CD**.  
* Create **multi-environment pipelines** for `dev`, `stage`, and `prod`.  
* Manage **secure credentials** and environment-specific configurations.  
* Integrate **security automation** with GitLab's SCA, SAST, DAST, Secret Detection, and IaC scanning.  

## 🧭 Summary

Once configured:

1. Pushing to `dev`, `main`, or `prod` triggers a full automated deployment to the corresponding environment.  
2. Local development is simple and uses PostgreSQL, while tests remain isolated with SQLite.  
3. Database seeding and data retention are controlled per environment (dev, stage, prod).  
4. Security scans run automatically and their findings appear in GitLab’s Vulnerability Report Dashboard for actionable remediation.  

This project is designed for learning purposes, demonstrating how **cloud deployments**, **infrastructure as code**, and **security automation** can be integrated into a developer-friendly workflow.

[1]: https://gitlab.com/ferran.verdes/static/-/raw/main/images/notes-app-with-gitlab-sca-sast-and-dast-architecture-overview.png
[2]: https://gitlab.com/ferran.verdes/static/-/raw/main/images/notes-app-with-gitlab-sca-sast-and-dast-pipeline-jobs.png
[3]: https://gitlab.com/ferran.verdes/static/-/raw/main/images/notes-app-with-gitlab-sca-sast-and-dast-vulnerability-report.png
