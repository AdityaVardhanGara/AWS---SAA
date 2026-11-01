# AWS SAA --- Module 8 Revision Notes

## Monitoring, Management & Governance

Exam-focused revision notes for CloudWatch, CloudTrail, AWS Config,
Organizations, Control Tower, Trusted Advisor, Systems Manager, and
CloudFormation.

------------------------------------------------------------------------

## 8.1 Monitoring & Observability

### Amazon CloudWatch

**Purpose:** Monitor AWS resources and applications through metrics,
logs, alarms, and dashboards.

**Metrics** - Metrics are time-ordered data points identified by a
namespace, metric name, and dimensions. - Common statistics: Average,
Sum, Minimum, Maximum, SampleCount. - AWS services publish standard
metrics; applications can publish custom metrics. - EC2 basic monitoring
is generally 5-minute; detailed monitoring is generally 1-minute. -
High-resolution custom metrics can use sub-minute periods. - Detailed
monitoring does not automatically provide guest OS memory or disk
metrics.

**CloudWatch Agent** - Collects guest OS metrics such as memory and disk
utilization. - Can collect application and custom log files. - CPU
utilization is a standard EC2 metric; memory generally requires the
agent.

**CloudWatch Logs** - Log group: collection of log streams. - Log
stream: sequence of log events from a source. - Log event: timestamp and
message. - Supports retention settings, KMS encryption, metric filters,
subscription filters, and export to S3. - Subscription filter
destinations include Lambda, Kinesis Data Streams, and Amazon Data
Firehose. - Logs Insights supports interactive log queries.

Example:

``` sql
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

**CloudWatch Alarms** - `OK`: metric is within threshold. - `ALARM`:
threshold condition is met. - `INSUFFICIENT_DATA`: insufficient data to
evaluate. - Metric alarms monitor metrics; composite alarms combine
alarm states; anomaly detection models expected behaviour. - Actions can
include SNS notifications and supported EC2 or Auto Scaling actions.

**Dashboards:** Combine metrics and visualizations from multiple AWS
services and Regions.

### CloudWatch vs EventBridge vs SNS vs SQS

  -----------------------------------------------------------------------
  Service                 Main purpose            Example
  ----------------------- ----------------------- -----------------------
  CloudWatch              Metrics, logs, alarms   Alert when CPU exceeds
                                                  80%

  EventBridge             Match and route events  Trigger on EC2 state
                                                  change

  SNS                     Notifications and       Email and Lambda
                          pub/sub fan-out         subscribers

  SQS                     Durable message queue   Buffer asynchronous
                                                  work
  -----------------------------------------------------------------------

Common patterns: - CloudWatch Alarm → SNS → email/Lambda. - EC2
state-change event → EventBridge rule → Lambda. - CloudWatch Logs
subscription filter → Lambda or Kinesis destination.

------------------------------------------------------------------------

## 8.2 Auditing, Compliance & Governance

### AWS CloudTrail

**Purpose:** Record AWS account activity and API calls for auditing and
investigation.

-   **Management events:** Control-plane operations, such as creating
    EC2 instances or changing security groups.
-   **Data events:** Data-plane activity, such as S3 object-level
    operations and Lambda invocations. Configure these for relevant
    resources; additional charges may apply.
-   **CloudTrail Insights:** Helps identify unusual patterns in API
    activity.
-   Event History provides the previous **90 days of management events
    per Region**.
-   Trails deliver events to S3 for longer-term retention and can also
    send events to CloudWatch Logs.
-   Organization trails collect activity across an AWS Organization.

**CloudTrail answers:** Who performed an API action, what action, and
when?

CloudTrail is not a CPU/memory monitoring service or a substitute for
database SQL auditing.

### AWS Config

**Purpose:** Record resource configurations and evaluate compliance.

Capabilities: - Configuration items, history, and snapshots. -
AWS-managed and custom Config rules. - Conformance packs. - Remediation
actions, including supported Systems Manager Automation or Lambda
workflows.

**Config answers:** What is the resource configuration, how did it
change, and is it compliant?

### CloudTrail vs Config

  -----------------------------------------------------------------------
  CloudTrail                          AWS Config
  ----------------------------------- -----------------------------------
  Records API activity                Records resource configuration
                                      state/history

  Identifies actor and API action     Evaluates configuration and
                                      compliance

  Who changed a security group?       Is SSH open to the world?

  Event-focused                       Resource-configuration-focused
  -----------------------------------------------------------------------

### AWS Organizations

**Purpose:** Centrally manage multiple AWS accounts.

-   Management account and member accounts.
-   Organizational Units (OUs) group accounts.
-   Consolidated billing combines eligible usage for billing.
-   Service Control Policies (SCPs) define the maximum permissions
    available to principals in affected accounts.

**SCP exam rules** - An SCP does **not** grant permissions. - IAM
identity/resource policies must still allow the action. - SCPs can
restrict the root user in member accounts. - SCPs generally do not
restrict principals in the Organizations management account. - SCPs are
guardrails, not replacements for IAM.

### AWS Control Tower

**Purpose:** Set up and govern a multi-account AWS landing zone.

-   Landing zone and Account Factory.
-   Guardrails and governance controls.
-   Dashboard and integration with Organizations.
-   Preventive guardrails commonly use SCPs.
-   Detective guardrails commonly use AWS Config rules.

### AWS Trusted Advisor

Provides recommendations across cost optimization, performance,
security, fault tolerance, and service limits. Check availability and
depth depend on support plan and current AWS features.

------------------------------------------------------------------------

## 8.3 Systems Management & Infrastructure as Code

### AWS Systems Manager

**Purpose:** Operate and maintain EC2 instances and supported
hybrid/multicloud servers.

A managed instance generally needs: - SSM Agent installed and running. -
Appropriate IAM permissions, commonly `AmazonSSMManagedInstanceCore`. -
Network connectivity to Systems Manager endpoints through internet/NAT
or configured VPC endpoints.

  -----------------------------------------------------------------------
  Feature                             Purpose
  ----------------------------------- -----------------------------------
  Session Manager                     Secure interactive access without
                                      inbound SSH/RDP ports

  Run Command                         Execute commands/scripts on one or
                                      many managed instances

  Patch Manager                       Scan/install patches using
                                      baselines and maintenance windows

  State Manager                       Maintain desired instance
                                      configuration over time

  Automation                          Run operational workflows

  Parameter Store                     Store configuration values and
                                      parameters

  Inventory                           Collect metadata about managed
                                      instances

  Fleet Manager                       View and manage fleet resources
  -----------------------------------------------------------------------

**Session Manager** - Avoids the need for inbound port 22 for this
access path. - Avoids distributing SSH keys for this access path. -
Sessions can be logged to S3 or CloudWatch Logs when configured. -
Requires agent, IAM permissions, and network connectivity.

**Exam trap:** Blocking port 22 alone does not enable Session Manager.
Agent, permissions, and connectivity are still required.

### Parameter Store vs Secrets Manager

  -----------------------------------------------------------------------
  Parameter Store                     Secrets Manager
  ----------------------------------- -----------------------------------
  General configuration and           Secrets such as database
  parameters                          credentials

  String, StringList, SecureString    Secret lifecycle management

  SecureString uses KMS encryption    Managed rotation integrations for
                                      supported secrets

  Good for application configuration  Good for credentials and rotation
                                      workflows
  -----------------------------------------------------------------------

### AWS CloudFormation

**Purpose:** Infrastructure as Code using declarative YAML or JSON
templates.

-   **Template:** Describes desired resources and configuration.
-   **Stack:** Deployed collection of resources managed together.
-   **Change Sets:** Preview proposed changes before applying an update.
-   **Drift Detection:** Finds differences between actual resource
    configuration and the template's expected configuration.
-   **Stack Policies:** Protect specified resources from unintended
    update actions.
-   **DeletionPolicy:** Controls what happens to supported resources
    when removed from a stack, such as Retain or Snapshot.
-   **Rollback:** CloudFormation can roll back failed stack operations.
-   **StackSets:** Deploy stacks across multiple AWS accounts and/or
    Regions.

### CloudFormation vs Systems Manager

  -----------------------------------------------------------------------
  Requirement                         Service
  ----------------------------------- -----------------------------------
  Provision VPC, EC2, RDS from a      CloudFormation
  template                            

  Preview infrastructure update       CloudFormation Change Sets

  Detect manual infrastructure        CloudFormation Drift Detection
  changes                             

  Deploy templates across             CloudFormation StackSets
  accounts/Regions                    

  Execute a script on hundreds of EC2 Systems Manager Run Command
  instances                           

  Access instances without inbound    Systems Manager Session Manager
  SSH                                 

  Patch a fleet                       Systems Manager Patch Manager

  Maintain instance state             Systems Manager State Manager
  -----------------------------------------------------------------------

**Mental model:** CloudFormation builds and manages infrastructure;
Systems Manager operates and maintains infrastructure.

------------------------------------------------------------------------

## 8.4 Architecture Scenarios & Exam Revision

### Pattern A --- Monitor and notify

Requirement: Notify operations when CPU exceeds a threshold.

``` text
EC2 metric → CloudWatch Alarm → SNS Topic → Email / Lambda subscribers
```

CloudWatch monitors the metric; SNS delivers notifications.

### Pattern B --- React to an AWS event

Requirement: Invoke Lambda when an EC2 instance changes state.

``` text
EC2 state-change event → EventBridge rule → Lambda
```

EventBridge matches event patterns and routes events to targets. It can
also match supported CloudTrail-delivered API events.

### Pattern C --- Detect and remediate non-compliance

Requirement: Detect an S3 bucket violating an encryption policy.

``` text
Resource configuration → AWS Config rule
                       → Non-compliant evaluation
                       → Remediation action
                       → Systems Manager Automation / Lambda
```

Config evaluates compliance; the remediation action performs corrective
work.

### Pattern D --- Govern multiple accounts

``` text
AWS Organizations
  ├── OU: Production
  │     └── Production account
  └── OU: Development
        └── Development account

SCPs define permission guardrails.
Control Tower helps establish and govern the landing zone.
Organization CloudTrail and Config support centralized visibility.
```

### Pattern E --- Operate a large EC2 fleet

-   Run Command: execute scripts across many instances.
-   Patch Manager: coordinate patching.
-   State Manager: maintain desired configuration.
-   Session Manager: managed interactive access without inbound SSH.

### Pattern F --- Deploy infrastructure consistently

-   CloudFormation templates provide repeatable infrastructure.
-   Change Sets preview proposed changes.
-   Drift Detection identifies out-of-band changes.
-   StackSets deploy across accounts and Regions.

### Common scenario-to-service mapping

  -----------------------------------------------------------------------
  Exam requirement                    Best-fit service or combination
  ----------------------------------- -----------------------------------
  Monitor CPU, latency, application   CloudWatch
  metrics                             

  Collect EC2 memory/disk metrics     CloudWatch Agent

  Search application logs             CloudWatch Logs Insights

  Notify when a metric crosses        CloudWatch Alarm → SNS
  threshold                           

  Identify who modified a security    CloudTrail
  group                               

  Investigate S3 object-level access  CloudTrail data events, if enabled

  Check bucket encryption compliance  AWS Config

  Restrict permissions across member  Organizations + SCP
  accounts                            

  Govern a multi-account landing zone Control Tower

  Execute commands across many        Systems Manager Run Command
  instances                           

  Patch EC2 instances                 Systems Manager Patch Manager

  Access EC2 without inbound SSH      Systems Manager Session Manager

  Deploy infrastructure across        CloudFormation StackSets
  accounts/Regions                    

  Detect manual changes to a          Drift Detection
  CloudFormation resource             

  Identify unusual API activity       CloudTrail Insights
  patterns                            
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 8.5 High-Value Exam Traps

1.  **CloudWatch vs CloudTrail:** CloudWatch monitors operational
    metrics/logs; CloudTrail records AWS API activity.
2.  **CloudTrail vs Config:** CloudTrail records who did what; Config
    records resource configuration and evaluates compliance.
3.  **EventBridge vs SNS:** EventBridge filters/routes events; SNS
    distributes notifications and messages to subscribers.
4.  **SCP vs IAM:** SCP sets the maximum permissions for affected
    accounts; IAM policies grant permissions.
5.  **Systems Manager vs CloudFormation:** Systems Manager operates
    resources; CloudFormation provisions infrastructure.
6.  **Session Manager:** No inbound port 22 is required, but agent,
    permissions, and network connectivity are.
7.  **EC2 memory metrics:** Usually require CloudWatch Agent; detailed
    monitoring alone is not enough.
8.  **CloudTrail Event History:** 90 days of management events per
    Region; use a trail for longer-term delivery/retention.
9.  **CloudTrail data events:** Configure relevant data-event selectors;
    do not assume all S3 object access is recorded automatically.
10. **Config remediation:** Config detects/evaluates; a remediation
    action changes the resource.
11. **CloudFormation Drift Detection:** Detects drift; it does not
    automatically fix all drift.
12. **StackSets:** Multi-account and multi-Region CloudFormation
    deployment.
13. **Control Tower:** Landing-zone governance built on Organizations
    and related services.
14. **Trusted Advisor:** Recommendation availability varies by support
    plan and current AWS features.

------------------------------------------------------------------------

## 8.6 Quick Self-Test

Try answering before looking at the answers.

1.  Which service identifies the IAM principal that changed a security
    group?
2.  Which service evaluates whether an S3 bucket complies with an
    encryption rule?
3.  Which service provides EC2 memory metrics?
4.  Which service runs a script across hundreds of managed EC2
    instances?
5.  Which CloudFormation feature previews a stack update?
6.  Which CloudFormation feature detects out-of-band resource changes?
7.  Which service routes an EC2 state-change event to Lambda?
8.  Which service provides secure instance access without inbound SSH?
9.  Do SCPs grant permissions?
10. Which service helps create and govern a multi-account landing zone?

### Answers

1.  CloudTrail.
2.  AWS Config.
3.  CloudWatch Agent publishing to CloudWatch.
4.  Systems Manager Run Command.
5.  Change Sets.
6.  Drift Detection.
7.  EventBridge.
8.  Systems Manager Session Manager.
9.  No. SCPs define maximum available permissions; IAM must still allow
    the action.
10. AWS Control Tower.

------------------------------------------------------------------------

## 8.7 Final Module 8 Checklist

-   [ ] Explain CloudWatch metrics, logs, alarms, dashboards, and the
    CloudWatch Agent.
-   [ ] Distinguish CloudWatch, CloudTrail, EventBridge, SNS, and SQS.
-   [ ] Distinguish CloudTrail management events, data events, and
    Insights.
-   [ ] Explain AWS Config rules, configuration history, and
    remediation.
-   [ ] Explain Organizations, OUs, SCPs, and consolidated billing.
-   [ ] Explain Control Tower and preventive vs detective guardrails.
-   [ ] Select the right Systems Manager capability for access,
    commands, patching, and configuration.
-   [ ] Distinguish Parameter Store from Secrets Manager.
-   [ ] Explain CloudFormation stacks, Change Sets, Drift Detection,
    DeletionPolicy, and StackSets.
-   [ ] Recognize the common architecture patterns in this module.

## Module 8 --- One-Page Service Selection

  If the question says...                         Think...
  ----------------------------------------------- ------------------------
  Metrics, alarms, dashboards, logs               CloudWatch
  Memory/disk metrics from an instance            CloudWatch Agent
  Who called an AWS API?                          CloudTrail
  S3 object-level API activity                    CloudTrail data events
  Resource configuration history/compliance       AWS Config
  Route events based on event patterns            EventBridge
  Notify multiple subscribers                     SNS
  Queue messages for processing                   SQS
  Multi-account management and billing            Organizations
  Permission guardrails across accounts           SCP
  Governed landing zone                           Control Tower
  Secure access without inbound SSH               Session Manager
  Run commands across a fleet                     Run Command
  Patch managed instances                         Patch Manager
  Maintain desired instance state                 State Manager
  Store configuration parameters                  Parameter Store
  Store and rotate supported credentials          Secrets Manager
  Define/provision AWS infrastructure as code     CloudFormation
  Preview infrastructure updates                  Change Sets
  Detect infrastructure drift                     Drift Detection
  Deploy CloudFormation across accounts/Regions   StackSets

**End of Module 8 --- Monitoring, Management & Governance.**
