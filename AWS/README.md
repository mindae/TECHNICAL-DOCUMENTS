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
| **2** | EC2, AMI, EBS, VPC, subnets, security groups; Lambda; ALB/NAT conceptual overview | Next |
| **3** | ECR/ECS/Fargate/EKS, RDS/Aurora, CloudWatch/CloudTrail, infrastructure as code; architecture recap | Planned |

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

## Day 2/3 safe lab guardrails

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

## References

- [AWS Free Tier](https://aws.amazon.com/free/)
- [AWS Free-plan account rules](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html)
- [AWS CLI documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [Amazon S3 documentation](https://docs.aws.amazon.com/s3/)
- [Amazon SQS documentation](https://docs.aws.amazon.com/sqs/)
- [Amazon DynamoDB documentation](https://docs.aws.amazon.com/amazondynamodb/)
- [Amazon EC2 Free Tier eligibility](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html)
