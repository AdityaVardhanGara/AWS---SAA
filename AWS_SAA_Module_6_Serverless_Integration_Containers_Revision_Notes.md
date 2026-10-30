# AWS Solutions Architect Associate --- Module 6

## Serverless, Application Integration & Containers

**Reading notes \| Lessons 6.1--6.9**

This module covers Lambda, API Gateway, SQS, SNS, EventBridge, Step
Functions, Fargate, ECS, and EKS. The objective is to understand the
engineering problem each service solves, its operating model,
reliability guarantees, limitations, production patterns, and how to
select it in SAA-C03 scenarios.

------------------------------------------------------------------------

# 6.0 --- Big picture

Modern applications separate request handling, compute, asynchronous
work, event routing, workflows, and container orchestration.

``` text
Clients -> API Gateway -> Lambda / ECS API -> Database
                              |
                              +-> SQS -> Workers
                              |
                              +-> EventBridge -> Rules -> Targets
                              |
                              +-> Step Functions -> Multi-step workflow

Container workloads -> ECS or EKS -> EC2 or Fargate compute
```

  -----------------------------------------------------------------------
  Requirement             Service to consider     Primary purpose
  ----------------------- ----------------------- -----------------------
  Run event-triggered     Lambda                  Function compute
  code                                            

  Expose managed APIs     API Gateway             API front door and
                                                  management

  Buffer asynchronous     SQS                     Durable work queue
  work                                            

  Broadcast notifications SNS                     Pub/sub fan-out

  Route events by content EventBridge             Event bus and rules

  Coordinate multi-step   Step Functions          Workflow orchestration
  work                                            

  Run containers without  Fargate                 Serverless container
  hosts                                           compute

  AWS-native              ECS                     Container scheduling
  orchestration                                   and services

  Managed Kubernetes      EKS                     Kubernetes control
                                                  plane on AWS
  -----------------------------------------------------------------------

**Core distinction:** Lambda executes functions; Fargate supplies
container compute; ECS and EKS orchestrate containers.

------------------------------------------------------------------------

# 6.1 --- AWS Lambda

## 6.1.1 Problem and model

Traditional servers require provisioning, patching, runtime management,
and capacity planning. For event-driven work, that overhead may be
unnecessary.

AWS Lambda runs code in response to events without requiring you to
manage servers. Serverless does not mean there are no servers; AWS
manages the execution infrastructure.

A function is configured with code, runtime, memory, timeout, IAM
execution role, environment variables, and optional VPC connectivity.

``` text
Event -> Lambda service -> execution environment -> handler -> result
```

## 6.1.2 Execution environments, cold starts, and state

A cold start occurs when Lambda initializes an execution environment,
including runtime startup and function initialization. A warm invocation
may reuse an existing environment, but reuse is not guaranteed.

Design functions to be stateless between invocations. Do not treat
process memory or `/tmp` as durable storage. Persist state in DynamoDB,
RDS, S3, queues, or another appropriate service.

**Provisioned Concurrency** pre-initializes a configured number of
environments to reduce cold-start latency. It has an additional cost.

## 6.1.3 Memory, CPU, timeout, and concurrency

-   Lambda memory allocation also affects allocated CPU.
-   Maximum execution duration is 15 minutes.
-   Concurrency is the number of simultaneous executions.
-   Reserved concurrency reserves capacity for a function and sets its
    concurrency ceiling.
-   Provisioned Concurrency keeps environments initialized.
-   Account and regional quotas still apply.

A useful approximation:

\[ `\text{Concurrency}`{=tex}
`\approx `{=tex}`\text{Request rate}`{=tex}
`\times `{=tex}`\text{Average duration}`{=tex} \]

For example, 100 requests/second at 0.2 seconds average duration implies
roughly 20 concurrent executions, before burst and variance effects.

## 6.1.4 Invocation models

  -----------------------------------------------------------------------
  Model                   Example                 Behavior
  ----------------------- ----------------------- -----------------------
  Synchronous             API Gateway -\> Lambda  Caller waits for
                                                  response

  Asynchronous            S3/EventBridge -\>      Event accepted for
                          Lambda                  later processing;
                                                  retries and failure
                                                  handling apply

  Poll-based              SQS/stream event source Lambda service polls
                          mapping                 and invokes with
                                                  batches
  -----------------------------------------------------------------------

Do not assume all triggers have identical retry semantics.

### Lambda with SQS

``` text
SQS -> event source mapping polls -> Lambda batch
                                  -> success: messages deleted
                                  -> failure: messages retried
```

Receiving does not delete a message. Visibility timeout, batch size,
concurrency, and partial batch failure handling matter. If a batch fails
without partial batch reporting, already-processed records may be
delivered again. Consumers must be idempotent.

## 6.1.5 VPC connectivity

Attach Lambda to a VPC when it needs private access to RDS, internal
services, or private caches. A VPC-attached function does not
automatically get internet access. Private-subnet egress may need NAT
Gateway; private AWS service access may use VPC endpoints.

IAM permission and network reachability are separate requirements.

## 6.1.6 Database connection pressure

Lambda can scale faster than a relational database can accept
connections. Hundreds of concurrent environments opening connections can
overwhelm the database.

Mitigations: - Reuse connections where practical, without depending on
reuse for correctness. - Set reserved concurrency to protect downstream
capacity. - Use RDS Proxy where appropriate. - Keep transactions
short. - Choose a data store suited to the access pattern.

**One sentence:** Lambda runs event-triggered functions without server
management; design around statelessness, invocation semantics,
concurrency, and downstream limits.

------------------------------------------------------------------------

# 6.2 --- Amazon API Gateway

## 6.2.1 Purpose

API Gateway provides a managed front door for APIs: routing,
authorization integration, throttling, stages, monitoring, and backend
integrations. It is not simply another load balancer.

## 6.2.2 API types

  Type            Typical use
  --------------- ------------------------------------------------
  REST API        Feature-rich API management
  HTTP API        Simpler, lower-cost HTTP APIs for common needs
  WebSocket API   Persistent two-way communication

## 6.2.3 Request path

``` text
Client -> API Gateway -> authorization / throttling / route
                       -> Lambda / HTTP backend / AWS service
```

## 6.2.4 Authorization

  -----------------------------------------------------------------------
  Mechanism                           Purpose
  ----------------------------------- -----------------------------------
  IAM authorization                   AWS identity signs requests with
                                      SigV4

  Cognito/JWT authorizer              Validates user tokens for supported
                                      APIs

  Lambda authorizer                   Custom authorization logic

  API key                             Client identification and
                                      usage-plan controls in supported
                                      configurations
  -----------------------------------------------------------------------

**Exam trap:** An API key is not strong authentication. Use IAM,
Cognito/JWT, or an appropriate authorizer.

## 6.2.5 Throttling, quotas, caching, and stages

-   Throttling protects backend systems from excessive request rates.
-   Usage plans associate API keys with throttling and quotas in
    supported REST API configurations.
-   Caching can reduce backend load for suitable read-heavy APIs.
-   Stages represent deployed environments such as dev, test, and prod.
-   Deployments publish API configuration to a stage.

Throttling limits do not guarantee backend success; account quotas,
burst behavior, and downstream capacity still matter.

## 6.2.6 API Gateway vs related services

  Service       Primary role
  ------------- -----------------------------------------------
  API Gateway   API management and integrations
  ALB           Layer 7 traffic distribution and HTTP routing
  NLB           Layer 4 load balancing
  CloudFront    Global content delivery and edge caching
  Route 53      DNS and routing policies

**One sentence:** API Gateway is a managed API front door with
authorization, throttling, routing, stages, and integrations---not
merely a load balancer.

------------------------------------------------------------------------

# 6.3 --- Amazon SQS

## 6.3.1 Purpose

SQS decouples producers and consumers by introducing a durable buffer. A
producer sends a message; a consumer polls, processes, and deletes it.

``` text
Producer -> SQS queue -> Consumer -> processing
```

It supports asynchronous work, buffering, retries, and independent
consumer scaling.

## 6.3.2 Standard vs FIFO

  -----------------------------------------------------------------------
  Feature                 Standard                FIFO
  ----------------------- ----------------------- -----------------------
  Throughput              Very high               Ordering constraints;
                                                  high-throughput mode
                                                  available

  Ordering                Best-effort             Strict within a message
                                                  group

  Delivery                At-least-once           Deduplication features,
                                                  but external side
                                                  effects still need
                                                  idempotency

  Use                     Independent work        Ordered work
  -----------------------------------------------------------------------

Different FIFO message groups can be processed concurrently.

## 6.3.3 Visibility timeout

When a consumer receives a message, SQS hides it temporarily. It is not
deleted.

``` text
Visible -> Receive -> hidden during visibility timeout
                         |
                  +------+------+
                  |             |
                Delete       No delete
                  |             |
                Done       Visible again
```

If processing exceeds visibility timeout, the message may become visible
while the original consumer is still processing it. This can cause
duplicate concurrent work. Set timeout appropriately and make consumers
idempotent.

## 6.3.4 Long polling

Long polling waits for messages instead of returning immediately with an
empty response. It reduces empty receives and polling overhead.

## 6.3.5 Dead-letter queues

A DLQ isolates messages that exceed a configured maximum receive count.
It helps diagnose poison messages and enables controlled redrive. It
does not fix the underlying error.

## 6.3.6 Lambda event source mapping

Lambda polls SQS and invokes with batches. Enable partial batch response
when appropriate so only failed records are retried rather than the
entire batch.

## 6.3.7 SQS vs Kafka

  -----------------------------------------------------------------------
  SQS                                 Kafka / MSK
  ----------------------------------- -----------------------------------
  Work queue                          Retained event log

  Messages normally removed after     Events retained by policy
  successful processing               

  Consumers compete for messages on   Consumer groups track offsets
  one queue                           

  Task distribution and buffering     Streaming, replay, multiple
                                      consumer groups
  -----------------------------------------------------------------------

**One sentence:** SQS is a durable work queue; receive does not delete,
visibility timeout controls temporary hiding, and consumers must
tolerate duplicates.

------------------------------------------------------------------------

# 6.4 --- Amazon SNS

## 6.4.1 Purpose

SNS is managed publish/subscribe messaging. A publisher sends to a
topic; SNS delivers to subscribers.

``` text
Publisher -> SNS topic -> SQS / Lambda / HTTPS / other subscribers
```

Use SNS when multiple independent systems need to be notified about the
same event.

## 6.4.2 SNS vs SQS

  -----------------------------------------------------------------------
  Dimension               SNS                     SQS
  ----------------------- ----------------------- -----------------------
  Model                   Push-based pub/sub      Pull-based queue

  Purpose                 Broadcast/fan-out       Buffer work

  Multiple subscribers    Each subscription gets  Consumers on one queue
                          delivery                compete

  Retention               Not a long-term event   Configurable queue
                          store                   retention
  -----------------------------------------------------------------------

A common pattern is SNS fan-out to separate SQS queues:

``` text
Order service -> SNS topic
                  |       |        |
                SQS     SQS      SQS
             Inventory  Email  Analytics
```

Each consumer has an independent backlog and can process at its own
rate.

## 6.4.3 Filtering and failure handling

Subscription filter policies allow subscribers to receive only matching
messages.

Distinguish failure stages: 1. SNS cannot deliver to a subscriber:
subscription DLQ can capture delivery failures where supported. 2. SNS
delivers to SQS, but the consumer fails: configure an SQS DLQ on that
queue.

SNS standard topics are high-throughput but do not guarantee ordering.
FIFO topics support ordering/deduplication with compatible subscription
patterns and constraints.

**One sentence:** SNS broadcasts to subscribers; SNS-to-SQS fan-out
gives each downstream consumer its own durable backlog.

------------------------------------------------------------------------

# 6.5 --- Amazon EventBridge

## 6.5.1 Purpose

EventBridge is a serverless event bus and content-aware event router. It
receives events from AWS services, custom applications, and supported
SaaS partners, then routes matching events to configured targets.

## 6.5.2 Event envelope

Typical fields include `source`, `detail-type`, `detail`, `time`,
`region`, and `account`.

``` json
{
  "source": "com.example.orders",
  "detail-type": "OrderCreated",
  "detail": {
    "orderId": "ORD-1001",
    "amount": 2500
  }
}
```

## 6.5.3 Event buses

  Bus       Purpose
  --------- -------------------------------
  Default   Supported AWS service events
  Custom    Application-defined events
  Partner   Supported SaaS partner events

## 6.5.4 Rules and targets

Rules match event patterns and route events to targets. Input
transformers can reshape event data for a target.

``` text
Event bus
  +-- OrderCreated rule -> Inventory SQS / Analytics Lambda
  +-- PaymentFailed rule -> Alert topic / incident workflow
```

Unmatched events are not automatically delivered to every target.

## 6.5.5 EventBridge vs SNS vs SQS vs Kafka

  Requirement                                              Service
  -------------------------------------------------------- -------------
  Content-based event routing                              EventBridge
  Broadcast to subscribers                                 SNS
  Durable work backlog                                     SQS
  Partitioned retained event stream and consumer offsets   Kafka/MSK

EventBridge supports archives and replay, but it is not a direct Kafka
replacement for high-volume partitioned streaming and consumer-managed
offsets.

## 6.5.6 Retries and DLQs

EventBridge retries failed target delivery according to its retry
policy. The default policy can retry for up to 24 hours and up to 185
attempts. A standard SQS DLQ can capture failed target deliveries when
configured.

Distinguish EventBridge target delivery failure from Lambda asynchronous
invocation failure and SQS consumer processing failure. These are
different stages with different retry/DLQ configurations.

Design consumers for duplicate delivery; use idempotency keys.

## 6.5.7 Scheduler, cross-account, and schemas

EventBridge Scheduler supports one-time, rate, and cron schedules, time
zones, flexible windows, retries, and DLQ configuration. Prefer
Scheduler for new scheduling use cases unless a specific requirement
calls for scheduled rules.

Cross-account event delivery uses event-bus permissions and appropriate
target roles.

Schema Registry discovers and manages event schemas and can generate
code bindings. It does not automatically enforce all business contracts
in application code.

## 6.5.8 Transactional outbox

A database write followed by an event publish is a dual-write problem.
The database may commit while event publishing fails.

Transactional outbox: 1. Write the business record and an outbox record
in one database transaction. 2. A separate publisher reads the outbox.
3. Publish the event. 4. Mark the outbox record delivered.

Consumers still need idempotency because retries can publish or deliver
duplicates.

**One sentence:** EventBridge is a content-aware event router; SNS
broadcasts, SQS buffers, and Kafka retains a partitioned event log.

------------------------------------------------------------------------

# 6.6 --- AWS Step Functions

## 6.6.1 Purpose

Step Functions orchestrates multi-step workflows and manages execution
state, branching, retries, timeouts, and recovery.

Example:

``` text
Validate order -> Authorize payment -> Reserve inventory
       -> Arrange shipping -> Notify customer
```

Without orchestration, each service may need to coordinate state and
recovery itself.

## 6.6.2 State machine and execution

-   State machine: reusable workflow definition.
-   Execution: one run of the workflow.
-   State: an individual step or control-flow element.

Workflows are defined using Amazon States Language (ASL), a JSON-based
language.

## 6.6.3 State types

  State      Purpose
  ---------- ---------------------------------------
  Task       Performs work or calls an integration
  Choice     Conditional branching
  Parallel   Runs branches concurrently
  Map        Runs a workflow over collection items
  Wait       Pauses
  Pass       Passes/transforms data
  Succeed    Successful terminal state
  Fail       Failed terminal state

## 6.6.4 JSON data flow

JSONPath-based fields:

-   `InputPath`: selects input for a state.
-   `Parameters`: constructs task parameters.
-   `ResultSelector`: shapes the result.
-   `ResultPath`: places the result into input.
-   `OutputPath`: selects output for the next state.

By default, a task result can replace input. Use `ResultPath`
deliberately. `ResultPath: null` discards the task result while
preserving input.

Step Functions also supports JSONata with different expression
semantics.

Keep payloads small; pass identifiers and references rather than large
documents.

## 6.6.5 Retry and Catch

Use `Retry` for transient errors with backoff and attempt limits. Use
`Catch` to route failures to recovery or compensation states.

Do not retry permanent business errors as if they were transient network
errors. Understand service-specific integration error names and terminal
failure behavior.

## 6.6.6 Standard vs Express

  -----------------------------------------------------------------------
  Dimension               Standard                Express
  ----------------------- ----------------------- -----------------------
  Maximum duration        Up to 1 year            Up to 5 minutes

  Execution semantics     Exactly-once workflow   Async at-least-once;
                          execution semantics     sync at-most-once

  Best fit                Long-running, auditable High-volume short
                          workflows               workflows

  History                 Execution history       CloudWatch Logs for
                          retained up to 90 days  execution logging
  -----------------------------------------------------------------------

Exactly-once workflow semantics do not guarantee exactly-once external
side effects. A payment may have succeeded even if the workflow timed
out before seeing the response. Use idempotency.

## 6.6.7 Service integrations

Step Functions can call supported AWS services directly.

-   Request-response: call and receive response.
-   `.sync`: start a supported job and wait for completion.
-   `.waitForTaskToken`: pause for an external callback.

Direct integrations reduce unnecessary Lambda wrappers. Use Lambda when
custom code is needed.

## 6.6.8 Parallel, Map, and Saga

Parallel runs branches concurrently. Map runs workflow logic for each
item. Distributed Map supports large-scale processing using child
workflow executions.

A Saga coordinates local transactions and compensating actions:

``` text
Reserve inventory -> Charge payment -> Create shipment
                                      |
                                  failure
                                      v
                         Refund payment -> Release inventory
```

Compensation is a new business action, not a magical rollback. It can
fail and require retry or manual reconciliation.

**One sentence:** Step Functions orchestrates workflows with state,
branching, retry, and recovery; external side effects still need
idempotency and compensation.

------------------------------------------------------------------------

# 6.7 --- AWS Fargate

## 6.7.1 Purpose

Fargate is a serverless compute engine for containers. You provide the
image and resource requirements; AWS manages the underlying compute
hosts.

Fargate works with ECS and EKS for eligible workloads. It is not an
orchestrator: ECS or Kubernetes decides what should run.

## 6.7.2 Task model

An ECS task is a running instance of a task definition. A task can
contain one or more tightly coupled containers.

``` text
ECR image -> ECS service -> Fargate tasks
                         +-> Task 1
                         +-> Task 2
                         +-> Task 3
```

Containers in a task share task-level networking and can communicate
over localhost where appropriate. Unrelated microservices should
generally not be placed in one task because they become coupled in
deployment and scaling.

## 6.7.3 Networking

ECS Fargate tasks use `awsvpc` networking. Each task receives an ENI in
a VPC subnet and task-level security groups.

Typical internet-facing design:

``` text
Internet -> public ALB -> private Fargate tasks -> private database
```

For Fargate tasks behind an ALB, use an IP target group, not an instance
target group.

## 6.7.4 Image pulls and endpoints

The task execution role authorizes startup actions such as pulling
images from ECR and publishing logs. Network connectivity is also
required.

Private subnet options include: - ECR interface endpoints. - S3 gateway
endpoint for image layers. - CloudWatch Logs interface endpoint. -
Secrets Manager endpoint if needed. - NAT Gateway for suitable outbound
access.

IAM permission does not create network connectivity.

## 6.7.5 CPU, memory, storage

Fargate requires supported CPU/memory combinations. Example: 1024 CPU
units = 1 vCPU; 2048 MiB = 2 GiB.

Right-size based on measured peak use, startup behavior, and headroom.

Fargate provides ephemeral task storage for temporary data. It is not
durable. Use S3 for objects, databases for application state, and EFS
for supported shared filesystem use cases.

## 6.7.6 IAM roles

  -----------------------------------------------------------------------
  Role                                Purpose
  ----------------------------------- -----------------------------------
  Task execution role                 Startup operations: image pull,
                                      logging, supported secret retrieval

  Task role                           AWS permissions used by application
                                      code
  -----------------------------------------------------------------------

## 6.7.7 Fargate Spot

Fargate Spot uses spare capacity at a discount and can be interrupted
with a two-minute warning. Good fits include retryable,
interruption-tolerant workers and batch processing. Do not treat it as
guaranteed capacity.

## 6.7.8 EC2 vs Fargate

  -----------------------------------------------------------------------
  ECS on EC2                          ECS on Fargate
  ----------------------------------- -----------------------------------
  You manage EC2 hosts                AWS manages underlying hosts

  You manage host fleet capacity      Scale ECS task count; no worker
                                      fleet

  More host-level control             Limited host control

  Can suit specialized host needs and Reduces infrastructure operations
  high steady utilization             
  -----------------------------------------------------------------------

**One sentence:** Fargate runs containers without host management; ECS
or EKS still controls desired workloads and you still manage scaling
policies, networking, IAM, and reliability.

------------------------------------------------------------------------

# 6.8 --- Amazon ECS

## 6.8.1 Purpose

ECS is AWS-native container orchestration. It schedules tasks, maintains
desired task counts, coordinates deployments, and integrates with AWS
networking and load balancing.

ECS can use EC2 or Fargate as compute.

## 6.8.2 Core concepts

  -----------------------------------------------------------------------
  Concept                 Meaning                 Kubernetes analogy
                                                  (approximate)
  ----------------------- ----------------------- -----------------------
  Cluster                 Logical grouping of ECS Cluster
                          resources               

  Task definition         Container blueprint     Pod template

  Task                    Running task-definition Pod
                          instance                

  Service                 Maintains desired tasks Deployment + ReplicaSet
                          and coordinates         
                          deployments             

  Container               Application             Container
                          process/image           

  Capacity provider       Compute capacity        Part of node-capacity
                          strategy                management
  -----------------------------------------------------------------------

These are analogies, not exact equivalences.

## 6.8.3 Task definitions

A task definition can specify image, CPU, memory, port mappings,
environment variables, secrets references, IAM roles, logs, volumes, and
networking settings.

A task definition is versioned. Registering a new revision does not
automatically update every service; the service must be updated to use
it.

## 6.8.4 Tasks vs services

Use an ECS service for a continuously running application that should
maintain desired task count.

Use a standalone task for one-off work such as migrations or
administrative jobs. EventBridge Scheduler can launch ECS tasks on a
schedule.

## 6.8.5 Desired state and reconciliation

If a service desires three tasks and one stops, ECS attempts to launch a
replacement. The service works continuously toward the desired state.

## 6.8.6 Capacity providers

Common capacity providers: - `FARGATE` - `FARGATE_SPOT` - EC2 Auto
Scaling group-backed capacity providers

A capacity provider strategy determines the compute capacity used by
tasks. It is distinct from service auto scaling, which changes desired
task count.

## 6.8.7 Service auto scaling

Application Auto Scaling adjusts ECS service desired count.

Example: - Minimum: 3 tasks. - Desired: 3 tasks. - Maximum: 20 tasks. -
Target CPU: 60%.

Approaches include target tracking, step scaling, and scheduled scaling.

Choose metrics that represent the bottleneck. Queue depth or
oldest-message age may be more useful than CPU for queue workers.

## 6.8.8 Load balancing and service discovery

ECS services integrate with ALB and NLB. Fargate tasks using `awsvpc`
require IP target groups for ALB. ALB can route by host and path. AWS
Cloud Map can provide service discovery.

## 6.8.9 Deployments

ECS supports rolling deployments and blue/green approaches through
supported deployment controllers and integrations.

Rolling deployment settings: - `minimumHealthyPercent`: minimum
percentage of desired tasks that must remain healthy. -
`maximumPercent`: maximum percentage of desired tasks allowed to be
running or pending during deployment.

For desired count 4, minimum healthy 100%, and maximum 200%, ECS can
keep at least four healthy tasks while allowing extra deployment
capacity, subject to available resources.

Blue/green runs old and new versions during transition and shifts
traffic after validation. It may require temporary extra capacity.

Design for graceful shutdown, compatible database migrations, health
checks, and safe old/new version coexistence.

## 6.8.10 Logs and metrics

ECS can forward container stdout/stderr to CloudWatch Logs using the
`awslogs` driver. FireLens can route logs to other supported
destinations. CloudWatch metrics, alarms, and Container Insights provide
operational visibility.

**One sentence:** ECS is AWS-native orchestration: task definitions
describe containers, services maintain and scale tasks, and capacity
providers determine compute strategy.

------------------------------------------------------------------------

# 6.9 --- Amazon EKS

## 6.9.1 Purpose

EKS is AWS's managed Kubernetes service. AWS manages the Kubernetes
control plane; you choose worker compute, commonly EC2 nodes or Fargate
for eligible pods.

## 6.9.2 Kubernetes architecture

Control-plane components: - API Server: Kubernetes API entry point. -
etcd: cluster state store. - Scheduler: selects nodes for pods. -
Controller Manager: reconciles desired and actual state. - Cloud
Controller Manager: cloud integration where applicable.

Worker-node components commonly include kubelet, container runtime,
networking components, and pods.

## 6.9.3 Declarative model

You declare desired state in Kubernetes manifests. A Deployment creates
and manages a ReplicaSet; the ReplicaSet creates pods; the scheduler
assigns pods to nodes; kubelets run containers.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
        - name: payment-api
          image: example/payment-api:v1
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
```

## 6.9.4 EKS compute choices

  -----------------------------------------------------------------------
  Option                              Responsibility
  ----------------------------------- -----------------------------------
  Managed node groups                 AWS helps provision/update EC2 node
                                      groups; you configure instance
                                      types and capacity

  Self-managed nodes                  You take more responsibility for
                                      EC2 lifecycle

  Fargate                             AWS provisions compute for eligible
                                      pods; you configure Fargate
                                      profiles and workloads
  -----------------------------------------------------------------------

Use EC2 for node-level control, GPUs, DaemonSets, privileged workloads,
or features unsupported by Fargate.

## 6.9.5 Networking

The AWS VPC CNI commonly assigns VPC-routable IPs to pods using ENIs and
subnet address capacity.

Watch for subnet IP exhaustion, ENI limits, pod density, security
groups, network policies, and endpoint connectivity.

The Kubernetes API endpoint is for cluster management; it is not the
public endpoint for your application.

## 6.9.6 Services and ingress

A Kubernetes Service gives a stable endpoint for a changing set of pods.

  -----------------------------------------------------------------------
  Type                                Purpose
  ----------------------------------- -----------------------------------
  ClusterIP                           Internal cluster endpoint

  NodePort                            Exposes port on worker nodes

  LoadBalancer                        Requests a cloud load balancer
                                      through supported integration
  -----------------------------------------------------------------------

Ingress defines HTTP/HTTPS routing. AWS Load Balancer Controller can
provision ALBs for supported Ingress resources and NLBs for supported
Service configurations.

## 6.9.7 Autoscaling

Two dimensions must be separated.

**HPA** changes pod replica count based on resource or supported
custom/external metrics.

**Cluster Autoscaler or Karpenter** provisions/adjusts EC2 worker-node
capacity when pods cannot be scheduled.

``` text
Traffic rises -> HPA adds pods -> scheduler cannot place all pods
              -> pods Pending -> node autoscaler adds nodes
              -> scheduler places pods
```

  -----------------------------------------------------------------------
  Mechanism                           Scales
  ----------------------------------- -----------------------------------
  HPA                                 Pods

  Cluster Autoscaler                  Node-group capacity based on
                                      unschedulable pods

  Karpenter                           Provisions/consolidates nodes based
                                      on workload needs and constraints
  -----------------------------------------------------------------------

With EKS Fargate, HPA can still scale pods, but you do not manage EC2
worker-node autoscaling for those Fargate pods.

## 6.9.8 IAM and security

Separate: 1. AWS IAM: AWS API permissions and identity. 2. Kubernetes
RBAC: permissions inside Kubernetes. 3. Pod identity: AWS permissions
for application workloads.

EKS supports workload IAM through IRSA and EKS Pod Identity. Avoid
long-lived keys in containers; use least-privilege workload roles.

## 6.9.9 Storage

  Kubernetes concept      Meaning
  ----------------------- ------------------------------------
  PersistentVolume        Storage resource
  PersistentVolumeClaim   Workload storage request
  StorageClass            Dynamic provisioning configuration
  CSI driver              Storage-system integration

AWS integrations include EBS CSI, EFS CSI, and supported FSx
integrations.

EBS is AZ-scoped and suited to block-storage use cases. EFS supports
shared filesystem access across AZs. For EKS Fargate, EFS is the
supported persistent filesystem integration; EBS volumes are not
supported for Fargate pods.

## 6.9.10 Observability

Common options: - CloudWatch Observability add-on and Container
Insights. - Fluent Bit for log forwarding. - Prometheus-compatible
metrics and Grafana. - OpenTelemetry and tracing backends.

Applications commonly write logs to stdout/stderr. An agent or
integration forwards them to CloudWatch Logs or another destination.
Collection must be configured; logs are not automatically collected from
every resource.

## 6.9.11 ECS vs EKS

  -----------------------------------------------------------------------
  Consideration           ECS                     EKS
  ----------------------- ----------------------- -----------------------
  Orchestrator            AWS-native              Kubernetes

  Workload definition     Task                    Kubernetes
                          definitions/services    manifests/APIs

  Ecosystem               AWS integrations        Broad Kubernetes
                                                  ecosystem

  Operational concepts    Generally fewer         More Kubernetes
                                                  concepts

  Portability             More AWS-specific       Kubernetes API
                                                  compatibility

  Compute                 EC2 or Fargate          EC2 or Fargate for
                                                  eligible pods
  -----------------------------------------------------------------------

Choose ECS when AWS-native orchestration is sufficient and simplicity is
valuable. Choose EKS when Kubernetes APIs, ecosystem tooling, existing
platforms, or portability justify the additional complexity.

**One sentence:** EKS provides an AWS-managed Kubernetes control plane;
you still manage workloads and choose EC2 for node control or Fargate to
avoid managing worker nodes.

------------------------------------------------------------------------

# 6.10 --- Cross-service architecture patterns

## Pattern A: API plus asynchronous work

``` text
Client -> API Gateway -> Lambda/ECS API -> Database
                              |
                              +-> SQS -> Worker
```

Use when the API should respond quickly and background processing can
finish later.

## Pattern B: Pub/sub fan-out

``` text
Order service -> SNS topic -> SQS Inventory -> Inventory worker
                          -> SQS Email -> Email worker
                          -> SQS Analytics -> Analytics worker
```

Use when independent consumers each need a durable backlog.

## Pattern C: Content-aware event routing

``` text
Services -> EventBridge
              +-> PaymentFailed -> Alert workflow
              +-> OrderCreated -> Inventory queue
              +-> ShipmentCreated -> Notification
```

Use when event content determines routing.

## Pattern D: Orchestrated workflow

``` text
Order -> Step Functions -> Validate -> Charge -> Reserve -> Ship
                                  failure -> compensate/reconcile
```

Use when business processes need explicit state, branching, retries,
timeouts, and compensation.

## Pattern E: Containerized microservices

``` text
Internet -> ALB -> ECS payment service
                 -> ECS order service
                 -> ECS worker service -> SQS
```

Choose ECS or EKS for orchestration; choose EC2 or Fargate for compute
separately.

------------------------------------------------------------------------

# 6.11 --- High-value comparisons and exam traps

## Lambda vs Fargate

  -----------------------------------------------------------------------
  Lambda                              Fargate
  ----------------------------------- -----------------------------------
  Function invocation                 Container task/pod

  Maximum 15-minute invocation        Long-running containers

  Invocation/concurrency model        Orchestrator controls task/pod
                                      count

  AWS manages execution environment   AWS manages underlying container
                                      compute
  -----------------------------------------------------------------------

## SQS vs SNS vs EventBridge

  Service       Question
  ------------- -----------------------------------------
  SQS           How do I buffer and distribute work?
  SNS           How do I broadcast to subscribers?
  EventBridge   How do I route events based on content?

## ECS vs EKS

  ECS                                      EKS
  ---------------------------------------- ------------------------------------
  AWS-native orchestrator                  Managed Kubernetes
  Task definitions/services                Pods/Deployments/Services/HPA
  Application Auto Scaling                 HPA plus optional node autoscaling
  Smaller Kubernetes operational surface   Broad Kubernetes ecosystem

## Common exam traps

1.  API keys are not strong authentication.
2.  SQS receive does not delete a message.
3.  Standard SQS is at-least-once; duplicates are possible.
4.  FIFO ordering is within a message group.
5.  SNS subscribers get independent deliveries; consumers on one SQS
    queue compete.
6.  EventBridge is not a durable work-queue replacement.
7.  EventBridge rules must match an event.
8.  Step Functions workflow semantics do not guarantee exactly-once
    external side effects.
9.  Fargate is compute, not orchestration.
10. ECS Fargate behind ALB uses IP target groups.
11. HPA scales pods, not EC2 nodes.
12. EKS Fargate does not support DaemonSets.
13. IAM permission and network connectivity are separate.
14. CloudWatch Logs is a common destination, not automatic collection
    for every resource.
15. A DLQ isolates repeated failures; it does not fix the root cause.

------------------------------------------------------------------------

# 6.12 --- Architecture decision trees

## Compute

``` text
Need to run code?
  |
  +-- Short event-driven function -> Lambda
  |
  +-- Containerized workload
         |
         +-- Need host/GPU/privileged control -> EC2
         |
         +-- No host control needed -> Fargate
```

## Messaging

``` text
Need asynchronous communication?
  |
  +-- Durable work backlog -> SQS
  |
  +-- Broadcast to subscribers -> SNS
  |
  +-- Content-based routing -> EventBridge
```

These services can be combined. For example, EventBridge can route to
SQS; SNS can fan out to multiple SQS queues.

## Container orchestration

``` text
Need Kubernetes APIs/ecosystem?
  |
  +-- Yes -> EKS
  |
  +-- No, AWS-native orchestration is enough -> ECS

Then choose compute:
  +-- Host-level control / specialized hardware -> EC2
  +-- Avoid worker-host management -> Fargate
```

------------------------------------------------------------------------

# 6.13 --- Production engineering principles

## 1. Design for duplicate delivery

Retries and distributed messaging can cause the same logical operation
to be attempted more than once.

Use idempotency keys, unique business identifiers, conditional writes,
and safe retry semantics.

## 2. Separate acceptance from completion

An API may accept an order while background processing continues. Make
the API contract clear: accepted does not mean every downstream action
is complete.

## 3. Protect downstream systems

Autoscaling consumers can overwhelm databases and external APIs. Use
concurrency limits, queue buffering, rate limits, connection pooling,
back-pressure, and retry budgets.

## 4. Make failures observable

Monitor: - Queue depth and oldest-message age. - Lambda errors,
throttles, duration, and concurrency. - ECS desired vs running task
count. - EKS pending pods and resource pressure. - Workflow failures and
timeouts. - DLQ counts. - API latency and error rates.

## 5. Treat deployment as a distributed-systems problem

Old and new application versions may run simultaneously. Use
backward-compatible APIs and schema changes, graceful shutdown, health
checks, and rollback strategies.

## 6. Apply least privilege

Separate infrastructure roles from application roles. Avoid long-lived
credentials in functions and containers.

------------------------------------------------------------------------

# 6.14 --- Final revision sheet

  -----------------------------------------------------------------------
  Service                 Core purpose            Remember
  ----------------------- ----------------------- -----------------------
  Lambda                  Function compute        Invocation model,
                                                  concurrency, 15-minute
                                                  limit

  API Gateway             API front door          Auth, throttling,
                                                  stages, integrations

  SQS                     Durable queue           Visibility timeout,
                                                  deletion, DLQ,
                                                  idempotency

  SNS                     Pub/sub                 Topics, subscriptions,
                                                  filters

  EventBridge             Event routing           Buses, patterns,
                                                  targets, archives

  Step Functions          Workflow orchestration  State machines, retry,
                                                  catch, Saga

  Fargate                 Serverless container    No worker-host
                          compute                 management

  ECS                     AWS-native container    Cluster, task
                          orchestration           definition, task,
                                                  service

  EKS                     Managed Kubernetes      Control plane, nodes,
                                                  pods, HPA
  -----------------------------------------------------------------------

## Module 6 mental model

-   Lambda runs a function.
-   API Gateway exposes and manages an API.
-   SQS queues work.
-   SNS broadcasts notifications.
-   EventBridge routes events.
-   Step Functions orchestrates workflows.
-   Fargate supplies container compute.
-   ECS orchestrates AWS-native containers.
-   EKS orchestrates Kubernetes workloads.

> Compute executes work. Orchestration decides what should run.
> Messaging decouples components. Workflow engines coordinate business
> processes. API gateways manage external access.

------------------------------------------------------------------------

# 6.15 --- Review checklist

Before moving on, be able to explain without notes:

1.  Why choose Lambda over Fargate, or vice versa?
2.  What differs between SQS, SNS, and EventBridge?
3.  Why does SQS need a visibility timeout?
4.  What differs between SNS and SQS DLQs?
5.  How does EventBridge differ from Kafka?
6.  Why is a transactional outbox useful?
7.  What differs between Step Functions Standard and Express?
8.  Why do external workflow side effects need idempotency?
9.  What differs between ECS task definition, task, and service?
10. What does Fargate remove from ECS operations?
11. How does ECS service auto scaling differ from capacity-provider
    behavior?
12. Why does an ALB target group for Fargate use IP targets?
13. What does EKS manage, and what remains your responsibility?
14. What differs between HPA and Cluster Autoscaler/Karpenter?
15. Why can't EKS Fargate run DaemonSets?
16. How do IRSA and EKS Pod Identity secure workloads?
17. When would you choose EKS over ECS?

**Completion checkpoint:** You should be able to design an application
using API Gateway or ALB for ingress, Lambda or containers for compute,
SQS/SNS/EventBridge for asynchronous communication, Step Functions for
workflows, ECS/EKS for orchestration, and CloudWatch for observability.

**Next module:** AWS Database Services --- RDS, Aurora, DynamoDB,
ElastiCache, database selection, replication, consistency, backups, and
production database architecture.
