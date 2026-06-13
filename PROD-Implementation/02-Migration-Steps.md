# Migration Steps — Learning to Production

These steps describe how to take the patterns proven in this session and harden them for
a real production environment. Steps reference the original Lab A/B/D sequence and explain
what changes.

---

## Step 1 — Provision a custom VPC via IaC

Replace the "default VPC" approach from Lab B with a purpose-built VPC.

```bash
# Example using CloudFormation or Terraform; pseudocode shown here
# At minimum the VPC needs:
#   - 2+ public subnets (different AZs) for the ALB
#   - 2+ private subnets (different AZs) for ECS tasks
#   - NAT Gateway (one per AZ for HA) OR VPC endpoints for ECR/S3/CWL/Secrets Manager
#   - Internet Gateway attached to the VPC (required for the public-subnet ALB)
```

Outputs to collect from IaC and use in subsequent steps:
- `<VPC_ID>` — the new VPC
- `<PUBLIC_SUBNET_ID_1>`, `<PUBLIC_SUBNET_ID_2>` — ALB subnets
- `<PRIVATE_SUBNET_ID_1>`, `<PRIVATE_SUBNET_ID_2>` — ECS task subnets

---

## Step 2 — Re-run Lab A with private subnets and no public IP

When running `m5_labA_deploy.sh` (or re-creating the task definition / service manually),
change the network configuration for the ECS service:

```bash
# In the service create / update call:
--network-configuration "awsvpcConfiguration={
  subnets=[<PRIVATE_SUBNET_ID_1>,<PRIVATE_SUBNET_ID_2>],
  securityGroups=[<TASK_SG_ID>],
  assignPublicIp=DISABLED
}"
```

Tasks in private subnets reach ECR and CloudWatch Logs via NAT or VPC endpoints.
They do **not** get public IPs, so the Lab A "smoke-test via public IP" step is no longer
applicable — test via the ALB instead.

---

## Step 3 — Re-run Lab B targeting the new VPC; add HTTPS

```bash
# Use the new VPC's public subnets for the ALB:
aws elbv2 create-load-balancer \
  --name truck-delay-alb \
  --type application \
  --scheme internet-facing \
  --subnets <PUBLIC_SUBNET_ID_1> <PUBLIC_SUBNET_ID_2> \
  --security-groups <ALB_SG_ID> \
  --region <AWS_REGION>

# Add the HTTPS listener (after ACM cert is issued and validated):
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN> \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=<ACM_CERT_ARN> \
  --default-actions "Type=forward,TargetGroupArn=<TG_ARN>" \
  --region <AWS_REGION>

# Redirect HTTP → HTTPS:
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN> \
  --protocol HTTP --port 80 \
  --default-actions 'Type=redirect,RedirectConfig={Protocol=HTTPS,Port=443,StatusCode=HTTP_301}' \
  --region <AWS_REGION>
```

The target group health check and port (8501) remain the same.

---

## Step 4 — Re-create CodeBuild + CodePipeline with scoped IAM roles

Instead of attaching `AmazonEC2ContainerRegistryFullAccess` to the CodeBuild role after
the fact, provision the CodeBuild role via IaC with a scoped ECR policy from the start:

```json
{
  "Effect": "Allow",
  "Action": [
    "ecr:GetAuthorizationToken"
  ],
  "Resource": "*"
},
{
  "Effect": "Allow",
  "Action": [
    "ecr:BatchCheckLayerAvailability",
    "ecr:InitiateLayerUpload",
    "ecr:UploadLayerPart",
    "ecr:CompleteLayerUpload",
    "ecr:PutImage"
  ],
  "Resource": "arn:aws:ecr:<AWS_REGION>:<AWS_ACCOUNT_ID>:repository/truck-delay-app"
}
```

For a production branching model, consider:
- `main` branch → deploys to **staging**
- `release/*` tags or a protected `production` branch → deploys to **prod**

This requires two separate CodePipelines (or one pipeline with environment-gated approval
stages) and two ECS service targets.

---

## Step 5 — Switch CodePipeline ECS deploy to CodeDeploy blue/green

Replace the "Amazon ECS" deploy action in CodePipeline with a **CodeDeploy** action:

1. Create a CodeDeploy application (`truck-delay-app-codedeploy`) and a deployment group
   (`truck-delay-dg`) of compute platform `ECS`
2. Create a second "green" target group (`truck-delay-tg-green`) with the same protocol,
   port, and health-check settings as the blue target group
3. Configure the deployment group with:
   - ECS cluster + service reference
   - Both target groups (blue = active, green = replacement)
   - The ALB listener ARN and the test listener (optional, for pre-swap validation)
   - Deployment config: `CodeDeployDefault.ECSAllAtOnce` (or a canary/linear config)
4. Update the CodePipeline deploy action to use `AWS CodeDeploy` and the deployment group

With this in place, a broken image is automatically rolled back before it takes production
traffic, and the old task set stays running until the new one passes health checks.

---

## Step 6 — Add Auto Scaling

```bash
# Register the ECS service as an Auto Scaling target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/m5-truck-delay-cluster/truck-delay-service \
  --min-capacity 2 \
  --max-capacity 10

# Target tracking on CPU utilization
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/m5-truck-delay-cluster/truck-delay-service \
  --policy-name truck-delay-cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

---

## Step 7 — DNS via Route 53

```bash
# Alias record (no IP to manage; ALB IPs can change):
aws route53 change-resource-record-sets \
  --hosted-zone-id <HOSTED_ZONE_ID> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.yourdomain.com.",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "<ALB_CANONICAL_HOSTED_ZONE_ID>",
          "DNSName": "<ALB_DNS_NAME>.",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

---

## Step 8 — Full end-to-end smoke test before going live

1. Push a known-good commit to the production branch
2. Confirm pipeline: Source → Build → Deploy all green
3. Confirm ECS service: `RolloutState: COMPLETED`, `runningCount` = `desiredCount` ≥ 2
4. Confirm target health: all targets `healthy` in the target group
5. `curl -I https://app.yourdomain.com` → `HTTP/2 200`
6. `curl -I http://app.yourdomain.com` → `HTTP/1.1 301 Moved Permanently` (redirect)
7. Deliberately push a broken image (e.g., `CMD ["sleep", "infinity"]` — no HTTP server)
   and confirm CodeDeploy rolls back automatically; old tasks stay healthy; users see no
   downtime
8. Restore the correct image and re-verify

Only declare the environment production after step 7 (the rollback test) passes.
