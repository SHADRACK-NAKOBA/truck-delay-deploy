# Step-by-Step Walkthrough

## Part 1 — Build & Push Image to ECR

**1.** Change into the repo root:
```bash
cd ~/Desktop/MLOps-Module-5  # or wherever truck-delay-deploy is checked out
```

**2.** Build the Docker image:
```bash
docker build -t truck-delay-app:v1 .
```

**3.** Smoke-test locally, then clean up:
```bash
docker run -d -p 8501:8501 --name truck-delay-test truck-delay-app:v1
# Open http://localhost:8501 — confirm dashboard loads
docker stop truck-delay-test && docker rm truck-delay-test
```

**4.** Authenticate Docker to ECR:
```bash
aws ecr get-login-password --region <AWS_REGION> \
  | docker login --username AWS \
    --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com
```

**5.** Tag the image for ECR:
```bash
docker tag truck-delay-app:v1 <ECR_REPO_URI>/truck-delay-app:v1
```

**6.** Push:
```bash
docker push <ECR_REPO_URI>/truck-delay-app:v1
```

**7.** Verify the image is in ECR:
```bash
aws ecr describe-images \
  --repository-name truck-delay-app \
  --region <AWS_REGION>
```

---

## Part 2 — Lab A (ECS Cluster + Fargate Service, automated)

**8.** Prepare the script files:
```bash
cd ~/Desktop/MLOps-Module-5/labA
chmod +x m5_labA_deploy.sh m5_labA_teardown.sh
```

**9.** Install `jq` if not already available (Git Bash):
```bash
mkdir -p ~/bin
curl -Lo ~/bin/jq.exe https://github.com/jqlang/jq/releases/latest/download/jq-windows-amd64.exe
chmod +x ~/bin/jq.exe
export PATH="$HOME/bin:$PATH"
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
```

**10.** Run the deploy script:
```bash
AWS_REGION=<AWS_REGION> bash m5_labA_deploy.sh
```

The script creates:
- ECS cluster: `m5-truck-delay-cluster`
- IAM role: `ecsTaskExecutionRole`
- CloudWatch log group: `/ecs/truck-delay-service`
- Task definition: `truck-delay-task` (Fargate, 256 CPU / 512 MB, container port 8501)
- Security group: opens inbound TCP 8501 from `0.0.0.0/0`
- ECS service: `truck-delay-service` (Fargate, `desiredCount=1`)

**11.** Verify the service is running:
```bash
aws ecs describe-services \
  --cluster m5-truck-delay-cluster \
  --services truck-delay-service \
  --region <AWS_REGION> \
  --query "services[0].{Running:runningCount,Desired:desiredCount}"
# Expected: {"Running": 1, "Desired": 1}
```

**12.** Get the task's public IP and smoke-test:
```bash
TASK_ARN=$(aws ecs list-tasks \
  --cluster m5-truck-delay-cluster \
  --service-name truck-delay-service \
  --region <AWS_REGION> \
  --query "taskArns[0]" --output text)

ENI_ID=$(aws ecs describe-tasks \
  --cluster m5-truck-delay-cluster \
  --tasks "$TASK_ARN" \
  --region <AWS_REGION> \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
  --output text)

PUBLIC_IP=$(aws ec2 describe-network-interfaces \
  --network-interface-ids "$ENI_ID" \
  --region <AWS_REGION> \
  --query "NetworkInterfaces[0].Association.PublicIp" \
  --output text)

echo "http://$PUBLIC_IP:8501"
```
Open the URL to confirm the dashboard loads.

---

## Part 3 — Lab B (Application Load Balancer)

> **Important:** Run `export MSYS_NO_PATHCONV=1` at the start of this section and keep it
> for the rest of the session. Git Bash mangles paths like `/_stcore/health` into Windows
> paths without it (see Issues #3).

**13.** Set environment variables:
```bash
export AWS_REGION=<AWS_REGION>
export MSYS_NO_PATHCONV=1

# Default VPC
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --region "$AWS_REGION" \
  --query "Vpcs[0].VpcId" --output text)

# All subnets in the default VPC (need at least 2 in different AZs for ALB)
SUBNETS=($(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --region "$AWS_REGION" \
  --query "Subnets[*].SubnetId" --output text))
```

**14.** Create the target group:
```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name truck-delay-tg \
  --protocol HTTP \
  --port 8501 \
  --vpc-id "$VPC_ID" \
  --target-type ip \
  --health-check-protocol HTTP \
  --health-check-path '/_stcore/health' \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --region "$AWS_REGION" \
  --query "TargetGroups[0].TargetGroupArn" --output text)
echo "TG_ARN=$TG_ARN"
```

**15.** Create the ALB security group (allows port 80 from anywhere):
```bash
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name truck-delay-alb-sg \
  --description "ALB SG for truck-delay" \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION" \
  --query "GroupId" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$ALB_SG_ID" \
  --protocol tcp --port 80 --cidr 0.0.0.0/0 \
  --region "$AWS_REGION"
```

**16.** Create the ALB (internet-facing; use 2 subnets in different AZs):
```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name truck-delay-alb \
  --type application \
  --scheme internet-facing \
  --subnets "${SUBNETS[0]}" "${SUBNETS[1]}" \
  --security-groups "$ALB_SG_ID" \
  --region "$AWS_REGION" \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)
echo "ALB_ARN=$ALB_ARN"
```

**17.** Create the HTTP:80 listener forwarding to the target group:
```bash
aws elbv2 create-listener \
  --load-balancer-arn "$ALB_ARN" \
  --protocol HTTP --port 80 \
  --default-actions "Type=forward,TargetGroupArn=$TG_ARN" \
  --region "$AWS_REGION"
```

**18.** Wait for the ALB to be fully available:
```bash
aws elbv2 wait load-balancer-available \
  --load-balancer-arns "$ALB_ARN" \
  --region "$AWS_REGION"
```

**19.** Get the ALB DNS name:
```bash
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns "$ALB_ARN" \
  --region "$AWS_REGION" \
  --query "LoadBalancers[0].DNSName" --output text)
echo "http://$ALB_DNS"
```

**20.** Register the running ECS task with the target group by updating the service:
```bash
TASK_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=truck-delay-task-sg" \
  --region "$AWS_REGION" \
  --query "SecurityGroups[0].GroupId" --output text)

aws ecs update-service \
  --cluster m5-truck-delay-cluster \
  --service truck-delay-service \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=truck-delay-app,containerPort=8501" \
  --force-new-deployment \
  --region "$AWS_REGION"
```

**21.** Poll until the deployment completes:
```bash
aws ecs describe-services \
  --cluster m5-truck-delay-cluster \
  --services truck-delay-service \
  --region "$AWS_REGION" \
  --query "services[0].deployments[0].{Status:rolloutState,Running:runningCount}"
# Wait until rolloutState = COMPLETED
```

**22.** Verify target health:
```bash
aws elbv2 describe-target-health \
  --target-group-arn "$TG_ARN" \
  --region "$AWS_REGION"
# Wait until TargetHealth.State = healthy
```

**23.** Open `http://<ALB_DNS_NAME>` in a browser — confirm dashboard.

**24.** Lock down the task security group (only allow 8501 from the ALB SG, not from the internet):
```bash
# Revoke the broad inbound rule opened by the deploy script
aws ec2 revoke-security-group-ingress \
  --group-id "$TASK_SG_ID" \
  --protocol tcp --port 8501 --cidr 0.0.0.0/0 \
  --region "$AWS_REGION"

# Allow only the ALB security group
aws ec2 authorize-security-group-ingress \
  --group-id "$TASK_SG_ID" \
  --protocol tcp --port 8501 \
  --source-group "$ALB_SG_ID" \
  --region "$AWS_REGION"
```

**25.** Verify the lockdown:
```bash
# Should time out (connection refused / no response):
curl --max-time 5 http://$PUBLIC_IP:8501

# Should return HTTP 200:
curl -I http://$ALB_DNS
```

---

## Part 4 — Lab D (CodeBuild + CodePipeline CI/CD)

**26.** Ensure `buildspec.yml` is at the repo root. The final working version is already
committed in this repo. Key points:
- All commands that contain `|`, `:=`, or `: ` (colon-space) must be wrapped in YAML single
  quotes to avoid YAML parse errors (see Issues #9)
- Env vars `AWS_ACCOUNT_ID`, `IMAGE_REPO_NAME`, `CONTAINER_NAME`, `AWS_DEFAULT_REGION` are
  supplied by the CodeBuild project — **not hardcoded** in `buildspec.yml`
- The `post_build` phase writes `imagedefinitions.json` for CodePipeline's ECS deploy action

**27.** Commit and push `buildspec.yml`:
```bash
git add buildspec.yml
git commit -m "ci: add CodeBuild buildspec"
git push origin main
```

**28.** Create a CodeStar/CodeConnections connection to GitHub:
```bash
aws codeconnections create-connection \
  --provider-type GitHub \
  --connection-name truck-delay-github-connection \
  --region "$AWS_REGION"
```
Then in the AWS Console: **Developer Tools → Connections → truck-delay-github-connection →
Update pending connection** — authorize via GitHub OAuth. Wait until status is `AVAILABLE`
before proceeding.

**29.** Create the CodeBuild project in the AWS Console (**CodeBuild → Build projects → Create
build project**):
- **Source:** GitHub via the connection `truck-delay-github-connection`; pick repo from dropdown
- **Branch:** `main`
- **Environment:** Amazon Linux 2, Standard image (latest), **Privileged mode: checked**
  (required for `docker build` inside CodeBuild)
- **Service role:** New service role (name auto-assigned: `codebuild-truck-delay-build-service-role`)
- **Environment variables:**
  - `AWS_ACCOUNT_ID` = `<AWS_ACCOUNT_ID>`
  - `IMAGE_REPO_NAME` = `truck-delay-app`
  - `CONTAINER_NAME` = `truck-delay-app`
  - `AWS_DEFAULT_REGION` = `<AWS_REGION>`

**30.** Grant the CodeBuild service role ECR push access:
```bash
aws iam attach-role-policy \
  --role-name codebuild-truck-delay-build-service-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess
```

**31.** Test the build standalone (CodeBuild → Start build) — confirm `Succeeded` before
wiring it into the pipeline.

**32.** Create the pipeline (**CodePipeline → Create pipeline**):
- **Source stage:** GitHub V2 via `truck-delay-github-connection`; repo = this repo;
  branch = `main`; trigger = Push to branch
- **Build stage:** CodeBuild project `truck-delay-build`; single output artifact
- **Test stage:** skip
- **Deploy stage:** Amazon ECS; cluster = `m5-truck-delay-cluster`;
  service = `truck-delay-service`; image definitions file = `imagedefinitions.json`

**33.** Set the health-check grace period (critical — without this, every pipeline-triggered
deploy will loop forever; see Issues #9):
```bash
aws ecs update-service \
  --cluster m5-truck-delay-cluster \
  --service truck-delay-service \
  --health-check-grace-period-seconds 90 \
  --region "$AWS_REGION"
```

**34.** Trigger an end-to-end test via `git push` (any commit):
```bash
git commit --allow-empty -m "ci: trigger pipeline smoke test"
git push origin main
```
Monitor in the CodePipeline console: **Source → Build → Deploy** must all go green.
Then confirm:
```bash
aws ecs describe-services \
  --cluster m5-truck-delay-cluster \
  --services truck-delay-service \
  --region "$AWS_REGION" \
  --query "services[0].deployments[0].rolloutState"
# Expected: "COMPLETED"

curl -I http://$ALB_DNS
# Expected: HTTP/1.1 200 OK
```

---

## Teardown (performed at end of this session)

**35.** Delete CodePipeline + CodeBuild:
```bash
aws codepipeline delete-pipeline --name truck-delay-pipeline --region "$AWS_REGION"
aws codebuild delete-project --name truck-delay-build --region "$AWS_REGION"

# Delete CodePipeline's auto-created S3 artifact bucket (name: codepipeline-<region>-<random>)
BUCKET=$(aws s3api list-buckets \
  --query "Buckets[?starts_with(Name,'codepipeline-$AWS_REGION')].Name | [0]" \
  --output text)
aws s3 rm s3://$BUCKET --recursive
aws s3api delete-bucket --bucket "$BUCKET" --region "$AWS_REGION"
```

**36.** Tear down ECS:
```bash
# Scale to 0 first so the service can be deleted cleanly
aws ecs update-service \
  --cluster m5-truck-delay-cluster \
  --service truck-delay-service \
  --desired-count 0 \
  --region "$AWS_REGION"

aws ecs delete-service \
  --cluster m5-truck-delay-cluster \
  --service truck-delay-service \
  --force \
  --region "$AWS_REGION"

aws ecs delete-cluster \
  --cluster m5-truck-delay-cluster \
  --region "$AWS_REGION"

# Deregister task definitions
for rev in 1 2 3; do
  aws ecs deregister-task-definition \
    --task-definition "truck-delay-task:$rev" \
    --region "$AWS_REGION" 2>/dev/null || true
done
```

**37.** Delete ALB resources (order matters — listener before ALB):
```bash
LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn "$ALB_ARN" \
  --region "$AWS_REGION" \
  --query "Listeners[0].ListenerArn" --output text)

aws elbv2 delete-listener --listener-arn "$LISTENER_ARN" --region "$AWS_REGION"
aws elbv2 delete-load-balancer --load-balancer-arn "$ALB_ARN" --region "$AWS_REGION"
aws elbv2 wait load-balancers-deleted --load-balancer-arns "$ALB_ARN" --region "$AWS_REGION"
aws elbv2 delete-target-group --target-group-arn "$TG_ARN" --region "$AWS_REGION"
aws ec2 delete-security-group --group-id "$ALB_SG_ID" --region "$AWS_REGION"
```

**38.** Delete CloudWatch log group:
```bash
aws logs delete-log-group \
  --log-group-name /ecs/truck-delay-service \
  --region "$AWS_REGION"
```

**39.** Verify — all sections should be empty except ECR:
```bash
echo "=== ECS clusters ===" && aws ecs list-clusters --region "$AWS_REGION"
echo "=== ALBs ===" && aws elbv2 describe-load-balancers --region "$AWS_REGION" \
  --query "LoadBalancers[?contains(LoadBalancerName,'truck')]"
echo "=== ECR images (preserved) ===" && aws ecr describe-images \
  --repository-name truck-delay-app --region "$AWS_REGION"
```
