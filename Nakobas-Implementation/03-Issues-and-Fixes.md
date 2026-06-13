# Issues and Fixes

Each entry follows the format: **Symptom / Cause / Fix**

---

## Issue 1 — `cp` missing destination operand

**Symptom:** Shell immediately errors: `cp: missing destination file operand after '<src>'`

**Cause:** Command typed as `cp <src>` without a destination argument (missing trailing `.`
to copy into the current directory).

**Fix:** Add the destination: `cp <src> .`

---

## Issue 2 — `jq: command not found` / `curl -o /usr/bin/jq.exe` → Permission denied

**Symptom:**
```
bash: jq: command not found
curl: (23) Failed writing body: Permission denied  (when targeting /usr/bin/jq.exe)
```

**Cause:** `jq` was not installed. The Git Bash install directory (`/usr/bin`) is inside
the Git for Windows install path, which requires Administrator privileges to write.

**Fix:** Install to the user's own `~/bin` instead:
```bash
mkdir -p ~/bin
curl -Lo ~/bin/jq.exe \
  https://github.com/jqlang/jq/releases/latest/download/jq-windows-amd64.exe
chmod +x ~/bin/jq.exe
export PATH="$HOME/bin:$PATH"
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
```

---

## Issue 3 — `CreateTargetGroup` → `ValidationError: Health check path 'C:/Program Files/Git/_stcore/health' must begin with '/'`

**Symptom:**
```
An error occurred (ValidationError) when calling the CreateTargetGroup operation:
Health check path 'C:/Program Files/Git/_stcore/health' must begin with '/'.
```

**Cause:** Git Bash path-mangling. When Git Bash sees a string that looks like an absolute
Unix path (e.g., `/_stcore/health`) in a command that isn't a `git` sub-command, it
auto-converts it to a Windows absolute path (`C:/Program Files/Git/_stcore/health`).

**Fix:** Set `MSYS_NO_PATHCONV=1` before the session:
```bash
export MSYS_NO_PATHCONV=1
```
Must be `export` (not a one-off inline prefix like `MSYS_NO_PATHCONV=1 aws ...`) so that
the variable is visible inside the shell's environment, including path evaluation in
argument processing.

---

## Issue 4 — `AuthorizeSecurityGroupIngress` → `InvalidPermission.Duplicate` / `revoke` returns `UnknownIpPermissions`

**Symptom:** Re-running the task-SG lockdown commands a second time produces:
```
An error occurred (InvalidPermission.Duplicate) ...
An error occurred (InvalidPermission.NotFound) / UnknownIpPermissions ...
```

**Cause:** The lockdown was already applied successfully on the first run. Running the same
`authorize` and `revoke` commands again hits rules that already exist / no longer exist.

**Fix:** These errors are harmless — they confirm the desired state is already in place.
No action needed. If automating, add `|| true` or check the current rules with
`describe-security-groups` before modifying.

---

## Issue 5 — CodeBuild "Create build project" → `Invalid project source: source location must be a valid GitHub repository URL`

**Symptom:** Console error when saving the new CodeBuild project.

**Cause:** The GitHub connection and repository weren't fully wired up on the first attempt
(the repo picker wasn't loaded from the connection, possibly because the connection status
was still `PENDING` rather than `AVAILABLE`).

**Fix:**
1. Confirm the connection status is `AVAILABLE` before opening CodeBuild:
   ```bash
   aws codeconnections list-connections --region "$AWS_REGION"
   ```
2. In the CodeBuild console, pick **GitHub** as the source provider, then select the
   connection from the **Connection** dropdown — once connected, pick the repository from
   the **Repository** dropdown. Do not hand-type a URL unless the UI only offers a text field
   and the connection is confirmed `AVAILABLE`.

---

## Issue 6 — "Role / policy already exists" when creating CodeBuild project after a failed attempt

**Symptom:**
```
Role with name codebuild-truck-delay-build-service-role already exists.
A policy called CodeBuildBasePolicy-truck-delay-build-<AWS_REGION> already exists.
```

**Cause:** A prior failed "Create build project" attempt had partially created IAM resources
(the service role and two managed policies) before failing. AWS Console's rollback was
incomplete — the policies were not cleaned up.

**Fix:** Manually delete the orphaned policies:
```bash
aws iam delete-policy \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/service-role/CodeBuildBasePolicy-truck-delay-build-<AWS_REGION>

aws iam delete-policy \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/service-role/CodeBuildCodeConnectionsSourceCredentialsPolicy-truck-delay-build-<AWS_REGION>
```
Then retry creating the project, choosing **New service role** (the name is now free again).
The role itself (`codebuild-truck-delay-build-service-role`) had already been auto-cleaned
by AWS; only the two managed policies were stranded.

> **Note:** The IAM user needed `iam:DeletePolicy` on `arn:aws:iam::<AWS_ACCOUNT_ID>:policy/service-role/*`
> to run these commands — added as part of Issue #7.

---

## Issue 7 — `AccessDeniedException: not authorized to perform: codebuild:ListProjects` / managed-policy quota exceeded

**Symptom (a):**
```
An error occurred (AccessDeniedException) when calling the ListProjects operation:
User arn:aws:iam::<AWS_ACCOUNT_ID>:user/<IAM_USERNAME> is not authorized to perform:
codebuild:ListProjects
```

**Symptom (b):**
```
An error occurred (LimitExceeded) when calling the AttachUserPolicy operation:
Cannot exceed quota for PoliciesPerUser: 10
```

**Cause:** The IAM user had no CodeBuild, CodePipeline, or CodeStar permissions.
Attempting to fix this by attaching a managed policy (`AWSCodeBuildAdminAccess`) failed
because the user was already at the AWS hard limit of **10 attached managed policies** per
user. This is a hard quota, not a permissions error.

**Diagnosis steps run:**
```bash
aws iam list-attached-user-policies --user-name <IAM_USERNAME>  # 10 policies — at limit
aws iam list-groups-for-user --user-name <IAM_USERNAME>          # no groups
aws iam list-user-policies --user-name <IAM_USERNAME>            # 2 inline policies
```

**Fix:** Inline policies do *not* count toward the managed-policy quota. Added a new
inline policy `mlops-codepipeline-policy` via:
```bash
cat > codepipeline-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:*",
        "codepipeline:*",
        "codestar-connections:*",
        "codeconnections:*",
        "iam:CreateRole", "iam:AttachRolePolicy", "iam:PassRole",
        "iam:GetRole", "iam:ListRoles", "iam:DeleteRole",
        "iam:DetachRolePolicy", "iam:DeletePolicy",
        "iam:ListAttachedRolePolicies",
        "logs:CreateLogGroup", "logs:CreateLogDelivery",
        "logs:DescribeLogGroups", "logs:DeleteLogGroup"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-user-policy \
  --user-name <IAM_USERNAME> \
  --policy-name mlops-codepipeline-policy \
  --policy-document file://codepipeline-policy.json
```

> `codepipeline-policy.json` is in `.gitignore` — never committed.

---

## Issue 8 — `put-user-policy --policy-document file:///tmp/foo.json` → "No such file or directory"

**Symptom:**
```
Error loading file /tmp/foo.json: [Errno 2] No such file or directory
```
even though the file was just created.

**Cause:** Same Git Bash `/tmp` path issue seen in Module 3. Git Bash's `/tmp` maps to the
Git Bash runtime temp, but the AWS CLI (a native Windows process) resolves `file:///tmp/...`
using the Windows file system path, where no such file exists.

**Fix:** Use a relative path in the current directory:
```bash
aws iam put-user-policy \
  --policy-document file://codepipeline-policy.json \
  ...
```

---

## Issue 9 — Pipeline-triggered build → `YAML_FILE_ERROR: Expected Commands[N] to be of string type: found subkeys instead`

**Symptom:** CodeBuild build phase fails immediately on pipeline-triggered runs with errors like:
```
Phase context status code: YAML_FILE_ERROR Message: Expected Commands[5] to be of string
type: found subkeys instead
```

**Cause:** Two separate YAML special-character issues in `buildspec.yml` — command strings
that are valid shell but invalid as *unquoted YAML plain scalars*:

| Character sequence | Where it appeared | Why YAML breaks |
|---|---|---|
| `\|` (pipe) | `COMMIT_HASH=$(echo $VAR \| cut -c 1-7)` | YAML treats `\|` as a block-scalar indicator |
| `:=` | `IMAGE_TAG=${COMMIT_HASH:=latest}` | `:` starts a YAML mapping key |
| `: ` (colon-space) | `echo "    Repository: $REPOSITORY_URI"` | YAML sees `key: value` even inside a shell double-quoted string, because the *line itself* is an unquoted YAML plain scalar |

Two rounds of fixes were needed: the `|`/`:=` lines first, then the `: ` echo lines.

**Fix:** Wrap every affected command in **YAML single quotes**:
```yaml
# Before (broken):
- COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
- IMAGE_TAG=${COMMIT_HASH:=latest}
- echo "    Repository: $REPOSITORY_URI"

# After (working):
- 'COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)'
- 'IMAGE_TAG=${COMMIT_HASH:=latest}'
- 'echo "    Repository: $REPOSITORY_URI"'
```

The final working `buildspec.yml` is committed at the repo root.

---

## Issue 10 — CodeBuild PRE_BUILD → `AccessDeniedException: not authorized to perform: ecr:GetAuthorizationToken`

**Symptom:**
```
[Container] PRE_BUILD: Running command ...
Error: AccessDeniedException: ... not authorized to perform: ecr:GetAuthorizationToken
```

**Cause:** CodeBuild auto-creates a service role (`codebuild-truck-delay-build-service-role`)
with only basic CodeBuild permissions. It has no ECR access.

**Fix:**
```bash
aws iam attach-role-policy \
  --role-name codebuild-truck-delay-build-service-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess
```

---

## Issue 11 — CodeBuild BUILD phase → `429 Too Many Requests / toomanyrequests: You have reached your unauthenticated pull rate limit`

**Symptom:** `docker build` fails pulling the base image:
```
Step 1/... : FROM python:3.12-slim
error pulling image configuration: ... 429 Too Many Requests - Server message:
toomanyrequests: You have reached your pull rate limit.
```

**Cause:** CodeBuild uses shared IP addresses that pull from Docker Hub anonymously.
Docker Hub rate-limits anonymous pulls by IP, and CodeBuild's shared pool gets throttled.

**Fix:** Switch the `Dockerfile`'s base image to the AWS ECR Public mirror:
```dockerfile
# Before:
FROM python:3.12-slim

# After:
FROM public.ecr.aws/docker/library/python:3.12-slim
```
ECR Public has no anonymous pull rate limit for traffic originating from AWS.

This change was also the real code change used to prove the full CI/CD loop end-to-end.

---

## Issue 12 — Pipeline Deploy stage fails: `Task failed ELB health checks`, infinite retry loop

**Symptom:** After a pipeline-triggered deploy, the ECS service keeps launching new task
revisions, each killed almost immediately, and the deploy never completes:
```
(service truck-delay-service) (deployment ...) deployment failed: tasks failed ELB
health checks in (target-group ...)
```

**Cause:** `healthCheckGracePeriodSeconds` was `0` (the default). The ALB began health-
checking new tasks immediately on registration. The Streamlit app takes ~30–60 seconds to
fully start and serve `/_stcore/health`. With a 30-second check interval and a 2-healthy-
threshold requirement, at least 60 seconds of consecutive healthy responses are needed
before a task can be marked healthy — but with a 0-second grace period, ECS killed the
task before it could ever accumulate 2 successes.

The deployment circuit breaker was also disabled, so instead of rolling back, ECS kept
spinning up new task revisions indefinitely.

**Fix:**
```bash
aws ecs update-service \
  --cluster m5-truck-delay-cluster \
  --service truck-delay-service \
  --health-check-grace-period-seconds 90 \
  --force-new-deployment \
  --region "$AWS_REGION"
```
Setting 90 seconds gives the container ample time to initialize before the ALB starts
counting health check results. The next deployment completed cleanly:
`RolloutState: COMPLETED`, old task drained, new task `healthy`.

**Production note:** Also enable the deployment circuit breaker with rollback
(`deploymentConfiguration.deploymentCircuitBreaker.enable=true,rollback=true`) or switch
to CodeDeploy blue/green, so a failed deploy rolls back automatically rather than looping.
