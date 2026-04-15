# JFrog Pipeline Templates for Azure DevOps

Secure, admin-controlled pipeline templates for integrating the JFrog Platform with Azure DevOps.

Devs choose what to enable. Admins control how it runs. No tokens stored anywhere — all auth is OIDC.

---

## Quick Start

Copy `examples/starter-pipeline.yml` to your repo, fill in the `TODO` fields, and uncomment what you need. See `examples/dev-pipeline.yml` for a working example.

```yaml
trigger: none

resources:
  repositories:
    - repository: jfrog-templates
      type: github
      name: 'carmineacanfora/jfrog-ado-templates'
      endpoint: 'your-github-connection'
      ref: refs/heads/main

pool:
  vmImage: 'ubuntu-latest'

extends:
  template: secured-pipeline.yml@jfrog-templates
  parameters:
    jfrogConnection: 'JFrog Platform - mycompany OIDC'
    jfrogUrl: 'mycompany.jfrog.io'
    buildName: 'my-app'
    preBuildSteps:
      - checkout: self
    enableNpm: true
    npmResolveRepo: 'my-npm-remote'
```

---

## How It Works

```
Dev pipeline                          Admin templates
─────────────                         ───────────────
preBuildSteps ──→ (validated) ──→     Setup JFrog OIDC
  - checkout                          NPM Install
  - setup node                        Xray Audit
  - setup buildx                      Docker Push
                                      Upload Artifacts
dockerBuildSteps ──→ (validated) ──→  Publish Build Info
  - docker build --load              Xray Build Scan
```

Devs provide **what** to build. Admin templates handle **all JFrog interactions**: authentication, uploads, scans, and build-info publishing.

---

## Pipeline Steps

| # | Step | Who | Required | What it does |
|---|------|-----|----------|--------------|
| 1 | Print Inputs | Admin | Always | Logs the pipeline configuration |
| 2 | preBuildSteps | **Dev** | Always | Checkout, Node install, Docker Buildx, etc. |
| 3 | Setup JFrog OIDC | Admin | Always | Verifies JFrog connection |
| 4 | NPM Install | Admin | Optional | Installs dependencies; runs curation audit on failure |
| 5 | Scan Code | Admin | Optional | Xray audit on project dependencies |
| 6 | Docker Build | **Dev** | Optional | Builds image locally with `--load` |
| 7 | Push Docker Image | Admin | Optional | Login, push, and collect build-info |
| 8 | Push Generic Artifact | Admin | Optional | Upload files to Artifactory |
| 9 | Push Helm Charts | Admin | Optional | Upload Helm packages |
| 10 | Publish Build Info | Admin | Optional | Collect env/git, publish with enforced `--env-exclude` |
| 11 | Scan Build | Admin | Optional | Xray scan on published build |

---

## Parameters

### Required

| Parameter | Description |
|-----------|-------------|
| `jfrogConnection` | ADO service connection (JFrog Platform V2, OIDC) |
| `jfrogUrl` | JFrog URL, e.g., `mycompany.jfrog.io` |
| `buildName` | Build name in Artifactory |

### Connection (optional)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `project` | `''` | JFrog project key |
| `envExclude` | `*password*;*secret*;*key*;*token*;...` | Env vars excluded from build-info (admin-enforced) |

### Dev Steps

| Parameter | Description |
|-----------|-------------|
| `preBuildSteps` | Runs before JFrog CLI is available. Use for checkout, tool setup. **JFrog CLI commands are blocked.** |
| `dockerBuildSteps` | Builds Docker image locally. **`docker push`, `docker login`, and JFrog CLI are blocked.** Tag as `jfrogUrl/dockerRepo/imageName:imageTag`, use `--load` not `--push`. |

### NPM

| Parameter | Default | Description |
|-----------|---------|-------------|
| `enableNpm` | `false` | Enable NPM install |
| `npmResolveRepo` | `''` | Artifactory NPM remote repo |
| `npmArguments` | `''` | Extra npm args, e.g., `--loglevel=notice` |
| `npmThreads` | `8` | Parallel download threads |
| `enableCurationAudit` | `false` | Run curation audit if install fails |

### Xray Audit

| Parameter | Default | Description |
|-----------|---------|-------------|
| `enableXrayAudit` | `false` | Scan source code dependencies |
| `xrayPackageManager` | `npm` | `npm`, `maven`, `gradle`, `pip`, `go`, `nuget` |
| `xrayFailBuild` | `true` | Fail on vulnerabilities |

### Docker

| Parameter | Default | Description |
|-----------|---------|-------------|
| `enableDockerPush` | `false` | Push image and collect build-info |
| `dockerRepo` | `''` | Artifactory Docker repo |
| `imageName` | `''` | Image name |
| `imageTag` | `$(Build.BuildId)` | Image tag |

### Uploads

| Parameter | Default | Description |
|-----------|---------|-------------|
| `enableUploadGeneric` | `false` | Upload generic artifacts |
| `genericSourcePattern` | `''` | File pattern, e.g., `rn/*.pdf` |
| `genericTargetRepo` | `''` | Target repo |
| `genericModuleName` | `''` | Module name for build-info |
| `enableUploadHelm` | `false` | Upload Helm charts |
| `helmSourcePattern` | `helm/*.tgz` | File pattern |
| `helmTargetRepo` | `''` | Target repo |
| `helmModuleName` | `''` | Module name for build-info |

### Build Info & Scan

| Parameter | Default | Description |
|-----------|---------|-------------|
| `enablePublishBuild` | `false` | Publish build-info (includes env/git collection) |
| `enableBuildScan` | `false` | Xray scan on published build |
| `buildScanFailBuild` | `true` | Fail on vulnerabilities |

---

## Security

### How devs are restricted

**Layer 1 — No credentials in dev steps.** All JFrog commands run through `JFrogCliV2@1` tasks inside admin templates. OIDC auth happens per-task. Dev steps run before JFrog CLI is configured.

**Layer 2 — Compile-time validation.** Dev steps (`preBuildSteps`, `dockerBuildSteps`) are scanned for blocked patterns before the pipeline starts. Blocked patterns include:

`jf `, `jfrog `, `install-cli.jfrog.io`, `jfrog-cli`, `env-exclude`, `build-publish`, `build-collect-env`, `build-add-git`, `curation-audit`, `build-scan`, `docker login`, `docker push` (in dockerBuildSteps only), `access-token`, `JF_ACCESS_TOKEN`

**Enforcement.** Use [Required Template Check](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops#required-template) on your service connection to ensure all pipelines extend from `secured-pipeline.yml`.

---

## Template Files

```
secured-pipeline.yml          ← Main template (devs extend this)
stages/
  jfrog-print-inputs.yml      ← Print pipeline config
  jfrog-setup.yml             ← Verify OIDC connection
  jfrog-npm-install.yml       ← NPM install + curation audit
  jfrog-xray-scan.yml         ← Xray source code audit
  jfrog-docker-push.yml       ← Docker login + push + build-info
  jfrog-generic-push.yml      ← Generic artifact upload
  jfrog-helm-push.yml         ← Helm chart upload
  jfrog-build-publish.yml     ← Collect env/git + publish build-info
  jfrog-build-scan.yml        ← Xray build scan
examples/
  starter-pipeline.yml        ← Copy this to get started
  dev-pipeline.yml            ← Working example
```

---

## Admin Guide

### Prerequisites

1. Install the [JFrog ADO Extension](https://marketplace.visualstudio.com/items?itemName=JFrog.jfrog-azure-devops-extension) (v2.11.0+).
2. Create a **JFrog Platform V2** service connection with **OpenID Connect Integration**.
3. Configure OIDC in JFrog: **Administration → General → Manage Integrations → OpenID Connect**. See [docs](https://jfrog.com/help/r/jfrog-platform-administration-documentation/configure-an-oidc-integration).
4. Add identity mappings. Use wildcards for flexibility: `sc://MyOrg/MyProject/*`.
5. If templates are on GitHub, create a GitHub service connection in ADO.

### Enforce template usage

1. Keep this repo with admin-only write access.
2. On your JFrog service connection → **Approvals and Checks** → **Required template**:
   - **Repository:** `carmineacanfora/jfrog-ado-templates`
   - **Ref:** `refs/heads/main`
   - **Path:** `secured-pipeline.yml`
3. Pipelines that don't extend this template will fail.

### Make steps mandatory

Remove the `enable*` toggle from `secured-pipeline.yml` and the step always runs.

### Version the templates

```yaml
ref: refs/tags/v1.0.0  # Pin to a release
```

---

## Known Limitations

**CI Server field** — Not populated when publishing via `JFrogCliV2@1`. The `--build-url` flag links back to the ADO run instead. Cosmetic only.

**OIDC with JFrogDocker@1 / JFrogPublishBuildInfo@1** — These tasks don't support OIDC with the Platform service connection. All interactions use `JFrogCliV2@1`.

**Build-info isolation** — Each `JFrogCliV2@1` task has its own CLI session. Build-info is collected and published within the same task. JFrog merges all publishes with the same build name and number.
