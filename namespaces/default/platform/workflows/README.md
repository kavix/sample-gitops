# GitOps Workflows

This directory contains OpenChoreo Workflow definitions that automate building, releasing, and promoting components through a GitOps-driven CI/CD pipeline.

## Available Workflows Overview

| Workflow | Purpose | Build Method |
|---|---|---|
| [docker-gitops-release](#docker-gitops-release) | Build from Dockerfile and release via GitOps | Docker (Podman) |
| [google-cloud-buildpacks-gitops-release](#google-cloud-buildpacks-gitops-release) | Build without Dockerfile and release via GitOps | Google Cloud Buildpacks |
| [react-gitops-release](#react-gitops-release) | Build React/SPA app and release via GitOps | Node.js + nginx |
| [bulk-gitops-release](#bulk-gitops-release) | Promote existing releases to a target environment | N/A (no build) |

> [!NOTE]  
> To learn more about how OpenChoreo workflows work, please refer to the [official documentation](https://openchoreo.dev/).

---

## Repository Roles & Architecture

OpenChoreo workflows distinguish between two repository destinations:

- **Application Source Repository (`repository.url`)**: Contains the application code (e.g. `https://github.com/openchoreo/sample-workloads.git`). Cloned during the **build phase**. Public repositories do not require a git token; private source repositories use `secret/git-token`.
- **GitOps Destination Repository (`gitopsRepoUrl`)**: Contains the infrastructure and deployment manifests (e.g. `https://github.com/<your-github-username>/sample-gitops.git`). Cloned during the **release phase** to generate release manifests and open pull requests using `secret/gitops-token`.

---

## Secret Prerequisites (OpenBao & ESO)

These workflows rely on the **External Secrets Operator (ESO)** and a `ClusterSecretStore` named `default` to synchronize secrets from OpenBao into per-WorkflowRun Kubernetes Secrets:

| Secret Key / Remote Path | Property | Required By | Purpose | Volume Behavior |
|---|---|---|---|---|
| `secret/git-token` | `git-token` | Build phase (source clone) | Authenticate to private application source repos | **Optional**; workflows fall back to public unauthenticated clone if secret is omitted. |
| `secret/gitops-token` | `git-token` | Release phase & bulk promotion | Authenticate to GitOps repo, push branches, and create PRs | **Mandatory**; workflow pods will wait for secret synchronization. |

To populate secrets in OpenBao:

```bash
# Secret for private application source repositories (optional for public repos)
kubectl exec -n openbao openbao-0 -- bao kv put secret/git-token git-token=<your_github_pat>

# Secret for GitOps repository release and PR operations (mandatory)
kubectl exec -n openbao openbao-0 -- bao kv put secret/gitops-token git-token=<your_github_pat>
```

> [!IMPORTANT]
> **Flux vs. Workflow Credentials:**
> Secrets in OpenBao authenticate OpenChoreo workflow pods. If your GitOps repository is private, Flux `source-controller` requires its own Kubernetes secret in the `flux-system` namespace.

---

## Build and Release Workflows

The following build-and-release workflows automate the build and deployment of OpenChoreo components in a GitOps setup:

1. [docker-gitops-release](#docker-gitops-release)
2. [google-cloud-buildpacks-gitops-release](#google-cloud-buildpacks-gitops-release)
3. [react-gitops-release](#react-gitops-release)

All three workflows follow the same high-level execution pattern:

**Build phase**
1. **`clone-source`**: Clones the source repository (supports private repositories via optional `git-token`).
2. **`build-image`**: Builds the container image using the workflow-specific builder.
3. **`push-image`**: Pushes the container image to the internal registry.
4. **`extract-descriptor`**: Extracts the workload descriptor from the source repository.

**Release phase**
1. **`clone-gitops`**: Clones the target GitOps repository using `gitops-token`.
2. **`create-feature-branch`**: Creates a release feature branch (`release/<component>-<timestamp>`).
3. **`generate-gitops-resources`**: Generates GitOps manifests (`Workload`, `ComponentRelease`, `ReleaseBinding`) using the `occ` CLI.
4. **`git-commit-push-pr`**: Commits changes, pushes the feature branch, and creates a pull request.

---

### docker-gitops-release

Workflow definition: [`docker-with-gitops-release.yaml`](./docker-with-gitops-release.yaml)

Use when your source repository has a **Dockerfile**.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `componentName` | string | yes | | Component name |
| `projectName` | string | yes | | Project name |
| `gitopsRepoUrl` | string | no | `https://github.com/openchoreo/sample-gitops` | GitOps repository URL |
| `gitopsBranch` | string | no | `main` | GitOps repository branch |
| `repository.url` | string | yes | | Source repository URL |
| `repository.revision.branch` | string | no | `main` | Source repository branch to check out |
| `repository.revision.commit` | string | yes | | Source repository Git commit SHA or reference |
| `repository.appPath` | string | no | `.` | Application path within the source repository |
| `docker.context` | string | no | `.` | Docker build context relative to the source repository root |
| `docker.filePath` | string | no | `./Dockerfile` | Dockerfile path relative to the source repository root |
| `workloadDescriptorPath` | string | no | `workload.yaml` | Path to workload descriptor relative to `repository.appPath` |

**WorkflowRun manifest:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: greeter-build-001
  namespace: default
spec:
  workflow:
    name: docker-gitops-release
    kind: Workflow
    parameters:
      componentName: greeter-service
      projectName: demo-project
      gitopsRepoUrl: https://github.com/<your-github-username>/sample-gitops
      gitopsBranch: main
      repository:
        url: https://github.com/openchoreo/sample-workloads.git
        revision:
          branch: main
          commit: "abc1234"
        appPath: /service-go-greeter
      docker:
        context: /service-go-greeter
        filePath: /service-go-greeter/Dockerfile
      workloadDescriptorPath: workload.yaml
```

---

### google-cloud-buildpacks-gitops-release

Workflow definition: [`google-cloud-buildpacks-gitops-release.yaml`](./google-cloud-buildpacks-gitops-release.yaml)

Use when your source repository **does not have a Dockerfile**. Buildpacks auto-detect the language and build a container image.

Supported languages: Go, Java (Maven/Gradle), Node.js, Python, .NET Core, Ruby, PHP.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `componentName` | string | yes | | Component name |
| `projectName` | string | yes | | Project name |
| `gitopsRepoUrl` | string | no | `https://github.com/openchoreo/sample-gitops` | GitOps repository URL |
| `gitopsBranch` | string | no | `main` | GitOps repository branch |
| `repository.url` | string | yes | | Source repository URL |
| `repository.revision.branch` | string | no | `main` | Branch to check out |
| `repository.revision.commit` | string | yes | | Git commit SHA or reference |
| `repository.appPath` | string | no | `.` | Application path within the repository |
| `buildpacks.builderImage` | string | no | `gcr.io/buildpacks/builder:v1` | Buildpacks builder image |
| `buildpacks.env` | string[] | no | `[]` | Build-time environment variables in `KEY=VALUE` format |
| `workloadDescriptorPath` | string | no | `workload.yaml` | Path to workload descriptor relative to `repository.appPath` |

**Example WorkflowRun manifest:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: reading-list-build-001
  namespace: default
spec:
  workflow:
    name: google-cloud-buildpacks-gitops-release
    kind: Workflow
    parameters:
      componentName: reading-list-service
      projectName: demo-project
      gitopsRepoUrl: https://github.com/<your-github-username>/sample-gitops
      gitopsBranch: main
      repository:
        url: https://github.com/openchoreo/sample-workloads.git
        revision:
          branch: main
          commit: "abc1234"
        appPath: /service-go-reading-list
      buildpacks:
        builderImage: gcr.io/buildpacks/builder:v1
        env: []
      workloadDescriptorPath: workload.yaml
```

---

### react-gitops-release

Workflow definition: [`react-gitops-release.yaml`](./react-gitops-release.yaml)

Use for **React and SPA** applications. Builds the app with Node.js, packages it into an nginx container with SPA routing configured.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `componentName` | string | yes | | Component name |
| `projectName` | string | yes | | Project name |
| `gitopsRepoUrl` | string | no | `https://github.com/openchoreo/sample-gitops` | GitOps repository URL |
| `gitopsBranch` | string | no | `main` | GitOps repository branch |
| `repository.url` | string | yes | | Source repository URL |
| `repository.revision.branch` | string | no | `main` | Branch to check out |
| `repository.revision.commit` | string | yes | | Git commit SHA or reference |
| `repository.appPath` | string | no | `.` | Application path within the repository |
| `react.nodeVersion` | string | no | `18` | Node.js version (`16`, `18`, `20`, `22`) |
| `react.buildCommand` | string | no | `npm run build` | Build command |
| `react.outputDir` | string | no | `build` | Frontend build output directory |
| `workloadDescriptorPath` | string | no | `workload.yaml` | Path to workload descriptor relative to `repository.appPath` |

**Example WorkflowRun manifest:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: frontend-build-001
  namespace: default
spec:
  workflow:
    name: react-gitops-release
    kind: Workflow
    parameters:
      componentName: frontend
      projectName: demo-project
      gitopsRepoUrl: https://github.com/<your-github-username>/sample-gitops
      gitopsBranch: main
      repository:
        url: https://github.com/openchoreo/sample-workloads.git
        revision:
          branch: main
          commit: "abc1234"
        appPath: /webapp-react-frontend
      react:
        nodeVersion: "20"
        buildCommand: "npm run build"
        outputDir: build
      workloadDescriptorPath: workload.yaml
```

---

## Promotion Workflow

### bulk-gitops-release

Workflow definition: [`bulk-gitops-release.yaml`](./bulk-gitops-release.yaml)

Use to **promote existing releases** to a target environment. Does not build images — generates `ReleaseBinding` manifests for components that already have `ComponentRelease` resources.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `scope.all` | boolean | no | `false` | Promote all projects |
| `scope.projectName` | string | yes | | Project name to promote; required placeholder when `all: true` |
| `gitops.repositoryUrl` | string | yes | | GitOps repository URL |
| `gitops.branch` | string | no | `main` | GitOps repository branch |
| `gitops.targetEnvironment` | string | no | `development` | Target environment name |
| `gitops.deploymentPipeline` | string | yes | | Deployment pipeline name |

**Example WorkflowRun manifest for promoting a single project to staging:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: promote-doclet-staging-001
  namespace: default
spec:
  workflow:
    name: bulk-gitops-release
    kind: Workflow
    parameters:
      scope:
        all: false
        projectName: doclet
      gitops:
        repositoryUrl: "https://github.com/<your-github-username>/sample-gitops"
        branch: main
        targetEnvironment: staging
        deploymentPipeline: standard
```

---

## Troubleshooting & Diagnostics

If a workflow execution is stuck, fails during clone/checkout, or encounters authentication issues, follow this guide to isolate and resolve the issue.

### Diagnostic Command Workflow

```bash
# 1. Check WorkflowRun status and rendered Argo Workflow reference
kubectl get workflowrun <workflowrun-name> -n default -o yaml

# 2. Check Argo Workflows in the cluster
kubectl get workflows.argoproj.io -A

# 3. Check Workflow pods and status
kubectl get pods -A
kubectl describe pod -n <workflow-namespace> <pod-name>

# 4. Check ExternalSecrets Operator & ClusterSecretStore
kubectl get pods -A | grep -i external-secrets
kubectl get clustersecretstore default
kubectl describe clustersecretstore default

# 5. Check ExternalSecrets created for the WorkflowRun
kubectl get externalsecret -A
kubectl describe externalsecret <workflowrun-name>-source-git-secret -n default
kubectl describe externalsecret <workflowrun-name>-gitops-git-secret -n default

# 6. Verify generated Secrets (safe byte count check, never echo credentials)
kubectl get secret <workflowrun-name>-source-git-secret -n default
kubectl get secret <workflowrun-name>-gitops-git-secret -n default
kubectl get secret <workflowrun-name>-gitops-git-secret -n default -o jsonpath='{.data.git-token}' | base64 -d | wc -c

# 7. Follow workflow execution logs
argo logs <workflow-name> -n <workflow-namespace> --follow
```

### Error-to-Cause Reference Table

| Error / Symptom | Potential Root Cause | Resolution |
|---|---|---|
| Pod stuck in `ContainerCreating` or `FailedMount` on `gitops-git-credentials` | `ExternalSecret` failed to sync `<workflowrun>-gitops-git-secret` from OpenBao. ESO is not running, `ClusterSecretStore/default` is unready, or `secret/gitops-token` is missing. | Verify ESO pod status, verify `kubectl describe clustersecretstore default`, and ensure `secret/gitops-token` is created in OpenBao with property `git-token`. |
| `clone-source` fails with HTTP 401/403 `Authentication failed` | Private source repository specified but `secret/git-token` has an invalid token or insufficient read permissions. | Update `secret/git-token` in OpenBao with a valid GitHub PAT with `repo` read access. |
| `clone-gitops` fails with HTTP 404 `Repository not found` | `gitopsRepoUrl` is pointing to `openchoreo/sample-gitops` (upstream) instead of the user fork, or repository URL is incorrect. | Explicitly pass `gitopsRepoUrl: https://github.com/<your-github-username>/sample-gitops` in the `WorkflowRun` parameters. |
| `clone-gitops` or `git-commit-push-pr` fails with HTTP 401/403 or `Permission denied` | `secret/gitops-token` PAT has expired or lacks write permissions (`repo` scope) to create branches and PRs on the fork. | Generate a GitHub PAT with `repo` scope and store it in OpenBao: `kubectl exec -n openbao openbao-0 -- bao kv put secret/gitops-token git-token=<pat>`. |
| Network error `Could not resolve host: github.com` | Kubernetes DNS or egress connectivity failure from the workflow pod. | Check CoreDNS pods (`kubectl get pods -n kube-system -l k8s-app=kube-dns`) and node network egress settings. |
