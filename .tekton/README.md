# Tekton Pipelines-as-Code for K8s Operator

This directory contains Tekton Pipelines-as-Code configuration for automated CI/CD workflows.

## Overview

The pipeline automates:
- **Linting**: Go code quality checks with golangci-lint
- **Testing**: Unit tests with coverage
- **Building**: Docker image build with Kaniko
- **Deployment**: Operator deployment to Kubernetes

## Prerequisites

1. **Kubernetes cluster** (v1.11.3+)
2. **Tekton Pipelines** installed on your cluster
3. **Tekton Pipelines-as-Code** (optional, for Git event automation)
4. **kubectl** configured to access your cluster

## Installation

### 1. Install Tekton Pipelines

```bash
# Install Tekton Pipelines
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Verify installation
kubectl get pods --namespace tekton-pipelines
```

### 2. Install Tekton Pipelines-as-Code (Optional)

For GitHub/GitLab webhook integration:

```bash
# Install Pipelines-as-Code
kubectl apply --filename https://raw.githubusercontent.com/openshift-pipelines/pipelines-as-code/stable/release.yaml

# Verify installation
kubectl get pods --namespace pipelines-as-code
```

### 3. Install Tekton Catalog Tasks

```bash
# Install git-clone task from Tekton Hub
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
```

### 4. Apply Tekton Resources

```bash
# Apply RBAC and ServiceAccount
kubectl apply -f .tekton/rbac.yaml

# Apply workspace PVC
kubectl apply -f .tekton/workspace-pvc.yaml

# Apply custom tasks
kubectl apply -f .tekton/tasks/

# Apply pipeline
kubectl apply -f .tekton/pipeline.yaml
```

## Configuration

### Docker Registry Credentials

Update `.tekton/rbac.yaml` with your registry credentials:

```yaml
stringData:
  username: "your-docker-username"
  password: "your-docker-password-or-token"
```

Or create the secret separately:

```bash
kubectl create secret docker-registry docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-username> \
  --docker-password=<your-password>
```

### Update PipelineRun Templates

Edit `.tekton/pull-request.yaml` and `.tekton/push-main.yaml`:
- Replace `{{registry}}` with your container registry URL (e.g., `docker.io/myuser`)
- Adjust branch names if needed
- Configure deployment namespace

## Usage

### Manual Execution

Run the pipeline manually:

```bash
kubectl create -f - <<EOF
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: operator-ci-manual-
spec:
  pipelineRef:
    name: operator-ci-cd
  params:
    - name: repo-url
      value: https://github.com/your-org/k8s-operator.git
    - name: revision
      value: main
    - name: image
      value: docker.io/your-username/k8s-operator:latest
    - name: namespace
      value: default
    - name: skip-deploy
      value: "false"
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
    - name: docker-credentials
      secret:
        secretName: docker-credentials
  serviceAccountName: tekton-pipeline-sa
EOF
```

### Watch Pipeline Execution

```bash
# List pipeline runs
kubectl get pipelineruns

# Watch a specific run
tkn pipelinerun logs <pipelinerun-name> -f

# Or with kubectl
kubectl logs -l tekton.dev/pipelineRun=<pipelinerun-name> -f
```

## Pipelines-as-Code Setup

### GitHub Integration

1. **Create a GitHub App** or use a webhook token
2. **Configure Pipelines-as-Code**:
   ```bash
   kubectl create secret generic pipelines-as-code-secret \
     -n pipelines-as-code \
     --from-literal=github.token=<your-github-token>
   ```

3. **Configure Repository**:
   - Add `.tekton/` directory to your repository
   - Pipelines-as-Code will automatically detect and run pipelines on PR/push events

### Available PipelineRuns

- **pull-request.yaml**: Runs on pull requests (tests only, no deploy)
- **push-main.yaml**: Runs on push to main/master (full CI/CD with deploy)

## Pipeline Structure

```
.tekton/
├── README.md                    # This file
├── pipeline.yaml                # Main pipeline definition
├── pull-request.yaml            # PipelineRun for PRs
├── push-main.yaml              # PipelineRun for main branch
├── rbac.yaml                   # RBAC and ServiceAccount
├── workspace-pvc.yaml          # Workspace storage
└── tasks/
    ├── go-test.yaml            # Go unit tests
    ├── go-lint.yaml            # Go linting
    ├── docker-build-push.yaml  # Container image build
    ├── deploy-operator.yaml    # Kubernetes deployment
    └── make.yaml               # Generic make task
```

## Customization

### Add Custom Tasks

Create new task files in `.tekton/tasks/`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: my-custom-task
spec:
  # ... task definition
```

### Modify Pipeline

Edit `.tekton/pipeline.yaml` to add/remove tasks or change execution order.

### Environment Variables

Tasks support various environment variables. Check individual task files for available options.

## Troubleshooting

### Check Pipeline Status
```bash
kubectl get pipelineruns
kubectl describe pipelinerun <name>
```

### View Logs
```bash
# All logs
tkn pipelinerun logs <name>

# Specific task
tkn pipelinerun logs <name> -t <task-name>
```

### Common Issues

1. **Image push fails**: Check docker-credentials secret
2. **Tests fail**: Verify Go version compatibility
3. **Deployment fails**: Check RBAC permissions

## Resources

- [Tekton Documentation](https://tekton.dev/docs/)
- [Tekton Pipelines-as-Code](https://pipelinesascode.com/)
- [Tekton Hub](https://hub.tekton.dev/)
- [Tekton Catalog](https://github.com/tektoncd/catalog)

## Next Steps

1. Configure your container registry credentials
2. Update image references in PipelineRun files
3. Push to your repository to trigger automated pipelines
4. Monitor pipeline execution in your Kubernetes cluster
