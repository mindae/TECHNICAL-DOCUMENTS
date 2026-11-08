# AWS for Java Developers & Architects — Hands-on Learning

> **Purpose:** Learn to use AWS to run and support real applications. Breadth first, implementation next, architecture depth when needed. Not a certification guide.
>
> **Repository safety:** This document contains only examples and placeholders. Never commit credentials, session tokens, account IDs, real resource ARNs/URLs, personal emails, `.aws` files, `.env` files, or logs containing sensitive data.

## Learning approach

- **Application first:** Begin with the problem, then decide whether AWS is the right tool.
- **CLI first, code second:** Run and understand a minimal AWS CLI operation before using the AWS SDK for Java/Spring Boot.
- **Explain commands on first use:** Record abbreviations, options, and expected output briefly.
- **Create → Verify → Use → Delete → Verify:** Do not leave cloud lab resources running between sessions.
- **Occasional Git revision:** Use `git status`, `git diff`, branches, commits, and history during actual development.
- **Safety:** Use the AWS **Free account plan**; do not upgrade to Paid or join an AWS Organization for these labs. Free-plan usage can still consume credits; verify eligibility before each resource creation.

## Three-phase plan

| Phase | Focus | Completion target |
|---|---|---|
| **1 — Basics (3 days)** | Core AWS services, identity, CLI, safe resource lifecycle, common architecture | Recognise AWS components and run basic labs |
| **2 — Intermediate (later)** | AWS SDK for Java, Spring Boot integrations, IAM roles, networking, ECS, Terraform, testing | Build and deploy a small application |
| **3 — Advanced (later)** | Availability, security, scaling, observability, reliability, cost and architecture trade-offs | Review and design production systems |

### Phase 1 schedule

| Day | Subjects | Status |
|---|---|---|
| **1** | Account safety and MFA; CLI login; S3, SQS, DynamoDB; basic IAM/cost concepts | **S3, SQS, DynamoDB completed and cleaned up**; IAM review is optional |
| **2** | EC2, AMI, EBS, VPC, subnets, security groups; AWS Budgets; Lambda and CloudWatch | **EC2 and Lambda completed, cleanup verified**; ALB/NAT walkthrough covered |
| **3** | ECR/ECS/Fargate/EKS, RDS/Aurora, CloudTrail, infrastructure as code; architecture recap | **Next — do not provision large/billed infrastructure solely for Pass 1** |

> Not every service must be provisioned. Expensive, unsupported, or continuously billed services are covered conceptually during Phase 1.

## Prerequisites and local tooling

- Mac with AWS CLI v2, Git, Java, Maven, Spring Boot, and Docker.
- AWS Free-plan account with root MFA enabled.
- Browser-based CLI login using temporary credentials.
- Local AWS emulator (Floci) retained for offline practice.
- Single chosen lab region. Example below: `ap-southeast-2` (replace if your project uses another region).

### Authentication and safe defaults

```bash
# Remove emulator or stale environment settings from this terminal.
unset AWS_PROFILE AWS_ENDPOINT_URL AWS_ENDPOINT
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

# Login through a browser; never create root access keys.
aws login --profile aws-lab --region ap-southeast-2

# Read-only verification: do not publish account identifiers in docs/logs.
aws sts get-caller-identity --profile aws-lab
aws configure get region --profile aws-lab
aws configure list --profile aws-lab
```

- `--profile aws-lab`: named real-AWS profile, separate from local emulator.
- `--region`: target regional endpoint; many AWS resources are regional.
- `sts get-caller-identity`: shows which identity is authenticated.
- `ExpiredToken`: browser-login credentials may need renewal; rerun `aws login` and check credential source.
- Do **not** commit `~/.aws/config`, `~/.aws/credentials`, or `~/.aws/login/cache`.

## Day 1 — Completed labs

### 1. Amazon S3 — Object storage

**Application use case:** Store employee documents, reports or user uploads independently of the Spring Boot host.

```bash
REGION=ap-southeast-2
PROFILE=aws-lab
BUCKET="aws-lab-$(date +%Y%m%d%H%M%S)"

aws s3 mb "s3://$BUCKET" --profile "$PROFILE" --region "$REGION"
echo 'Hello S3' > aws-test.txt
aws s3 cp aws-test.txt "s3://$BUCKET/aws-test.txt" --profile "$PROFILE" --region "$REGION"
aws s3api head-object --bucket "$BUCKET" --key aws-test.txt --profile "$PROFILE" --region "$REGION"

# CLEANUP
aws s3 rm "s3://$BUCKET/aws-test.txt" --profile "$PROFILE" --region "$REGION"
aws s3 rb "s3://$BUCKET" --profile "$PROFILE" --region "$REGION"
# A 404/NoSuchBucket response from head-bucket after deletion is expected.
aws s3api head-bucket --bucket "$BUCKET" --profile "$PROFILE" --region "$REGION"
rm -f aws-test.txt
```

- `mb` = make bucket; `rb` = remove bucket; `cp` = copy; `rm` = remove; `ls` = list.
- `s3api head-object` retrieves metadata without downloading the object.
- `--endpoint-url http://localhost:4566` is used **only for the local emulator**, never in the real-AWS commands above.
- **Verified:** Object and bucket were removed.

### 2. Amazon SQS — Message queue

**Application use case:** Publish a payment-completed event; a downstream service processes it asynchronously.

```bash
REGION=ap-southeast-2
PROFILE=aws-lab
QUEUE_NAME=aws-lab-payment-events

aws sqs create-queue --queue-name "$QUEUE_NAME" --profile "$PROFILE" --region "$REGION"
QUEUE_URL=$(aws sqs get-queue-url --queue-name "$QUEUE_NAME" --query QueueUrl --output text --profile "$PROFILE" --region "$REGION")

aws sqs send-message --queue-url "$QUEUE_URL" \
  --message-body '{"paymentId":"PAY-DEMO","status":"SUCCESS","amount":500}' \
  --profile "$PROFILE" --region "$REGION"
aws sqs receive-message --queue-url "$QUEUE_URL" \
  --max-number-of-messages 1 --visibility-timeout 60 --wait-time-seconds 2 \
  --profile "$PROFILE" --region "$REGION"

# For a disposable queue, delete the queue and its contents after the lab.
aws sqs delete-queue --queue-url "$QUEUE_URL" --profile "$PROFILE" --region "$REGION"
aws sqs list-queues --queue-name-prefix "$QUEUE_NAME" --profile "$PROFILE" --region "$REGION"
```

- `send-message` publishes; `receive-message` fetches but **does not acknowledge/delete**.
- `ReceiptHandle` is used by `delete-message`; don't confuse it with `MessageId`.
- A message not deleted before its visibility timeout can be redelivered; consumers should be idempotent.
- `--wait-time-seconds` enables long polling (up to the specified seconds).
- **Verified:** Test message was received; queue was deleted; no matching queue listed.

### 3. Amazon DynamoDB — Key/value and document NoSQL

**Application use case:** Fast lookup of payment status by payment ID; design keys around access patterns.

```bash
REGION=ap-southeast-2
PROFILE=aws-lab
TABLE=aws-lab-payments

aws dynamodb create-table --table-name "$TABLE" \
  --attribute-definitions AttributeName=paymentId,AttributeType=S \
  --key-schema AttributeName=paymentId,KeyType=HASH \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
  --table-class STANDARD --profile "$PROFILE" --region "$REGION"

aws dynamodb wait table-exists --table-name "$TABLE" --profile "$PROFILE" --region "$REGION"
aws dynamodb put-item --table-name "$TABLE" \
  --item '{"paymentId":{"S":"PAY-DEMO"},"status":{"S":"SUCCESS"},"amount":{"N":"500"}}' \
  --profile "$PROFILE" --region "$REGION"
aws dynamodb get-item --table-name "$TABLE" \
  --key '{"paymentId":{"S":"PAY-DEMO"}}' --profile "$PROFILE" --region "$REGION"

# CLEANUP
aws dynamodb delete-table --table-name "$TABLE" --profile "$PROFILE" --region "$REGION"
aws dynamodb wait table-not-exists --table-name "$TABLE" --profile "$PROFILE" --region "$REGION"
aws dynamodb list-tables --profile "$PROFILE" --region "$REGION"
```

- `S` = string, `N` = number represented as a JSON string, `HASH` = partition key.
- `PROVISIONED` sets RCU/WCU; evaluate Free-plan allowance before repeating.
- A missing `BillingModeSummary.BillingMode` field in `describe-table` can display as `None`; inspect actual configuration rather than assuming failure.
- **Verified:** Retrieved the sample item; after deletion, table list was empty.

## Day 2 — Completed labs (real AWS)

### 4. EC2 — Provision a Linux VM, inspect network/disk, terminate

**Application use case:** Run a Spring Boot JAR or Docker container on an AWS virtual machine when OS-level control is useful.

**Observed:** Default VPC with three subnets across distinct Availability Zones; no EC2 instances before the lab. Selected a Free Tier-eligible `t3.micro` (x86_64, 1 GiB memory) and Free Tier-eligible Amazon Linux 2023 AMI, using an 8 GiB `gp3` root EBS volume with `DeleteOnTermination=true`. Launched without public IP; private IP assigned; instance reached `running`. **This lab did not include an SSH/SSM login or Java deployment.** Instance was subsequently terminated.

```bash
REGION=ap-southeast-2
PROFILE=aws-lab

# READ ONLY: VPC, subnet, instance type and AMI lookup
aws ec2 describe-vpcs --profile "$PROFILE" --region "$REGION"
aws ec2 describe-subnets --profile "$PROFILE" --region "$REGION"
aws ec2 describe-instance-types --instance-types t3.micro \
  --query 'InstanceTypes[].{Type:InstanceType,FreeTier:FreeTierEligible}' \
  --profile "$PROFILE" --region "$REGION"

AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text \
  --profile "$PROFILE" --region "$REGION")

aws ec2 describe-images --image-ids "$AMI_ID" \
  --query 'Images[].{Name:Name,FreeTier:FreeTierEligible,BlockDevices:BlockDeviceMappings}' \
  --profile "$PROFILE" --region "$REGION"
```

- `AMI` = Amazon Machine Image (template for launching a VM).
- `EBS` = Elastic Block Store (persistent block storage); `gp3` = general-purpose SSD volume.
- `CpuCredits=standard` avoids T3 Unlimited surplus credit charges but may throttle CPU when credits run out.
- `HttpTokens=required` requires IMDSv2 to access instance metadata.
- `PublicIP=None` prevents direct internet access; it also means **there was no remote shell configured for this lab**.

**Resource lifecycle / cleanup commands:** These require an instance ID obtained from *your* account. The following do not launch an instance.

```bash
# Set this from your own run-instances result, not from a public README.
# INSTANCE_ID=i-REPLACE_WITH_YOUR_LAB_INSTANCE_ID

aws ec2 terminate-instances --instance-ids "$INSTANCE_ID" \
  --profile "$PROFILE" --region "$REGION"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID" \
  --profile "$PROFILE" --region "$REGION"

# After termination, inspect EBS volumes/snapshots; inspect only lab resources.
aws ec2 describe-volumes --profile "$PROFILE" --region "$REGION" \
  --query 'Volumes[].{Volume:VolumeId,State:State,Size:Size}' --output table
aws ec2 describe-snapshots --owner-ids self --profile "$PROFILE" --region "$REGION" \
  --query 'Snapshots[].{Snapshot:SnapshotId,State:State}' --output table
```

**Verified:** The EC2 instance was terminated. Its AMI's root-volume mapping specified automatic deletion. **Individual EBS-volume deletion was not explicitly verified in the recorded lab output**; inspect account resources and billing rather than assuming the entire account is empty.

### 5. AWS Budgets — Keep a permanent cost monitor

**Observed:** Created an AWS cost budget named `aws-lab-zero-spend`; the **actual threshold shown in the console was $1.00 per month**, despite its name. Notification email was verified. This budget intentionally remains in place between learning sessions.

- A budget is **an alert, not an automatic hard cap**; billable usage and credit consumption can appear with a delay.
- The AWS Free account plan and its promotional-credit rules are distinct from a traditional AWS Paid plan.
- Additional credit offers may appear in the account's Explore AWS dashboard; do not launch resources only to claim credits.
- The account showed an additional $20 credit after the EC2 lab, but the specific qualifying activity was not independently confirmed in the notes.

### 6. Lambda — Deploy, invoke, inspect logs, delete

**Application use case:** Small event-driven receipt-processing function that does not require an always-on Spring Boot server.

**Observed:** A Python 3.14 Lambda called `aws-lab-payment-receipt`, with 128 MiB memory, 10-second timeout, and a minimal IAM execution role for CloudWatch logging. Invoked with a test payment event; function returned `RECEIPT_GENERATED`. CloudWatch included application logs and `START`/`END`/`REPORT` lines. One invocation reported a roughly 3 ms handler duration and roughly 108 ms billed duration, including initialization (cold start).

**Example source** (`lambda_function.py`):

```python
def lambda_handler(event, context):
    payment_id = event.get("paymentId", "UNKNOWN")
    print(f"Generating receipt for {payment_id}")
    return {"paymentId": payment_id, "result": "RECEIPT_GENERATED"}
```

A Lambda function needs an IAM role with a *trust policy* allowing `lambda.amazonaws.com` to assume it, and appropriate *permissions policies*. For this lab the role was attached to `AWSLambdaBasicExecutionRole` for basic logging. Do not grant broad administrator permissions.

```bash
REGION=ap-southeast-2
PROFILE=aws-lab
FUNCTION=aws-lab-payment-receipt
ROLE=aws-lab-lambda-role

# Deploying an existing function requires a ZIP package and execution-role ARN.
# The local code file and deployment ZIP must not contain secrets.
zip lambda-function.zip lambda_function.py

# Invoke a deployed function; response body goes to the local file.
aws lambda invoke --function-name "$FUNCTION" \
  --payload '{"paymentId":"PAY-DEMO","amount":500}' \
  --cli-binary-format raw-in-base64-out \
  --log-type Tail --profile "$PROFILE" --region "$REGION" response.json
cat response.json

# Inspect logs (read-only)
aws logs tail "/aws/lambda/$FUNCTION" --since 10m \
  --profile "$PROFILE" --region "$REGION"

# CLEANUP, after the lab (irreversible)
aws lambda delete-function --function-name "$FUNCTION" \
  --profile "$PROFILE" --region "$REGION"
aws logs delete-log-group --log-group-name "/aws/lambda/$FUNCTION" \
  --profile "$PROFILE" --region "$REGION"
aws iam detach-role-policy --role-name "$ROLE" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole \
  --profile "$PROFILE"
aws iam delete-role --role-name "$ROLE" --profile "$PROFILE"
```

**Verified cleanup:** `lambda list-functions` was `[]`; matching CloudWatch log groups were `[]`; IAM role lookup returned `NoSuchEntity` as expected. No public Lambda URL or scheduled trigger was created.

### Architecture note — Public traffic to private applications

`Client → public ALB → private ECS/Fargate tasks → data services` is a common pattern. ALB handles HTTP(S) listeners, target groups and health checks; security groups govern traffic; NAT Gateways or VPC endpoints may allow private workloads to reach external or AWS APIs. We deliberately **did not provision ALBs, NAT Gateways, EKS or RDS in Day 2**.

## Java/Spring Boot: Same API, different endpoint

In the existing `aws-learning` Spring Boot lab, the AWS SDK for Java v2 calls S3 and SQS.

| Local emulator | Real AWS |
|---|---|
| `endpointOverride(http://localhost:4566)` | No endpoint override; use AWS regional endpoint |
| Dummy/local credentials profile | Browser login or IAM role credentials |
| Local-only behaviour | Real IAM permissions, network settings, and usage reporting |

**Production rule:** Prefer IAM task/instance roles; never hardcode access keys in Java source, YAML, environment files committed to Git, or a README. Use profile/environment settings for developer machines only.

## IAM and networking cheat sheet

- **IAM principal:** Identity making a request; **policy:** what it is permitted to do; **role:** assumable identity with temporary credentials.
- **EC2:** virtual machine; **AMI:** machine image; **EBS:** attached block storage.
- **VPC:** logically isolated network; **subnet:** network subdivision in one AZ; **route table:** traffic paths.
- **Security group:** stateful network firewall; **ALB:** HTTP(S) request routing; **NAT Gateway:** common outbound IPv4 route for private resources.
- **ECR:** image registry; **ECS:** container orchestration; **Fargate:** managed container capacity; **EKS:** managed Kubernetes control plane; **Lambda:** event-triggered functions.

## Day 3 safe lab guardrails

- Before creating EC2/Lambda/ECR resources, inspect **current account-plan eligibility**, exact region, pricing, and expected credit usage.
- Avoid NAT Gateways, internet-facing ALBs, EKS clusters, and RDS instances during introductory labs unless a specific approved, verified low-cost exercise requires them.
- An **EC2 stop is not full cleanup**: terminate lab instances and inspect attached EBS volumes, snapshots, elastic IPs/public IPv4 usage, security groups and key pairs as applicable.
- Delete Lambda functions, versions/aliases if created, CloudWatch log groups, container images/repositories, and other associated lab resources when appropriate.
- Review the AWS Billing/Free Tier dashboard after each session; cost data may be delayed.
- Don't upgrade from the Free plan or join AWS Organizations just for a lesson.

## Git mini-revision during cloud learning

```bash
git status -sb                 # Brief branch and working-tree status
git diff                      # Review unstaged changes
git diff --cached             # Review staged changes
git add AWS/README.md
git diff --cached -- AWS/README.md
git commit -m "docs(aws): add phase-1 learning notes"
```

Before pushing, check that **only documentation** is staged. `git add AWS/README.md` is safer than `git add .` for this exercise. Use `git fetch` and compare incoming changes before pull during a future Git revision lab.

### Suggested `.gitignore` entries

```gitignore
# Credentials and local environment
.env
.env.*
!.env.example
.aws/
**/.aws/
*.pem
*.key
*.p12
*.pfx
*.tfstate
*.tfstate.*
.terraform/

# Build output / local logs
**/target/
**/build/
*.log
```

> **Check tracked files too:** `.gitignore` does not remove secrets already committed or protect against secrets copied into Markdown. If a credential is ever committed, revoke/rotate it and treat Git history as exposed.

## Pending Day 3 — Brief, application-first

- ECR: push/pull container images; check pricing and repository cleanup before creating.
- ECS / Fargate / EKS: how Spring Boot containers are scheduled and exposed; prefer architecture or read-only inspection if live deployments consume credits.
- RDS / Aurora and ElastiCache: use cases and operational trade-offs; create only if current Free-plan eligibility and deletion costs are verified.
- SNS / EventBridge / Step Functions: messaging patterns and orchestration versus choreography.
- CloudWatch / CloudTrail, Secrets Manager / KMS: observability and security concepts.
- Terraform / CloudFormation: Infrastructure as Code principles; no unreviewed `apply`.
- Close out with a resource inventory and Billing/Free Tier check.

## References

- [AWS Free Tier](https://aws.amazon.com/free/)
- [AWS Free-plan account rules](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html)
- [AWS CLI documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [Amazon S3 documentation](https://docs.aws.amazon.com/s3/)
- [Amazon SQS documentation](https://docs.aws.amazon.com/sqs/)
- [Amazon DynamoDB documentation](https://docs.aws.amazon.com/amazondynamodb/)
- [Amazon EC2 Free Tier eligibility](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html)
