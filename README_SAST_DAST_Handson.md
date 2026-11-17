# SAST and DAST Hands-on Lab

A practical guide to setting up a local Todo application and integrating
**SAST (Static Application Security Testing)** with SonarQube and **DAST
(Dynamic Application Security Testing)** with OWASP ZAP using GitHub
Actions and a Windows self-hosted runner.

> **Lab architecture:** Windows laptop + local MongoDB + Angular
> frontend + Node.js backend + Docker Desktop + SonarQube Server + OWASP
> ZAP + GitHub Actions self-hosted runner.

------------------------------------------------------------------------

## 1. Lab architecture and request flow

``` text
Developer laptop
│
├── Todo frontend (Angular) ─────────────── http://localhost:4200
│             │
│             └── Calls backend API
│
├── Todo backend (Node.js) ─────────────── http://localhost:3001
│             │
│             └── Connects to local MongoDB
│
├── MongoDB ────────────────────────────── local database: todo_db
│
├── Docker Desktop
│    ├── SonarQube Server ──────────────── http://localhost:9000
│    ├── PostgreSQL (SonarQube database)
│    └── OWASP ZAP (temporary scan container)
│
└── GitHub Actions self-hosted runner
     ├── Checks out the repository
     ├── Runs SonarScanner for frontend and backend (SAST)
     ├── Checks that the applications are reachable
     ├── Runs OWASP ZAP against frontend and backend (DAST)
     └── Uploads ZAP HTML reports as workflow artifacts
```

### What each tool does

  -----------------------------------------------------------------------
  Tool                                Purpose
  ----------------------------------- -----------------------------------
  Git                                 Version control for application
                                      code and workflow files

  GitHub                              Hosts the repository and GitHub
                                      Actions workflow

  Visual Studio Code                  Edits and manages the application
                                      and workflow files

  Node.js and npm                     Runs the backend and manages
                                      JavaScript dependencies

  MongoDB                             Stores Todo application data

  Docker Desktop                      Runs SonarQube, PostgreSQL and
                                      OWASP ZAP containers

  SonarQube                           Analyses source code for
                                      code-quality and security issues
                                      (SAST)

  SonarScanner                        Runs from the GitHub Actions job
                                      and sends analysis to SonarQube
                                      Server

  OWASP ZAP                           Tests running web applications for
                                      potential security issues (DAST)

  GitHub Actions runner               Executes the CI/CD workflow on the
                                      Windows machine
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 2. Install the prerequisites

Install the following on the Windows laptop:

1.  **Git** -- https://git-scm.com/downloads
2.  **GitHub account** -- https://github.com/
3.  **Visual Studio Code** -- https://code.visualstudio.com/
4.  **Node.js (LTS)** -- https://nodejs.org/
5.  **MongoDB Community Server** --
    https://www.mongodb.com/try/download/community
6.  **MongoDB Shell (`mongosh`)** --
    https://www.mongodb.com/try/download/shell
7.  **Docker Desktop** --
    https://www.docker.com/products/docker-desktop/
8.  **Angular CLI** -- install after Node.js using the command below.

Verify the installations in PowerShell:

``` powershell
git --version
node --version
npm --version
mongosh --version
docker --version
docker compose version
```

Install Angular CLI:

``` powershell
npm install -g @angular/cli
ng version
```

Start Docker Desktop and make sure the Docker engine is running:

``` powershell
docker info
```

> **Note:** Docker Desktop must remain running while the workflow uses
> Docker to launch OWASP ZAP.

------------------------------------------------------------------------

## 3. Create the GitHub repository

1.  Sign in to GitHub.
2.  Create a new repository, for example: `SAST_and_DAST_Handson`.
3.  Choose the repository visibility appropriate for your lab.
4.  Do not commit passwords, API keys, SonarQube tokens or other
    secrets.
5.  Clone the repository to your laptop.

Example:

``` powershell
cd "$HOME\Documents\My_Learnings\2026"
git clone https://github.com/<GITHUB_USERNAME>/SAST_and_DAST_Handson.git
cd SAST_and_DAST_Handson
```

Replace `<GITHUB_USERNAME>` with your GitHub username.

------------------------------------------------------------------------

## 4. Create a GitHub Personal Access Token (PAT)

A PAT can be used to authenticate Git operations over HTTPS when Git
prompts for credentials.

1.  In GitHub, open **Settings → Developer settings → Personal access
    tokens**.
2.  Create a fine-grained token where possible.
3.  Give it access only to the required repository and the permissions
    needed for your Git operations.
4.  Copy the token when GitHub displays it. Store it securely.

### Save credentials using Git Credential Manager

Check whether Git Credential Manager is available:

``` powershell
git credential-manager --version
```

Set Git to use Git Credential Manager:

``` powershell
git config --global credential.helper manager
```

When you perform an authenticated Git operation, enter your GitHub
username and use the PAT as the password if prompted. Git Credential
Manager can securely save the credentials.

**Do not** put the PAT directly into the remote URL, source code,
workflow YAML or a README file. Do not share it in screenshots or chat.

------------------------------------------------------------------------

## 5. Arrange the local project folders

Inside the cloned repository, arrange the application folders as
follows. The folder names below are examples; keep them consistent with
the paths used in your workflow.

``` text
SAST_and_DAST_Handson/
│
├── Mean_Stack_Todo_App_Backend-main/
│   ├── package.json
│   ├── package-lock.json
│   ├── dbConfig.js
│   ├── index.js
│   └── ...
│
├── Mean_Stack_Todo_App_Frontend-main/
│   ├── package.json
│   ├── package-lock.json
│   ├── angular.json
│   └── src/
│       └── ...
│
├── SonarQube/
│   └── docker-compose.yml
│
└── .github/
    └── workflows/
        └── sast-dast.yml
```

The frontend and backend folders must be inside the repository root if
you use the single-checkout workflow described later.

------------------------------------------------------------------------

## 6. Configure the backend to use local MongoDB

1.  Make sure the MongoDB service is running.
2.  Open the backend project.
3.  Find `dbConfig.js`.
4.  Update the MongoDB connection string to use the local MongoDB
    instance and the `todo_db` database.

A typical local connection string is:

``` javascript
mongodb://127.0.0.1:27017/todo_db
```

Use the connection-string variable and configuration structure already
expected by the application's `dbConfig.js`. Do not replace unrelated
application configuration blindly.

### Create/check the database

Open PowerShell and run:

``` powershell
mongosh
```

In the MongoDB shell:

``` javascript
use todo_db
db.createCollection("connection_test")
show dbs
```

MongoDB creates a database when data is first stored in it. Creating a
collection as above makes the database visible. If the application
already creates its own collections, you can let the application do that
instead.

Exit the shell:

``` javascript
exit
```

> If MongoDB authentication is enabled, include the appropriate
> credentials in the connection string and keep them out of Git. For a
> local-only learning setup without authentication, the example
> connection string may be sufficient.

------------------------------------------------------------------------

## 7. Install and run the backend application

Open PowerShell and navigate to the backend directory:

``` powershell
cd "<REPOSITORY_PATH>\Mean_Stack_Todo_App_Backend-main"
```

Install dependencies:

``` powershell
npm install
```

Start the backend:

``` powershell
node index.js
```

Keep this terminal open while testing the application.

Confirm that the backend is listening on port `3001`. The exact startup
message depends on the application.

Test the API endpoint in another PowerShell window:

``` powershell
Invoke-WebRequest -Uri "http://localhost:3001/tasks" -UseBasicParsing
```

The endpoint should return a successful response. If it returns `404`,
inspect the backend route definitions and use the correct API path. The
route may be `/task` or `/tasks`; use the actual route defined by the
application consistently in the frontend, readiness check and ZAP scan.

------------------------------------------------------------------------

## 8. Configure and run the frontend application

1.  Open the frontend project.
2.  Find `src/app/service/webrequest.service.ts`.
3.  Update the backend API base URL to point to the local backend, using
    the actual route and port configured by the application.

For example, the API base URL may use:

``` text
http://localhost:3001
```

Do not blindly replace the URL if the service already appends `/tasks`;
avoid accidentally creating a duplicated path such as `/tasks/tasks`.

Open another PowerShell window and navigate to the frontend directory:

``` powershell
cd "<REPOSITORY_PATH>\Mean_Stack_Todo_App_Frontend-main"
```

Install dependencies:

``` powershell
npm install
```

Start Angular:

``` powershell
ng serve --host 0.0.0.0 --port 4200
```

Keep the terminal open.

Test the frontend in a browser:

``` text
http://localhost:4200/
```

The frontend should load and should be able to communicate with the
backend. If the browser reports a CORS error, configure the backend to
allow the frontend development origin (`http://localhost:4200`) for the
required API methods. Do not use unrestricted CORS as a production
solution.

### Local application checks

  -------------------------------------------------------------------------------
  Component               URL                             Expected result
  ----------------------- ------------------------------- -----------------------
  Frontend                `http://localhost:4200/`        Angular application
                                                          loads

  Backend                 `http://localhost:3001/tasks`   Actual API route
                                                          returns a successful
                                                          response

  MongoDB                 `127.0.0.1:27017`               Backend can connect to
                                                          the database
  -------------------------------------------------------------------------------

------------------------------------------------------------------------

## 9. Run SonarQube Server and PostgreSQL using Docker Compose

SonarQube Server stores analysis results and provides the web interface.
PostgreSQL is its supporting database. The SonarScanner used in the
GitHub Actions workflow connects to this server; the scan action does
not start the SonarQube Server.

Create a folder named `SonarQube` in the repository root:

``` powershell
mkdir SonarQube
cd SonarQube
```

Create `docker-compose.yml` with the following content:

``` yaml
services:
  sonarqube-db:
    image: postgres:16-alpine
    container_name: sonarqube-db
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: "ReplaceWithAStrongLocalPassword"
      POSTGRES_DB: sonarqube
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    depends_on:
      - sonarqube-db
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://sonarqube-db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: "ReplaceWithAStrongLocalPassword"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    restart: unless-stopped

volumes:
  postgres_data:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
```

Replace the example database password with a strong local lab password.
The password must match in both services. Do not use a real production
password in a committed compose file; for a shared or production
environment, use a secrets-management approach.

### Start SonarQube and PostgreSQL

From the `SonarQube` folder:

``` powershell
docker compose up -d
```

Check container status:

``` powershell
docker compose ps
```

View SonarQube logs while it starts:

``` powershell
docker compose logs -f sonarqube
```

Wait until startup completes, then open:

``` text
http://localhost:9000
```

Sign in and change the default administrator password when prompted.

### Create SonarQube projects

In the SonarQube portal, create two projects with these project keys:

  Project         Project key
  --------------- -----------------
  Todo frontend   `todo-frontend`
  Todo backend    `todo-backend`

The project keys must match the values in the GitHub Actions workflow.

------------------------------------------------------------------------

## 10. Create a SonarQube token

The GitHub Actions scanner needs a token to authenticate with SonarQube
Server.

1.  Open `http://localhost:9000`.
2.  Sign in.
3.  Open **My Account → Security** (or the equivalent account security
    page in your SonarQube version).
4.  Generate a token with a descriptive name, such as
    `github-actions-sast`.
5.  Copy the token immediately and store it securely.
6.  Ensure the token's user has permission to execute analysis on both
    `todo-frontend` and `todo-backend`.

**Important:** Never hardcode the token in the workflow. If a token has
been exposed in a chat, log, screenshot or committed file, revoke it and
generate a replacement.

### Add GitHub Actions secrets

In the GitHub repository:

1.  Open **Settings → Secrets and variables → Actions**.
2.  Select **New repository secret**.
3.  Add:

  Secret name        Value
  ------------------ -------------------------------------
  `SONAR_TOKEN`      The newly generated SonarQube token
  `SONAR_HOST_URL`   `http://localhost:9000`

`localhost:9000` works here because the self-hosted runner and SonarQube
Server are on the same Windows machine. If they run on different
machines, use a server address reachable from the runner.

------------------------------------------------------------------------

## 11. Run OWASP ZAP using Docker

OWASP ZAP is used for DAST. In this lab, ZAP is launched as a temporary
Docker container by the GitHub Actions workflow. You do not need to keep
a separate ZAP server container running.

First, confirm Docker can pull and run the ZAP image:

``` powershell
docker pull ghcr.io/zaproxy/zaproxy:stable
```

You can run a manual baseline scan from PowerShell after the frontend is
running:

``` powershell
docker run --rm ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://host.docker.internal:4200
```

For this particular Windows/Docker Desktop setup, `host.docker.internal`
may not work for ZAP even if it works for another container. In earlier
lab testing, the Docker gateway address `192.168.65.254` successfully
reached the frontend. Verify the current gateway and use the address
that works on your machine:

``` powershell
docker run --rm ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://192.168.65.254:4200
```

For the backend, use the actual working API route, for example:

``` powershell
docker run --rm ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://192.168.65.254:3001/tasks
```

The IP above is specific to the earlier lab environment and may change.
Do not assume it is a permanent Docker Desktop address.

### What does the ZAP baseline scan do?

-   Crawls the target that you provide.
-   Performs passive security checks on the responses it observes.
-   Reports potential security issues and hardening recommendations.
-   Does not perform a full active penetration test.

Scanning one API path does not automatically test every backend
endpoint. For broader API coverage, use an OpenAPI/Swagger definition
with an appropriate ZAP API scan, or otherwise provide ZAP with the full
API scope.

------------------------------------------------------------------------

## 12. Create a Windows self-hosted GitHub Actions runner

The self-hosted runner executes the workflow on your laptop. It must be
online whenever the workflow runs.

1.  Open the GitHub repository.
2.  Go to **Settings → Actions → Runners**.
3.  Select **New self-hosted runner**.
4.  Select **Windows** and the appropriate architecture (normally x64).
5.  Follow the commands displayed by GitHub to download, configure and
    run the runner.

Open PowerShell with elevated privileges if required by the installation
method or if you intend to install the runner as a Windows service.
Follow the commands and instructions shown by GitHub for your runner.

During configuration, give the runner a clear name, such as
`sast_dast_runner`, and add any custom labels you want to use.

Start the runner using the generated `run.cmd` command, or install and
start it as a service using GitHub's instructions.

### Verify runner labels

The workflow uses:

``` yaml
runs-on: [self-hosted, Windows, X64]
```

These are standard labels for a Windows x64 self-hosted runner. The
runner's display name is not automatically a label. If you want to
target a custom label such as `sast_dast_runner`, add that label during
runner configuration and include it in `runs-on`.

### Runner prerequisites

The runner machine must have:

-   Docker Desktop running and accessible to the runner account.
-   SonarQube Server running and reachable at `http://localhost:9000`.
-   Frontend running on port `4200`.
-   Backend running on port `3001`.
-   Network access to GitHub and the required container/image
    registries.

The current workflow checks whether the applications are running; it
does not start them automatically.

------------------------------------------------------------------------

## 13. Create the GitHub Actions workflow

In the repository root, create:

``` text
.github/workflows/sast-dast.yml
```

The workflow below assumes:

-   Frontend and backend are in the same GitHub repository.
-   The folders have the names shown in the example project structure.
-   Both applications are already running on the self-hosted runner
    machine.
-   SonarQube Server is running on the same machine as the runner.
-   The backend GET endpoint is `/tasks` and returns a successful
    response.
-   `192.168.65.254` is the currently verified Docker gateway address
    for ZAP to reach the Windows host. Change it if your environment
    uses a different reachable address.

``` yaml
name: SAST and DAST Security Scan

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read

jobs:
  security-scan:
    name: SAST and DAST
    runs-on: [self-hosted, Windows, X64]
    timeout-minutes: 30

    steps:
      # Step 1: Checkout source code
      - name: Checkout repository
        uses: actions/checkout@v4

      # Step 2: SAST - Angular frontend
      - name: SAST - Frontend SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v8.3.0
        with:
          projectBaseDir: Mean_Stack_Todo_App_Frontend-main
          skipSignatureVerification: true
          args: >
            -Dsonar.projectKey=todo-frontend
            -Dsonar.projectName=Todo-App-Frontend
            -Dsonar.sources=src
            -Dsonar.exclusions=**/*.spec.ts
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

      # Step 3: SAST - Node.js backend
      - name: SAST - Backend SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v8.3.0
        with:
          projectBaseDir: Mean_Stack_Todo_App_Backend-main
          skipSignatureVerification: true
          args: >
            -Dsonar.projectKey=todo-backend
            -Dsonar.projectName=Todo-App-Backend
            -Dsonar.sources=.
            -Dsonar.exclusions=**/node_modules/**,**/images/**,**/Kubernetes_manifests/**
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

      # Step 4: Check that both applications are reachable
      - name: Check frontend and backend availability
        shell: powershell
        run: |
          $targets = @(
            "http://localhost:4200/",
            "http://localhost:3001/tasks"
          )

          foreach ($url in $targets) {
            Write-Host "Checking $url"
            $available = $false

            for ($i = 1; $i -le 12; $i++) {
              try {
                $response = Invoke-WebRequest `
                  -Uri $url `
                  -UseBasicParsing `
                  -TimeoutSec 5

                Write-Host "Available: $url (HTTP $([int]$response.StatusCode))"
                $available = $true
                break
              }
              catch {
                Write-Host "Waiting for application... Attempt $i of 12"
                Start-Sleep -Seconds 5
              }
            }

            if (-not $available) {
              throw "Application is not reachable: $url"
            }
          }

      # Step 5: DAST - Angular frontend
      - name: DAST - Frontend OWASP ZAP Scan
        shell: powershell
        run: |
          docker run --rm `
            -v "${{ github.workspace }}:/zap/wrk/:rw" `
            ghcr.io/zaproxy/zaproxy:stable `
            zap-baseline.py `
            -t http://192.168.65.254:4200 `
            -r /zap/wrk/frontend-zap-report.html `
            -I

          if ($LASTEXITCODE -ne 0) {
            exit $LASTEXITCODE
          }

      # Step 6: DAST - Node.js backend
      - name: DAST - Backend OWASP ZAP Scan
        shell: powershell
        run: |
          docker run --rm `
            -v "${{ github.workspace }}:/zap/wrk/:rw" `
            ghcr.io/zaproxy/zaproxy:stable `
            zap-baseline.py `
            -t http://192.168.65.254:3001/tasks `
            -r /zap/wrk/backend-zap-report.html `
            -I

          if ($LASTEXITCODE -ne 0) {
            exit $LASTEXITCODE
          }

      # Step 7: Upload ZAP HTML reports
      - name: Upload ZAP HTML reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: OWASP-ZAP-Security-Reports
          path: |
            frontend-zap-report.html
            backend-zap-report.html
          if-no-files-found: warn
          retention-days: 14
```

### Important workflow notes

-   `SonarSource/sonarqube-scan-action` runs the scanner; it does not
    start the SonarQube Server.
-   `SONAR_TOKEN` is read from GitHub Actions secrets. Never paste the
    actual token into the YAML file.
-   `skipSignatureVerification: true` is a lab convenience. Prefer
    installing GPG on the runner and verifying the scanner download in a
    hardened setup.
-   `shell: powershell` uses Windows PowerShell. This avoids the earlier
    `pwsh: command not found` error on a runner that does not have
    PowerShell 7 installed.
-   `-I` tells the ZAP baseline scan not to fail the job solely because
    of WARN findings. It does not mean the warnings are fixed or
    unimportant.
-   The ZAP reports are written into the mounted GitHub workspace and
    uploaded as workflow artifacts.
-   The ZAP target must be reachable from inside the Docker container,
    not merely from the Windows host.
-   If frontend and backend are in separate GitHub repositories, this
    single-checkout workflow must be adapted to check out both
    repositories or split into separate workflows.

------------------------------------------------------------------------

## 14. Run the GitHub Actions pipeline

Before running the workflow, confirm:

-   [ ] GitHub repository contains both application folders and the
    workflow file.
-   [ ] MongoDB is running and the backend can connect to `todo_db`.
-   [ ] Backend is running and its configured API endpoint responds
    successfully.
-   [ ] Frontend is running and can call the backend.
-   [ ] SonarQube and PostgreSQL containers are healthy/running.
-   [ ] Both SonarQube projects exist.
-   [ ] GitHub Actions secret `SONAR_TOKEN` is configured and authorized
    for both projects.
-   [ ] GitHub Actions secret `SONAR_HOST_URL` is configured.
-   [ ] The self-hosted runner is online.
-   [ ] Docker Desktop is running and the runner can execute `docker`
    commands.
-   [ ] The ZAP Docker target address has been verified.

Commit and push the workflow:

``` powershell
git status
git add .
git commit -m "Add SAST and DAST GitHub Actions workflow"
git push origin main
```

Or run it manually:

1.  Open the repository on GitHub.
2.  Select **Actions**.
3.  Select **SAST and DAST Security Scan**.
4.  Select **Run workflow**.
5.  Choose the `main` branch and start the workflow.

------------------------------------------------------------------------

## 15. Review the scan results

### SonarQube (SAST)

Open:

``` text
http://localhost:9000
```

Select the frontend or backend project and review:

-   Bugs
-   Vulnerabilities
-   Security Hotspots
-   Code Smells
-   Quality Gate status

A Security Hotspot is code that requires review; it is not automatically
a confirmed vulnerability. Review the finding and its context.

### OWASP ZAP (DAST)

In GitHub:

1.  Open the completed workflow run.
2.  Open the **Artifacts** section.
3.  Download `OWASP-ZAP-Security-Reports`.
4.  Extract the ZIP file.
5.  Open `frontend-zap-report.html` and `backend-zap-report.html` in a
    browser.

Review each finding's risk, confidence, evidence and affected URL.
Validate findings before treating them as confirmed vulnerabilities. A
scan with no FAIL findings does not prove that an application is secure.

------------------------------------------------------------------------

## 16. Troubleshooting

  -----------------------------------------------------------------------
  Error / symptom                     What to check
  ----------------------------------- -----------------------------------
  `GPG is not available`              Install GPG and make it available
                                      in PATH, or use
                                      `skipSignatureVerification: true`
                                      for the lab

  `pwsh: command not found`           Use `shell: powershell`, or install
                                      PowerShell 7 and ensure `pwsh` is
                                      in PATH

  SonarQube connection refused        Confirm SonarQube container is
                                      running and `http://localhost:9000`
                                      is reachable from the runner

  SonarQube                           Check token validity, secret name
  authentication/authorization        and project analysis permissions
  failure                             

  Frontend/backend availability check Start the application, check its
  fails                               port and verify the exact API route

  ZAP cannot connect to target        Check Docker Desktop networking and
                                      test the target URL from a
                                      temporary container

  ZAP report not uploaded             Check the Docker bind mount, report
                                      path and artifact upload step

  Backend scan finds only 404 pages   Confirm the target is a real API
                                      route; a root URL returning 404 is
                                      not a useful API scan target

  MongoDB connection failure          Check MongoDB service status, port,
                                      database URI and authentication
                                      settings
  -----------------------------------------------------------------------

Useful commands:

``` powershell
# Docker containers
docker ps -a

# SonarQube logs
docker logs sonarqube

# PostgreSQL logs
docker logs sonarqube-db

# Check local application ports
Test-NetConnection localhost -Port 4200
Test-NetConnection localhost -Port 3001
Test-NetConnection localhost -Port 9000

# Check the Docker engine
docker info
```

------------------------------------------------------------------------

## 17. Security and lab limitations

1.  Use this setup for learning and testing, not as a production
    security architecture without further hardening.
2.  Do not commit PATs, SonarQube tokens, database passwords or other
    credentials.
3.  Revoke and replace any credential that has been exposed.
4.  Keep the SonarQube Server and its database persistent; do not remove
    their named volumes when you want to retain analysis data.
5.  A ZAP baseline scan is a useful passive DAST starting point, not a
    full penetration test.
6.  Scanning one frontend URL or one backend endpoint does not guarantee
    complete application coverage.
7.  Run scans against an application and environment you own or are
    explicitly authorized to test.
8.  Review and validate scanner findings; automated tools can report
    false positives and can miss issues.
9.  The Angular development server is suitable for a lab. For more
    realistic testing, scan a production-like build deployed to a
    controlled test environment.
10. The example workflow assumes the applications are already running on
    the self-hosted runner machine. A later improvement is to automate
    application startup and cleanup within the workflow.

------------------------------------------------------------------------

## 18. End-to-end execution sequence

Use this order when setting up the lab from scratch:

1.  Install Git, GitHub access, VS Code, Node.js, MongoDB, Angular CLI
    and Docker Desktop.
2.  Create the GitHub repository and clone it.
3.  Create a PAT if needed for HTTPS Git authentication and save it
    securely using Git Credential Manager.
4.  Place the frontend and backend application folders inside the
    repository.
5.  Configure the backend MongoDB connection and create/use `todo_db`.
6.  Install backend dependencies and start the backend.
7.  Configure the frontend API URL, install dependencies and start
    Angular.
8.  Verify the frontend and backend endpoints.
9.  Create the SonarQube Compose file and start SonarQube plus
    PostgreSQL.
10. Create the frontend and backend projects in SonarQube.
11. Generate a SonarQube token and save it as a GitHub Actions secret.
12. Pull the OWASP ZAP image and verify that a ZAP container can reach
    the applications.
13. Configure and start the Windows self-hosted GitHub Actions runner.
14. Add the workflow file under `.github/workflows/`.
15. Verify all required secrets, services, ports and Docker networking.
16. Push the workflow to GitHub or run it manually.
17. Review the SonarQube analysis and download the ZAP HTML reports.

**Expected outcome:** The GitHub Actions workflow checks out the source
code, performs SonarQube analysis on the frontend and backend, verifies
application availability, runs OWASP ZAP baseline scans against the two
targets, and uploads the resulting HTML reports.
