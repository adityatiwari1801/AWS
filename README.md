# Event-Driven AWS Serverless Data Pipeline

An end-to-end event-driven ETL pipeline built on AWS using **Amazon S3**, **AWS Lambda**, **AWS Glue (Workflows, Crawlers, Jobs, Data Catalog)**, **IAM**, and **Amazon CloudWatch**. The pipeline automatically triggers upon uploading raw CSV datasets, executes transformations in Python, cataloging input/output schemas, and outputs clean transformed datasets back to Amazon S3.

---

## 📁 Project Structure

```text
AWS/
├── Screenshots/                        # Directory containing all AWS console step and log output images
├── glue_job.py                         # AWS Glue Python script executing data transformations
├── ipl_players_extra.csv               # Raw sample dataset uploaded to S3 input bucket
├── lambda_function.py                  # S3 event-driven Lambda function triggering Glue Workflow
└── README.md                           # Comprehensive pipeline documentation
```

---

## 🏗️ Architecture Diagram

![Architecture Diagram](/architecture_diagram_lowlevel.png)

---

## 🛠️ Step-by-Step Implementation & Image Documentation

### 1. Amazon S3 Bucket Setup & Input Data
Amazon S3 acts as both the raw ingestion layer (landing bucket) and target storage layer (output bucket).

* **Input Storage (`bootcamp-input/input/`)**
  * **Image**: 

![S3-1](Screenshots/1.%20S3%20Input%20Bucket%20Created%20with%20subfolders.png)

![S3-data](Screenshots/1.1%20Souce%20CSV%20view.png)

  * **What it shows**: The `bootcamp-input` S3 bucket configured with an `input/` directory containing raw files (`ipl_players_extra.csv` and `ipl_players.csv`).
  * **How it works**: Serving as the data landing zone, uploading a file here automatically emits an `s3:ObjectCreated:Put` notification to AWS Lambda.


* **Output Storage (`bootcamp-output/`)**
  * **Image**: 

![S3-2](Screenshots/2.%20S3%20Output%20Bucket%20Created%20with%20subfolders.png)

  * **What it shows**: The `bootcamp-output` bucket prepared with subfolder target locations like `ipl_players_extra/`.
  * **How it works**: Holds processed CSV partitions created by the AWS Glue Python script upon ETL run completion.

---

### 2. IAM Roles & Security Management
Secure service-to-service communication is enforced via strict AWS IAM roles following the principle of least privilege.

* **IAM Service Roles Overview**
  * **Image**: 

  ![IAM](Screenshots/17.%20IAM%20Roles.png)

  * **What it shows**: List of IAM roles provisioned for the architecture (`Bootcamp-Lambda`, `Bootcamp-Glue`, `Bootcamp-Glue-Role`, `Bootcamp-Crawler`, `Bootcamp-Crawler-Input`).
  * **How it works**: Assumed by Lambda, Glue Jobs, and Crawlers to authenticate actions against S3 and CloudWatch without hardcoding credentials.

* **Glue Execution Role Configuration**
  * **Image**: *(IAM Role `Bootcamp-Glue` Details)*

  ![Glue_Role](Screenshots/4.%20Glue.png)

  * **What it shows**: Permissions attached to the `Bootcamp-Glue` role including managed policy `AWSGlueServiceRole` and custom inline policy `Glue-access-S3`.
  * **How it works**: Grants AWS Glue execution access to read from `bootcamp-input`, write to `bootcamp-output`, and record metadata in the Glue Data Catalog.

* **Lambda Execution Role Configuration**
  * **Image**: *(IAM Role `Bootcamp-Lambda` Details)*

    ![Lambda_Role](Screenshots/4.%20Lamda.png)

  * **What it shows**: Permissions attached to the `Bootcamp-Lambda` role, including the AWS managed policy `AWSLambdaBasicExecutionRole`, a customer managed policy `ETL`, and inline policies `Glue_only` and `Start-Glue-Job`.
  * **How it works**: Grants the Lambda function basic execution permissions to write logs to CloudWatch and the necessary privileges to trigger and start the AWS Glue workflow.

---


### 3. AWS Lambda Trigger Automation
AWS Lambda provides serverless orchestration between S3 ingestion and Glue Workflow processing.

* **Lambda Function Definition**
  * **Image**: 

![Lambda](Screenshots/3.%20Lamda%20before.png)

  * **What it shows**: The `bootcamp-function` AWS Lambda function page connected to an S3 bucket trigger, with embedded code editor (`lambda_function.py`).
  * **How it works**: When S3 notifies Lambda of an upload, `lambda_handler` extracts `object_key`, calculates `input_path` and `output_path`, and passes these parameters when starting `glue-pipeline`.

* **CloudWatch Lambda Log Stream**
  * **Image**: 

![Log1](Screenshots/5.%20Cloudwatch%20Lambda%20overview.png)

  * **What it shows**: The CloudWatch Log Group `/aws/lambda/bootcamp-function` showing generated log streams per event invocation.
  * **How it works**: Tracks every execution instance for monitoring, debugging, and audit trails.

* **Lambda Event Processing Verification**
  * **Image**: 

![Log2](Screenshots/6.%20Cloudwatch%20Lambda%20logs.png)

  * **What it shows**: Detailed CloudWatch log event entries capturing extracted S3 event metadata (`input path: s3://bootcamp-input/input/ipl_players_extra.csv`) and confirming `"Glue workflow started successfully"` with `Workflow RunId`.
  * **How it works**: Proves end-to-end trigger integration between S3 event notifications and Glue workflow initialization.

---

### 4. AWS Glue Workflow, Crawlers & Data Catalog
AWS Glue orchestrates schema discovery, data modeling, and python execution.

* **Glue Workflow Graph Architecture**
  * **Image**: 

![Pipe_graph](Screenshots/7.%20Glue%20Pipeline%20graph.png)

  * **What it shows**: The visual pipeline graph inside `glue-pipeline` showing DAG execution nodes: `glue-start` -> `Glue_Input` -> `inter1` -> `glue_job` -> `Inter2` -> `Glue_output`.
  * **How it works**: Defines event conditional triggers (`inter1` fires after `Glue_Input` completes, `Inter2` fires after `glue_job` completes) to automate pipeline state progression.

* **Glue Crawlers Registration**
  * **Image**: 

![Crawlers](Screenshots/8.%20Glue%20Crawler.png)

  * **What it shows**: Status page of crawlers `Glue_Input` and `Glue_output` in `Ready` state with `Succeeded` last run state.
  * **How it works**: Crawlers inspect S3 data sources to automatically infer schemas, partition structures, and formats, synchronizing table definitions into the Glue Data Catalog.

* **Crawler CloudWatch Logs**
  * **Images**:

![inp](Screenshots/9.%20Glue_Input_Logs.png)

![out](Screenshots/10.%20Glue_Output_Logs.png)

  * **What it shows**: Detailed benchmark logs for crawlers showing schema discovery actions like `"Classification complete, writing results to database glue_db"` and `"Finished writing to Catalog"`.
  * **How it works**: Verifies that table schemas were parsed from CSVs and written into metadata tables without structural errors.

* **Glue Data Catalog Database**
  * **Image**: 

![Catalog](Screenshots/11.%20Glue_Catalog_DB.png)

  * **What it shows**: The `glue_db` database entry under AWS Glue Data Catalog.
  * **How it works**: Centralizes metadata table definitions created by crawlers, enabling queryability via AWS Athena or Glue ETL scripts.

* **Workflow Run Execution History**
  * **Image**: 

![Work_run](Screenshots/12.%20Glue_workflow_Run.png)

  * **What it shows**: Historical list of 6 workflow executions showing status `Completed` with durations (~1 min 55s to 2 min 24s).
  * **How it works**: Validates pipeline stability across multiple test uploads and manual executions.

* **Detailed Workflow Visual Execution Details**
  * **Image**: 

![run_details](Screenshots/13.%20Glue_workflow_run_details.png)

  * **What it shows**: Live DAG execution state highlighting completed nodes in green, along with dynamic runtime parameters:
    * `INPUT_PATH`: `s3://bootcamp-input/input/ipl_players_extra.csv`
    * `OUTPUT_PATH`: `s3://bootcamp-output/ipl_players_extra/`
  * **How it works**: Confirms runtime parameters passed from Lambda were passed down into workflow job properties.

---

### 5. ETL Data Transformation & Output Validation
The Python ETL script processes raw data using standard Pandas transformations.

* **Glue Script Logic & Transformation**
  * **Image**: 

![Transformation](Screenshots/14.1%20Glue_Job%20Transformation.png)

  * **What it shows**: The source snippet of `glue_job.py` showing data cleaning and column transformation logic:
    ```python
    df["COUNTRY"] = df["COUNTRY"].astype("string").str.upper()
    df["adjusted_salary"] = (df["SALARY_CR"] * 0.90).map(round_half_up)
    ```
  * **How it works**:
    1. Reads incoming CSV dataset specified by `INPUT_PATH`.
    2. Standardizes country names into uppercase format (e.g. `India` -> `INDIA`).
    3. Calculates a 10% reduced `adjusted_salary` based on original player salary in Crores (`SALARY_CR`).
    4. Writes partitioned output back to S3 target bucket.

* **Glue Job Execution Logs**
  * **Image**: 

![Log3](Screenshots/14.%20Glue_Job_logs.png)

  * **What it shows**: CloudWatch execution log for `glue_job.py` printing `"Read 1 input file(s)... Wrote 124 rows to s3://bootcamp-output/ipl_players_extra/part-...csv"`.
  * **How it works**: Validates file I/O operations and confirms 124 rows were written into target location.

* **Final Output S3 Artifact**
  * **Image**: 

![Final_output](Screenshots/15.%20Bootcamp-Output%20Bucket(Final).png)

  * **What it shows**: The `bootcamp-output/ipl_players_extra/` bucket containing generated output CSV file (`part-9ba91a1ecdeb415c9574bf327673cf52.csv`).
  * **How it works**: Stores final transformed datasets ready for downstream analytics or consumption.

* **Final Data Inspection**
  * **Image**: 

![Final](Screenshots/16.%20Final%20CSV%20view.png)

  * **What it shows**: VS Code preview of transformed CSV content showing updated schema:
    `PLAYER_ID, SALARY_CR, DEBUT_YEAR, COUNTRY, adjusted_salary`
  * **How it works**: Displays transformed data rows (e.g., `COUNTRY` converted to uppercase like `NEW ZEALAND`, `AUSTRALIA`, `AFGHANISTAN`, and `adjusted_salary` computed accurately).

---

## ⚡ How to Run

1. **Deploy S3 Buckets**: Create `bootcamp-input` and `bootcamp-output` buckets.
2. **Setup IAM Roles**: Configure `Bootcamp-Lambda` and `Bootcamp-Glue` roles with standard S3/CloudWatch/Glue policy attachments.
3. **Deploy Lambda Function**: Create function `bootcamp-function`, deploy `lambda_function.py`, and add S3 event notification on `bootcamp-input`.
4. **Configure Glue Workflow**: Build `glue-pipeline` containing crawlers (`Glue_Input`, `Glue_output`) and job (`glue_job`).
5. **Trigger Pipeline**: Upload `ipl_players_extra.csv` into `s3://bootcamp-input/input/`.
6. **Verify Output**: Monitor progress in Glue Workflow console or CloudWatch logs, and inspect transformed output in `s3://bootcamp-output/ipl_players_extra/`.