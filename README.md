<img width="2400" height="1261" alt="AWS SECRE" src="https://github.com/user-attachments/assets/55e6ecf4-f87d-4f7e-89a8-229fba2b13bc" />

# Secret Management with Secrets Manager and Lambda

## **Problem**

Organizations often hardcode sensitive information like database credentials, API keys, and configuration data directly into their application code or environment variables. This creates significant security risks as secrets become visible to developers, exposed in version control systems, and difficult to rotate without code deployments. When applications fail or are compromised, these hardcoded credentials can lead to data breaches and unauthorized access to critical systems.

## **Solution**


AWS Secrets Manager provides centralized, encrypted storage for sensitive information with automatic rotation capabilities, while Lambda functions offer serverless compute that can securely retrieve secrets at runtime. This combination eliminates hardcoded credentials by storing secrets in an encrypted service and retrieving them programmatically when needed. The AWS Parameters and Secrets Lambda Extension provides optimized performance with local caching, reducing API calls and improving function execution speed.

## Architecture Diagram



<img width="584" height="524" alt="image" src="https://github.com/user-attachments/assets/57a24f39-c2c2-470f-9539-ff863f058ae2" />


**Breakdown of what is happening:**

1. **The secret is stored** — Secrets Manager holds the database credentials (host, username, password, etc.) as one encrypted JSON object.
2. **Lambda gets just enough permission** — its IAM role can only read this one secret, nothing else in the account.
3. **The caching extension is attached to Lambda** — it's a layer that runs alongside the function and listens on `localhost:2773`.
4. **Lambda asks the extension for the secret** — instead of calling Secrets Manager directly, it asks the extension running next to it.
5. **The extension checks its cache first** — first time, it's empty, so it fetches from Secrets Manager and saves a copy (for 5 minutes). After that, it just hands back the saved copy — faster, and without calling Secrets Manager again.
6. **Lambda returns the safe parts only** — host, port, database name, username are returned; the password is deliberately left out of the response.
7. **CloudWatch captures logs and metrics** — execution logs (with 14-day retention) and alarms for errors/duration monitor the function's health, without ever logging the actual secret value.

## Prerequisites



- AWS account with administrative permissions for Secrets Manager and Lambda
- AWS CLI v2 installed and configured (`aws configure`)
- Terraform >= 1.0 installed
- Basic understanding of AWS IAM roles and policies
- Basic knowledge of serverless computing concepts
- Estimated cost: ~ϵ0.40–ϵ0.50/month if left running (Secrets Manager secret storage + minimal Lambda/CloudWatch usage); this lab was destroyed immediately after testing

## Tools & Services Used



- **AWS Secrets Manager** — encrypted storage for the sample database credentials
- **AWS Lambda** (Python 3.11) — serverless compute that retrieves and uses the secret
- **AWS Parameters and Secrets Lambda Extension** — Lambda layer providing a local caching HTTP proxy (`localhost:2773`) for secret retrieval
- **AWS IAM** — least-privilege execution role scoped to a single secret ARN
- **Amazon CloudWatch** — Logs (with 14-day retention) and metric alarms for errors/duration
- **Terraform** — Infrastructure as Code, including the `terraform-aws-modules/secrets-manager/aws` community module

## Preparation



This project's Terraform configuration is split across five files:

- **`versions.tf`** — pins the required Terraform version and providers (`aws`, `random`, `archive`), configures the AWS provider with default tags, and declares `aws_caller_identity` / `aws_region` data sources for live account context.
- **`variables.tf`** — defines all configurable inputs (project name, environment, Lambda settings, secret recovery window, rotation settings, etc.) with validation blocks.
- **`main.tf`** — the core resources: the Secrets Manager secret (via the `terraform-aws-modules/secrets-manager/aws` module), the Lambda execution IAM role and least-privilege policy, the zipped Lambda deployment package (via `archive_file` + `templatefile()`), the Lambda function itself (with the extension layer attached), a CloudWatch log group, and CloudWatch alarms for errors and duration.
- **`lambda_function.py.tpl`** — a Python template rendered by Terraform's `templatefile()` function; it calls the local extension endpoint to retrieve the secret, parses the JSON, and returns the non-sensitive fields (password excluded) in the HTTP-style response.
- **`outputs.tf`** — exposes the secret ARN/name, Lambda ARNs, IAM role ARN, log group name, ready-to-run AWS CLI test commands, and summary info (security posture, estimated costs, extension config).

## Steps


### **1. Define input variables and computed locals**

`variables.tf` establishes all configurable inputs with validation rules (e.g., environment must be one of `dev/staging/prod/test`, Lambda timeout must be 3–900 seconds) and a `locals` block that generates unique, collision-free resource names using a `random_string` suffix.

<img width="1050" height="721" alt="image-2" src="https://github.com/user-attachments/assets/8ecf6a1b-cec2-498e-83c1-b616980cc811" />


### **2. Create the Secrets Manager secret via a community module**

`main.tf` uses the `terraform-aws-modules/secrets-manager/aws` module to provision the secret, storing sample database credentials as JSON. `ignore_secret_changes = true` prevents Terraform from fighting with secret rotation if enabled later.

<img width="825" height="526" alt="image-3" src="https://github.com/user-attachments/assets/a6e2db51-d94c-4cce-81a4-140116e6c620" />


### **3. Build the least-privilege IAM role**

An IAM role is created for Lambda with the AWS-managed `AWSLambdaBasicExecutionRole` (CloudWatch Logs) attached, plus a custom inline-equivalent policy scoped to `secretsmanager:GetSecretValue` on only the specific secret's ARN, not all secrets in the account.


<img width="665" height="364" alt="image-4" src="https://github.com/user-attachments/assets/ea1a18e5-c6bb-43b6-8771-b9c5225f8390" />
<img width="739" height="598" alt="image-17" src="https://github.com/user-attachments/assets/29293908-14af-46ff-a07e-dc237ae17395" />



### **4. Package the Lambda function code**

The `archive_file` data source renders `lambda_function.py.tpl` via `templatefile()` (injecting the real secret name into the code) and zips it into a deployable package. No manual `zip` commands required.

<img width="680" height="238" alt="image-18" src="https://github.com/user-attachments/assets/6f061d2e-602e-4cae-83d8-0ac5f2ec4947" />


### **5. Deploy the Lambda function with the Secrets Extension layer**

The Lambda function is created with the AWS Parameters and Secrets Lambda Extension attached as a layer, resolved dynamically via SSM Parameter Store to avoid hardcoding a region-specific account ID. Environment variables configure the extension's local cache (enabled, size, max connections, port).

<img width="893" height="615" alt="image-19" src="https://github.com/user-attachments/assets/233a3f71-9a88-4df8-a6cd-4b75940a7554" />


### **6. Add CloudWatch logging and alarms**

A dedicated CloudWatch log group with 14-day retention avoids indefinite, uncontrolled log storage costs. Two metric alarms watch for Lambda errors and execution duration approaching the configured timeout.

<img width="749" height="814" alt="image-20" src="https://github.com/user-attachments/assets/256db45f-79f2-4b97-b4ab-a1da453bf033" />


## Validation & Testing

---

#### **1. Invoke the Lambda function**

Confirms the function runs successfully and retrieves the secret through the extension.

```bash
aws lambda invoke \
  --function-name $(terraform output -raw lambda_function_name) \
  --payload '{}' \
  response.json && cat response.json
```

<img width="1075" height="162" alt="image-21" src="https://github.com/user-attachments/assets/f4222d88-7495-49de-85d7-bcaec1ad12f6" />


#### **2. Check CloudWatch Logs**

Confirms the extension successfully retrieved the secret, without ever logging the actual sensitive values.

```bash
aws logs tail $(terraform output -raw lambda_log_group_name) --since 5m --region us-west-2
```

<img width="1296" height="270" alt="image-22" src="https://github.com/user-attachments/assets/12c8af13-d3e7-427e-bd32-3df9553f0070" />


#### **3. Verify the secret directly in Secrets Manager**

Confirms the raw secret value in Secrets Manager matches what the Lambda function retrieved end-to-end.

```bash
aws secretsmanager get-secret-value \
  --secret-id $(terraform output -raw secret_name) \
  --query 'SecretString' --output text | jq .
```

<img width="644" height="143" alt="image-23" src="https://github.com/user-attachments/assets/25943735-4b1e-4923-8458-152c6bf015f2" />


### Links
