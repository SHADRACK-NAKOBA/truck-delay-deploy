# Prerequisites

## Software

| Tool | Notes |
|------|-------|
| Docker Desktop | Must be running before any `docker build` / `docker run` steps |
| AWS CLI v2 | Configured with credentials for `<AWS_REGION>` |
| `jq` | JSON processor — required by `m5_labA_deploy.sh`. **Windows Git Bash gotcha:** install to `~/bin`, not `/usr/bin` — the Git install directory is not writable (see Issues #2). |
| Git Bash | Used as the shell for all AWS CLI commands in this session |

### Installing `jq` on Windows Git Bash (the right way)

```bash
mkdir -p ~/bin
curl -Lo ~/bin/jq.exe https://github.com/jqlang/jq/releases/latest/download/jq-windows-amd64.exe
chmod +x ~/bin/jq.exe
export PATH="$HOME/bin:$PATH"
# Persist across sessions:
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
```

## AWS IAM permissions

The IAM user (`<IAM_USERNAME>`) needs the following actions. Below are the gaps discovered
during this session and how each was resolved:

| Service | Permissions needed | How it was granted |
|---------|-------------------|-------------------|
| ECR | `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage` | Pre-existing `eksctl-ecr-permissions` inline policy |
| ECS | `ecs:*` (full) | Added to `mlops-deploy-policy` inline policy |
| ALB / target groups | `elasticloadbalancing:*` (EC2 full covers `elbv2:*`) | Pre-existing EC2 permissions |
| IAM | `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PassRole`, `iam:GetRole`, `iam:ListRoles`, `iam:DeleteRole`, `iam:DetachRolePolicy`, `iam:DeletePolicy`, `iam:ListAttachedRolePolicies` | Added via new inline policy `mlops-codepipeline-policy` (see Issues #7) |
| CloudWatch Logs | `logs:CreateLogGroup`, `logs:CreateLogDelivery`, `logs:DescribeLogGroups`, `logs:DeleteLogGroup` | Added to inline policy |
| CodeBuild | `codebuild:*` | Added to `mlops-codepipeline-policy` inline policy (Issues #7) |
| CodePipeline | `codepipeline:*` | Added to `mlops-codepipeline-policy` inline policy (Issues #7) |
| CodeStar / CodeConnections | `codestar-connections:*`, `codeconnections:*` | Added to `mlops-codepipeline-policy` inline policy (Issues #7) |
| S3 | `s3:*` (for CodePipeline artifact bucket) | Pre-existing permissions |

> **Note on the 10-managed-policy quota:** AWS enforces a hard limit of 10 *managed* policies
> per IAM user. During this session the user was already at 10, so additional permissions had
> to be granted as an *inline* policy (`mlops-codepipeline-policy`) — inline policies do not
> count toward the managed-policy quota. See Issues #7 for details.

## GitHub repository

This repo must have the following at its **root** (not nested under a subdirectory):

- `Dockerfile`
- `app.py`
- `requirements.txt`
- `buildspec.yml`
- `artifacts/`

This is because `buildspec.yml` uses `BUILD_CONTEXT=.` — the repo root IS the Docker build
context for CodeBuild.

## Script files

Copy from course materials into a working folder (e.g. `~/Desktop/MLOps-Module-5/labA/`):

- `m5_labA_deploy.sh` — creates the ECS cluster, task definition, service, and associated resources
- `m5_labA_teardown.sh` — tears down everything created by the deploy script

```bash
chmod +x m5_labA_deploy.sh m5_labA_teardown.sh
```
