# Event-Driven AWS Serverless Data Pipeline

An end-to-end event-driven ETL pipeline built on AWS using **Amazon S3**, **AWS Lambda**, **AWS Glue (Workflows, Crawlers, Jobs, Data Catalog)**, **IAM**, and **Amazon CloudWatch**. The pipeline automatically triggers upon uploading raw CSV datasets, executes transformations in Python, catalogs input/output schemas, and outputs clean transformed datasets back to Amazon S3.

The pipeline can be stood up two ways, covered together in one flow below:
1. **Manually**, click-by-click through the AWS Console.
2. **Via CI/CD**, using GitHub Actions + AWS CloudFormation (Infrastructure as Code) with an IAM OIDC deploy role — no long-lived keys required at the identity-federation layer.

Both paths provision the exact same resource names (`bootcamp-input`, `bootcamp-output`, `bootcamp-function`, `glue-pipeline`, etc.), so the verification screenshots apply either way.

---

## 📁 Project Structure

```text
AWS/
├── .github/
│   └── workflows/
│       ├── deploy.yml                  # CI/CD pipeline: packages Lambda, uploads assets, deploys/updates CloudFormation stack, seeds sample CSV
│       └── destroy.yml                 # CI/CD pipeline: empties S3 buckets and tears down the CloudFormation stack (manual, confirmation-gated)
├── infra/
│   ├── bootstrap.yaml                  # One-time stack: GitHub OIDC provider, GitHub Actions deploy role, assets bucket (deploy this by hand first)
│   ├── template.yaml                   # Main pipeline stack: IAM roles, S3 buckets, Lambda, Glue DB/Crawlers/Job/Workflow/Triggers
│   ├── import-template.yaml            # One-time import template used to adopt pre-existing console-created resources into the stack
│   └── import-resources.json           # Resource identifiers mapped during the one-time CloudFormation import
├── Screenshots/                        # AWS console steps, CloudWatch logs, and CI/CD run evidence
│   └── Billing Status/                 # AWS Budgets / SNS billing-alarm setup and confirmation screenshots
├── glue_job.py                         # AWS Glue Python script executing data transformations
├── ipl_players_extra.csv               # Raw sample dataset uploaded to S3 input bucket
├── lambda_function.py                  # S3 event-driven Lambda function triggering the Glue Workflow
├── architecture_diagram_lowlevel.png   # Low-level architecture diagram
├── 1. Bootstrap.png                    # CLI evidence: one-time bootstrap stack deployed
├── 2. Deploy ARN.png                   # CLI evidence: OIDC provider + deploy role ARN lookup
└── README.md                           # Comprehensive pipeline documentation
```

---

## 🏗️ Architecture Diagram

![Architecture Diagram](/AWS/architecture-with-cicd.png)

**Flow:** `S3 (input/)` → `s3:ObjectCreated:Put` event → `Lambda (bootcamp-function)` → `Glue Workflow (glue-pipeline)` → `Crawler (Glue_Input)` → `Glue Job (glue_job.py)` → `Crawler (Glue_output)` → `S3 (output/)`, with every stage emitting logs to **CloudWatch** and schema metadata to the **Glue Data Catalog**.

---

## 💰 Step 0 — Billing Guardrail (AWS Budgets via CLI)

Before provisioning any pipeline resources, a cost guardrail is put in place so spend never goes unnoticed. This is done first, ahead of both the manual and CI/CD paths.

1. **Define a monthly budget stack.** A small CloudFormation template declares an `AWS::Budgets::Budget` (monthly cost budget) plus an `AWS::SNS::Topic` + `AWS::SNS::Subscription` so budget alerts land in an email inbox.
2. **Allow Budgets to publish to SNS.** An `AWS::SNS::TopicPolicy` explicitly allows the `budgets.amazonaws.com` service principal to call `sns:Publish` on the alert topic — otherwise alert emails silently never arrive.
3. **Deploy it from the AWS CLI**, passing the monthly limit and alert email as parameter overrides:

   ```powershell
   aws cloudformation deploy `
     --stack-name account-billing-alarm `
     --template-file cloudformation/00-billing-alarm.yaml `
     --region ap-south-1 `
     --parameter-overrides MonthlyBudgetLimitUSD=5 AlertEmail=<your-email>
   ```

   ![AWS CLI Billing Deploy](Screenshots/Billing%20Status/aws%20cli.png)

   * **What it shows**: `aws cloudformation deploy` creating/updating the `account-billing-alarm` stack with a `$5` monthly limit, run from VS Code's integrated terminal.
   * **How it works**: CloudFormation resolves the changeset, waits for `stack create/update` to complete, and prints `Successfully created/updated stack - account-billing-alarm`.

4. **Confirm the SNS email subscription.** AWS sends a confirmation email immediately after the stack is created — the subscription stays in `PendingConfirmation` until it is confirmed.

   ![SNS Subscription Confirmed](Screenshots/Billing%20Status/aws%20confirmation.png)![Confirmation](Screenshots/Billing%20Status/Confirmation.png)

   * **What it shows**: The SNS "Subscription confirmed!" page after clicking the confirmation link, and the AWS Budgets console showing budget health.
   * **How it works**: Once confirmed, the subscription is `Active`; any future threshold breach (e.g. forecasted vs. actual spend) triggers an email via the SNS topic.

5. **Verify in the console.** Under **Billing and Cost Management → Budgets**, the `monthly-account-budget` shows `Health status: Healthy`, `Budget amount: $5.00`, `Period: Monthly`, confirming the guardrail is live before any billable pipeline resources are created.

> ⚠️ Do this step **once per AWS account**, before running either the manual setup or the first CI/CD deploy, so runaway resources trigger a cost alert instead of going unnoticed.

---

## 🚀 Step 1 — Provisioning the Pipeline (Manual + CI/CD, in one flow)

The two approaches are presented together per stage — pick either column, or read both to understand how the CI/CD pipeline automates what was previously done by hand.

### 1. Amazon S3 Bucket Setup & Input Data

**Manual (Console):**
* Create the `bootcamp-input` bucket with an `input/` prefix, and the `bootcamp-output` bucket.

**CI/CD (CloudFormation, `infra/template.yaml`):**
* `InputBucket` and `OutputBucket` resources are declared in the template; the input bucket carries an `s3:ObjectCreated:Put` `NotificationConfiguration` (filtered to `input/` prefix, `.csv` suffix) wired directly to the Lambda ARN.
* The `deploy.yml` workflow additionally seeds the `input/` prefix marker via `aws s3api put-object --bucket bootcamp-input --key input/` so the folder exists even before the first upload.

![S3-1](Screenshots/1.%20S3%20Input%20Bucket%20Created%20with%20subfolders.png)
![S3-data](Screenshots/1.1%20Souce%20CSV%20view.png)

* **What it shows**: The `bootcamp-input` S3 bucket configured with an `input/` directory containing raw files (`ipl_players_extra.csv`, `ipl_players.csv`).
* **How it works**: Serving as the data landing zone, uploading a file here automatically emits an `s3:ObjectCreated:Put` notification to AWS Lambda.

![S3-2](Screenshots/2.%20S3%20Output%20Bucket%20Created%20with%20subfolders.png)

* **What it shows**: The `bootcamp-output` bucket prepared with subfolder target locations like `ipl_players_extra/`.
* **How it works**: Holds processed CSV partitions created by the AWS Glue Python script upon ETL run completion.

---

### 2. IAM Roles & Security Management

**Manual (Console):** Roles were created and attached one at a time via IAM → Roles.

**CI/CD (CloudFormation):** All five roles are declared as `AWS::IAM::Role` resources in `infra/template.yaml`, each scoped to least-privilege inline policies:
* `Bootcamp-Lambda` — `AWSLambdaBasicExecutionRole` + inline `Start-Glue-Job` (`glue:StartWorkflowRun`, `glue:GetWorkflowRun`).
* `Bootcamp-Glue` — `AWSGlueServiceRole` + inline `Glue-access-S3` (read input bucket, write output bucket, read assets bucket).
* `Bootcamp-Crawler-Input` — read-only access to `bootcamp-input`.
* `Bootcamp-Crawler` — read-only access to `bootcamp-output`.
* `Bootcamp-Glue-Role` — kept for console parity (spare role, not attached to any resource).

Separately, `infra/bootstrap.yaml` provisions `GitHubActions-BootcampPipeline-Deploy`, an OIDC-federated role GitHub Actions assumes (no static AWS keys stored for this role) to run the deploy/destroy workflows.

![IAM](Screenshots/17.%20IAM%20Roles.png)

* **What it shows**: List of IAM roles provisioned for the architecture (`Bootcamp-Lambda`, `Bootcamp-Glue`, `Bootcamp-Glue-Role`, `Bootcamp-Crawler`, `Bootcamp-Crawler-Input`).
* **How it works**: Assumed by Lambda, Glue Jobs, and Crawlers to authenticate actions against S3 and CloudWatch without hardcoding credentials.

![Glue_Role](Screenshots/4.%20Glue.png)

* **What it shows**: Permissions attached to the `Bootcamp-Glue` role including managed policy `AWSGlueServiceRole` and custom inline policy `Glue-access-S3`.
* **How it works**: Grants AWS Glue execution access to read from `bootcamp-input`, write to `bootcamp-output`, and record metadata in the Glue Data Catalog.

![Lambda_Role](Screenshots/4.%20Lamda.png)

* **What it shows**: Permissions attached to the `Bootcamp-Lambda` role, including `AWSLambdaBasicExecutionRole`, a customer-managed `ETL` policy, and inline policies `Glue_only` / `Start-Glue-Job`.
* **How it works**: Grants Lambda basic execution permissions to write logs to CloudWatch, plus privileges to trigger and start the AWS Glue workflow.

---

### 3. Bootstrap the CI/CD Foundation (one-time, CLI)

This step has no manual-console equivalent — it exists solely to enable the CI/CD path, and only needs to run once per AWS account/repo.

1. Deploy `infra/bootstrap.yaml` once, by hand, from the CLI — **before** the first `Deploy` workflow run:

   ```powershell
   aws cloudformation deploy `
     --stack-name bootcamp-bootstrap `
     --template-file infra/bootstrap.yaml `
     --capabilities CAPABILITY_NAMED_IAM `
     --region ap-south-1 `
     --parameter-overrides `
       GitHubOrg=<your-github-org> `
       GitHubRepo=AWS `
       AssetsBucketName=bootcamp-pipeline-assets-<account-id>
   ```

   ![Bootstrap](1.%20Bootstrap.png)

   * **What it shows**: The bootstrap stack (`bootcamp-bootstrap`) being deployed, creating the GitHub OIDC provider, the `GitHubActions-BootcampPipeline-Deploy` IAM role, and the assets S3 bucket.
   * **How it works**: This stack is intentionally isolated from `infra/template.yaml` so the pipeline stack's own deploy/destroy cycle never touches the credentials that manage it.

2. Retrieve the deploy role ARN and confirm the OIDC provider registered correctly:

   ```powershell
   aws iam list-open-id-connect-providers
   aws cloudformation describe-stacks --stack-name bootcamp-bootstrap --region ap-south-1 --query "Stacks[0].Outputs" --output table
   ```

   ![Deploy ARN](2.%20Deploy%20ARN.png)

   * **What it shows**: The registered OIDC provider (`token.actions.githubusercontent.com`) and the stack outputs — `DeployRoleArn` and `AssetsBucketNameOut`.
   * **How it works**: GitHub Actions later assumes `DeployRoleArn` via `sts:AssumeRoleWithWebIdentity`, scoped to this repo (`repo:<org>/<repo>:*`), to run every subsequent deploy/destroy.

3. Add the outputs as GitHub repo **Variables**/**Secrets** (`AWS_REGION`, `ASSETS_BUCKET`, `CFN_STACK_NAME`, `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` for the IAM-user path used inside `deploy.yml`).

---

### 4. AWS Lambda Trigger Automation

**Manual (Console):** Function created via Lambda console, code pasted into the inline editor, S3 trigger attached manually.

**CI/CD (`deploy.yml` + `infra/template.yaml`):**
1. `zip lambda_function.zip lambda_function.py` — packages the function.
2. `aws s3 cp` uploads the zip (and `glue_job.py`) to the bootstrap-created assets bucket.
3. `AWS::Lambda::Function` (`BootcampFunction`) in `infra/template.yaml` pulls its code from that S3 key, with `GLUE_WORKFLOW_NAME` injected as an environment variable.
4. `AWS::Lambda::Permission` + the bucket's `NotificationConfiguration` wire the S3 → Lambda trigger declaratively — no manual click-through needed.

![Lambda](Screenshots/3.%20Lamda%20before.png)

* **What it shows**: The `bootcamp-function` AWS Lambda function page connected to an S3 bucket trigger, with embedded code editor (`lambda_function.py`).
* **How it works**: When S3 notifies Lambda of an upload, `lambda_handler` extracts `object_key`, calculates `input_path` and `output_path`, and passes these parameters when starting `glue-pipeline`.

![Log1](Screenshots/5.%20Cloudwatch%20Lambda%20overview.png)

* **What it shows**: The CloudWatch Log Group `/aws/lambda/bootcamp-function` showing generated log streams per event invocation.
* **How it works**: Tracks every execution instance for monitoring, debugging, and audit trails.

![Log2](Screenshots/6.%20Cloudwatch%20Lambda%20logs.png)

* **What it shows**: Detailed CloudWatch log event entries capturing extracted S3 event metadata (`input path: s3://bootcamp-input/input/ipl_players_extra.csv`) and confirming `"Glue workflow started successfully"` with `Workflow RunId`.
* **How it works**: Proves end-to-end trigger integration between S3 event notifications and Glue workflow initialization — identical outcome whether the resources were created by hand or by CloudFormation.

---

### 5. AWS Glue Workflow, Crawlers & Data Catalog

**Manual (Console):** Crawlers, job, workflow, and triggers created and wired one at a time in the Glue console.

**CI/CD (`infra/template.yaml`):** `AWS::Glue::Database`, `AWS::Glue::Crawler` (×2), `AWS::Glue::Job`, `AWS::Glue::Workflow`, and three `AWS::Glue::Trigger` resources declare the entire DAG as code:
* `glue-start` (`ON_DEMAND`) → runs `Glue_Input` crawler.
* `inter1` (`CONDITIONAL`, fires on `Glue_Input` = `SUCCEEDED`) → runs `glue_job`.
* `Inter2` (`CONDITIONAL`, fires on `glue_job` = `SUCCEEDED`) → runs `Glue_output` crawler.

![Pipe_graph](Screenshots/7.%20Glue%20Pipeline%20graph.png)

* **What it shows**: The visual pipeline graph inside `glue-pipeline` showing DAG execution nodes: `glue-start` → `Glue_Input` → `inter1` → `glue_job` → `Inter2` → `Glue_output`.
* **How it works**: Conditional triggers automate pipeline state progression exactly as declared in `infra/template.yaml`.

![Crawlers](Screenshots/8.%20Glue%20Crawler.png)

* **What it shows**: Status page of crawlers `Glue_Input` and `Glue_output` in `Ready` state with `Succeeded` last run state.
* **How it works**: Crawlers inspect S3 data sources to infer schemas/partitions/formats and sync table definitions into the Glue Data Catalog.

![inp](Screenshots/9.%20Glue_Input_Logs.png)
![out](Screenshots/10.%20Glue_Output_Logs.png)

* **What it shows**: Benchmark logs for crawlers showing `"Classification complete, writing results to database glue_db"` and `"Finished writing to Catalog"`.
* **How it works**: Verifies table schemas were parsed from CSVs and written into metadata tables without structural errors.

![Catalog](Screenshots/11.%20Glue_Catalog_DB.png)

* **What it shows**: The `glue_db` database entry under AWS Glue Data Catalog.
* **How it works**: Centralizes metadata table definitions created by crawlers, enabling queryability via Athena or Glue ETL scripts.

![Work_run](Screenshots/12.%20Glue_workflow_Run.png)

* **What it shows**: Historical list of workflow executions showing status `Completed` with durations (~1 min 55s to 2 min 24s).
* **How it works**: Validates pipeline stability across multiple test uploads and manual/CI-triggered executions.

![run_details](Screenshots/13.%20Glue_workflow_run_details.png)

* **What it shows**: Live DAG execution state highlighting completed nodes in green, with dynamic runtime parameters:
  * `INPUT_PATH`: `s3://bootcamp-input/input/ipl_players_extra.csv`
  * `OUTPUT_PATH`: `s3://bootcamp-output/ipl_players_extra/`
* **How it works**: Confirms runtime parameters passed from Lambda flow down into workflow job properties.

---

### 6. ETL Data Transformation & Output Validation

The Python ETL script (`glue_job.py`) processes raw data using standard Pandas transformations — identical logic regardless of how the surrounding infrastructure was provisioned.

![Transformation](Screenshots/14.1%20Glue_Job%20Transformation.png)

* **What it shows**: The source snippet of `glue_job.py` showing data cleaning and column transformation logic:
  ```python
  df["COUNTRY"] = df["COUNTRY"].astype("string").str.upper()
  df["adjusted_salary"] = (df["SALARY_CR"] * 0.90).map(round_half_up)
  ```
* **How it works**:
  1. Reads the incoming CSV dataset specified by `INPUT_PATH`.
  2. Standardizes country names into uppercase format (e.g. `India` → `INDIA`).
  3. Calculates a 10%-reduced `adjusted_salary` from the original player salary in Crores (`SALARY_CR`).
  4. Writes partitioned output back to the S3 target bucket.

![Log3](Screenshots/14.%20Glue_Job_logs.png)

* **What it shows**: CloudWatch execution log for `glue_job.py` printing `"Read 1 input file(s)... Wrote 124 rows to s3://bootcamp-output/ipl_players_extra/part-...csv"`.
* **How it works**: Validates file I/O operations and confirms 124 rows were written into the target location.

![Final_output](Screenshots/15.%20Bootcamp-Output%20Bucket(Final).png)

* **What it shows**: The `bootcamp-output/ipl_players_extra/` bucket containing the generated output CSV file.
* **How it works**: Stores final transformed datasets ready for downstream analytics or consumption.

![Final](Screenshots/16.%20Final%20CSV%20view.png)

* **What it shows**: VS Code preview of transformed CSV content showing updated schema: `PLAYER_ID, SALARY_CR, DEBUT_YEAR, COUNTRY, adjusted_salary`.
* **How it works**: Displays transformed data rows (e.g., `COUNTRY` converted to uppercase like `NEW ZEALAND`, `AUSTRALIA`, `AFGHANISTAN`) with `adjusted_salary` computed accurately.

---

### 7. CI/CD Pipeline Execution (GitHub Actions)

This stage has no manual equivalent — it is what replaces steps 1–6 above with a single, repeatable, auditable pipeline run.

1. **Trigger.** `deploy.yml` runs on a push to `main`/`ci/cd-CF` touching `infra/**`, `lambda_function.py`, `glue_job.py`, or the workflow file itself — or manually via `workflow_dispatch`.
2. **Authenticate.** `aws-actions/configure-aws-credentials@v4` configures AWS credentials for the run (IAM user secrets in this repo's current setup).
3. **Package & upload.** Lambda is zipped and, together with `glue_job.py`, uploaded to the assets bucket created in bootstrap.
4. **Import mode (one-time, optional).** If `workflow_dispatch` is run with `import_mode: true`, a CloudFormation **IMPORT** changeset (`infra/import-template.yaml` + `infra/import-resources.json`) adopts pre-existing console-created resources into the stack instead of recreating them.
5. **Create/update the stack.** `aws cloudformation deploy` applies `infra/template.yaml` idempotently — first run creates everything, subsequent runs update only what changed.
6. **Seed & verify.** The `input/` prefix marker is seeded, the run waits 45s for the S3 notification config to propagate, then `ipl_players_extra.csv` is uploaded automatically to trigger the live pipeline end-to-end.

![Deploy Runs](Screenshots/Deploy%20Part%201.png)

* **What it shows**: GitHub Actions **Deploy** workflow run history — a mix of manual (`workflow_dispatch`) and push-triggered runs against `main` and a feature branch.
* **How it works**: Every infra/code change is deployed the same way, giving a full audit trail of what changed and when.

![Deploy Steps](Screenshots/Deploy%20Part%202.png)

* **What it shows**: A successful `Deploy #22` run — `Configure AWS credentials` → `Package Lambda` → `Upload assets` → `Create base resources` → `Seed input/ prefix marker`, completed in 2m 37s.
* **How it works**: Each named step maps 1:1 to the manual actions described in sections 1–5 above, now fully automated.

![Deploy Trigger](Screenshots/Deploy%20Part%203.png)

* **What it shows**: The final steps — `sleep 45` to let the S3 event notification config propagate, then `aws s3 cp ipl_players_extra.csv s3://bootcamp-input/input/...` to auto-trigger the pipeline.
* **How it works**: Confirms the CI/CD run doesn't just provision infrastructure — it also proves the pipeline works, in the same run.

![Stack Resources](Screenshots/Deploy%20Part%204.png)

* **What it shows**: The `bootcamp-pipeline` CloudFormation stack's **Resources** tab — 17 resources in `UPDATE_COMPLETE`/`CREATE_COMPLETE` state, matching every IAM role, S3 bucket, Lambda function, and Glue asset described above.
* **How it works**: Confirms CloudFormation, not manual console clicks, now owns and tracks the full resource graph.

![Deploy Summary](Screenshots/Deploy%20Part%205.png)

* **What it shows**: Full log tail of a completed deploy run, including `Show stack outputs` and the final `Trigger pipeline` step.
* **How it works**: `GITHUB_STEP_SUMMARY` captures the stack outputs (bucket names, Lambda ARN, workflow name) directly in the Actions run summary for quick reference.

---

### 8. Tearing It Down (CI/CD Destroy)

1. `destroy.yml` is a manual-only (`workflow_dispatch`) workflow that requires typing the exact CloudFormation stack name as a confirmation input — a deliberate guardrail against accidental deletion.
2. It empties both S3 buckets (`aws s3 rm --recursive`) since CloudFormation cannot delete non-empty buckets.
3. It deletes the `bootcamp-pipeline` stack and waits for `stack-delete-complete`.

![Destroy Trigger](Screenshots/Destroy%20Part%201.png)

* **What it shows**: The **Destroy** workflow's manual trigger form, requiring the operator to type `bootcamp-pipeline` to confirm deletion.
* **How it works**: `if [ "${{ inputs.confirm_stack_name }}" != "${{ vars.CFN_STACK_NAME }}" ]; then exit 1; fi` — a hard stop if the typed name doesn't match.

![Destroy Steps](Screenshots/Destroy%20Part%202.png) ![Destroy Steps 2](Screenshots/Destroy%20Part%203.png)

* **What it shows**: The buckets being emptied and the stack deletion in progress.
* **How it works**: Guarantees a clean, repeatable teardown with no orphaned S3 objects blocking the CloudFormation delete.

![Stack Deleted](Screenshots/Destroy%20Part%204.png)

* **What it shows**: The `bootcamp-pipeline` stack's Resources tab post-destroy — all 17 resources in `DELETE_COMPLETE`, with only `bootcamp-bootstrap` and `account-billing-alarm` remaining active.
* **How it works**: Confirms the pipeline stack is fully torn down while the CI/CD foundation (bootstrap) and the billing guardrail stay intact for the next deploy.

---

## ⚡ How to Run

### Option A — Manual (Console)
1. **Set up billing guardrail**: Deploy the AWS Budgets/SNS alarm stack via CLI (Step 0) so spend is monitored from the start.
2. **Deploy S3 Buckets**: Create `bootcamp-input` (with `input/` prefix) and `bootcamp-output` buckets.
3. **Set up IAM Roles**: Configure `Bootcamp-Lambda`, `Bootcamp-Glue`, `Bootcamp-Crawler-Input`, `Bootcamp-Crawler` roles with the policies described in Step 1.2.
4. **Deploy Lambda Function**: Create function `bootcamp-function`, deploy `lambda_function.py`, and add an S3 event notification on `bootcamp-input`.
5. **Configure Glue Workflow**: Build `glue-pipeline` containing crawlers (`Glue_Input`, `Glue_output`) and job (`glue_job`), wired with `glue-start`, `inter1`, `Inter2` triggers.
6. **Trigger Pipeline**: Upload `ipl_players_extra.csv` into `s3://bootcamp-input/input/`.
7. **Verify Output**: Monitor progress in the Glue Workflow console or CloudWatch logs, and inspect transformed output in `s3://bootcamp-output/ipl_players_extra/`.

### Option B — CI/CD (GitHub Actions + CloudFormation)
1. **Set up billing guardrail**: Deploy the AWS Budgets/SNS alarm stack via CLI (Step 0), same as the manual path — do this once per account.
2. **Bootstrap (one-time)**: Deploy `infra/bootstrap.yaml` via CLI to create the GitHub OIDC provider, deploy role, and assets bucket.
3. **Configure repo secrets/variables**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `ASSETS_BUCKET`, `CFN_STACK_NAME`, `INPUT_BUCKET`, `OUTPUT_BUCKET`.
4. **Run `Deploy`**: Push to `main`/`ci/cd-CF` (with changes under `infra/**`, `lambda_function.py`, or `glue_job.py`) or trigger manually — this packages Lambda, deploys/updates `infra/template.yaml`, seeds the `input/` prefix, and auto-uploads a sample CSV to prove the pipeline end-to-end.
5. **Verify Output**: Check the workflow's `GITHUB_STEP_SUMMARY` for stack outputs, then inspect `s3://bootcamp-output/ipl_players_extra/` and the Glue workflow run history.
6. **Run `Destroy`** (when needed): Trigger manually, typing the exact stack name to confirm — this empties both buckets and deletes the `bootcamp-pipeline` stack.
