# Module 5 — Production-Style Deployment: ECS Fargate + ALB + CI/CD

**Project:** FreshBasket Truck Delay Classification — Deployment Unit

## What this repo is

`truck-delay-deploy` is the deployable unit for Module 5. It contains:

- `Dockerfile` — at repo root
- `app.py` — Streamlit dashboard
- `requirements.txt`
- `artifacts/` — trained model artifacts
- `buildspec.yml` — CodeBuild build spec (`BUILD_CONTEXT=.`, builds from repo root)

The image content is functionally identical to Module 4's image; it is simply rebuilt fresh
from this repo's root so that `buildspec.yml`'s `BUILD_CONTEXT=.` resolves correctly during
CI/CD.

## Parts completed this session

| Part | Description | Outcome |
|------|-------------|---------|
| **Part 1** | Build & push image to ECR | Image `truck-delay-app:v1` + `:latest` + `:<commit-sha>` pushed |
| **Lab A** | ECS cluster + Fargate task + service | Automated via `labA/m5_labA_deploy.sh`; task running, reachable on port 8501 |
| **Lab B** | Application Load Balancer + target group | Stable public URL on port 80; task SG locked to ALB-only access; health checks passing |
| **Lab D** | CodeBuild + CodePipeline CI/CD | `git push` → auto build → auto ECS deploy; full loop proven with a real code change |

## Final live state (before teardown)

- Dashboard reachable at `http://<ALB_DNS_NAME>` (port 80, no port number needed in URL)
- Full CI/CD loop proven end-to-end: a real code change (Dockerfile base-image fix from
  Docker Hub to ECR Public mirror) was committed, pushed, auto-built by CodeBuild, and
  auto-deployed by CodePipeline + ECS — `RolloutState: COMPLETED`, target `healthy`,
  dashboard reachable

## Teardown

**Session ended with full teardown of all M5 AWS resources:**

- CodePipeline, CodeBuild project, CodePipeline's auto-created S3 artifact bucket
- ECS service (scaled to 0, then deleted with `--force`), ECS cluster, task definitions deregistered
- ALB listener, ALB, target group, ALB security group
- CloudWatch log group `/ecs/truck-delay-service`

**Intentionally preserved:** ECR repository `truck-delay-app` and its images (`v1`, `latest`,
`<commit-sha>`) — confirmed via final verification script. Low cost to retain; required for
re-runs without rebuilding from scratch.

**Not touched (low/no cost):** `ecsTaskExecutionRole`, `mlops-codepipeline-policy` inline
IAM policy, CodeStar GitHub connection `truck-delay-github-connection`.
