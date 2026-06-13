# FAQ

---

**Q: Why did the first CI/CD deploy get stuck in an infinite retry loop instead of failing cleanly?**

Two root causes combined:

1. `healthCheckGracePeriodSeconds` was `0` (the default). The ALB started health-checking
   new tasks the moment they registered, before the Streamlit app had finished initializing.
   With a 30-second check interval and a 2-healthy-threshold requirement, a task needed at
   least 60 consecutive seconds of healthy responses — but with a 0-second grace period, ECS
   killed the task before it could ever accumulate two successes.

2. The ECS deployment circuit breaker was disabled (also the default). With no circuit
   breaker, ECS never declared the deployment permanently failed — it just kept launching new
   task revisions that got killed in the same way.

Fixed by setting `healthCheckGracePeriodSeconds=90`. In production, also enable the
deployment circuit breaker with rollback, or switch to CodeDeploy blue/green, so a bad
deploy rolls back automatically after a defined number of failures.

---

**Q: Why use `public.ecr.aws/docker/library/python:3.12-slim` instead of `python:3.12-slim`?**

Docker Hub enforces anonymous pull rate limits by IP. CodeBuild's managed build environments
share a pool of IP addresses; those IPs collectively exhaust Docker Hub's rate limit quickly
during busy periods (or just because many teams use the same IPs). The result is a `429 Too
Many Requests` error that fails the build with no code change on our part.

`public.ecr.aws/docker/library/python:3.12-slim` is AWS's ECR Public mirror of the official
Docker Hub library images. For builds running inside AWS infrastructure (CodeBuild,
EC2, ECS), pulls from ECR Public do not count against Docker Hub's anonymous rate limit.
The image content is identical; only the registry differs.

---

**Q: Why is the task's port 8501 not reachable directly from the internet, even though the task has a public IP?**

Intentional. The task's security group (created by `m5_labA_deploy.sh`) originally opened
port 8501 from `0.0.0.0/0`. During Lab B, that broad inbound rule was revoked and replaced
with a rule allowing port 8501 only from the ALB security group.

This means:
- All public traffic enters via `http://<ALB_DNS_NAME>:80` (eventually `https://...443`)
- The ALB security group is the only source allowed to reach port 8501 on tasks
- Direct connections to `http://<task-public-ip>:8501` are silently dropped by the SG

This is defense-in-depth: even if the task has a public IP (which it has in the default
VPC / public subnet setup), it is not directly reachable. In a production VPC the task
would be in a private subnet with no public IP at all, making this a non-issue.

---

**Q: What's the difference between this repo and Module 4's repo?**

Same application code and the same Dockerfile pattern. The structural difference is where
the `Dockerfile`, `app.py`, `requirements.txt`, and `artifacts/` live:

- Module 4: nested under a subdirectory (e.g., `labs/module4/app/`)
- Module 5 (this repo): at the **repo root**

This matters because `buildspec.yml` sets `BUILD_CONTEXT=.` — CodeBuild runs `docker build .`
with the repo root as the build context. If the `Dockerfile` were nested, CodeBuild would
not find it without an explicit `DOCKERFILE_PATH` override. This repo is structured so the
repo root IS the deployable unit, which is the simplest pattern for a single-service
deployment.

---

**Q: How do I bring this deployment back up after the teardown in this session?**

The ECR images were preserved (`v1`, `latest`, `<commit-sha>`), so you can skip the image
rebuild if the code hasn't changed.

1. **(Optional) Rebuild the image** — only if code has changed since the last push:
   ```bash
   docker build -t truck-delay-app:v1 .
   docker tag truck-delay-app:v1 <ECR_REPO_URI>/truck-delay-app:v1
   docker push <ECR_REPO_URI>/truck-delay-app:v1
   ```

2. **Re-run Lab A** (`m5_labA_deploy.sh`) — creates the ECS cluster, task definition,
   and service. The `ecsTaskExecutionRole` was preserved, so the script should reuse it.

3. **Re-run Lab B** — create the target group, ALB SG, ALB, listener, update the ECS
   service to register with the target group, then lock down the task SG.

4. **Re-run Lab D** — create the CodeBuild project and CodePipeline. Note:
   - Issue #6 (orphaned IAM policies) may recur if a previous partially-created project
     left stale policies; check and delete them first.
   - Issue #7 (managed-policy quota) is permanently fixed — the `mlops-codepipeline-policy`
     inline policy was preserved.
   - The GitHub connection `truck-delay-github-connection` was preserved; check its status
     is still `AVAILABLE` before wiring it into CodeBuild.
   - Remember to set `healthCheckGracePeriodSeconds=90` on the ECS service (step 33 in the
     walkthrough) before the first pipeline-triggered deploy.

---

**Q: Why did the YAML `buildspec.yml` errors only surface on pipeline-triggered builds, not on the standalone CodeBuild test?**

They surfaced on both — but the standalone test (step 31) was run *after* the `buildspec.yml`
had already been fixed. The YAML errors (Issue #9) were discovered during the first
pipeline-triggered run (step 34 equivalent). The fix was committed and pushed, which itself
triggered another pipeline run, which was the one that confirmed the YAML was correct. So
the standalone test in the walkthrough implicitly assumes a fixed `buildspec.yml`.

If you ever need to test `buildspec.yml` YAML validity without triggering a full build:

```bash
# Install the CodeBuild local agent (Docker required):
# https://docs.aws.amazon.com/codebuild/latest/userguide/use-codebuild-agent.html

# Or validate YAML syntax alone (catches structural issues but not CodeBuild semantics):
python3 -c "import yaml,sys; yaml.safe_load(open('buildspec.yml'))" && echo "YAML OK"
```
