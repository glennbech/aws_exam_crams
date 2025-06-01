## CodeCommit

**Overview:**
AWS CodeCommit is a fully managed source control service that hosts Git repositories. It’s similar to GitHub or Bitbucket, but natively integrated into the AWS ecosystem, making it easier to tie into CodePipeline and other DevOps tooling.

**Key Points & Context:**

* **IAM Roles for Power Users:**

  * Create a dedicated IAM role (e.g., `CodeCommitPowerUser`) that grants permissions to perform repository operations such as `GitPull`, `GitPush`, `CreateBranch`, and `MergePullRequest`.
  * Ensure least-privilege by scoping the role to specific repositories or repository prefixes.
  * Example policy snippets:

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "codecommit:GitPull",
            "codecommit:GitPush",
            "codecommit:CreateBranch",
            "codecommit:MergePullRequestByFastForward",
            "codecommit:CreatePullRequest"
          ],
          "Resource": "arn:aws:codecommit:us-east-1:123456789012:MyRepo"
        }
      ]
    }
    ```

**Suggested Reading:**

* [AWS CodeCommit User Guide](https://docs.aws.amazon.com/codecommit/latest/userguide/what-is-codecommit.html)
* [IAM Policies for CodeCommit](https://docs.aws.amazon.com/codecommit/latest/userguide/auth-and-access-control-iam.html)

---

## Codebuild 

---

## **AWS CodeBuild: Exam Cram**

### **What is CodeBuild?**

* **AWS CodeBuild** is a fully managed continuous integration (CI) service.
* **Purpose:** Compiles source code, runs tests, and produces deployable software artifacts.

---

### **Key Concepts & Responsibilities**

| Feature               | Description                                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------------ |
| **Build Environment** | The Docker image or environment where your code is built. Can be AWS-managed or custom.          |
| **Buildspec File**    | `buildspec.yml` — a YAML file that defines build commands, phases, artifacts, and env variables. |
| **Input Source**      | CodeBuild can pull code from S3, CodeCommit, GitHub, Bitbucket, etc.                             |
| **Artifacts**         | Output files produced by your build, stored in S3.                                               |
| **Logs**              | Stored in CloudWatch Logs (and optionally S3).                                                   |
| **IAM Role**          | Grants CodeBuild permission to access resources (S3, CodeCommit, etc).                           |

---

### **The `buildspec.yml` File**

Defines exactly what CodeBuild should do at each phase.

**Typical Structure:**

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Installing dependencies
      - npm install
  pre_build:
    commands:
      - echo Running tests
      - npm test
  build:
    commands:
      - echo Build started on `date`
      - npm run build
  post_build:
    commands:
      - echo Build completed on `date`

artifacts:
  files:
    - build/**/*

env:
  variables:
    NODE_ENV: production
```

---

#### **Phases Explained**

* **install**: Install dependencies or set up the environment.
* **pre\_build**: Pre-compile steps (testing, linting).
* **build**: The main build process.
* **post\_build**: Steps to run after build (cleanup, notifications, upload, etc.).

---

### **CodeBuild vs CodeDeploy vs CodePipeline**

* **CodeBuild**: *Builds* and *tests* code (not for deployment).
* **CodeDeploy**: *Deploys* built artifacts to EC2/Lambda/ECS/On-Prem.
* **CodePipeline**: *Orchestrates* workflows, chaining together CodeBuild, CodeDeploy, etc.

---

### **Lifecycle Hooks (in CodeBuild)**

Unlike CodeDeploy, **CodeBuild does not use AppSpec files or deployment lifecycle hooks.**
Instead, it relies on the `phases` in the `buildspec.yml` to run scripts at each stage.

---

### **Exam Tips:**

* **You must define `buildspec.yml`** or set build commands inline in the console.
* **Artifacts** are output to S3 (must be defined in `buildspec.yml` or console).
* **Environment variables** can be set in the file, in the console, or pulled from Parameter Store/Secrets Manager.
* **Custom Docker images** can be used for specialized build environments.
* **IAM permissions** are crucial for access to sources/artifacts/logs.
* **Build Badge**: Can display status of latest build for open-source projects.

---

### **Summary Table**

| CodeBuild Feature | Notes                                   |
| ----------------- | --------------------------------------- |
| `buildspec.yml`   | Defines the whole build workflow        |
| Phases            | install, pre\_build, build, post\_build |
| Artifacts         | Output files, go to S3                  |
| Logs              | CloudWatch by default                   |
| No AppSpec/Hook   | All steps are in buildspec, not AppSpec |

---

### **Example: Minimal buildspec.yml**

```yaml
version: 0.2
phases:
  build:
    commands:
      - echo "Compiling app..."
artifacts:
  files:
    - '**/*'
```

---

### **More elaborate: Building a docker image** 

```
version: 0.2

env:
  variables:
    IMAGE_REPO_NAME: "codex-trader"
    IMAGE_TAG: "codebuild-${CODEBUILD_BUILD_NUMBER}"
    AWS_REGION: "eu-west-1"

phases:
  pre_build:
    commands:
      - echo "Hello world"
      - echo Logging in to Amazon ECR...
      - aws --version
      - AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - echo AWS_ACCOUNT_ID $AWS_ACCOUNT_ID
      - echo AWS_REGION $AWS_REGION
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
      - IMAGE_TAG_WITH_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_REPO_NAME:${CODEBUILD_BUILD_NUMBER}"  
  build:
    commands:
      - docker build -t $IMAGE_TAG_WITH_URI .

  post_build:
    commands:
      - echo Pushing the Docker image to ECR...
      - docker push $IMAGE_TAG_WITH_URI
      - echo Writing image definitions file...
      - printf '[{"name":"container-name","imageUri":"%s"}]' $IMAGE_TAG_WITH_URI > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```


**In summary:**

* CodeBuild is *all about building* — you define every step in `buildspec.yml` (no AppSpec, no deployment hooks).
* You’re responsible for managing dependencies, scripts, build artifacts, and logs using the YAML configuration.

---

Let me know if you want practice questions, common exam traps, or advanced use cases!



## CodeDeploy

**Overview:**
AWS CodeDeploy automates code deployments to EC2 instances, on-premises servers, Lambda functions, and ECS. It uses an application, deployment group, and deployment structure. For Lambda/ECS, it can handle traffic shifting; for EC2/on-premises, it relies on the CodeDeploy Agent and lifecycle event hooks.

### Components

1. **Application**

   * Logical reference to the code you want to deploy (e.g., “MyWebApp”).

2. **Deployment Group**

   * Specifies which instances or resource group to target (e.g., Auto Scaling group, tags, or EC2 instance IDs).

3. **Deployment Configuration & Strategies**

   * **All at Once:** Deploy to all instances simultaneously. Fast but high risk.
   * **Rolling:** Deploy in batches; each batch waits for the previous batch to succeed.
   * **Canary (Minimum In-Service):** Deploy to a small subset (e.g., 10%), validate, then flip traffic to the rest.
   * **Blue/Green:** Provision a new environment, deploy there, switch traffic, and then decommission the old environment. (Primarily for Lambda/ECS.)

### AppSpec File Structure & Hooks

* **appspec.yml** defines files to copy, permissions, and lifecycle event hooks.

  ```yaml
  version: 0.0
  os: linux
  files:
    - source: /
      destination: /var/www/html
  hooks:
    BeforeInstall:
      - location: scripts/stop_server.sh
        timeout: 300
        runas: root
    AfterInstall:
      - location: scripts/install_dependencies.sh
        timeout: 300
        runas: root
    ApplicationStart:
      - location: scripts/start_server.sh
        timeout: 300
        runas: root
    ValidateService:
      - location: scripts/verify_status.sh
        timeout: 300
        runas: root
  ```

  * **BeforeInstall / AfterInstall:** Pre- and post-install scripts (e.g., stop services, backup files, adjust permissions).
  * **ApplicationStart:** Start or restart the application service.
  * **ValidateService:** Run health checks (e.g., curl to endpoint) to ensure deployment succeeded; returning non-zero will trigger a rollback.

**Suggested Reading:**

* [AWS CodeDeploy User Guide: AppSpec File Reference](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html)
* [Deployment Configurations](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html)

---

## CodePipeline

**Overview:**
AWS CodePipeline orchestrates a continuous delivery workflow by chaining together multiple stages (Source, Build, Test, Deploy). It integrates natively with CodeCommit, CodeBuild, CodeDeploy, and CloudFormation, among others.

### Key Concepts & Common Pitfalls

1. **S3 as Artifact Store**

   * Artifacts (source code, build outputs) are stored in an S3 bucket. This bucket must have versioning enabled if you plan to use S3 actions (e.g., S3Deploy).
   * If using KMS encryption for pipeline artifacts, ensure the KMS key policy grants `codepipeline.amazonaws.com` permission to `Encrypt` and `Decrypt`.

2. **Cross-Account Deployments**

   * When deploying CloudFormation stacks or using CodeDeploy in another account, you must:

     1. Create a customer-managed KMS key in the “source” account (Account A) and allow the CloudPipeline service role plus the target account (Account B) to use it.
     2. In Account B, create an IAM role that CodePipeline can assume (grant `sts:AssumeRole` permission).
     3. Update the S3 bucket policy in Account A to grant read access to the pipeline in Account B.
     4. For CloudFormation, attach an IAM service role to the CloudFormation action in the pipeline so the DevOps user only needs `iam:PassRole` to that CFN service role.

3. **IAM Capabilities for CloudFormation Actions**

   * When adding a CloudFormation deploy stage, check “Enable IAM role in pipeline” or “Allow IAM capabilities” so that the pipeline can create/update resources requiring IAM (e.g., roles, instance profiles).

4. **KMS and S3 Versioning**

   * **Common Misconception:** “You must enable versioning on the input bucket for CodePipeline to work.”

     * **Reality:** Versioning is required only if your pipeline uses Amazon S3 as a source or artifact location with KMS encryption.

**Suggested Reading:**

* [AWS CodePipeline User Guide](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html)
* [Cross-Account Pipeline Setup](https://aws.amazon.com/blogs/devops/cross-account-codepipeline-for-multi-account-cicd/)

---

## CloudFormation

**Overview:**
AWS CloudFormation (CFN) allows you to template and provision AWS resources in a predictable, repeatable way. Key DevOps exam topics include drift detection, custom resources, and service roles.

### Core Topics

1. **Stack Roles & Least-Privilege Deployment**

   * Create a **CloudFormation Service Role** granting CFN permission to provision resources (e.g., EC2, S3, IAM).
   * Grant the DevOps engineer only the `iam:PassRole` permission to pass that service role to CloudFormation. This adheres to least-privilege.

2. **Drift Detection & AWS Config Integration**

   * **Drift Detection:** CFN compares actual resource configurations to the template. A stack is:

     * **IN\_SYNC (Compliant)** if no differences.
     * **DRIFTED (Non-compliant)** if there are differences.
   * AWS Config’s managed rule `cloudformation-stack-drift-detection-check` uses the CFN API `DetectStackDrift`. If throttled or unavailable, AWS Config marks the rule as NON\_COMPLIANT by default.
   * **Note:** CloudFormation does not support drift detection on [Custom Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-resource.html#aws-resource-custom), so those resources are always skipped.

3. **Custom Resources & Lambda Callbacks**

   * When you need to perform something CFN can’t do natively, you write a Lambda-backed custom resource. CFN expects a SUCCESS or FAILED response via a pre-signed S3 URL (via `cfn-response` library or direct HTTPS PUT).

     * **Common Pitfall:** Forgetting to send the callback response causes the stack to hang until timeout (\~1 hour).
     * **Best Practice:** In your Lambda handler, after performing custom logic, call `send` using the response URL from the `event.ResponseURL`.

4. **cfn-init & cfn-signal**

   * **cfn-init:** Executed on EC2 instances to read metadata from the CFN template (e.g., packages to install, files to create, services to start).
   * **cfn-signal:** After initialization and application installation, instances signal back to the Auto Scaling group or CFN WaitCondition that they’re ready. Useful in rolling or wait conditions to ensure resources are fully configured before proceeding.

5. **Auto Remediation via AWS Config**

   * Example: Enforce `s3-bucket-logging-enabled` rule.
   * Steps to configure auto remediation:

     1. Enable AWS Config in the account and region.
     2. Create or choose a remediation action (e.g., `AWS-ConfigureS3BucketLogging`).
     3. Provide an IAM role (AutomationAssumeRole) that AWS Systems Manager (SSM) can assume to perform the remediation (must have `pass-role` permission).
     4. Associate the remediation action with the Config rule and select automatic remediation.
   * If a bucket is non-compliant (no logging), Config automatically triggers an SSM automation document to enable server access logging.

**Suggested Reading:**

* [AWS CloudFormation User Guide](https://docs.aws.amazon.com/cloudformation/latest/userguide/Welcome.html)
* [Drift Detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-drift.html)
* [Creating Custom Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)
* [AWS Config Auto Remediation](https://aws.amazon.com/blogs/mt/aws-config-auto-remediation-s3-compliance/)

---

## Elastic Beanstalk

**Overview:**
AWS Elastic Beanstalk (EB) abstracts away infrastructure management for web applications. You deploy code, and EB handles capacity provisioning, load balancing, scaling, and application health.

### Key Topics

1. **.ebextensions**

   * Custom configuration files in YAML or JSON placed under the `.ebextensions/` directory in your source bundle.
   * Use `.ebextensions/*.config` to modify instance configuration—install packages, run container commands, set environment variables, create files, or run scripts before the application is deployed.
   * Example: `db-migration.config`

     ```yaml
     commands:
       run_migrations:
         command: "python manage.py migrate"
         cwd: "/var/app/current"
         ignoreErrors: false
     ```
   * Use `container_commands` instead of `commands` if you need to run commands after application and web server have been set up.

2. **Db Migrations in Beanstalk**

   * To run database migrations on deploy, create an `.ebextensions/db-migration.config` file with a `commands` block that executes your migration scripts.
   * Consider `lock_mode: true` to ensure only one instance runs migrations (to avoid race conditions).

3. **EB Deployment Options**

   * **All at Once:** Quickest, but downtime occurs.
   * **Rolling:** Deploy to a few instances at a time; can be slow if you have many instances.
   * **Rolling with Additional Batch:** Provision a new batch of instances, deploy there, then terminate the old batch, reducing downtime.
   * **Immutable:** Launch a parallel group of instances, validate health, then shift traffic. Lowest risk but doubles the capacity temporarily.

**Suggested Reading:**

* [AWS Elastic Beanstalk Developer Guide: .ebextensions](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions.html)
* [Elastic Beanstalk Deployment Policies](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.rolling-version-deploy.html)

---

## Auto Scaling & Lifecycle Hooks

**Overview:**
Amazon EC2 Auto Scaling maintains application availability by ensuring the correct number of instances. Advanced topics include lifecycle hooks, rebalancing behavior, and cross-account KMS for encrypted AMIs.

### Auto Scaling Group Capacity & Rebalancing

* **Temporary Capacity Increase:**

  * When an Auto Scaling group (ASG) needs to rebalance across Availability Zones (e.g., after AZ failure or user-requested rezoning), it can exceed the `MaxSize` by up to 10% (or one instance, whichever is larger) to prevent downtime. Once new instances are healthy, old instances terminate, returning to within `MaxSize`.
  * This margin only applies when the group is at or near max capacity and requires rebalancing.

### Lifecycle Hooks

* Lifecycle hooks allow you to perform custom actions when instances launch or terminate, pausing the transition and giving time for inspections or custom setup/cleanup.
* **Key Transitions:**

  1. **Pending\:Wait** – Triggered after instance launch but before entering service. You might:

     * Install specialized software.
     * Register instance in external CMDB.
     * Configure application-specific parameters.
     * **Actions:** Respond with `CONTINUE` (proceed with launch), `ABANDON` (terminate immediately), or let it time out (configured `DefaultResult`).
  2. **Pending\:Proceed** – Transitional state; once you send `CONTINUE`, the instance continues launching.
  3. **Terminating\:Wait** – Triggered when instance receives scale-in signal but before termination. You might:

     * Drain connections from load balancer.
     * Copy logs off-instance.
     * Release external licenses.
     * **Actions:** Same as `Pending:Wait`.
  4. **Terminating\:Proceed** – Transitional state; after `CONTINUE`, the instance terminates.

**Suggested Reading:**

* [Auto Scaling Lifecycle Hooks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html)
* [Auto Scaling Rebalancing](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-mixed-instances.html#as-group-rebalancing)

---

## Amazon EC2 AMI Encryption & Cross-Account Sharing

**Overview:**
To meet security/compliance, you may need to share encrypted AMIs across accounts. AWS only allows sharing AMIs backed by snapshots encrypted with customer-managed KMS keys (not AWS-managed keys).

### Steps to Share an Encrypted AMI

1. **In Account A (Source):**

   * **Create an Encrypted AMI:** Copy the unencrypted AMI, specify your customer-managed KMS key (CMK) for encryption.
   * **Modify KMS Key Policy:**

     * Add a statement granting the DevOps (or pipeline) IAM role in Account B `kms:CreateGrant`, `kms:Encrypt`, `kms:Decrypt`, `kms:DescribeKey`, and so on.
     * Example KMS key policy snippet:

       ```json
       {
         "Sid": "AllowAccountBToCreateGrant",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::ACCOUNT_B_ID:role/AutoScalingServiceRole"
         },
         "Action": [
           "kms:CreateGrant",
           "kms:DescribeKey",
           "kms:Decrypt"
         ],
         "Resource": "*"
       }
       ```
   * **Share the AMI:** Modify AMI permissions to allow Account B to copy/launch. Ensure underlying snapshots are shared in encrypted form.

2. **In Account B (Target):**

   * **Create a KMS Grant:**

     * Using the KMS console or CLI, create a grant on the CMK in Account A with `GranteePrincipal` set to the Auto Scaling service-linked role ARN in Account B (`arn:aws:iam::ACCOUNT_B_ID:role/aws-service-role/autoscaling.amazonaws.com/AWSServiceRoleForAutoScaling`).
     * This grant allows the ASG to decrypt the snapshot when launching instances.
   * **Launch the ASG:**

     * Reference the shared AMI ID (the encrypted AMI). The ASG can now launch instances encrypted with the CMK because the grant delegates permissions.

**Suggested Reading:**

* [Sharing Encrypted AMIs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sharing-amis.html#share-encrypted-ami)
* [KMS Grants & Cross-Account Access](https://docs.aws.amazon.com/kms/latest/developerguide/grants.html)

---

## AWS Config & Auto Remediation

**Overview:**
AWS Config continuously evaluates resource configurations against desired baselines. Auto Remediation allows you to automatically correct non-compliant resources using SSM Automation Documents or AWS-managed remediation actions.

### Auto Remediation Workflow

1. **Enable AWS Config:** In each region/account where you want compliance checks.
2. **Choose or Create a Config Rule:**

   * Example: `s3-bucket-logging-enabled`.
3. **Select a Remediation Action:**

   * AWS-managed: e.g., `AWS-ConfigureS3BucketLogging`.
   * Custom: Write your own SSM Automation Document to remediate resources (e.g., apply tags, change configurations).
4. **AutomationAssumeRole:**

   * The role that Config uses to execute the SSM automation must be assumable by SSM (`sts:AssumeRole`).
   * The user creating the remediation action must have `iam:PassRole` permission to pass that role.
5. **Associate & Enable Automatic Remediation:**

   * Once associated, when AWS Config identifies a non-compliant resource, it triggers the SSM document.
   * If remediation fails or the resource remains non-compliant, Config can retry according to settings.

**Suggested Reading:**

* [AWS Config Auto Remediation](https://docs.aws.amazon.com/config/latest/developerguide/remediation-overview.html)
* [AWS-ConfigureS3BucketLogging Action Details](https://docs.aws.amazon.com/config/latest/developerguide/ssm-doc-aws-config-automate-rule.html)

---

## Amazon EventBridge

**Overview:**
EventBridge is a serverless event bus that can ingest events from AWS services, SaaS providers, or custom applications. You can define rules to filter the events and route them to targets like Lambda, SNS, SQS, or Kinesis.

### Key Use Cases & Examples

1. **Input Transformer:**

   * Transforms incoming event JSON to a customized payload before sending it to a target. Useful when the target expects a specific structure or fewer fields.
   * Example: Extract only `detail.jobName` and `detail.status` from an AWS Batch or Glue job event.

2. **AWS Glue Job Retry Failure Notification:**

   * **Rule:** Match events where `source = aws.glue` and `detail.jobRunState = FAILED` and `detail.attempt = >1`.
   * **Target:** Lambda function processes the event, filters for retry failures, and publishes to SNS for the security or DevOps team.
   * **Correct Pattern:** Use EventBridge to detect Glue job retries that fail, not rely on basic CloudWatch alarms.

3. **CloudWatch Event to Trigger SSM Automation:**

   * Example: When AWS Trusted Advisor detects a low-utilized EC2 instance, it emits an EventBridge event.
   * Use a Rule to match `source = aws.trustedadvisor` and `detail.checkId = “LowUtilizedInstances”`, route to a Lambda or SSM Automation document workflow with a manual approval step for instance termination.

**Suggested Reading:**

* [Amazon EventBridge User Guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/what-is-amazon-eventbridge.html)
* [EventBridge Input Transformer](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-transform-target-input.html)
* [AWS Glue Event Patterns](https://docs.aws.amazon.com/glue/latest/dg/monitor-continuous-logging.html#glue-cloudwatch-events)

---

## AWS Lambda Deployment Strategies with CodeDeploy & SAM

**Overview:**
For serverless applications, AWS Serverless Application Model (SAM) simplifies CloudFormation templates and integrates with CodeDeploy to perform traffic shifting (canary or linear deployments) and automatic rollbacks.

### Key Concepts

1. **SAM & CloudFormation Integration:**

   * Define functions, APIs, and permissions in a `template.yaml` using shorthand SAM syntax.
   * During `sam deploy`, SAM transforms into CloudFormation and provisions resources.

2. **Traffic Shifting with CodeDeploy:**

   * **PreTraffic & PostTraffic Hooks:** Use Lambda functions to run integration or health checks before shifting traffic.
   * **Canary Deployment:** Send a small percentage of traffic (e.g., 10%) to the new version for a set period, validate, then shift 100% if healthy.
   * **Automatic Rollbacks:** Define CloudWatch alarms (e.g., errors, latency) that CodeDeploy watches. If alarm triggers, CodeDeploy rolls back to the previous version.

3. **Rollback Conditions:**

   * Any CloudWatch alarm (e.g., `Errors > 1` for 5 minutes) can trigger automatic rollback. Ensure alarms are defined and attached to the deployment group.

**Suggested Reading:**

* [AWS SAM Developer Guide: Canary Deployment](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-canary-deployment.html)
* [CodeDeploy for Lambda](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-lambda.html)

---

## Amazon CloudWatch & Logging

**Overview:**
CloudWatch provides monitoring via metrics, logs, and alarms. Key DevOps scenarios include cross-account observability and real-time log processing.

### Cross-Account Observability

* **Monitoring Account vs. Source Accounts:**

  * Configure one or more accounts as centralized “monitoring” accounts.
  * Source accounts publish their metrics, logs, and traces to the monitoring account.
  * Two onboarding methods:

    1. **AWS Organizations:** Automatically onboards all member accounts.
    2. **Individual Account Linking:** Manually link each source account.

* **Shared Telemetry:**

  * Metrics (CloudWatch Metrics)
  * Log groups (CloudWatch Logs)
  * Traces (X-Ray)

* **Use Case:** A security operations team in an AWS Organization wants full-stack visibility across dev, test, and prod.

### CloudWatch Logs to S3 & Kinesis

* **Log Subscriptions & Destinations:**

  * Create a **Log Destination** in the centralized account (e.g., an AWS Kinesis Data Firehose).
  * In each source account, create a **Subscription Filter** that routes logs (JSON) to the Firehose.
  * Configure Firehose to deliver to an S3 bucket for long-term storage, analytics, or Athena queries.

* **Encryption with KMS:**

  * When CloudWatch Logs are at rest, you can specify a customer-managed CMK to satisfy compliance (e.g., FIPS 140-2).
  * Ensure the CMK policy allows `logs.amazonaws.com` to use it.

* **Real-Time Filters:**

  * Use CloudWatch metric filters on logs (e.g., detect `DeleteBucket` API calls in CloudTrail logs).
  * Create CloudWatch Alarms on these custom metrics to alert or trigger Lambda/SSM Automations for remediation.

### AWS WAF Logging

* **Log Destination Requirements:**

  * S3 bucket names for WAF logs must start with `aws-waf-logs-`.
  * Logs are published every 5 minutes; max file size is 75 MB.
  * If a file reaches 75 MB before 5 minutes, it’s delivered early and a new file is started.

**Suggested Reading:**

* [CloudWatch Cross-Account Observability](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Cross-Account_Observability.html)
* [CloudWatch Logs Subscriptions & Destinations](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Subscriptions.html)
* [AWS WAF Logging to S3](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html)

---

## Amazon ECS (EC2) & ECR Logging

**Overview:**
When running Docker containers on ECS Classic (EC2 launch type), you can stream logs from containers to CloudWatch Logs by configuring the log driver in the task definition.

### Correct Configuration

* **awslogs Log Driver:**

  * In your ECS task definition:

    ```json
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-region": "eu-west-1",
        "awslogs-group": "/ecs/my-application",
        "awslogs-stream-prefix": "ecs"
      }
    }
    ```
  * **IAM Instance Role:**

    * Attach a role to the EC2 instances (the ECS container instances) with permissions:

      ```json
      {
        "Effect": "Allow",
        "Action": [
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:CreateLogGroup"
        ],
        "Resource": "*"
      }
      ```
  * No need for sidecar or CloudWatch Agent on the instance; the Docker daemon forwards container logs directly.

**Common Incorrect Approaches:**

* Mapping `/var/log` on instance + installing CloudWatch Agent: unnecessary complexity.
* Sidecar container to run the CloudWatch Agent: not required with the `awslogs` driver.

**Suggested Reading:**

* [Amazon ECS Task Logging](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html)
* [CloudWatch Logs IAM Permissions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-logging-cloudwatch-permissions.html)

---

## AWS Systems Manager (SSM)

**Overview:**
AWS Systems Manager provides operational data, automation, and management capabilities for AWS resources and on-premises servers. Key DevOps exam topics include hybrid activation, inventory, patching, and SSM Automation.

### Hybrid Fleet Management

* **Activation Process:**

  1. **Create IAM Service Role for SSM:**

     * Allows managed instances (either EC2 or on-prem) to perform `ssm:AssumeRole`.
     * Example policy includes SSM permissions like `ssm:SendCommand`, `ssm:GetParameters`, etc.
  2. **Generate Activation Code & ID:**

     * In the Systems Manager console (Hybrid Activations → Create Activation).
     * Associates an internal `mi-` managed instance profile.
  3. **Register On-Prem Servers:**

     * Install the SSM agent on on-prem servers.
     * Run activation commands (`amazon-ssm-agent` with `--activation-code` and `--activation-id`).
     * On-prem servers appear with prefix `mi-` in the SSM console.

### SSM Inventory & Compliance

* **SSM Inventory:**

  * Automatically collects metadata about managed instances (e.g., installed applications, OS details, network configurations).
  * Use SSM Inventory along with AWS Config to correlate resource configuration changes over time.

* **Patch Management:**

  * **Custom Patch Baseline:** Define custom rules (e.g., allowed CVEs, excluded patches); attach to patch groups (e.g., “Production” tag).
  * **Maintenance Windows:** Schedule recurring windows (e.g., every Sunday at 2 AM); include Run Command `AWS-RunPatchBaseline` to apply patches.
  * For minimal disruption:

    1. Define maintenance window with a weekly recurrence.
    2. Ensure your ASG instances drain connections before patching.
    3. Use SSM Automation documents to orchestrate patching and validate success, then call Auto Scaling to refresh instances if needed.

### SSM Automation with CloudWatch Events

* **Trust Advisor Low-Utilized EC2:**

  * Rule: EventBridge matches `source = aws.trustedadvisor` & `detail.checkId = “LowUtilizedInstances”`.
  * Target: Lambda function that triggers an SSM Automation document. Document can:

    * Pause for manual approval (SSM Automation “aws\:waitForApproval”).
    * If approved, proceed to terminate instance via `aws:terminateEC2Instance` action.

**Suggested Reading:**

* [AWS Systems Manager User Guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html)
* [Hybrid Activations](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-managedinstances.html)
* [SSM Inventory](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-inventory.html)
* [Patch Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-patch.html)

---

## Amazon EFS Replication & Data Replication Patterns

**Overview:**
While Amazon EFS provides Regional file systems, replicating data across Regions (or to other services) requires building custom solutions. AWS also now offers Amazon EFS Replication, but manual patterns can involve EC2 and Kinesis.

### Custom Cross-Region Replication

* **Replication Cluster (EC2-Based):**

  1. **In eu-west-1:** Launch an EC2 Auto Scaling group that monitors a custom CloudWatch metric for “file lag” (e.g., difference between last file timestamp on a primary EFS vs. replication target).
  2. **Sync to S3 in ap-southeast-2:**

     * Use a process (e.g., rsync or AWS DataSync) to push new files from primary EFS to an S3 bucket.
  3. **In ap-southeast-2:** Launch another replication cluster that reads from the S3 bucket and writes to a standby EFS in that region.

* **Consider AWS DataSync:** AWS DataSync now supports cross-region EFS replication natively, removing the need for custom EC2.

**Suggested Reading:**

* [Amazon EFS Replication Documentation](https://docs.aws.amazon.com/efs/latest/ug/replication.html)
* [AWS DataSync User Guide](https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html)

---

## Amazon API Gateway

**Overview:**
API Gateway provides a front door for applications to access data, business logic, or functionality from backend services (e.g., Lambda, Step Functions, HTTP endpoints).

### Key Topics

1. **Canary Releases:**

   * Use stages and deployments to shift a percentage of traffic to a new deployment (e.g., 10% for 10 minutes, then 100% if stable).
   * Configure in the console under “Stage → Deployment Settings → Canary”.

2. **Service Integrations:**

   * **Direct AWS Service Integration:** Without Lambda, API Gateway can call supported AWS services directly (e.g., S3, DynamoDB) using AWS credentials you configure in the integration.
   * **Step Functions Integration:**

     * Create a Step Functions state machine.
     * In API Gateway method integration, choose “Step Functions” and specify the ARN of the state machine.
     * API Gateway can pass the request payload to Step Functions and return the output as the REST response.

3. **Regional vs. Edge-Optimized Endpoints:**

   * **Regional:** Best for clients within the same AWS Region; lower latency for regional clients.
   * **Edge-Optimized:** Uses CloudFront to cache and reduce latency for users globally.

**Suggested Reading:**

* [Amazon API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
* [Canary Deployment for API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/canary-release.html)

---

## AWS WAF

**Overview:**
AWS WAF protects web applications or APIs by filtering malicious requests. To analyze traffic, you can enable logging and route logs to CloudWatch, Kinesis, or S3.

### Logging to S3

* **Bucket Naming Convention:**

  * Must start with `aws-waf-logs-` (for example, `aws-waf-logs-mycompany-app`).
* **Log Delivery:**

  * AWS WAF publishes logs every 5 minutes; each file ≤75 MB. If size limit is reached earlier, it delivers and starts a new file.
  * **Destination Options:**

    * CloudWatch Log Group
    * S3 bucket
    * Kinesis Data Firehose

**Suggested Reading:**

* [AWS WAF Logging and Metrics](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html)

---

## API Gateway Canary Release

**Overview:**
API Gateway supports canary deployments at the stage level. You specify traffic weights and canary deployment steps directly in the console or via CloudFormation.

**Key Steps:**

1. Create or select a stage (e.g., `prod`).
2. Under “Deployment” settings, enable “Canary deployment” and specify:

   * **Percent traffic to canary** (e.g., 10%).
   * **Stage variables** to isolate canary configuration (e.g., new Lambda alias).
3. Monitor metrics (Latency, 5XX errors) on the canary stage via CloudWatch.
4. If stable, ramp to 100%; if unstable, roll back by disabling canary.

**Suggested Reading:**

* [API Gateway Canary Release](https://docs.aws.amazon.com/apigateway/latest/developerguide/canary-release.html)

---

## AWS KMS: Throttling Considerations

**Context:**
When you encrypt large numbers of S3 objects using server-side encryption with KMS (SSE-KMS), you may hit a throttling threshold (10,000 requests per second against the CMK).

* **Solution:**

  * Use **SSE-S3** (Amazon-managed S3 keys) for large-scale, high-throughput write workloads where specific CMK control isn’t mandatory.
  * If you need CMK, consider batching writes, using multiple CMKs with key rotation or parallelization to distribute load.

**Suggested Reading:**

* [Understanding KMS Request Quotas](https://docs.aws.amazon.com/kms/latest/developerguide/service_limits.html)
* [S3 Encryption Options](https://docs.aws.amazon.com/AmazonS3/latest/dev/serv-side-encryption.html)

---

## Amazon S3: Enforcing HTTPS Only

**Requirement:**
Ensure that requests to the S3 bucket are only over HTTPS, denying any HTTP requests.

* **Bucket Policy Example:**

  ```json
  {
    "Version":"2012-10-17",
    "Statement":[
      {
        "Sid":"AllowSSLRequestsOnly",
        "Effect":"Deny",
        "Principal":"*",
        "Action":"s3:*",
        "Resource":["arn:aws:s3:::my-secure-bucket","arn:aws:s3:::my-secure-bucket/*"],
        "Condition":{
          "Bool":{
            "aws:SecureTransport":"false"
          }
        }
      }
    ]
  }
  ```

  * The condition `"aws:SecureTransport":"false"` matches HTTP requests; `Deny` blocks them.
  * Always explicitly **deny** HTTP even if IAM policies grant `s3:GetObject` over HTTPS.

**Suggested Reading:**

* [Amazon S3 Bucket Policy Examples](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-bucket-policies.html)

---

## Amazon API Gateway & Step Functions

**Overview:**
API Gateway can integrate directly with AWS Step Functions, eliminating the need for a Lambda function as intermediary.

* **Integration Steps:**

  1. Create a Step Functions state machine with input schema matching the expected payload.
  2. In API Gateway, create or update a resource method → Integration Type: **AWS Service**.

     * Service: `Step Functions`
     * Action: `StartSyncExecution` (for synchronous) or `StartExecution` (for asynchronous)
     * IAM Role: Create an IAM Role that API Gateway can assume with `states:StartExecution`.
  3. Map method request data to Step Functions input via mapping templates.

**Suggested Reading:**

* [API Gateway Integration with Step Functions](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-stepfunctions-integration.html)

---

## Amazon CloudTrail → CloudWatch Logs with KMS

**Overview:**
To audit API calls, CloudTrail can stream logs to CloudWatch Logs. For compliance (e.g., FIPS 140-2), encrypt CloudWatch Log Groups with a customer-managed CMK.

* **Steps:**

  1. **Enable CloudTrail** with a new or existing Trail. Check “Send to CloudWatch Logs.”
  2. **Create a CloudWatch Logs log group** (e.g., `/aws/cloudtrail/audit-logs`).
  3. **KMS Encryption:**

     * Create/choose a KMS CMK with a policy granting `logs.amazonaws.com` permission to `Encrypt`, `Decrypt`, and `GenerateDataKey`.
     * In CloudWatch Logs, specify the CMK ARN for encryption.

* **Use Cases:**

  * Real-time detection of critical API calls (e.g., `DeleteBucket`).
  * Set up metric filters on the log group (e.g., pattern: `DeleteBucket`) → Create custom metrics → CloudWatch Alarm → SNS notification.

**Suggested Reading:**

* [CloudTrail Integration with CloudWatch Logs](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-cloudwatch-alarms-with-logs.html)
* [CloudWatch Logs Encryption](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/encrypt-log-data.html)

---

## AWS Config Aggregator

**Overview:**
AWS Config Aggregator collects resource configuration and compliance data across multiple accounts and regions into a single account.

* **Use Case:**

  * Identify EC2 instances not launched from a recent, approved shared AMI.
  * Create a **Custom Config Rule** (e.g., using a Lambda function) that checks the AMI ID of every EC2 instance against a list of approved AMI IDs (or tags).
  * Deploy this rule to all accounts via CloudFormation StackSets.
  * In the aggregator account, view compliance results and generate summaries.

**Suggested Reading:**

* [AWS Config Aggregator](https://docs.aws.amazon.com/config/latest/developerguide/aggregate-data.html)
* [Creating Custom AWS Config Rules](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config_develop-rules.html)

---

## AWS WAF & Traffic Logging

**Overview:**
AWS WAF can log web ACL traffic to CloudWatch Logs, Kinesis Data Firehose, or S3. Enabling logging helps you analyze malicious or unwanted traffic patterns.

* **Enable Logging:**

  1. In WAF Console → Web ACL → Logging and Metrics → Enable Logging.
  2. Select **S3 bucket** as destination; bucket name must start with `aws-waf-logs-`.
  3. Or choose CloudWatch Logs group or Kinesis Data Firehose stream.

* **Retention & Size:**

  * Log files published every 5 minutes; rotate if >75 MB.
  * Use Athena to query WAF logs stored in S3 for threat hunting.

**Suggested Reading:**

* [AWS WAF Logging and Metrics](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html)

---

## Systems Manager Patch Management & Custom Baselines

**Scenario:**
A financial services firm needs daily vulnerability scans of security-hardened AMIs (driven by CVEs). Instances launch via an Auto Scaling group using the latest hardened AMI.

### Proposed Solution

1. **Custom Patch Baseline Using SSM:**

   * Define a custom patch baseline that includes rules to exclude non-compliant patches or allow specific CVE-based patches.
   * Associate the baseline with a patch group tag (e.g., `SecurityHardened`).
2. **Maintenance Window:**

   * Set a maintenance window to run once daily (e.g., every morning at 3 AM).
   * Add `AWS-RunPatchBaseline` as a task.
   * Configure maintenance window targets using tags (ASG instances typically have a tag).
3. **Minimize Disruption:**

   * Use `allow-reboot: false` or `rebootOption: NoReboot` if patching without reboots is possible; otherwise schedule a controlled reboot.
   * Use `baseline override` to exclude patches that require a full reboot (for AMI validation, you might want daily patch reporting rather than forcing reboots on production).

**Suggested Reading:**

* [SSM Patch Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-patch.html)
* [Automating Patch Compliance](https://aws.amazon.com/blogs/mt/building-a-compliant-patch-management-solution-with-aws-systems-manager/)

---

## AWS OpsWorks (Optional)

**Overview:**
AWS OpsWorks is a configuration management service that uses Chef or Puppet. It’s less studied in the DevOps Pro exam but still relevant for certain organizations.

* **Use Cases:**

  * Configuration as Code with Chef recipes or Puppet modules.
  * Automating infrastructure provisioning and lifecycle.

**Suggested Reading:**

* [AWS OpsWorks User Guide](https://docs.aws.amazon.com/opsworks/latest/userguide/welcome.html)

---

## CloudWatch Retention & Aggregation

* **Retention Settings:**

  * CloudWatch Logs: Default is “Never Expire”; you can adjust per log group (e.g., 30 days, 90 days) to manage storage costs.
  * Metrics: Detailed metrics (1-minute granularity) stored for 15 days; 5-minute metrics for 63 days; 1-hour metrics for 455 days.

* **Metric Math & Dashboards:**

  * Aggregate metrics using math expressions (e.g., `SUM`, `AVG`, `ANOMALY_DETECTION_BAND`) to create service health dashboards.

**Suggested Reading:**

* [CloudWatch Logs Retention](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/RetentionAndStorage.html)
* [CloudWatch Metric Math](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/using-metric-math.html)

---

## API Gateway Endpoints

1. **Regional Endpoints:**

   * Intended for clients in the same AWS Region.
   * Lower latency for in-region requests; integrates with regional custom domains.

2. **Edge-Optimized Endpoints:**

   * Use CloudFront to cache responses at edge locations.
   * Best for clients distributed globally to reduce latency.

3. **Private Endpoints (VPC):**

   * Accessible only from within a VPC via an interface VPC endpoint (powered by AWS PrivateLink).
   * Useful for internal microservices or intranet APIs.

**Suggested Reading:**

* [Choosing the Right Endpoint Type](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-custom-domain.html#choose-endpoint-type)

---

## AWS Trusted Advisor Integration

**Use Case:**
Automatically detect low-utilized EC2 instances and remediate.

* **Workflow:**

  1. **Enable Trusted Advisor** in all accounts.
  2. **CloudWatch Event Rule:**

     * Source: `aws.trustedadvisor`
     * Check: `LowUtilizedInstances`
  3. **Lambda Function Target:**

     * Parses the event, extracts instance IDs.
     * Triggers an SSM Automation run to (a) notify team, (b) gather confirmation, and (c) terminate instances if approved.

* **Important Note:**

  * The AWS-RunShellScript or a custom SSM document can decide to terminate or hibernate instances after gathering logs.

**Suggested Reading:**

* [Automating Trusted Advisor with CloudWatch Events](https://aws.amazon.com/blogs/mt/automate-aws-trusted-advisor-checks-with-cloudwatch-events/)

---

## AWS Inspector & Programmatic Runs

**Overview:**
Amazon Inspector is an automated security assessment service for EC2 instances and container images.

* **Invoke Inspector Programmatically:**

  1. Use AWS SDK or CLI:

     ```bash
     aws inspector2 start-findings-report
     aws inspector2 list-findings --filters ...
     ```
  2. Automate via Lambda or SSM Automation Document—schedule daily runs, send findings to SNS or Slack.

**Suggested Reading:**

* [Amazon Inspector User Guide](https://docs.aws.amazon.com/inspector/latest/userguide/inspector2.html)

---

## Examination Tips & ‘Wrong Answer’ Context

* **CodePipeline KMS & Versioning:**

  * **Wrong:** “Create a KMS key and enable versioning on the input bucket to work with CodePipeline.”
  * **Clarification:** Versioning is required only if using S3 source actions with KMS-encrypted buckets.

* **AWS Config Drift Detection Errors:**

  * **Issue:** `Rate Exceeded` error for `DetectStackDrift`.
  * **Note:** Config marks the rule NON\_COMPLIANT if throttled; increasing API limits or reducing frequency can help.

* **CloudFormation Custom Resources:**

  * **Pitfall:** CFN does not detect drift on custom resources. Rely on AWS Config or manual checks.

* **Glue Job Failure Notifications:**

  * **Wrong:** “Configure EventBridge for Glue and set SNS to notify directly on retry failures.”
  * **Correct:** Use a Lambda as a target to filter events for job retry failures and then publish to SNS.

* **ECS Logging Best Practice:**

  * **Wrong:** Install CloudWatch Agent on EC2, map `/var/log` or use sidecar.
  * **Correct:** Use `awslogs` log driver in task definition with proper IAM instance role.

* **Beanstalk AfterInstall Hook Limitations:**

  * **Pitfall:** “Using AfterInstall to verify app health.”
  * **Note:** AfterInstall is executed before `ApplicationStart`; health checks should occur in `ValidateService` or `ApplicationStart`.

* **SSM Inventory vs. AWS Config:**

  * **Misconception:** “SSM Inventory can track resources without tags.”
  * **Clarification:** Inventory only reports installed software/config on managed instances. AWS Config is needed for resource configurations and tag compliance.

---

## Summary & Next Steps

These reorganized notes group related topics by AWS service, provide additional context, and include suggested reading so you can drill deeper into each area. As you study for the AWS Certified DevOps Engineer – Professional exam:

1. **Review AWS Whitepapers & FAQs:**

   * Focus on services you’re less familiar with (e.g., EFS replication, cross-account pipelines).
   * Understand best practices, exam-style scenarios, and typical “gotchas.”

2. **Hands-On Labs:**

   * Spin up sample pipelines with cross-account CloudFormation stages.
   * Practice CodeDeploy hooks using a simple web application on EC2.
   * Experiment with SAM canary deployments for Lambda functions.

3. **Practice Exam Questions:**

   * Identify patterns (e.g., cross-account IAM, KMS grants, least-privilege CFN roles).
   * Review AWS-ish “Why is Option X wrong?” explanations to understand the nuances.
