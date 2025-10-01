# S3 Bootstrap Source Migration (2025-10)

This note captures **all** of the changes and runbooks required to operate the bootstrap pipeline when its source is hosted in S3 instead of CodeCommit. Keep it around so “future-me” can skim it and re-run the exact same flow without thinking.

---

## 1. Summary of Code Changes

| File | Purpose |
|------|---------|
| `src/template.yml` | Adds `BootstrapSourceProvider` / `BootstrapSourceObjectKey` parameters, toggles CodePipeline stages based on provider, extends IAM + bucket policies, and teaches the `InitialCommit` custom resource to bundle the repository into S3 (preserving executable bits). |
| `src/lambda_codebase/initial_commit/bootstrap_repository/adf-bootstrap/deployment/regional.yml` | Rewrites `DeploymentFrameworkRegionalPipelineBucketPolicy` to avoid hard-coded principals. Creation succeeds even when roles are created later in the template. |
| `src/lambda_codebase/initial_commit/bootstrap_repository/adf-bootstrap/global.yml` | (Currently untouched – include only if future edits happen.) |
| `docs/s3-bootstrap-migration.md` | This document. |

Everything else remains stock ADF.

---

## 2. Preconditions

- All commands run from the repository root (`/Users/monicacolangelo/Dropbox/WORK/repos/aws-deployment-framework`).
- Management account region for the stack is **us-east-1** (ADF requirement). `DeploymentAccountMainRegion`, etc. can be EU.
- The `BootstrapTemplatesBucket` is the one exported from the stack (currently something like `aws-deployment-framework-bootstraptemplatesbucket-…`).
- AWS CLI profile already set up (Administrator in the management account).

---

## 3. Updating the bootstrap repository contents

The bootstrap archive **must** contain runtime files (`adfconfig.yml`, generated examples, etc.), not only the sources under `src/…`. The safe way is to start from the archive created by the Lambda during the initial install, overlay the local changes, and rebuild.

### 3.1 Retrieve the last good archive (optional but recommended)

```bash
BUCKET=<BootstrapTemplatesBucket>
KEY=bootstrap-source/bootstrap_repository.zip

# Inspect existing versions (the one created by the Lambda is usually the oldest)
aws s3api list-object-versions \
  --bucket "$BUCKET" \
  --prefix "$KEY" \
  --region us-east-1

# Optionally download a previous version to keep as reference
aws s3api get-object \
  --bucket "$BUCKET" \
  --key "$KEY" \
  --version-id <VERSION_ID> \
  /tmp/bootstrap_repository_reference.zip \
  --region us-east-1
```

### 3.2 Build a fresh archive from the current workspace

```bash
cd /Users/monicacolangelo/Dropbox/WORK/repos/aws-deployment-framework

TMP=$(mktemp -d)
# Copy the entire bootstrap_repository folder (generated files included)
rsync -a --delete src/lambda_codebase/initial_commit/bootstrap_repository/ "$TMP/"

# Apply overrides (only the files you changed). Example:
rsync -a src/lambda_codebase/initial_commit/bootstrap_repository/adf-bootstrap/global.yml "$TMP/adf-bootstrap/"
rsync -a src/lambda_codebase/initial_commit/bootstrap_repository/adf-bootstrap/deployment/regional.yml "$TMP/adf-bootstrap/deployment/"

# Create the zip with the correct root layout
(cd "$TMP" && zip -rq "$OLDPWD/bootstrap_repository.zip" .)
rm -rf "$TMP"

# (Optional) sanity check
zipinfo bootstrap_repository.zip | head
```

If `zipinfo` shows `adf-build/…`, `adf-bootstrap/…`, `adfconfig.yml`, etc. at the top level you’re good.

### 3.3 Upload to S3

```bash
aws s3 cp bootstrap_repository.zip \
  s3://$BUCKET/$KEY \
  --region us-east-1
```

---

## 4. Redeploy the main stack when templates change

Any edit in `src/template.yml` or the custom resources requires a full `sam build`/`sam package`/`cloudformation deploy`. Use Docker (SAM needs it for the Lambda layers).

```bash
rm -rf .aws-sam

sam build \
  --template-file src/template.yml \
  --build-dir .aws-sam/build \
  --use-container \
  --region us-east-1

sam package \
  --template-file .aws-sam/build/template.yaml \
  --output-template-file packaged-template.yml \
  --s3-bucket tch-adf-artifacts-use1-<suffix> \
  --region us-east-1

aws cloudformation deploy \
  --stack-name aws-deployment-framework \
  --template-file packaged-template.yml \
  --s3-bucket tch-adf-artifacts-use1-<suffix> \
  --s3-prefix templates/adf \
  --region us-east-1 \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides \
      BootstrapSourceProvider=S3 \
      BootstrapSourceObjectKey=$KEY \
      DeploymentAccountName=ADFdeployment \
      DeploymentAccountEmailAddress=monica.colangelo+adfdeployment@gmail.com \
      DeploymentAccountAlias=tch-adf-deployment \
      DeploymentAccountId=327756948098 \
      DeploymentAccountMainRegion=eu-central-1 \
      DeploymentAccountTargetRegions=eu-central-1,eu-west-1 \
      ProtectedOUs=ou-1r5g-dli3za77 \
      MainNotificationEndpoint=monica.colangelo+adfnotifications@gmail.com \
      LogLevel=INFO \
      AllowBootstrappingOfManagementAccount=No
```

> Tip: keep `packaged-template.yml` out of git (add to `.gitignore`).

---

## 5. Run the bootstrap pipeline

1. Go to CodePipeline (`us-east-1`) → `aws-deployment-framework-bootstrap-pipeline`.
2. Click **Release change**. The stages should finish in ~10 minutes:
   - **BootstrapSource** reads the S3 object.
   - **EnableBootstrappingViaJumpRole** and **Restrict…** manipulate the jump role.
   - **UploadAndUpdateBaseStacks** kicks CodeBuild + Step Functions to propagate templates/SSM.
3. Monitor the Step Functions in the deployment account (operation should create/update stacks `adf-global-base`, `adf-regional-base-*`).

Validation checklist:
- Stack `aws-deployment-framework` → `CREATE_COMPLETE` / `UPDATE_COMPLETE`.
- Deployment account (327756948098) CloudFormation: `adf-global-base`, `adf-regional-base-eu-central-1`, `adf-regional-base-eu-west-1` → `CREATE_COMPLETE`.
- Step Functions: `adf-pipeline-management` exists and last execution `SUCCEEDED`.
- SSM parameters (/adf/…) updated with the latest timestamp.

---

## 6. Troubleshooting cheatsheet

| Symptom | Likely cause | Remedy |
|---------|--------------|--------|
| `BootstrapSource` stage → "Insufficient permissions" | Wrong zip layout (missing `adf-build` or `adfconfig.yml`). | Rebuild the archive using the rsync+zip instructions above, then re-upload. |
| `DeploymentFrameworkRegionalPipelineBucketPolicy` → `Invalid principal` | Bucket policy referenced roles before they existed. | The updated template already applies wildcard principal + conditions; redeploy the stack. |
| Step Function `adf-account-bootstrapping` → CloudFormation `ROLLBACK_COMPLETE` | S3 policy invalid or missing generated files in zip. | Fix the policy/zip, rerun the bootstrap pipeline. |
| `ERROR: Could not open requirements file adf-build/requirements.txt` | Archive missing required directories. | Same as above: rebuild from the full `bootstrap_repository`. |
| `sam build` → `Docker is unreachable` | Docker not running on the build machine. | Start Docker Desktop (or disable `--use-container`). |

Keep CloudTrail/CloudWatch logs handy: they pinpoint the exact failing action.

---

## 7. Automating the upload (TODO)

- Add a GitHub Action (or GitLab pipeline) that runs on push:
  1. `rsync` + `zip` the bootstrap repository.
  2. Upload to S3 using an IAM role (`aws-actions/configure-aws-credentials`).
  3. Optionally trigger the pipeline via `aws codepipeline start-pipeline-execution`.
- Store S3 bucket name and key in repository secrets so the workflow stays generic.

Until then, follow the manual process documented above whenever the bootstrap templates change.
