# AWS Foundations — First Pass (3-Day Hands-on Lab)

> **Audience:** Java/Spring Boot engineers moving from local Docker/Kubernetes and traditional enterprise infrastructure to AWS.  
> **Approach:** Breadth-first, AWS CLI-first, small real deployments, verify, clean up.  
> **Scope:** Introductory learning record; not a production deployment guide.

## Contents

1. [Learning map](#1-learning-map)
2. [Operating safely](#2-operating-safely)
3. [Day 1 — S3, SQS, DynamoDB](#3-day-1--s3-sqs-dynamodb)
4. [Day 2 — EC2, Lambda, CloudWatch and networking](#4-day-2--ec2-lambda-cloudwatch-and-networking)
5. [Day 3 — ECR and ECS Fargate](#5-day-3--ecr-and-ecs-fargate)
6. [ECS versus EKS](#6-ecs-versus-eks)
7. [RDS and Aurora](#7-rds-and-aurora)
8. [CloudTrail](#8-cloudtrail)
9. [CloudFormation and Terraform](#9-cloudformation-and-terraform)
10. [Troubleshooting log](#10-troubleshooting-log)
11. [Cleanup and billing checklist](#11-cleanup-and-billing-checklist)
12. [Phase 2 roadmap](#12-phase-2-roadmap)

## 1. Learning map

| Day | Services and concepts | What was actually practiced |
| --- | --- | --- |
| 1 | IAM / CLI login, S3, SQS, DynamoDB, AWS Budgets | Real-cloud S3, SQS, DynamoDB create/use/delete; budget configured; local emulator practice |
| 2 | VPC, subnet, security groups, EC2, Lambda, CloudWatch, load-balancer concepts | Launch/terminate a private EC2 instance; create/invoke/delete Lambda and logs/role |
| 3 | ECR, ECS Fargate, EKS comparison, RDS, Aurora concepts, CloudTrail, CloudFormation | Push ARM64 Java image to ECR; run/stop Fargate task; create/delete PostgreSQL RDS; inspect CloudTrail events; deploy/delete CloudFormation stack |

**Important scope boundaries:** We did **not** deploy a production ECS Service, EKS cluster, application load balancer, multi-AZ RDS cluster, or production secrets/network topology. Those are future exercises.

## 2. Operating safely

### Profiles, region and identity

Examples in this document use the configured CLI profile `aws-lab` and Sydney region `ap-southeast-2`. Substitute your own profile and region if different.

```bash
aws --version
aws login --profile aws-lab --region ap-southeast-2
aws sts get-caller-identity --profile aws-lab
```

**Do not paste or commit account IDs, session tokens, access keys, password outputs or unredacted infrastructure dumps.** `sts get-caller-identity` output can include account-identifying information; inspect it locally only.

Always distinguish:

- **AWS real cloud:** `--profile aws-lab --region ap-southeast-2`, without an emulator endpoint.
- **Local emulator (Floci):** Use its own profile and explicit local endpoint. Successful emulator operations do not imply a resource exists on AWS.

### Cost policy

- Use a promotional-credit-backed/free account where available, but assume some resources can incur charges.
- Keep existing budget alerts active; a budget notification is **not** a hard spending cap.
- Avoid leaving EC2, Fargate tasks, RDS databases, NAT Gateways, ALBs, EKS clusters, public IPv4 addresses, or provisioned storage running unnecessarily.
- Confirm deletion with `describe`, `list`, and/or AWS CLI waiters; a successful deletion *request* is not always final deletion.
- Recheck billing later because estimates can lag resource use.
- Never create a production-style public security group for a learning exercise by default.

### Our billing checkpoint

At the conclusion of this pass, the AWS **promotional credits page** showed $180 issued, ~$179.99 estimated remaining and ~$0.01 estimated credits used. These figures are a historical snapshot, **not** current charges or a guarantee about the amount payable. Credits included the initial $100 plus $20 learning rewards for EC2, a budget, Lambda and RDS. Credits and eligibility depend on the account and offer terms.

## 3. Day 1 — S3, SQS, DynamoDB

### S3 — object storage

**Use for:** files, reports, artifacts, static assets, backups. **Not:** a relational database or a POSIX filesystem replacement.

Terms: **bucket** (container), **object** (data + metadata), **key** (object path-like identifier), **region**, **versioning**, **encryption**, **bucket policy**.

Example with a deliberately generic and globally unique bucket name (choose your own):

```bash
BUCKET='replace-with-globally-unique-lab-name'
REGION='ap-southeast-2'
PROFILE='aws-lab'

aws s3api create-bucket \
  --bucket "$BUCKET" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION" \
  --profile "$PROFILE"

printf 'hello from AWS\n' > example.txt
aws s3 cp example.txt "s3://$BUCKET/example.txt" --profile "$PROFILE" --region "$REGION"
aws s3api head-object --bucket "$BUCKET" --key example.txt \
  --profile "$PROFILE" --region "$REGION"
aws s3 rm "s3://$BUCKET/example.txt" --profile "$PROFILE" --region "$REGION"
aws s3 rb "s3://$BUCKET" --profile "$PROFILE" --region "$REGION"
```

We also tested S3 access from Java AWS SDK v2 via a Spring Boot wrapper against the local emulator.

### SQS — reliable asynchronous messaging

**Use for:** decoupling producers and consumers, buffering bursts and retrying work. Standard queues normally provide **at-least-once delivery**: consumers must be idempotent. Visibility timeout hides a received message temporarily; `receive-message` does **not** remove it. Delete it after processing.

```bash
aws sqs create-queue --queue-name aws-lab-payment-events \
  --profile aws-lab --region ap-southeast-2

QUEUE_URL=$(aws sqs get-queue-url --queue-name aws-lab-payment-events \
  --query QueueUrl --output text --profile aws-lab --region ap-southeast-2)

aws sqs send-message --queue-url "$QUEUE_URL" \
  --message-body '{"paymentId":"PAY301","status":"CREATED"}' \
  --profile aws-lab --region ap-southeast-2

aws sqs receive-message --queue-url "$QUEUE_URL" \
  --wait-time-seconds 5 --visibility-timeout 30 \
  --profile aws-lab --region ap-southeast-2

# After successful processing, delete using the receipt handle returned by receive-message:
# aws sqs delete-message --queue-url "$QUEUE_URL" --receipt-handle "$HANDLE" ...

aws sqs delete-queue --queue-url "$QUEUE_URL" \
  --profile aws-lab --region ap-southeast-2
```

We practiced receive/delete and visibility-timeout behaviour locally and on AWS. In Java, prefer producer/consumer separation and idempotent processing.

### DynamoDB — managed NoSQL key/value and document database

**Use for:** access patterns built around partition keys and optional sort keys; low-latency managed key-based access. Design the data model around queries, unlike a normalized relational schema.

Our lab used a small provisioned-capacity table and a sample payment record, then removed the table:

```bash
aws dynamodb create-table \
  --table-name aws-lab-payments \
  --attribute-definitions AttributeName=paymentId,AttributeType=S \
  --key-schema AttributeName=paymentId,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
  --profile aws-lab --region ap-southeast-2

aws dynamodb wait table-exists --table-name aws-lab-payments \
  --profile aws-lab --region ap-southeast-2

aws dynamodb put-item \
  --table-name aws-lab-payments \
  --item '{"paymentId":{"S":"PAY301"},"status":{"S":"CREATED"}}' \
  --profile aws-lab --region ap-southeast-2

aws dynamodb get-item \
  --table-name aws-lab-payments \
  --key '{"paymentId":{"S":"PAY301"}}' \
  --profile aws-lab --region ap-southeast-2

aws dynamodb delete-table --table-name aws-lab-payments \
  --profile aws-lab --region ap-southeast-2
aws dynamodb wait table-not-exists --table-name aws-lab-payments \
  --profile aws-lab --region ap-southeast-2
```

## 4. Day 2 — EC2, Lambda, CloudWatch and networking

### VPC fundamentals

- **VPC:** isolated logical AWS network.
- **Subnet:** address range within a VPC and one availability zone.
- **Route table:** decides where network packets go.
- **Internet Gateway:** routing option for internet-facing VPC traffic.
- **Security group:** stateful firewall applied to resources.
- **NACL:** stateless subnet-level traffic filtering.
- **Public subnet** means its route table can route to an internet gateway. A resource still needs suitable IP addressing and security rules.
- **Private subnet** usually has no direct inbound internet route; outbound access may use NAT or private VPC endpoints.

We inspected the default VPC/subnets/security groups and deliberately avoided publishing services to the internet.

### EC2

We launched an Amazon Linux 2023 **x86_64 `t3.micro`** instance using an official AMI, IMDSv2, a small gp3 root volume and **no public IPv4**. We inspected its running state, then terminated the instance. The root-volume deletion setting was enabled, although independent EBS-volume verification was not captured in the earlier notes.

Useful read-only checks:

```bash
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=pending,running,stopping,stopped \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,Type:InstanceType}' \
  --profile aws-lab --region ap-southeast-2

aws ec2 describe-volumes \
  --query 'Volumes[].{ID:VolumeId,State:State,Size:Size,Attachments:Attachments[].InstanceId}' \
  --profile aws-lab --region ap-southeast-2
```

**Architecture:** An EC2 instance is a machine you manage. ECS/Fargate and Lambda shift parts of that operational responsibility away from you.

### Lambda

We created an IAM role with `AWSLambdaBasicExecutionRole`, deployed a small Python handler, invoked it successfully, inspected its CloudWatch START/END/REPORT logs, then removed the function, log group and IAM role.

Key terms: **handler**, **execution role**, **timeout**, **memory**, **cold start / init duration**, **invocation**, **CloudWatch log group**.

```bash
aws lambda list-functions \
  --query 'Functions[].{Name:FunctionName,Runtime:Runtime}' \
  --profile aws-lab --region ap-southeast-2
```

**Compare:** Lambda runs event-triggered functions; Fargate runs container tasks/services; EC2 exposes server-level control.

### Application Load Balancer (ALB) — concept only

A common production path is `Client -> ALB listener -> Target group -> ECS tasks (private subnets)`. The ALB security group controls inbound user traffic; the ECS task group should allow only the traffic needed from the ALB. We did not deploy an ALB in the first pass.

## 5. Day 3 — ECR and ECS Fargate

### ECR — a container image registry

**Docker Hub and Amazon ECR are registries.** ECS can pull images from either. ECR integrates with IAM and AWS deployment workflows.

We selected a local Spring Boot `employee-svc:1.0` image built for `linux/arm64`, with `java -jar app.jar` and port `8081/tcp`.

```bash
# Examples assume you've authenticated with AWS CLI and Docker is running.
aws ecr create-repository \
  --repository-name aws-lab-employee-svc \
  --image-tag-mutability IMMUTABLE \
  --profile aws-lab --region ap-southeast-2

REPO_URI=$(aws ecr describe-repositories \
  --repository-names aws-lab-employee-svc \
  --query 'repositories[0].repositoryUri' --output text \
  --profile aws-lab --region ap-southeast-2)

aws ecr get-login-password --profile aws-lab --region ap-southeast-2 \
  | docker login --username AWS --password-stdin "${REPO_URI%/*}"

docker image inspect employee-svc:1.0 \
  --format '{{.Os}}/{{.Architecture}} {{json .Config.Entrypoint}} {{json .Config.ExposedPorts}}'

docker tag employee-svc:1.0 "$REPO_URI:1.0"
docker push "$REPO_URI:1.0"

aws ecr describe-images --repository-name aws-lab-employee-svc \
  --query 'imageDetails[].{Tags:imageTags,Digest:imageDigest,Size:imageSizeInBytes}' \
  --profile aws-lab --region ap-southeast-2

# Only after downstream workloads have been removed:
aws ecr delete-repository --repository-name aws-lab-employee-svc --force \
  --profile aws-lab --region ap-southeast-2
```

**Tag vs digest:** `1.0` is a human-readable version tag; `sha256:...` identifies a particular image manifest. OCI indexes, platform manifests and provenance metadata may appear as multiple ECR entries, some untagged. Do not assume three entries mean three distinct application builds.

**Shell detail:** In Zsh, `${REPO_URI%/*}` removes the repository path, leaving the registry host for `docker login`.

### ECS entities

| Resource | Purpose |
| --- | --- |
| Cluster | Logical management boundary |
| Task Definition | Versioned blueprint: image, CPU, RAM, port, IAM, logging, runtime platform |
| Task | One running/stopped instance of a task definition |
| Service | Maintains a desired task count and manages service deployments |
| Execution role | ECS infrastructure permissions to pull ECR images and publish logs |
| Task role | Permissions the *application* uses to call AWS APIs |

Our successful one-off deployment used:

- ECS cluster `aws-lab-cluster`.
- Fargate task, `0.5 vCPU` / `1 GiB`, Linux ARM64.
- Port `8081`; `awsvpc` networking.
- A dedicated security group with **no inbound rules**.
- A temporary public IP in an internet-routed default subnet for outbound ECR/logging access; **no exposed public API**.
- CloudWatch `awslogs` driver and an IAM task execution role.

Illustrative task-definition excerpt:

```json
{
  "family": "aws-lab-employee-svc",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "runtimePlatform": {
    "operatingSystemFamily": "LINUX",
    "cpuArchitecture": "ARM64"
  },
  "executionRoleArn": "REPLACE_WITH_EXECUTION_ROLE_ARN",
  "containerDefinitions": [
    {
      "name": "employee-svc",
      "image": "REPLACE_WITH_ECR_IMAGE_URI:1.0",
      "essential": true,
      "portMappings": [{"containerPort": 8081, "protocol": "tcp"}]
    }
  ]
}
```

The excerpt omits log configuration and should be adapted before registering as a complete task definition.

**Lifecycle:** `register-task-definition -> create-cluster -> run-task -> describe-tasks -> logs tail -> stop-task -> wait tasks-stopped -> cleanup`. Our container reached `RUNNING`; Tomcat was initialized on 8081. The provided log excerpt did **not** establish a final `Started ...` application-ready message. ECS reported `healthStatus: UNKNOWN` because no health check was configured. `RUNNING` is **not** synonymous with application readiness.

Cleanup was verified: task STOPPED, Task Definition INACTIVE, cluster deleted, dedicated SG deleted, CloudWatch log group deleted, IAM execution role deleted, ECR repository deleted; four corresponding resource-list checks returned empty arrays.

### Production improvements reserved for Phase 2

ECS Service, ALB target group + listener, readiness/health checks, private subnets and VPC endpoints or justified NAT, auto scaling, deployment rollback, application task role, secret delivery, monitoring/alarms and security policies.

## 6. ECS versus EKS

**ECS/EKS are orchestrators; Fargate/EC2 are compute choices.**

| ECS | EKS |
| --- | --- |
| AWS-native task/service model | Managed Kubernetes control plane |
| Fewer platform components to operate | Familiar Kubernetes API, Deployments, Services, Helm, operators |
| Good default when fully AWS-based and no Kubernetes standard | Useful when the organization requires Kubernetes tooling or a common K8s platform |
| Can run on Fargate or EC2 | Can run supported workloads on Fargate, EC2 or EKS Auto Mode |

Mental mappings (not exact equivalence): ECS task ≈ Kubernetes Pod; ECS task definition ≈ Pod specification; ECS Service spans responsibilities often split between Kubernetes Deployment and Service.

IAM choices differ: ECS **task role** is for app permissions; EKS uses mechanisms such as **Pod Identity** or **IRSA**, depending on compute/setup. Compute compatibility also differs: check ARM64 support for the chosen EKS execution model before reusing our ARM64 image. We did not create a billable EKS control plane in Phase 1.

## 7. RDS and Aurora

### Managed relational databases

**RDS** manages database infrastructure, engine installation and many backup/patch/failover operations. The application/DBA teams still own schemas, access control, query tuning, transaction behaviour, sizing and recovery objectives.

**Aurora** is an AWS-designed MySQL/PostgreSQL-compatible engine family with distributed storage; it is not automatically a better choice for every workload and is not an Oracle-compatible engine.

**Single-AZ vs Multi-AZ** concerns availability. **Backups/snapshots** concern recovery. Neither substitutes for the other. Traditional RDS Multi-AZ instances have standby infrastructure; some newer Multi-AZ cluster architectures also have readable standby instances.

### Our PostgreSQL deployment

We verified that PostgreSQL 17.11 on `db.t4g.micro` with gp3 was orderable in the selected region. We created a DB subnet group covering two availability zones and a dedicated security group with no inbound rules. We then provisioned:

- RDS PostgreSQL 17.11; instance `db.t4g.micro`; 20 GiB gp3.
- Database name `employeedb`.
- Single-AZ and **not publicly accessible**.
- AWS-managed master password in Secrets Manager; no password in CLI history.
- Automated backup retention `0` and deletion protection disabled **for the disposable lab only**.

The instance reached `available`, its private endpoint was issued on port `5432`, and managed-secret status became `active`. We did not open DB ingress or run SQL from the Mac, because a private RDS endpoint is not reachable directly without a suitable network path.

**Three separate connection gates:** network connectivity/security-group authorization; database authentication; SQL object permissions (authorization). For a future Spring Boot JDBC connection:

```properties
spring.datasource.url=jdbc:postgresql://REPLACE_WITH_PRIVATE_DB_ENDPOINT:5432/employeedb
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

In production, avoid using the master login as the application's everyday DB user.

**Deletion:** `delete-db-instance --skip-final-snapshot --delete-automated-backups`, then `aws rds wait db-instance-deleted`. We verified no DB instances, DB snapshots, retained automated backups, or listed Secrets Manager entries, and removed the lab DB subnet group and SG. **Never blindly skip the final snapshot for production data.**

## 8. CloudTrail

**CloudWatch:** application logs, metrics, alarms.  
**CloudTrail:** AWS API activity: who requested an operation, when, through which identity/API; event history typically includes 90 days of management events without creating a trail.

We inspected the real `CreateDBInstance` and `DeleteDBInstance` events for our lab. CloudTrail recorded `userIdentity.type = AssumedRole`, `eventSource = rds.amazonaws.com`, AWS CLI as the user agent, and `skipFinalSnapshot = true`. `errorCode = null` meant no API error was recorded; the **RDS deletion waiter and empty DB list** proved the asynchronous deletion finished.

Read-only event summary:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteDBInstance \
  --max-items 10 \
  --query 'Events[].{Time:EventTime,Action:EventName,User:Username}' \
  --profile aws-lab --region ap-southeast-2
```

Safe filtered detail (requires `jq`):

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteDBInstance \
  --max-items 10 \
  --query 'Events[].CloudTrailEvent' --output json \
  --profile aws-lab --region ap-southeast-2 \
| jq -r '.[] | fromjson
  | select(.requestParameters.dBInstanceIdentifier == "aws-lab-postgres")
  | {eventTime,eventSource,eventName,identityType:.userIdentity.type,
     database:.requestParameters.dBInstanceIdentifier,
     skipFinalSnapshot:.requestParameters.skipFinalSnapshot,errorCode}'
```

Avoid publishing raw audit events; they may expose identity and environment metadata.

## 9. CloudFormation and Terraform

**Infrastructure as Code (IaC)** defines desired resources in source files instead of relying on a sequence of manual console operations.

| CloudFormation | Terraform |
| --- | --- |
| AWS-native YAML/JSON templates | HCL and providers |
| Stack is AWS-managed | Terraform state tracks managed resources |
| Change Sets preview changes | `terraform plan` previews changes |
| Stack create/update/delete | `init`, `plan`, `apply`, `destroy` |

### Our first CloudFormation stack

Template, safe for a public learning repository:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Temporary AWS learning lab
Resources:
  LearningLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws-lab/cloudformation-demo
      RetentionInDays: 1
```

```bash
aws cloudformation validate-template \
  --template-body file://aws-lab-cloudformation.yaml \
  --profile aws-lab --region ap-southeast-2

aws cloudformation create-stack \
  --stack-name aws-lab-cfn-stack \
  --template-body file://aws-lab-cloudformation.yaml \
  --profile aws-lab --region ap-southeast-2

aws cloudformation wait stack-create-complete \
  --stack-name aws-lab-cfn-stack \
  --profile aws-lab --region ap-southeast-2

aws cloudformation list-stack-resources --stack-name aws-lab-cfn-stack \
  --profile aws-lab --region ap-southeast-2

aws cloudformation delete-stack --stack-name aws-lab-cfn-stack \
  --profile aws-lab --region ap-southeast-2

aws cloudformation wait stack-delete-complete \
  --stack-name aws-lab-cfn-stack \
  --profile aws-lab --region ap-southeast-2
```

We observed `CREATE_COMPLETE` for the stack and managed `AWS::Logs::LogGroup`, then confirmed the stack's log group was gone after deletion. `describe-stacks --stack-name ...` subsequently returned “stack does not exist,” which is compatible with successful deletion; deleted-stack history may require the original stack ID.

**CloudFormation logical ID vs physical ID:** `LearningLogGroup` refers to the YAML resource, while `/aws-lab/cloudformation-demo` is the concrete AWS log group.

**Terraform caution:** never commit `terraform.tfstate`, `.terraform/`, secret `.tfvars` or credentials. The state may contain sensitive data even when the `.tf` file looks harmless.

## 10. Troubleshooting log

| Symptom | Cause / insight | Resolution |
| --- | --- | --- |
| `InvalidParameterValue: Input can't contain control characters` for RDS subnet group | AWS CLI `--output text` placed tab-separated subnet IDs in a scalar; macOS Zsh does not split scalars like Bash by default | Pass subnet IDs as separate arguments or a Zsh/Bash array: `--subnet-ids "${SUBNET_IDS[@]}"` |
| CloudFormation `TemplateBody, value: 0` | YAML file had been truncated to zero bytes | `wc -c`, inspect file, restore here-document, validate again |
| `docker images ls --filter references=...` invalid | Incorrect command form and filter key | Use `docker image ls --filter 'reference=*aws-lab-employee-svc*'` |
| ECR list showed additional untagged manifest records | OCI image/index/attestation metadata may produce multiple records | Inspect image manifest types; don't assume separate application versions |
| ECS `RUNNING` with health `UNKNOWN` | No container health check configured | Configure health check/readiness in later production-oriented lab |
| `describe-stacks` reports stack not found after deletion | Stack was removed; name-based lookup may not retrieve history | Verify waiter and managed resources, or use historical stack ID if available |
| RDS API delete event had no error | API request was accepted, but deletion remained asynchronous | Use `aws rds wait db-instance-deleted` plus list verification |

### Shell technique: safe here-documents

```bash
cat > aws-lab-cloudformation.yaml <<'EOF'
# Paste YAML here
EOF

wc -c aws-lab-cloudformation.yaml
```

A bare `>` truncates an existing file before writing. `cat file` only reads it. Keep the terminator `EOF` alone on a line.

### Git hygiene

```bash
git status -sb
git diff -- README.md
# git diff doesn't display untracked files until they have been staged.
git add README.md
git diff --cached -- README.md
```

Review changes and redact environment-specific details before committing to a **public** repository.

## 11. Cleanup and billing checklist

**What was verified by the end of the first pass:**

- [x] Day 1 test S3 bucket/object, SQS queue and DynamoDB table deleted.
- [x] EC2 lab instance terminated. **Additional independent root-EBS-volume inventory remains a sensible account-wide safety check.**
- [x] Lambda function, log group and IAM role removed.
- [x] ECS Fargate task `STOPPED`; cluster, execution role, security group, log group and ECR repository deleted; list checks empty.
- [x] RDS instance deleted; snapshots and retained automated backups absent; DB subnet group and SG removed; Secrets Manager listing empty.
- [x] CloudFormation stack and managed log group deleted.
- [x] AWS promotional credit snapshot reviewed; budget alert retained.

**Read-only periodic inventory:**

```bash
aws ec2 describe-instances --filters Name=instance-state-name,Values=pending,running,stopping,stopped \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}' \
  --profile aws-lab --region ap-southeast-2
aws ec2 describe-volumes --query 'Volumes[].{ID:VolumeId,State:State,Size:Size}' \
  --profile aws-lab --region ap-southeast-2
aws ecs list-clusters --profile aws-lab --region ap-southeast-2
aws rds describe-db-instances --query 'DBInstances[].DBInstanceIdentifier' \
  --profile aws-lab --region ap-southeast-2
aws ecr describe-repositories --query 'repositories[].repositoryName' \
  --profile aws-lab --region ap-southeast-2
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query 'StackSummaries[].StackName' \
  --profile aws-lab --region ap-southeast-2
```

**Limitations:** Regional checks don't prove that no resources exist in every AWS region or service. Billing and cost reports can lag. On some account types, credits and available services have special restrictions. Use the AWS console's cost and resource inventory views as a secondary cross-check.

## 12. Phase 2 roadmap

**Keep Phase 1 and Phase 2 separate.** The goal now is confidence with service selection and operational vocabulary; next comes real application integration.

1. **VPC and IAM deeper dive:** private/public subnets, routing, NAT vs VPC endpoints, least privilege, IAM role assumption, security-group-to-security-group permissions, SSM connectivity.
2. **Production-style ECS:** deploy Spring Boot service via ECS Service, ALB and target group; health checks, rolling updates, task and execution roles, Secrets Manager, autoscaling and alarms. Compare ARM64 and x86_64 considerations.
3. **Java SDK / Spring integration:** S3 uploads, SQS message producer/consumer, DynamoDB access; credentials provider chain, retries, timeouts, idempotency and local testing.
4. **Managed data access:** private RDS access from ECS, app-specific DB users, TLS, HikariCP settings, migrations and RDS backup/failover concepts.
5. **IaC and delivery:** CloudFormation change sets, Terraform state/plan/apply/destroy, CI/CD to ECR/ECS, configuration and drift.
6. **Observability and operations:** CloudWatch metrics/alarms, structured logs, CloudTrail investigation and cost attribution.
7. **EKS only when useful:** managed nodes or Auto Mode, Kubernetes IAM identity, networking, Ingress/load balancers and Kubernetes operations on AWS.

### Guiding architecture questions

For every service, ask: **Who invokes it? Who owns its data? What permissions does it need? How is it reached? How does it fail? How is it observed? What does it cost when idle? How do we delete it safely?**

---

*Learning notes prepared from completed hands-on exercises. Examples deliberately omit account numbers, actual ARNs, resource IDs, endpoint hosts, secrets and personal paths. Revalidate versions, prices and service behaviour against current AWS documentation before production use.*
