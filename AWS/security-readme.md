# AWS Security Fundamentals — IAM, Roles, Policies, and Security Groups

> **Audience:** Java/Spring Boot developers and architects learning AWS.  
> **Purpose:** A standalone, offline reference you can read at work without ChatGPT.  
> **Scope:** Fundamentals, practical microservice examples, troubleshooting, and safe exercises.  
> **Safety:** All names and ARNs below are illustrative. Never commit access keys, passwords, session tokens, or downloaded secret values to Git.

## 1. The single most important distinction

AWS security becomes much simpler when you separate **AWS API authorization** from **network access control**.

| Question | AWS mechanism | Example |
|---|---|---|
| **Who can call an AWS API, on which resource?** | IAM identities and policies; sometimes resource-based policies | May `employee-svc` call `s3:GetObject`? |
| **Who may assume a role?** | Role trust policy, along with other applicable authorization controls | May ECS tasks assume an execution role? |
| **Which network packets can flow?** | Security groups, routing, network ACLs, etc. | Can `employee-svc` reach RDS on TCP 5432? |
| **Once connected, what may the app do?** | Application/database authorization | May PostgreSQL user execute `SELECT`? |

**Critical:** An IAM policy is not a firewall. A security group does not grant IAM permissions. An open PostgreSQL port does not provide a database username or table privileges.

### Familiar analogies (useful, but not exact)

- **Linux:** identity + group/file permissions resembles the general notion of principals and access rights.
- **Spring Security:** authenticated principal + authorities resembles an authorization decision.
- **AWS role is different from `ROLE_ADMIN`:** an IAM role is an **assumable AWS identity** that yields temporary credentials; it is not merely a string authority assigned to a user.

## 2. IAM vocabulary: identities, roles, policies

### IAM user

An IAM user is a long-lived identity within an AWS account. It may have console credentials and/or access keys if configured. For human workforce access, prefer federation / IAM Identity Center and short-lived credentials where supported, rather than distributing long-lived IAM user keys.

### IAM policy

A policy is a JSON permissions document. Common elements:

| Field | Meaning |
|---|---|
| `Effect` | `Allow` or `Deny` |
| `Action` | API actions, e.g. `s3:GetObject` |
| `Resource` | Resources the statement applies to (often ARN syntax) |
| `Condition` | Optional constraints, such as source context or tags |
| `Principal` | Used in resource-based and role trust policies to identify who the statement applies to; typically **not** present in identity-based permissions policies |

**Example: read objects from one specific S3 bucket**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadEmployeeDocuments",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example-employee-documents/*"
    }
  ]
}
```

This allows retrieving objects within the example bucket. **It does not** grant `s3:PutObject`, `s3:DeleteObject`, or bucket listing. To list objects, you'd generally add `s3:ListBucket` on `arn:aws:s3:::example-employee-documents` (without `/*`). The bucket name is a placeholder, not one we created.

#### AWS-managed, customer-managed, inline

- **AWS-managed policy:** Created and maintained by AWS; reusable, e.g. `AmazonECSTaskExecutionRolePolicy`.
- **Customer-managed policy:** Created and maintained in your AWS account; reusable across suitable principals.
- **Inline policy:** Embedded into exactly one IAM user, group, or role; not independently reusable.

One role may have several permissions policies, and one managed policy may be attached to multiple IAM identities. **Sharing a policy does not mean sharing a role or credentials.**

### IAM role

A role is an IAM identity that a trusted caller can **assume**. The caller obtains temporary credentials for a role session. A role commonly has:

1. A **trust policy** — who may assume it.
2. **Permissions policies** — what the assumed role may do.

**Trust policy example: ECS tasks may assume this role**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ecs-tasks.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

This policy **does not** grant S3/ECR access itself. Attaching an S3 policy to the role **does not** make the role assumable by everyone. Cross-account and human role assumption can also require permission to call `sts:AssumeRole` and satisfy other conditions.

**Mental model:** Trust policy = *who can become this identity?* Permissions policy = *what can this identity do?*

## 3. What we actually did in our ECS lab

Our temporary Spring Boot image was stored in ECR and run as an ECS Fargate task.

```text
Developer's AWS identity
    |
    | permission to register task / run task
    v
ECS service
    |
    | assumes TASK EXECUTION ROLE
    |--> pull private image from ECR
    |--> write container logs to CloudWatch Logs
    |
    v
Fargate task launches Java container
    |
    | optionally uses a distinct TASK ROLE
    |--> Java AWS SDK calls S3 / SQS / DynamoDB etc.
    v
Your Spring Boot code
```

### Execution role vs task role — do not mix them up

| Role | Used by | Typical permissions |
|---|---|---|
| **Task execution role** | ECS/Fargate infrastructure | ECR pull, CloudWatch log publishing, certain secret/config retrieval during task initialization |
| **Task role** | Application processes in the running container | S3 read/write, SQS send/receive, DynamoDB calls, etc. |

We created an ECS **execution role**, trusted `ecs-tasks.amazonaws.com`, and attached the AWS-managed `AmazonECSTaskExecutionRolePolicy`. We did **not** need a task role for that introductory deployment. Both roles can have the same service trust principal but are attached to different ECS task-definition fields (`executionRoleArn` vs `taskRoleArn`).

### Java SDK example — no hardcoded access keys

Suppose we give `employee-svc` a task role with the illustrative S3 read policy above:

```java
try (S3Client s3 = S3Client.create()) {
    s3.getObject(
        GetObjectRequest.builder()
            .bucket("example-employee-documents")
            .key("employee-101.pdf")
            .build(),
        Paths.get("/tmp/employee-101.pdf")
    );
}
```

*Illustrative AWS SDK for Java v2 snippet; imports and error handling omitted.* The default credential provider chain can obtain ECS task-role credentials automatically. **Do not place AWS access keys in Spring `application.properties` or source code.**

An ECS execution role that can pull from ECR does **not** automatically permit your Java application to read S3 objects.

## 4. Security groups: AWS network firewalls

A **VPC security group** governs traffic for network interfaces of associated resources (e.g. EC2 instance, Fargate task ENI, RDS DB instance). Rules describe protocol, port/range, and allowed source/destination. Security groups are **stateful**: allowed connection responses are tracked automatically.

### Our RDS lab

We created a dedicated security group with:

```json
{
  "Inbound": [],
  "Outbound": [
    { "Protocol": "all", "Destination": "0.0.0.0/0" }
  ]
}
```

This is a human-friendly conceptual representation, **not** the exact AWS API schema. `Inbound: []` meant no new inbound connections were permitted by that security group. Outbound allow-all **did not** make the database reachable from the internet. We also set RDS to **not publicly accessible**.

### Better enterprise pattern: security group referencing security group

```text
[ECS employee-svc]  security group: employee-sg
         |
         | TCP 5432
         v
[RDS PostgreSQL]   security group: database-sg

RDS inbound rule:
    Type: PostgreSQL / TCP
    Port: 5432
    Source: employee-sg    (SG reference, not a public CIDR)
```

This authorizes traffic from instances/interfaces associated with the allowed source group, subject to VPC connectivity, routing and other network controls. It does **not** grant PostgreSQL credentials or query privileges.

A security group is **not attached to an IAM user, role, or permissions policy**. It is associated with VPC resources/interfaces. One SG can be referenced in rules of another SG; the reference is a **network rule**, not IAM role assumption.

### Network troubleshooting, in order

1. Is the application using the correct hostname and port?
2. Can the client route to that destination (VPC/subnets/routes/VPN/endpoints)?
3. Does the destination SG allow inbound from the client's SG or appropriate address?
4. Does the client's SG permit the necessary outbound connection?
5. Are NACLs, DNS and other network controls compatible?
6. Once TCP connects, do PostgreSQL authentication and database privileges allow the request?

**Note:** Security-group rules allow traffic, not AWS API calls. Conversely, IAM authorization does not open TCP connections.

## 5. Identity policies vs resource-based policies

Not every AWS permissions policy attaches to an IAM identity.

- **Identity-based policy:** attached to a user, group or role; e.g. task role may call `s3:GetObject` on specified objects.
- **Resource-based policy:** attached to a resource, e.g. S3 bucket policy or SQS queue policy; declares principals and permitted actions.
- **Trust policy:** a special resource-based policy on an IAM role governing who may assume it.

Both types can affect the final authorization result. Other possible controls include service control policies (SCPs), permissions boundaries, session policies, resource control policies (RCPs), VPC endpoint policies and service-specific restrictions. Policy evaluation can differ between same-account and cross-account requests.

**Rules of thumb:**

- No applicable `Allow` usually means implicit denial.
- An applicable explicit `Deny` overrides an `Allow`.
- Cross-account access usually requires authorization on both sides, subject to the exact policy evaluation rules.
- Avoid believing "role has Allow" implies success without checking relevant resource/organization controls.

### S3 example: permissions vs network

An ECS task can successfully contact S3 over HTTPS yet receive HTTP `403 AccessDenied`: first investigate the application's task role and applicable S3/bucket/organization policies, not the PostgreSQL SG.

For private connectivity to S3, VPC endpoints and their endpoint policies can become relevant in enterprise designs. This is separate from opening application container port 8081.

## 6. Full architecture: what protects what?

```text
                         Human developer
                               |
                   SSO / assumed AWS role
                               |
            IAM allows AWS API deployment actions
                               |
                               v
             ECS task definition + ECS cluster
                  /                      \
        Execution role                 Task role
        ECR + log access              app's AWS API access
                  \                      /
                   [employee-svc Fargate]
                    Security Group: A
                         |          \
                 TCP 5432 |           \ HTTPS API
                         v            v
                    [RDS]            [S3]
                  Security Group B    IAM/resource policies
                  PG credentials
                  PG SQL privileges
```

For a successful RDS SQL request, **network access + DB authentication + DB authorization** are essential. For an S3 SDK request, **AWS identity + IAM/resource-policy authorization** are essential, and network connectivity to the S3 endpoint must also exist.

## 7. Common failures and the first place to look

| Symptom | First investigation | Don't immediately change |
|---|---|---|
| ECS cannot pull ECR image with `AccessDenied` | Execution role policy and ECR registry/repository authorization | App task role or RDS SG |
| ECS image pull times out | Outbound network route, DNS, ECR access path/endpoints | IAM alone |
| Java `s3:GetObject` gets `AccessDenied` | Task role and applicable S3/resource/org policies | Container port mapping |
| Java cannot connect to RDS TCP 5432 | Route, SG inbound/outbound, DNS, NACL | S3 policy |
| TCP connects, PostgreSQL says password invalid | Database credentials/auth method | Security group |
| SQL connects but `permission denied for table` | Database grants/roles | IAM |
| Caller cannot assume role | Role trust policy and applicable `sts:AssumeRole` controls | Security group |
| CloudTrail shows `AssumedRole` | Inspect `userIdentity`, session context and session issuer | Assume it was a permanent IAM user |

### CloudTrail connection to our real labs

We looked up `CreateDBInstance` and `DeleteDBInstance` events and saw `identityType: AssumedRole` and `userAgent: aws-cli/...`. This illustrated:

- **CloudTrail:** who/which AWS principal requested which **AWS API operation**, and when.
- **CloudWatch:** application/system logs and operational metrics, such as Java startup logs.
- A successful API request is not necessarily the same as completion of an asynchronous workflow: we separately used an RDS deletion waiter to confirm the database was gone.

Do not publish raw CloudTrail identity/session details, account IDs, session tokens or full account-specific ARNs in a public learning repository.

## 8. Safe, practical exercises — first UI pass

> **DO:** Explore read-only AWS Console pages and answer questions.  
> **DO NOT:** Attach policies, grant broad permissions, open SG ports, or deploy billable resources.  
> **STOP AT:** Understanding which control serves which purpose before changing anything.

### Exercise A — IAM role anatomy

In the AWS Console go to **IAM → Roles**. Inspect a suitable existing role, preferably a service-linked or non-production learning role. Look for:

1. **Trust relationships:** Which `Principal`/service can assume it? Which STS action is permitted?
2. **Permissions:** Which AWS-managed or customer-managed policies are attached?
3. **Policy JSON:** Which actions/resources are allowed or denied?

Write in your own words: *Who can assume this role? What AWS actions does the resulting session get?*

**Caution:** AWS-managed service-linked roles may have special restrictions; do not edit them just for practice.

### Exercise B — VPC security group anatomy

In **EC2 → Security Groups**, inspect a non-production group's inbound and outbound rules. Identify:

- Which VPC it belongs to.
- Whether an inbound rule uses a CIDR like `0.0.0.0/0` or references another security group.
- Protocol and port range.
- Whether it is associated with a resource/network interface.

Question: *Would attaching an S3 read IAM policy open TCP port 5432?* **No.**

### Exercise C — Decide which access to grant (paper-only)

For an ECS application that must **receive from an SQS queue** and **write to PostgreSQL**:

1. Its **task role** needs relevant SQS API permissions, scoped to the required queue (e.g. `sqs:ReceiveMessage`, `sqs:DeleteMessage`, often `sqs:GetQueueAttributes`).
2. ECS infrastructure's **execution role** needs its own ECR/log permissions.
3. Its ECS **security group** requires appropriate outbound network access to the database.
4. The RDS **security group** permits inbound TCP 5432 from the application's SG.
5. PostgreSQL must authorize the application's database identity.

No single IAM policy replaces all five controls.

## 9. Self-check quiz — answer first, then reveal

1. An ECS task cannot pull a private ECR image because its **execution role lacks permission**. Fix IAM or the RDS SG?
2. `employee-svc` cannot reach RDS because RDS's SG has **no inbound rule**. Change IAM trust policy or SG ingress?
3. Java reaches S3 but gets `AccessDenied` for `GetObject`. Look at application task role/bucket policy or container port 8081?
4. A role allows `s3:GetObject`, but someone cannot assume it. Which policy controls **who may assume** the role?
5. Can a single customer-managed IAM policy be attached to multiple roles?
6. Is an IAM role exactly equivalent to a Spring Security `ROLE_ADMIN` authority?
7. If RDS SG permits TCP 5432, can an unauthenticated client automatically query tables?
8. What is the difference between ECS **execution role** and **task role**?

<details>
<summary>Answer key (open after attempting)</summary>

1. **IAM execution-role permissions**; check ECR authorization as applicable. Also check network if the symptom is a timeout instead.
2. **RDS SG inbound** TCP 5432 from the appropriate ECS security group; verify routes too.
3. **Task role, resource policies, and other applicable AWS permission controls**.
4. **Role trust policy**, plus other applicable authorization controls for assumption.
5. **Yes.** A reusable managed policy can be attached to multiple identities.
6. **No.** An IAM role is an assumable identity providing temporary credentials.
7. **No.** Database authentication and SQL grants are separate.
8. **Execution role** serves ECS infrastructure (image pull/logging); **task role** serves the code inside the container (AWS API calls).

</details>

## 10. Practical guardrails for enterprise development

- Prefer short-lived/federated credentials for people and IAM roles for workloads.
- Apply **least privilege**: exact actions, resources and relevant conditions.
- Never add `AdministratorAccess` just to resolve an unexplained `AccessDenied`.
- Keep app task roles distinct from infrastructure execution roles.
- Treat security-group references as network access controls, not identity grants.
- For databases, grant 5432 from a narrowly scoped application SG, not the entire internet.
- Prefer private database endpoints and managed/separate secret storage; never print real secret values into logs.
- Prefer IaC and PR review for production permission/SG changes; use the console for investigation and approved operations.
- Audit relevant changes through CloudTrail; monitor workloads via CloudWatch.
- Clean up any learning resources and check for residual snapshots, secrets, SGs and logs.

## 11. One-page memory aid

```text
IAM User / SSO identity = WHO I AM (human/API principal)
IAM Role                = IDENTITY THAT CAN BE ASSUMED (temporary credentials)
Role Trust Policy       = WHO MAY ASSUME THAT ROLE?
Permissions Policy      = WHAT AWS ACTIONS MAY THE IDENTITY PERFORM?
Resource Policy         = WHO MAY ACCESS THIS PARTICULAR AWS RESOURCE?
Security Group          = WHICH NETWORK CONNECTIONS ARE ALLOWED?
DB Credentials          = MAY THIS CLIENT LOGIN TO THE DATABASE?
DB Grants               = WHAT SQL MAY THIS DATABASE USER EXECUTE?
CloudTrail              = WHO CALLED WHICH AWS API, AND WHEN?
CloudWatch              = WHAT DID THE APPLICATION/INFRASTRUCTURE REPORT?
```

**Remember:** For an AWS integration failure, identify whether the problem is **identity, API authorization, networking, or application/database authorization** *before* changing permissions.

---

### Useful AWS documentation (read later when internet access is available)

- [IAM identities: users, groups, roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id.html)
- [How IAM policy evaluation works](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
- [IAM roles and role trust policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [ECS task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)
- [ECS task IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)
- [VPC security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [CloudTrail Event history](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html)

*Prepared as an offline-friendly Phase 1 supplemental security reference. Example names are illustrative and do not grant any access by themselves.*
