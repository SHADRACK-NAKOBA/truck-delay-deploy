# Teardown — Full Verified Cleanup

All M5 AWS resources were deleted at the end of this session. Commands are shown in the
order they were run. All commands used `--region <AWS_REGION>`.

---

## 1. CodePipeline

```bash
aws codepipeline delete-pipeline \
  --name truck-delay-pipeline \
  --region "$AWS_REGION"
```

## 2. CodeBuild project

```bash
aws codebuild delete-project \
  --name truck-delay-build \
  --region "$AWS_REGION"
```

## 3. CodePipeline's auto-created S3 artifact bucket

CodePipeline creates a bucket named `codepipeline-<region>-<random-id>`. It must be
emptied before deletion:

```bash
# Find the bucket
BUCKET=$(aws s3api list-buckets \
  --query "Buckets[?starts_with(Name,'codepipeline-$AWS_REGION')].Name | [0]" \
  --output text)

# Empty and delete
aws s3 rm s3://$BUCKET --recursive
aws s3api delete-bucket --bucket "$BUCKET" --region "$AWS_REGION"
```

## 4. ECS service — scale to 0, then delete

```bash
aws ecs update-service \
  --cluster m5-truck-delay-cluster \
  --service truck-delay-service \
  --desired-count 0 \
  --region "$AWS_REGION"

# Wait for tasks to drain, then force-delete the service
aws ecs delete-service \
  --cluster m5-truck-delay-cluster \
  --service truck-delay-service \
  --force \
  --region "$AWS_REGION"
```

## 5. ECS cluster

```bash
aws ecs delete-cluster \
  --cluster m5-truck-delay-cluster \
  --region "$AWS_REGION"
```

## 6. Task definitions — deregister all revisions

```bash
for rev in 1 2 3 4 5; do
  aws ecs deregister-task-definition \
    --task-definition "truck-delay-task:$rev" \
    --region "$AWS_REGION" 2>/dev/null || true
done
```

## 7. ALB listener + ALB + wait

```bash
LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn "$ALB_ARN" \
  --region "$AWS_REGION" \
  --query "Listeners[0].ListenerArn" --output text)

aws elbv2 delete-listener \
  --listener-arn "$LISTENER_ARN" \
  --region "$AWS_REGION"

aws elbv2 delete-load-balancer \
  --load-balancer-arn "$ALB_ARN" \
  --region "$AWS_REGION"

aws elbv2 wait load-balancers-deleted \
  --load-balancer-arns "$ALB_ARN" \
  --region "$AWS_REGION"
```

## 8. Target group

```bash
aws elbv2 delete-target-group \
  --target-group-arn "$TG_ARN" \
  --region "$AWS_REGION"
```

## 9. ALB security group

```bash
aws ec2 delete-security-group \
  --group-id "$ALB_SG_ID" \
  --region "$AWS_REGION"
```

## 10. CloudWatch log group

```bash
aws logs delete-log-group \
  --log-group-name /ecs/truck-delay-service \
  --region "$AWS_REGION"
```

---

## Final verification

The following script was run to confirm all resources were gone:

```bash
echo "=== ECS clusters ==="
aws ecs list-clusters --region "$AWS_REGION"

echo "=== ECS services ==="
aws ecs list-services \
  --cluster m5-truck-delay-cluster \
  --region "$AWS_REGION" 2>/dev/null || echo "(cluster already deleted)"

echo "=== ALBs ==="
aws elbv2 describe-load-balancers \
  --region "$AWS_REGION" \
  --query "LoadBalancers[?contains(LoadBalancerName,'truck')]"

echo "=== Target groups ==="
aws elbv2 describe-target-groups \
  --region "$AWS_REGION" \
  --query "TargetGroups[?contains(TargetGroupName,'truck')]"

echo "=== CloudWatch log groups ==="
aws logs describe-log-groups \
  --log-group-name-prefix /ecs/truck-delay \
  --region "$AWS_REGION"

echo "=== CodePipelines ==="
aws codepipeline list-pipelines --region "$AWS_REGION"

echo "=== CodeBuild projects ==="
aws codebuild list-projects --region "$AWS_REGION"

echo "=== ECR images (PRESERVED — intentional) ==="
aws ecr describe-images \
  --repository-name truck-delay-app \
  --region "$AWS_REGION"
```

**Result:** All `===` sections were empty except the ECR section.

---

## What was intentionally preserved

| Resource | Reason |
|----------|--------|
| ECR repository `truck-delay-app` with images `v1`, `latest`, `<commit-sha>` | Low cost; required to re-run Parts 2–4 without rebuilding from scratch |
| IAM role `ecsTaskExecutionRole` | Low cost; reusable across re-runs |
| IAM inline policy `mlops-codepipeline-policy` on `<IAM_USERNAME>` | Zero cost; needed for any future CodeBuild/CodePipeline work |
| CodeStar GitHub connection `truck-delay-github-connection` | Zero cost; saves the OAuth re-authorization step on re-runs |

## Optional cleanup (not performed, no urgency)

If a full clean slate is desired in the future:

```bash
# Delete ecsTaskExecutionRole
aws iam detach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name ecsTaskExecutionRole

# Remove the inline policy from the IAM user
aws iam delete-user-policy \
  --user-name <IAM_USERNAME> \
  --policy-name mlops-codepipeline-policy

# Delete the CodeStar connection
aws codeconnections delete-connection \
  --connection-arn <GITHUB_CONNECTION_ARN> \
  --region "$AWS_REGION"
```
