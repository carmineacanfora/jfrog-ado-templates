# JFrog Pipeline Templates for Azure DevOps

Admin-controlled pipeline templates for secure JFrog Platform integration in Azure DevOps.

## Overview

These templates provide a standardized, secure way for development teams to interact with the JFrog Platform (Artifactory, Xray, Curation) from Azure DevOps pipelines. All JFrog interactions are admin-controlled — devs choose what to enable and provide configuration, but cannot modify how the steps execute.

### Security Layers

- **Layer 1:** JFrog CLI is installed *after* dev pre-build steps. Dev steps have no access to JFrog credentials.
- **Layer 2:** Dev steps (`preBuildSteps`, `dockerBuildSteps`) are validated at compile time. Any step containing JFrog CLI commands, install scripts, or sensitive flags is blocked before the pipeline runs.
- **Enforcement:** Use the [Required Template Check](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops#required-template) on your JFrog service connections to ensure all pipelines extend from `secured-pipeline.yml`.

## Quick Start

1. Reference the template repo in your pipeline:

```yaml
trigger: none

resources:
  repositories:
    - repository: jfrog-templates
      type: git
      name: 'MyProject/jfrog-pipeline-templates'
      ref: refs/heads/main

extends:
  template: secured-pipeline.yml@jfrog-templates
  parameters:
    jfrogUrl: 'mycompany.jfrog.io'
    oidcProviderName: 'ado-oidc'
    buildName: 'my-app'
    preBuildSteps:
      - checkout: self
    enableNpm: true
    npmResolveRepo: 'my-npm-remote'
```

2. See `examples/starter-pipeline.yml` for a full template with all parameters.
3. See `examples/dev-pipeline.yml` for a working example.

## Pipeline Flow

| #  | Step                        | Control | Required | Description |
|----|-----------------------------|---------|----------|-------------|
| 1  | Print Inputs                | Admin   | Always   | Shows what the dev enabled |
| 2  | Dev preBuildSteps           | Dev     | Always   | Checkout, tooling setup (validated, no CLI) |
| 3  | JFrog CLI Setup             | Admin   | Always   | Installs CLI and authenticates via OIDC |
| 4  | NPM Install                 | Admin   | Optional | With curation audit on failure |
| 5  | Xray Audit                  | Admin   | Optional | Source code dependency scan |
| 6  | Docker Authenticate         | Admin   | Optional | Login to JFrog Docker registry |
| 7  | Docker Build                | Dev     | Optional | Dev controls how image is built (validated) |
| 8  | Docker Push                 | Admin   | Optional | Pushes image and adds to build-info |
| 9  | Upload Generic              | Admin   | Optional | Upload generic artifacts |
| 10 | Upload Helm Charts          | Admin   | Optional | Upload Helm chart packages |
| 11 | Collect Build Info          | Admin   | Optional | Collect environment and git info |
| 12 | Publish Build Info          | Admin   | Optional | Publish with enforced --env-exclude |
| 13 | Xray Build Scan             | Admin   | Optional | Scan published build for vulnerabilities |

## Parameters Reference

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `jfrogUrl` | string | JFrog Platform URL (e.g., `mycompany.jfrog.io`) |
| `oidcProviderName` | string | OIDC provider name configured in JFrog |
| `buildName` | string | Build name used in build-info |

### Optional Connection Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `oidcAudience` | string | `''` | OIDC audience (if required) |
| `project` | string | `''` | JFrog project key |
| `envExclude` | string | `*password*;*secret*;*key*;*token*;*credentials*;*ACCESS*;*PRIVATE*` | Env vars to exclude from build-info (admin-enforced) |

### Dev Steps

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `preBuildSteps` | stepList | `[]` | Steps that run before JFrog CLI is available. Use for checkout, tool setup, etc. **JFrog CLI commands are blocked.** |
| `dockerBuildSteps` | stepList | `[]` | Steps that build the Docker image locally. **`docker push`, `docker login`, and JFrog CLI commands are blocked.** |

### NPM Install

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableNpm` | boolean | `false` | Enable NPM install via JFrog CLI |
| `npmResolveRepo` | string | `''` | Artifactory NPM remote repo for dependency resolution |
| `npmArguments` | string | `''` | Additional npm arguments (e.g., `--loglevel=notice`) |
| `npmThreads` | number | `5` | Number of threads for parallel download |
| `enableCurationAudit` | boolean | `false` | Run curation audit if NPM install fails |

### Xray Audit

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableXrayAudit` | boolean | `false` | Enable Xray source code audit |
| `xrayPackageManager` | string | `npm` | Package manager: `npm`, `maven`, `gradle`, `pip`, `go`, `nuget` |
| `xrayFailBuild` | boolean | `true` | Fail the build if vulnerabilities are found |

### Docker

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableDockerAuth` | boolean | `false` | Enable Docker login to JFrog |
| `enableDockerPush` | boolean | `false` | Enable Docker push and build-info |
| `dockerRepo` | string | `''` | Artifactory Docker repository |
| `imageName` | string | `''` | Docker image name |
| `imageTag` | string | `$(Build.BuildId)` | Docker image tag |

**Important:** When using Docker, the dev provides `dockerBuildSteps` to build the image locally with `--load` (not `--push`). The admin template handles the push. The image must be tagged as `jfrogUrl/dockerRepo/imageName:imageTag`.

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
| `enableCollectBuildInfo` | boolean | `false` | Collect environment variables and git info |
| `enablePublishBuild` | boolean | `false` | Publish build-info to Artifactory |

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
| `stages/jfrog-setup.yml` | Installs and configures JFrog CLI via OIDC |
| `stages/jfrog-npm-install.yml` | NPM install with optional curation audit |
| `stages/jfrog-xray-scan.yml` | Xray source code audit |
| `stages/jfrog-docker-login.yml` | Docker login to JFrog |
| `stages/jfrog-docker-push.yml` | Docker push and build-info |
| `stages/jfrog-generic-push.yml` | Generic artifact upload |
| `stages/jfrog-helm-push.yml` | Helm chart upload |
| `stages/jfrog-build-collect-info.yml` | Collect environment and git info |
| `stages/jfrog-build-publish.yml` | Publish build-info (enforced env-exclude) |
| `stages/jfrog-build-scan.yml` | Xray build scan |

## Dev Step Restrictions

The following patterns are **blocked** in `preBuildSteps` and `dockerBuildSteps`:

- `jf ` / `jfrog ` — any JFrog CLI command
- `install-cli.jfrog.io` / `jfrog-cli` — CLI installation attempts
- `env-exclude` / `build-publish` / `build-collect-env` / `build-add-git` — build-info manipulation
- `curation-audit` / `build-scan` — scan commands
- `docker login` — auth (handled by admin template)
- `docker push` — push (handled by admin template, only in `dockerBuildSteps`)
- `access-token` / `JF_ACCESS_TOKEN` — credential access

If you need a capability not covered by these templates, contact your platform admin.

## For Admins

### Enforcing Template Usage

1. Push this repo with admin-only write access.
2. On your JFrog-related service connections, add a **Required Template Check**:
   - Repository: `jfrog-pipeline-templates`
   - Ref: `refs/heads/main`
   - Path: `secured-pipeline.yml`
3. Any pipeline not extending this template will fail.

### Making Steps Mandatory

To make a step mandatory (e.g., build scan must always run), remove its `enable*` toggle from `secured-pipeline.yml` and hardcode it:

```yaml
# Always runs — no toggle
- template: stages/jfrog-build-scan.yml
  parameters:
    buildName: ${{ parameters.buildName }}
    buildNumber: '$(Build.BuildId)'
    project: ${{ parameters.project }}
```

### Versioning

Use git tags/branches to version the templates:

```yaml
resources:
  repositories:
    - repository: jfrog-templates
      type: git
      name: 'MyProject/jfrog-pipeline-templates'
      ref: refs/tags/v1.0.0    # Pin to a version
```
