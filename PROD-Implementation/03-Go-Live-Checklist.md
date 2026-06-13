# Go-Live Checklist

Run through every item below before declaring the environment production. Each item
should be verified by observation, not assumed.

---

## HTTPS & networking

- [ ] `curl -I https://app.yourdomain.com` returns `HTTP/2 200`
- [ ] `curl -I http://app.yourdomain.com` returns `HTTP/1.1 301 Moved Permanently`
      and `Location: https://...`
- [ ] ACM certificate status is `Issued` (not `Pending validation`)
- [ ] ACM certificate covers the exact domain (or wildcard) used by the ALB listener
- [ ] ECS tasks are in private subnets with `assignPublicIp=DISABLED` — confirm no tasks
      have a public IP in `describe-tasks`

## Availability

- [ ] ECS service `runningCount` ≥ 2, tasks spread across ≥ 2 AZs
      (`describe-tasks` → check `availabilityZone` on each task)
- [ ] ALB target group shows all registered targets as `healthy`
- [ ] `healthCheckGracePeriodSeconds` is set appropriately for this image's startup time
      (90 s was correct for the current Streamlit app — re-validate after any significant
      dependency or model change)
- [ ] Auto Scaling policy is attached; min capacity ≥ 2

## CI/CD & deploy safety

- [ ] **Blue/green rollback test passed**: deliberately deployed a broken image; CodeDeploy
      detected health-check failure and rolled back automatically; no user-visible downtime
- [ ] CodePipeline → CodeDeploy deploy action is active (not the rolling ECS action)
- [ ] Deployment circuit breaker is enabled with rollback on the ECS service as a fallback
- [ ] A manual rollback procedure is documented and has been rehearsed:
      ```bash
      aws ecs update-service \
        --cluster <CLUSTER_NAME> \
        --service truck-delay-service \
        --task-definition truck-delay-task:<PREVIOUS_REVISION> \
        --force-new-deployment \
        --region <AWS_REGION>
      ```

## IAM

- [ ] CodeBuild service role has only scoped ECR push access (not `ECRFullAccess`)
- [ ] No `iam:CreateRole` / `iam:AttachRolePolicy` in any CI/CD role
- [ ] All infrastructure roles are provisioned via IaC; no manually created roles in the
      production account
- [ ] ECS task role (if any runtime AWS API calls are needed) is scoped to exactly the
      resources the application reads/writes

## Secrets

- [ ] No plaintext secrets in `buildspec.yml` environment variables or ECS task definition
      `environment` blocks
- [ ] All runtime secrets in Secrets Manager / SSM; referenced in task definition `secrets`
      block

## Observability

- [ ] CloudWatch alarm: `ECS/CPUUtilization` > 80% sustained → SNS → email/PagerDuty
- [ ] CloudWatch alarm: `ECS/MemoryUtilization` > 80% sustained
- [ ] CloudWatch alarm: `RunningTaskCount` < `DesiredTaskCount` (any value > 0) —
      fires immediately if a task exits unexpectedly
- [ ] ALB alarm: `HTTPCode_ELB_5XX_Count` > 10 in 5 minutes
- [ ] ALB alarm: `UnHealthyHostCount` > 0 sustained for > 2 minutes
- [ ] CloudWatch Logs: application logs are flowing to `/ecs/truck-delay-service`
      (or a renamed production log group); log retention is set (e.g., 30 days)

## Cost

- [ ] AWS Budgets alarm set for the production account (or the relevant cost allocation tag)
- [ ] ECR lifecycle policy is active — confirm stale images are expiring on schedule

## Image hygiene

- [ ] `Dockerfile` base image is pinned by digest (`@sha256:<digest>`), not just tag
- [ ] The latest build used the ECR Public mirror (`public.ecr.aws/docker/library/python:...`)
      for the base image — no Docker Hub pulls in CI

## DNS

- [ ] Route 53 alias record points to the ALB; TTL is appropriate
- [ ] Downstream services / monitoring that reference the raw ALB DNS name have been updated
      to use the canonical domain name

## Runbook

- [ ] A brief runbook exists covering:
      - How to trigger a manual deploy
      - How to roll back to the previous task revision
      - How to scale the service up/down manually
      - Who to page if `UnHealthyHostCount` > 0 persists
