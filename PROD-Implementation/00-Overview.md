# Module 5 — Production Assessment

## What was built

Module 5 is the closest this spine project gets to a real production deployment pattern:

```
GitHub push
    → CodePipeline (Source)
    → CodeBuild (docker build + docker push to ECR)
    → ECS Fargate deploy
    → ALB (HTTP:80)
    → Users
```

This is a complete container-based CI/CD pipeline: every `git push` to `main` produces a new
Docker image in ECR and deploys it to a running ECS Fargate service behind an Application
Load Balancer.

---

## What is learning-grade in this implementation

| Area | What was done | Why it's learning-grade |
|------|--------------|------------------------|
| **VPC** | Default VPC, public subnets for ECS tasks | Default VPC has no network isolation; tasks have public IPs directly |
| **Encryption** | HTTP only (port 80) | No ACM certificate, no HTTPS, no HTTP→HTTPS redirect |
| **Scale** | `desiredCount=1`, no auto scaling | Single task, single AZ effectively; any task failure = downtime |
| **IAM** | `AmazonEC2ContainerRegistryFullAccess` on CodeBuild role; broad `codebuild:*` / `codepipeline:*` inline policy on IAM user | Far beyond least privilege; a compromised CI role has push access to all ECR repos |
| **Deploy strategy** | Rolling ECS deploy (CodePipeline ECS action) | No blue/green; a bad deploy with circuit breaker disabled = endless retry loop (hit in this session as Issue #12) |
| **Secrets** | No runtime secrets in this app; env vars in buildspec are non-sensitive IDs | Pattern does not yet use Secrets Manager / SSM injection for sensitive values |
| **Base image** | `public.ecr.aws/docker/library/python:3.12-slim` by tag | Not pinned by digest; tag can be mutated upstream |
| **ECR lifecycle** | No lifecycle policy | Commit-tagged images accumulate indefinitely |
| **Health check grace period** | Hard-coded 90s via CLI after the fact | Should be set in IaC from day one; was discovered only after Issue #12 |

---

## What is genuinely prod-grade in this implementation

| Area | What was done | Why it counts |
|------|--------------|---------------|
| **Container isolation** | App runs in a Fargate task (no shared OS with other workloads) | Serverless compute; no EC2 instance patching |
| **Image registry** | ECR with immutable tags (`v1`, `latest`, commit SHA) | Private registry; images auditable by commit |
| **Load balancer** | ALB with health checks (`/_stcore/health`) | Traffic only goes to healthy tasks; ALB is the only public entry point |
| **Task SG lockdown** | Task security group only allows port 8501 from ALB SG; not from internet | Defense-in-depth; direct task access blocked even though tasks have public IPs |
| **CI/CD** | Full automated pipeline: push → build → deploy | Reproducible, auditable; no manual `docker push` or `aws ecs update-service` |
| **Build environment** | ECR Public mirror for base image | Eliminates Docker Hub pull rate limit failures in CI |
| **Teardown discipline** | Full teardown with verification script | No orphaned resources running up cost after the session |
