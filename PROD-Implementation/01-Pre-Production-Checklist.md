# Pre-Production Checklist

Work through these items before promoting this deployment pattern to a production environment.
Items are roughly ordered by risk reduction impact.

---

## Networking & encryption

- [ ] **HTTPS / TLS**
  - Request an ACM certificate for the real domain (or wildcard)
  - Add an HTTPS:443 listener on the ALB; attach the ACM cert
  - Add an HTTP:80 listener rule that redirects all traffic to HTTPS (`HTTP 301`)
  - Remove or repurpose the current HTTP:80 forward-to-TG rule
  - ACM certificates auto-renew; no manual renewal process needed

- [ ] **Custom VPC with private subnets**
  - Create a VPC with at least 2 AZs: public subnets (ALB), private subnets (ECS tasks)
  - Add a NAT Gateway in each public subnet (one per AZ for HA) so tasks in private subnets
    can reach ECR, S3, CloudWatch, and Secrets Manager without public IPs
  - Alternatively, replace NAT with VPC Interface Endpoints for ECR (`ecr.api`, `ecr.dkr`),
    S3 Gateway Endpoint, CloudWatch Logs, and Secrets Manager — lower ongoing cost than NAT
    for this traffic pattern
  - Move `assignPublicIp=DISABLED` for ECS tasks (tasks in private subnets do not need
    and should not have public IPs)
  - Move the ALB to the public subnets only

- [ ] **ALB access logs**
  - Enable ALB access logging to S3 for audit and debugging

---

## Availability & resilience

- [ ] **Multi-task / multi-AZ**
  - Set `desiredCount` ≥ 2 in the ECS service
  - Pin subnets across ≥ 2 AZs in the task network configuration
  - The ALB already spans multiple AZs; the target group will distribute traffic once ≥ 2
    tasks are registered

- [ ] **Auto Scaling**
  - Register an Application Auto Scaling target on the ECS service
  - Attach a Target Tracking policy on `ECSServiceAverageCPUUtilization` (e.g., target 60%)
  - Consider a second policy on ALB request count per target for bursty traffic
  - Set `minCapacity` ≥ 2 and a reasonable `maxCapacity`

- [ ] **Deployment circuit breaker with rollback**
  - In the ECS service's `deploymentConfiguration`, set:
    ```json
    "deploymentCircuitBreaker": { "enable": true, "rollback": true }
    ```
  - This would have prevented Issue #12's infinite retry loop automatically
  - Or switch the CodePipeline ECS deploy action to **CodeDeploy blue/green** for zero-
    downtime deploys with automatic rollback on health-check failure

- [ ] **Tuned health check grace period**
  - 90 seconds was correct for this image; re-validate if the image's startup time changes
    (larger models, more imports, etc.)
  - Set `healthCheckGracePeriodSeconds` in IaC, not as a post-hoc CLI fix

---

## IAM — least privilege

- [ ] **CodeBuild service role — ECR**
  - Replace `AmazonEC2ContainerRegistryFullAccess` with a scoped policy allowing push to
    exactly one ECR repository:
    ```json
    {
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "arn:aws:ecr:<AWS_REGION>:<AWS_ACCOUNT_ID>:repository/truck-delay-app"
    }
    ```

- [ ] **IAM user — CI/CD permissions**
  - Replace the broad `codebuild:*` / `codepipeline:*` / `iam:*` inline policy with
    per-resource scoped policies
  - Remove `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:DeleteRole`, etc. once all
    infrastructure roles are stable and provisioned via IaC — CI should not be able to
    create or modify its own IAM roles

- [ ] **ECS task execution role**
  - `ecsTaskExecutionRole` uses `AmazonECSTaskExecutionRolePolicy` (AWS managed) — acceptable,
    but consider scoping the Secrets Manager / SSM read actions to only the ARNs this
    service needs once secrets are in use

- [ ] **Principle: provision roles via IaC**
  - Once the environment is stable, remove all manual `iam:CreateRole` / `iam:AttachRolePolicy`
    grants from human user policies; roles should only be created/modified via CloudFormation
    or Terraform in a controlled pipeline

---

## Secrets management

- [ ] **No plaintext secrets in buildspec or task definitions**
  - Any future runtime secrets (DB passwords, API keys, third-party tokens) must go in
    **AWS Secrets Manager** or **SSM Parameter Store**
  - Inject them via the task definition's `secrets` block, not `environment`
  - Reference them in `buildspec.yml` only as SSM parameters using the `parameter-store`
    syntax — never hardcode values

---

## Image hygiene

- [ ] **Pin base image by digest**
  ```dockerfile
  FROM public.ecr.aws/docker/library/python:3.12-slim@sha256:<digest>
  ```
  Tags like `3.12-slim` can be updated upstream; a digest pin makes the build reproducible.

- [ ] **ECR lifecycle policy**
  - Add a lifecycle policy to the `truck-delay-app` repository to expire untagged images
    after N days and keep only the last M tagged images:
  ```json
  {
    "rules": [
      {
        "rulePriority": 1,
        "description": "Expire untagged images after 14 days",
        "selection": { "tagStatus": "untagged", "countType": "sinceImagePushed",
                       "countUnit": "days", "countNumber": 14 },
        "action": { "type": "expire" }
      },
      {
        "rulePriority": 2,
        "description": "Keep last 10 tagged images",
        "selection": { "tagStatus": "tagged", "tagPrefixList": ["v", "sha"],
                       "countType": "imageCountMoreThan", "countNumber": 10 },
        "action": { "type": "expire" }
      }
    ]
  }
  ```

---

## Observability & cost

- [ ] **CloudWatch alarms**
  - ECS service: `CPUUtilization` > 80% (sustained), `MemoryUtilization` > 80%,
    `RunningTaskCount` < `DesiredTaskCount`
  - ALB: `HTTPCode_ELB_5XX_Count` > threshold, `TargetResponseTime` p99 > threshold,
    `UnHealthyHostCount` > 0

- [ ] **Cost alarm**
  - Create a CloudWatch billing alarm for the AWS account (or the relevant cost allocation
    tags) so unexpected resource cost is caught early

- [ ] **Structured logging**
  - The current container logs to CloudWatch Logs via `awslogs` driver; add log insights
    queries or a log-based metric for application errors

---

## DNS

- [ ] **Route 53 alias record**
  - Once HTTPS is set up, point the real domain at the ALB via an alias A record in Route 53
  - Update any downstream services or health-check endpoints that reference the raw ALB DNS
    name
