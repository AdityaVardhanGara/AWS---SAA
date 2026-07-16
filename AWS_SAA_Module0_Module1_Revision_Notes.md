# AWS Solutions Architect Associate -- Revision Notes

## Module 0 -- Cloud Foundations & Module 1 -- IAM

> These notes are optimized for **AWS SAA revision**, focusing on
> concepts, architectural thinking, and common exam traps.

------------------------------------------------------------------------

# Module 0 -- Cloud Foundations

## Why Cloud Exists

-   Traditional infrastructure requires capacity planning, hardware
    procurement, maintenance, and disaster planning.
-   Cloud shifts infrastructure to **on-demand managed services**.
-   Four fundamental resource types:
    -   **Compute** -- EC2, Lambda, ECS, EKS
    -   **Storage** -- S3, EBS, EFS
    -   **Networking** -- VPC, Route53, ELB, CloudFront
    -   **Memory** -- ElastiCache, MemoryDB

### Characteristics of Cloud

-   On-demand provisioning
-   Elastic scaling
-   Global infrastructure
-   API-driven
-   Managed services

------------------------------------------------------------------------

## Virtualization vs Containers

### Virtual Machines

-   Virtualize **hardware**.
-   Each VM has its own OS.
-   Strong isolation.
-   Used heavily by cloud providers for multi-tenancy.

### Containers

-   Virtualize the **OS**.
-   Share the host kernel.
-   Lightweight and fast.
-   Isolation via Linux namespaces and cgroups.

### Kubernetes

-   Does **not** virtualize.
-   Orchestrates containers.
-   Schedules workloads across nodes.

Typical AWS stack:

Application → Container → Container Runtime → Linux → EC2 (VM) → Nitro
Hypervisor → Physical Server

------------------------------------------------------------------------

## AWS Global Infrastructure

### Region

-   Independent geographic area.
-   Contains multiple AZs.
-   Used for latency, compliance, disaster recovery.

### Availability Zone (AZ)

-   One or more isolated data centers.
-   High-speed private connectivity.
-   Deploy production workloads across multiple AZs.

### Local Zone

-   Extension of a Region.
-   Lower latency for specific metro areas.

### Edge Location

-   Used primarily by CloudFront and Route53.
-   Brings content closer to users.

------------------------------------------------------------------------

## Architect Concepts

### Scalability

Ability to handle increasing load while maintaining performance.

-   Vertical Scaling (Scale Up)
-   Horizontal Scaling (Scale Out)

AWS generally prefers horizontal scaling.

------------------------------------------------------------------------

### Elasticity

Automatically scale resources based on demand.

------------------------------------------------------------------------

### Reliability

System behaves correctly and preserves data integrity.

------------------------------------------------------------------------

### Availability

Users can access the service.

------------------------------------------------------------------------

### Fault Tolerance

System continues operating despite component failures.

------------------------------------------------------------------------

### Disaster Recovery

Recover from Region-level failures.

Common strategies: - Backup & Restore - Pilot Light - Warm Standby -
Multi-site Active/Active

------------------------------------------------------------------------

### Cost Optimization

Always choose the lowest operational cost that satisfies business
requirements.

------------------------------------------------------------------------

## Control Plane vs Data Plane

### Control Plane

Management APIs: - Create - Delete - Configure

### Data Plane

Actual workload execution: - CPU - Storage - Network - Database traffic

AWS separates these two for resilience.

------------------------------------------------------------------------

## EC2 Launch Flow (Conceptual)

User → IAM Authentication → EC2 API → Control Plane → Scheduler →
Availability Zone → Nitro Hypervisor → VM Creation → EBS Attachment →
VPC/Subnet → Security Group → Linux Boot → Running Instance

------------------------------------------------------------------------

# Module 1 -- IAM

## Core Flow

Every AWS API request follows:

Authenticate → Authorize → Execute

------------------------------------------------------------------------

## IAM Components

### Users

-   Long-lived identities.
-   Can have console login and access keys.
-   Modern AWS minimizes their usage.

### Groups

-   Collection of users.
-   Aggregate policies.
-   Cannot contain other groups.
-   Cannot be attached to Roles.

### Roles

-   No long-lived credentials.
-   Assumed temporarily.
-   Used by AWS services and federated identities.

------------------------------------------------------------------------

## Policies

Policies are JSON documents defining permissions.

Everything ultimately evaluates to policies.

------------------------------------------------------------------------

### Identity Policies

Attached to: - User - Group - Role

Grant permissions to identities.

------------------------------------------------------------------------

### Resource Policies

Attached to resources: - S3 - SQS - SNS - Lambda - KMS (etc.)

Grant or deny access from the resource side.

------------------------------------------------------------------------

## Policy Evaluation

Default: - Implicit Deny

Then: - Explicit Allow

Finally: - Explicit Deny overrides everything.

**Exam Rule:** Explicit Deny always wins.

------------------------------------------------------------------------

## Least Privilege

Grant only the permissions required.

Reduce blast radius if credentials are compromised.

------------------------------------------------------------------------

## IAM Roles

Purpose: Eliminate long-lived credentials.

Every Role has:

### Trust Policy

Who may assume the role?

### Permission Policy

What permissions are granted after assumption?

------------------------------------------------------------------------

## STS (Security Token Service)

Issues temporary credentials: - Access Key - Secret Key - Session
Token - Expiration

Temporary credentials are preferred over permanent keys.

------------------------------------------------------------------------

## AssumeRole

Used for: - EC2 - Lambda - ECS - EKS (IRSA) - Cross-account access -
Identity federation

------------------------------------------------------------------------

## EC2 Role Credential Flow

Application → Instance Metadata Service (IMDS) → STS → Temporary
Credentials → AWS Service

No credentials stored in code.

------------------------------------------------------------------------

## IMDSv2

Improves protection against SSRF attacks by requiring a session token
before metadata access.

------------------------------------------------------------------------

## Users vs Roles

Users - Permanent credentials - Mainly legacy/human use

Roles - Temporary credentials - Preferred for workloads

------------------------------------------------------------------------

## Permission Boundary

Maximum permissions an identity can ever receive.

Effective Permissions = Identity Policies ∩ Permission Boundary

Does NOT grant permissions.

------------------------------------------------------------------------

## AWS Organizations

Manage multiple AWS Accounts.

Typical structure: - Management Account - Production - Development -
Security - Logging - Sandbox

------------------------------------------------------------------------

## Organizational Units (OU)

Logical grouping of accounts.

SCPs are commonly attached here.

------------------------------------------------------------------------

## Service Control Policy (SCP)

Maximum permissions available within an Account or OU.

Characteristics: - Does NOT grant permissions. - Restricts
permissions. - Even Administrator cannot bypass.

Typical use cases: - Restrict Regions - Block expensive instance types -
Prevent disabling CloudTrail - Compliance guardrails

------------------------------------------------------------------------

## Identity Policy vs Permission Boundary vs SCP

  Component             Grants   Restricts   Scope
  --------------------- -------- ----------- -----------------
  Identity Policy       Yes      No          User/Group/Role
  Resource Policy       Yes      Yes         Resource
  Permission Boundary   No       Yes         User/Role
  SCP                   No       Yes         Account/OU

------------------------------------------------------------------------

## Common Exam Patterns

✔ EC2 accessing S3 → IAM Role

✔ Lambda accessing DynamoDB → Execution Role

✔ Cross-account access → AssumeRole

✔ Company-wide restrictions → SCP

✔ Prevent privilege escalation → Permission Boundary

✔ Resource-specific permissions → Resource Policy

✔ Always prefer temporary credentials over IAM Users

✔ Explicit Deny always wins

------------------------------------------------------------------------

# High-Yield Takeaways

1.  AWS is fundamentally a policy evaluation engine.
2.  Roles are preferred over Users.
3.  STS provides temporary credentials.
4.  Trust Policy = Who can assume?
5.  Permission Policy = What can they do?
6.  Permission Boundaries cap identities.
7.  SCPs cap accounts.
8.  Identity Policies grant permissions.
9.  Resource Policies protect resources.
10. Multi-AZ = High Availability.
11. Multi-Region = Disaster Recovery.
12. Edge Locations = CloudFront caching.
13. Horizontal Scaling \> Vertical Scaling for most AWS architectures.
