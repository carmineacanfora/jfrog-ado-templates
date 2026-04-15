# JFrog Pipeline Templates for Azure DevOps

Admin-controlled pipeline templates for secure JFrog Platform integration in Azure DevOps using OIDC authentication.

## Overview

These templates provide a standardized, secure way for development teams to interact with the JFrog Platform (Artifactory, Xray, Curation) from Azure DevOps pipelines. All JFrog interactions are admin-controlled — devs choose what to enable and provide configuration, but cannot modify how the steps execute.

All JFrog commands run through `JFrogCliV2@1` tasks with OIDC authentication via an ADO service connection. No long-lived tokens are stored anywhere.

### Security Layers

- **Layer 1:** All JFrog commands run through `JFrogCliV2@1` which handles OIDC auth per-task. Dev steps have no access to JFrog credentials.
- **Layer 2:** Dev steps (`preBuildSteps`, `dockerBuildSteps`) are validated at compile time. Any step containing JFrog CLI commands, install scripts, or sensitive flags is blocked before the pipeline runs.
- **Enforcement:** Use the [Required Template Check](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops#required-template) on your JFrog service connections to ensure all pipelines extend from `secured-pipeline.yml`.

## Quick Start

1. Reference the template repo in your pipeline:

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

2. See `examples/starter-pipeline.yml` for a full template with all parameters.
3. See `examples/dev-pipeline.yml` for a working example.

## Prerequisites

- **JFrog ADO Extension:** Install the [JFrog Extension](https://marketplace.visualstudio.com/items?itemName=JFrog.jfrog-azure-devops-extension) (v2.11.0+) in your ADO organization.
- **JFrog Platform Service Connection:** Create a **JFrog Platform V2** service connection in ADO with **OpenID Connect Integration** authentication.
- **OIDC Integration in JFrog:** Configure an OIDC integration in JFrog Platform matching your ADO organization. See [JFrog OIDC Documentation](https://jfrog.com/help/r/jfrog-platform-administration-documentation/configure-an-oidc-integration).
- **GitHub Service Connection:** If the template repo is on GitHub, create a GitHub service connection in ADO with access to the repo.

## Pipeline Flow

| #  | Step                        | Control | Required | Description |
|----|-----------------------------|---------|----------|-------------|
| 1  | Print Inputs                | Admin   | Always   | Shows what the dev enabled |
| 2  | Dev preBuildSteps           | Dev     | Always   | Checkout, tooling setup (validated, no JFrog CLI) |
| 3  | Setup JFrog OIDC            | Admin   | Always   | Verifies JFrog connection via OIDC |
| 4  | NPM Install                 | Admin   | Optional | With curation audit on failure |
| 5  | Scan Code (Xray)            | Admin   | Optional | Source code dependency scan |
| 6  | Docker Build                | Dev     | Optional | Dev controls how image is built (validated) |
| 7  | Push Docker Image           | Admin   | Optional | Docker login, push, and build-info |
| 8  | Push Generic Artifact       | Admin   | Optional | Upload generic artifacts |
| 9  | Push Helm Charts            | Admin   | Optional | Upload Helm chart packages |
| 10 | Publish Build Info          | Admin   | Optional | Collect env/git and publish (enforced --env-exclude) |
| 11 | Scan Build (Xray)           | Admin   | Optional | Scan published build for vulnerabilities |

## Parameters Reference

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `jfrogConnection` | string | ADO service connection name (JFrog Platform V2 with OIDC) |
| `jfrogUrl` | string | JFrog Platform URL (e.g., `mycompany.jfrog.io`) |
| `buildName` | string | Build name used in build-info |

### Optional Connection Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `project` | string | `''` | JFrog project key |
| `envExclude` | string | `*password*;*secret*;*key*;*token*;*credentials*;*ACCESS*;*PRIVATE*` | Env vars to exclude from build-info (admin-enforced) |

### Dev Steps

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `preBuildSteps` | stepList | `[]` | Steps that run before any JFrog interaction. Use for checkout, tool setup, etc. **JFrog CLI commands are blocked.** |
| `dockerBuildSteps` | stepList | `[]` | Steps that build the Docker image locally. **`docker push`, `docker login`, and JFrog CLI commands are blocked.** Use `--load` flag (not `--push`). |

### NPM Install

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableNpm` | boolean | `false` | Enable NPM install via JFrog CLI |
| `npmResolveRepo` | string | `''` | Artifactory NPM remote repo for dependency resolution |
| `npmArguments` | string | `''` | Additional npm arguments (e.g., `--loglevel=notice`) |
| `npmThreads` | number | `5` | Number of threads for parallel download |
| `enableCurationAudit` | boolean | `false` | Run curation audit if NPM install fails |

### Xray Audit (Scan Code)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableXrayAudit` | boolean | `false` | Enable Xray source code audit |
| `xrayPackageManager` | string | `npm` | Package manager: `npm`, `maven`, `gradle`, `pip`, `go`, `nuget` |
| `xrayFailBuild` | boolean | `true` | Fail the build if vulnerabilities are found |

### Docker

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableDockerPush` | boolean | `false` | Enable Docker login, push, and build-info collection |
| `dockerRepo` | string | `''` | Artifactory Docker repository |
| `imageName` | string | `''` | Docker image name |
| `imageTag` | string | `$(Build.BuildId)` | Docker image tag |

**Important:** The dev provides `dockerBuildSteps` to build the image locally with `--load` (not `--push`). The admin template handles Docker login, push, and build-info. The image must be tagged as `jfrogUrl/dockerRepo/imageName:imageTag`.

### Upload Generic Artifacts

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableUploadGeneric` | boolean | `false` | Enable generic artifact upload |
| `genericSourcePattern` | string | `''` | Source file pattern (e.g., `rn/*.pdf`) |
| `genericTargetRepo` | string | `''` | Target Artifactory repository |
| `genericModuleName` | string | `''` | Module name for build-info |

### Upload Helm Charts

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableUploadHelm` | boolean | `false` | Enable Helm chart upload |
| `helmSourcePattern` | string | `helm/*.tgz` | Source file pattern |
| `helmTargetRepo` | string | `''` | Target Artifactory repository |
| `helmModuleName` | string | `''` | Module name for build-info |

### Build Info

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enablePublishBuild` | boolean | `false` | Collect env/git info and publish build-info to Artifactory |

### Build Scan

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableBuildScan` | boolean | `false` | Enable Xray build scan |
| `buildScanFailBuild` | boolean | `true` | Fail the build if vulnerabilities are found |

## Template Files

| File | Description |
|------|-------------|
| `secured-pipeline.yml` | Main extends template — devs extend this |
| `stages/jfrog-print-inputs.yml` | Prints pipeline configuration |
| `stages/jfrog-setup.yml` | Verifies JFrog connection via OIDC |
| `stages/jfrog-npm-install.yml` | NPM install with optional curation audit |
| `stages/jfrog-xray-scan.yml` | Xray source code audit |
| `stages/jfrog-docker-push.yml` | Docker login, push, and build-info |
| `stages/jfrog-generic-push.yml` | Generic artifact upload |
| `stages/jfrog-helm-push.yml` | Helm chart upload |
| `stages/jfrog-build-publish.yml` | Collect env/git and publish build-info (enforced env-exclude) |
| `stages/jfrog-build-scan.yml` | Xray build scan |

## Dev Step Restrictions

The following patterns are **blocked** in `preBuildSteps` and `dockerBuildSteps`:

- `jf ` / `jfrog ` — any JFrog CLI command
- `install-cli.jfrog.io` / `jfrog-cli` — CLI installation attempts
- `env-exclude` / `build-publish` / `build-collect-env` / `build-add-git` — build-info manipulation
- `curation-audit` / `build-scan` — scan commands
- `docker login` — auth (handled by admin template)
- `docker push` — push (handled by admin template, blocked in `dockerBuildSteps` only)
- `access-token` / `JF_ACCESS_TOKEN` — credential access

If you need a capability not covered by these templates, contact your platform admin.

## For Admins

### Setting Up OIDC

1. **ADO:** Create a **JFrog Platform V2** service connection with **OpenID Connect Integration**.
2. **JFrog:** Create an OIDC integration matching your ADO organization at **Administration → General → Manage Integrations → OpenID Connect**.
3. **JFrog:** Add identity mappings using the subject claim. Use wildcards for flexibility: `sc://MyOrg/MyProject/*` matches all service connections in the project.
4. Run a test pipeline to verify the connection.

### Enforcing Template Usage

1. Push this repo with admin-only write access.
2. On your JFrog service connection, add a **Required Template Check**:
   - Go to the service connection → **Approvals and Checks** → **Add check → Required template**
   - **Repository type:** GitHub (or Azure Repos)
   - **Repository:** `carmineacanfora/jfrog-ado-templates`
   - **Ref:** `refs/heads/main`
   - **Path:** `secured-pipeline.yml`
3. Any pipeline using this service connection must extend from `secured-pipeline.yml`.

### Making Steps Mandatory

To make a step mandatory (e.g., build scan must always run), remove its `enable*` toggle from `secured-pipeline.yml` and hardcode it.

### Versioning

Use git tags to version the templates:

```yaml
resources:
  repositories:
    - repository: jfrog-templates
      type: github
      name: 'carmineacanfora/jfrog-ado-templates'
      endpoint: 'your-github-connection'
      ref: refs/tags/v1.0.0
```

### Known Limitations

- **CI Server field:** The "CI Server" field in Artifactory build-info is not populated when using `JFrogCliV2@1` for publishing. This is cosmetic and does not affect functionality. The `--build-url` flag is used to link back to the ADO run instead.
- **OIDC with JFrogDocker@1 / JFrogPublishBuildInfo@1:** These tasks do not support OIDC with the JFrog Platform service connection type. All JFrog interactions use `JFrogCliV2@1` instead.
- **Build-info per CLI session:** Each `JFrogCliV2@1` task has an isolated CLI session. Build-info is published within the same task that collects it (Docker push publishes its own build-info, uploads publish theirs, etc.). JFrog merges all publishes with the same build name and number.
