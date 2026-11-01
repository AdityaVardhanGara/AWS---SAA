# AWS SAA --- Module 9 Revision Notes

## Architecture, Migration & Cost Optimization

These notes consolidate the four Module 9 submodules. Dedicated exam
preparation and mock tests remain separate.

------------------------------------------------------------------------

# 9.1 Architecture & Well-Architected Design

## AWS Well-Architected Framework

  -----------------------------------------------------------------------
  Pillar                  Focus                   Examples
  ----------------------- ----------------------- -----------------------
  Operational Excellence  Operate and improve     CloudWatch,
                          workloads               CloudFormation, Systems
                                                  Manager

  Security                Protect systems and     IAM, KMS, WAF, least
                          data                    privilege

  Reliability             Recover from failures   Multi-AZ, Auto Scaling,
                                                  Route 53, backups

  Performance Efficiency  Use resources           EC2, Lambda,
                          efficiently             CloudFront, ElastiCache

  Cost Optimization       Avoid unnecessary       Savings Plans, Spot,
                          spending                lifecycle policies

  Sustainability          Reduce environmental    Efficient utilization,
                          impact                  managed services,
                                                  Graviton where suitable
  -----------------------------------------------------------------------

Availability concerns whether a system can be accessed. Reliability also
includes correct operation and recovery from failures.

## Core architecture principles

-   **Design for failure:** instances, AZs, databases and networks can
    fail. Use redundancy, health checks, failover, backups and recovery
    plans.
-   **Decouple components:** SQS for queues, SNS for pub/sub,
    EventBridge for event routing, Step Functions for workflow
    orchestration.
-   **Scale horizontally:** add instances/workers rather than relying
    only on larger servers.
-   **Use managed services where appropriate:** RDS, SQS and Lambda can
    reduce operational work, but are not automatically cheaper or always
    suitable.
-   **Automate:** infrastructure as code, deployments, monitoring and
    recovery.

## Availability and deployment scope

  Concept             Meaning
  ------------------- --------------------------------------------------
  Region              Independent geographic AWS infrastructure area
  Availability Zone   Isolated infrastructure location within a Region
  Multi-AZ            Resources deployed across multiple AZs
  Multi-Region        Resources deployed across multiple Regions
  Fault tolerance     Continue operating despite component failures
  Disaster recovery   Restore service after a major disruption

Typical highly available architecture:

``` text
Users → Route 53 → CloudFront/WAF → ALB
                                  |
                         +--------+--------+
                         |                 |
                        AZ-A              AZ-B
                       EC2/ECS           EC2/ECS
                         +--------+--------+
                                  |
                              RDS/Aurora
```

-   ALB distributes HTTP/HTTPS requests across healthy targets.
-   Auto Scaling adjusts fleet capacity and can replace unhealthy
    instances.
-   Configure ELB health checks in the Auto Scaling group when
    replacement based on target health is required.
-   RDS Multi-AZ primarily provides high availability/failover, not
    ordinary read scaling.
-   Read replicas primarily support read scaling and can support certain
    DR designs.
-   Multi-Region is generally more complex and expensive than Multi-AZ.

## RTO and RPO

-   **RTO:** Maximum acceptable time to restore service.
-   **RPO:** Maximum acceptable data-loss window, expressed as time.

Example: RTO 30 minutes means restore service within 30 minutes; RPO 5
minutes means tolerate at most 5 minutes of data loss.

## Load balancing

  -------------------------------------------------------------------------
  Feature           ALB               NLB                 GWLB
  ----------------- ----------------- ------------------- -----------------
  Layer             7                 4                   Network traffic
                                                          inspection
                                                          architecture

  Main use          HTTP/HTTPS apps   TCP/UDP/TLS traffic Security
                                                          appliance fleets

  Path/host routing Yes               No                  No
                                      application-level   
                                      routing             

  Static IP         Not typical ALB   Static IP per AZ;   Gateway Load
                    model             Elastic IP          Balancer
                                      association         endpoints
                                      supported           

  Key feature       Content-aware     Low latency/high    GENEVE
                    routing           throughput          encapsulation
  -------------------------------------------------------------------------

Choose ALB for `/payments` vs `/users` routing; NLB for static IP and
TCP/UDP needs; GWLB for virtual firewall and inspection appliances.

## EC2 Auto Scaling

-   Minimum capacity: lowest capacity to maintain.
-   Desired capacity: target capacity.
-   Maximum capacity: upper limit.

  -----------------------------------------------------------------------
  Policy                  Behaviour               Example
  ----------------------- ----------------------- -----------------------
  Target tracking         Maintain a target       Average CPU around 50%
                          metric                  

  Step scaling            Different adjustments   Add 2 at 70%, add 4 at
                          for alarm ranges        90%

  Simple scaling          One adjustment per      Add 2 instances
                          alarm                   

  Scheduled scaling       Scale at known times    Increase before
                                                  business hours

  Predictive scaling      Forecast demand from    Prepare for recurring
                          history                 peaks
  -----------------------------------------------------------------------

Load balancer distributes traffic; Auto Scaling changes fleet capacity.

## Caching and content delivery

  -----------------------------------------------------------------------
  Feature                 CloudFront              ElastiCache
  ----------------------- ----------------------- -----------------------
  Purpose                 CDN                     In-memory application
                                                  cache

  Typical data            Images, CSS, JS,        Sessions, hot records,
                          videos, cacheable       query results
                          responses               

  Location                Global edge locations   Regional deployment in
                                                  VPC

  Benefit                 Lower user latency and  Lower
                          origin load             application/database
                                                  latency and load
  -----------------------------------------------------------------------

CloudFront concepts: - Origin: source such as S3 or ALB. - Distribution:
delivery configuration. - Cache behaviour: path matching and
cache/origin rules. - TTL: how long content remains cached. -
Invalidation: request removal of objects from edge caches. - OAC: lets
CloudFront access a private S3 origin without public bucket access.

## Resilient application patterns

**Queue-based load leveling**

``` text
API → SQS → Workers → Database/downstream service
```

Buffers spikes, decouples producers and consumers, and permits
independent scaling.

**Dead-letter queue:** stores messages that repeatedly fail after the
configured receive threshold.

**Retries:** exponential backoff (e.g. 1, 2, 4, 8 seconds) with jitter
and a maximum retry count.

**Idempotency:** repeated execution produces the same intended business
outcome. Use idempotency keys for payment APIs; DynamoDB conditional
writes/records can help. SQS FIFO deduplication applies within its
supported deduplication window.

**Saga:** coordinates distributed business steps and compensating
actions. Step Functions can orchestrate workflows.

**Graceful degradation:** preserve core functionality when a
nonessential dependency fails.

## Disaster recovery strategies

  ---------------------------------------------------------------------------
  Strategy          Description           Relative cost     Recovery
  ----------------- --------------------- ----------------- -----------------
  Backup & Restore  Restore               Lowest            Slowest
                    infrastructure/data                     
                    after disaster                          

  Pilot Light       Keep essential core   Low--medium       Faster
                    components running                      

  Warm Standby      Maintain scaled-down  Medium--high      Faster
                    functional                              
                    environment                             

  Multi-Site        Serve traffic in      Highest           Fastest potential
  Active/Active     multiple locations                      recovery
  ---------------------------------------------------------------------------

Exact RTO/RPO depends on implementation.

### Multi-Region choices

-   **Route 53:** DNS routing policies including latency, weighted,
    failover and geolocation.
-   **Global Accelerator:** Anycast static IPs and routing over AWS
    global network to healthy regional endpoints.

  -----------------------------------------------------------------------
  Aurora Global Database              DynamoDB Global Tables
  ----------------------------------- -----------------------------------
  Relational SQL                      NoSQL key-value/document

  One primary write Region in         Multi-Region, multi-active writes
  standard configuration              

  Secondary Regions read-only until   Replicas can accept writes
  promotion/failover                  

  Asynchronous cross-Region           Asynchronous cross-Region
  replication                         replication

  Global reads and regional DR        Global distributed NoSQL
  -----------------------------------------------------------------------

Asynchronous replication can cause nonzero data loss during failover.
Multi-Region does not automatically mean zero RPO.

------------------------------------------------------------------------

# 9.2 AWS Migration & Data Transfer

## The 6 Rs

  -----------------------------------------------------------------------
  Strategy                Meaning                 Example
  ----------------------- ----------------------- -----------------------
  Rehost                  Move as-is              VM to EC2

  Replatform              Limited optimization    Self-managed PostgreSQL
                                                  to RDS PostgreSQL

  Refactor / Re-architect Redesign application    Monolith to
                                                  microservices

  Repurchase              Replace with another    Self-hosted CRM to SaaS
                          product                 

  Retain                  Keep workload where it  Legacy system retained
                          is                      

  Retire                  Decommission unused     Shut down obsolete
                          application             system
  -----------------------------------------------------------------------

## Server and application migration

**AWS Application Migration Service (MGN)** - Lift-and-shift physical
servers, VMs and cloud servers to AWS. - Install replication agent on
source servers. - Continuously replicate block-level data to a staging
area. - Launch test instances, validate, then cut over to production
instances. - Useful for migrating many servers to EC2 with minimal
redesign.

**AWS Migration Hub** - Centralized tracking across supported migration
tools. - Groups servers into applications/migration groups. - Does not
itself migrate servers or databases.

**AWS Application Discovery Service** - Discovers server inventory,
utilization, software and network dependencies. - Helps plan migration
waves and right-size target infrastructure.

## Database migration

**AWS DMS** - Migrates database data. - Full load copies existing
data. - Change Data Capture (CDC) captures ongoing source changes. -
Typical sequence: configure endpoints → full load → CDC → validate →
cutover.

**Homogeneous vs heterogeneous**

  Type            Example                          Schema conversion
  --------------- -------------------------------- ----------------------
  Homogeneous     PostgreSQL → RDS PostgreSQL      Usually not required
  Homogeneous     MySQL → RDS MySQL                Usually not required
  Heterogeneous   Oracle → PostgreSQL              Usually required
  Heterogeneous   SQL Server → Aurora PostgreSQL   Usually required

**SCT / DMS Schema Conversion** - Converts supported database objects
and code: tables, indexes, data types, procedures, views, functions. -
Some objects need manual conversion. - Schema conversion handles
structure/code; DMS moves data and ongoing changes.

Typical flow:

``` text
Oracle → SCT / DMS Schema Conversion → PostgreSQL schema
       → DMS full load + CDC → RDS/Aurora PostgreSQL
```

## Data transfer and synchronization

**AWS DataSync** - Managed movement/synchronization of file and object
data. - Examples: NFS/SMB to S3, EFS or supported FSx; supported AWS
storage transfers. - Supports scheduled/incremental transfers, integrity
verification, parallelized movement and bandwidth controls. - Not a
database-aware migration engine.

**AWS Transfer Family** - Managed SFTP, FTPS, FTP and AS2 endpoints. -
Common use: partners upload files directly to S3 or EFS.

**S3 Transfer Acceleration** - Accelerates uploads to S3 for
geographically distant clients using nearby AWS edge locations and the
AWS network.

Distinguish: CloudFront = content delivery/cache; Transfer Acceleration
= faster S3 uploads; DataSync = managed bulk transfer/synchronization.

## Storage Gateway

  Gateway           Interface              Main use
  ----------------- ---------------------- ----------------------------------
  S3 File Gateway   NFS / SMB              File access backed by S3 objects
  Volume Gateway    iSCSI                  Block storage and volume backups
  Tape Gateway      Virtual Tape Library   Replace physical tape backup

Volume Gateway: - Cached volumes: primary data in S3, frequently
accessed data cached locally. - Stored volumes: primary data stored
locally, asynchronous backups to AWS.

## Snow Family

  -----------------------------------------------------------------------
  Device                              Typical use
  ----------------------------------- -----------------------------------
  Snowcone                            Small portable edge computing/data
                                      transfer

  Snowball Edge                       Large-scale offline transfer and
                                      supported edge computing

  Snowmobile                          Extremely large data-center-scale
                                      transfer, subject to availability
  -----------------------------------------------------------------------

Use when online transfer is impractical or too slow. Physical workflow:
request → receive → copy data → return → AWS imports data.

## Hybrid connectivity

  -----------------------------------------------------------------------
  Feature                 Site-to-Site VPN        Direct Connect
  ----------------------- ----------------------- -----------------------
  Connectivity            IPsec tunnels over      Dedicated network
                          internet                connection

  Encryption              IPsec encrypted         Not encrypted by
                                                  default

  Setup                   Usually faster          Can take longer

  Performance             Depends on internet     More consistent
                          path                    

  Use                     Quick hybrid            High-volume enterprise
                          connection/backup       connectivity
  -----------------------------------------------------------------------

Direct Connect can be combined with VPN for encryption. A common design
uses Direct Connect as primary and VPN as backup.

## Migration service selection

  Requirement                       Service
  --------------------------------- -------------------------------
  Lift-and-shift servers to EC2     MGN
  Track migration progress          Migration Hub
  Discover server dependencies      Application Discovery Service
  Migrate database data             DMS
  Convert database schema/code      SCT / DMS Schema Conversion
  Replicate database changes        DMS with CDC
  Transfer NFS/SMB files            DataSync
  Managed SFTP/FTPS/FTP/AS2         Transfer Family
  Accelerate distant S3 uploads     S3 Transfer Acceleration
  S3-backed NFS/SMB                 S3 File Gateway
  Hybrid iSCSI block storage        Volume Gateway
  Virtual tape backup               Tape Gateway
  Physical large-scale transfer     Snow Family
  Encrypted internet connectivity   Site-to-Site VPN
  Dedicated network connectivity    Direct Connect

Migration sequence: discover → assess/choose 6R → establish landing
zone/connectivity → group waves → replicate → test → cut over →
validate/rollback plan → decommission source only after approval.

Continuous replication does not guarantee zero downtime. Coordinate
application writes and database cutover to avoid inconsistent data.

------------------------------------------------------------------------

# 9.3 AWS Cost Optimization

## EC2 purchasing models

  ---------------------------------------------------------------------------
  Model                   Behaviour               Suitable workload
  ----------------------- ----------------------- ---------------------------
  On-Demand               Pay for usage, no       Unpredictable/short-lived
                          long-term commitment    

  Reserved Instances      1- or 3-year billing    Stable matching usage
                          commitment              

  Savings Plans           Commit to eligible      Predictable compute spend
                          spend per hour          

  Spot                    Spare capacity; can be  Fault-tolerant
                          interrupted             batch/flexible workers

  Dedicated Instances     Hardware dedicated to   Isolation requirements
                          one customer            

  Dedicated Hosts         Physical server         Licensing/compliance
                          dedicated with          needing host control
                          host-level              
                          visibility/control      
  ---------------------------------------------------------------------------

## Savings Plans vs Reserved Instances

  -----------------------------------------------------------------------
  Feature                 Savings Plans           Reserved Instances
  ----------------------- ----------------------- -----------------------
  Commitment              Dollar amount of        Matching instance
                          eligible usage per hour attributes/capacity
                                                  type

  Term                    1 or 3 years            1 or 3 years

  Payment                 No, Partial, All        No, Partial, All
                          Upfront                 Upfront

  Flexibility             Depends on plan         Depends on RI type
  -----------------------------------------------------------------------

-   **Compute Savings Plans:** broad flexibility across EC2, Fargate and
    Lambda.
-   **EC2 Instance Savings Plans:** targeted to an EC2 instance family
    in a Region.
-   **Standard RIs:** generally higher discount potential, less
    flexibility.
-   **Convertible RIs:** can be exchanged for configurations with
    different attributes, subject to AWS rules.

### Billing discount vs capacity reservation

-   Billing discount reduces matching usage charges.
-   Capacity reservation reserves EC2 capacity in a specified AZ.
-   Regional RI generally provides a discount but not AZ-specific
    capacity.
-   Zonal RI includes capacity reservation for matching configuration.
-   On-Demand Capacity Reservations reserve capacity without a long-term
    pricing commitment.

## Spot Instances

-   Use spare EC2 capacity at discounts.
-   Can be interrupted; many interruption scenarios provide a two-minute
    notice.
-   Use checkpoints/restartable work, retries and queues.
-   Spot Fleet / EC2 Fleet can request capacity across instance types
    and AZs.
-   Good for batch, CI/CD, analytics, stateless workers, rendering.
-   Avoid relying exclusively on Spot for interruption-intolerant
    workloads.

## S3 storage cost optimization

  -----------------------------------------------------------------------
  Storage class           Access pattern          Considerations
  ----------------------- ----------------------- -----------------------
  S3 Standard             Frequent access         No retrieval fee;
                                                  higher storage cost

  S3 Intelligent-Tiering  Unknown/changing access Monitoring/automation
                                                  charge; automatic
                                                  tiering for eligible
                                                  objects

  S3 Standard-IA          Infrequent, rapid       Retrieval charges;
                          retrieval               30-day minimum

  S3 One Zone-IA          Infrequent, recreatable Single AZ; retrieval
                                                  charges; 30-day minimum

  Glacier Instant         Rare access,            Retrieval charges;
  Retrieval               millisecond retrieval   90-day minimum

  Glacier Flexible        Archive,                Retrieval charges;
  Retrieval               minutes-to-hours        90-day minimum
                          options                 

  Glacier Deep Archive    Long-term archive       Lowest storage cost;
                                                  typically hours to
                                                  retrieve; 180-day
                                                  minimum
  -----------------------------------------------------------------------

Lifecycle example:

``` text
Day 0       → S3 Standard
After 30d   → S3 Standard-IA
After 90d   → S3 Glacier Flexible Retrieval
After 365d  → S3 Glacier Deep Archive
After 7 yrs → Expire/delete if retention permits
```

Consider transition requests, retrieval fees, minimum storage duration,
early deletion charges, object size and versioned bucket lifecycle
rules.

Intelligent-Tiering automatically moves eligible objects based on access
patterns. Monitoring/automation charges apply; small objects may not
benefit.

## EBS, EFS and FSx

  EBS volume          Typical use
  ------------------- ---------------------------------------------------
  gp3                 General-purpose SSD; configurable IOPS/throughput
  gp2                 Older general-purpose SSD
  io2 Block Express   High-performance IOPS-intensive
  st1                 Throughput-intensive HDD
  sc1                 Low-cost infrequently accessed HDD

EBS optimization: - Right-size volumes. - Consider gp3 where
requirements fit; IOPS and throughput can be configured independently of
size within supported limits. - Delete unattached volumes and obsolete
snapshots. - Apply backup retention.

EFS provides Standard and IA storage classes and lifecycle management.
One Zone classes may cost less but offer lower AZ resilience. FSx
provides managed file systems such as Windows, Lustre, NetApp ONTAP and
OpenZFS.

Remember: S3 = object; EBS = block; EFS = managed file; FSx =
specialized managed file systems.

## Networking costs

Potential costs: - Internet data transfer out. - Cross-AZ and
cross-Region transfer. - NAT Gateway hourly and per-GB processing. -
Public IPv4 address charges. - Load balancer hourly/usage charges. -
CloudFront requests and transfer. - Direct Connect port and data
transfer charges, depending on configuration.

**NAT optimization:** For private EC2 instances accessing S3, use an S3
Gateway Endpoint and route S3 traffic through it rather than NAT.

-   S3 and DynamoDB Gateway Endpoints have no additional endpoint hourly
    or data-processing charges.
-   Interface Endpoints use PrivateLink and have hourly and
    data-processing charges.

CloudFront optimization: - Set appropriate TTLs. - Avoid forwarding
unnecessary cookies, headers and query strings. - Use compression. -
Reduce repeated origin requests.

Do not sacrifice required availability merely to reduce cross-AZ costs.

## Cost tools

  Tool                 Purpose
  -------------------- ---------------------------------------
  Cost Explorer        Analyze historical spend and forecast
  AWS Budgets          Set cost/usage thresholds and alerts
  CUR / Data Exports   Detailed billing and usage records
  Compute Optimizer    Resource sizing recommendations
  Trusted Advisor      Broader best-practice recommendations

-   Cost Explorer filters by service, Region, account and tags; supports
    comparisons and forecasts.
-   Budgets support cost/usage and Savings Plans/RI utilization/coverage
    budgets, with alerts and supported actions.
-   CUR/Data Exports deliver detailed billing/usage records to S3 for
    Athena/Redshift analysis.
-   Compute Optimizer recommends configurations for supported EC2, Auto
    Scaling, EBS, Lambda and ECS resources.
-   Trusted Advisor checks cost, performance, security, fault tolerance
    and service limits; availability depends on support plan/current
    features.

### Cost allocation

Use tags such as `Environment=Production`, `Team=Payments`,
`Application=PaymentAPI`.

-   Activate user-defined cost allocation tags in billing settings.
-   Cost categories group spending using business rules.
-   Tag policies help standardize tags across an organization.

------------------------------------------------------------------------

# 9.4 Architecture Scenarios & Design Decisions

## Highly available e-commerce application

``` text
Users
  |
Route 53
  |
CloudFront + WAF
  |
ALB
  |
  +----------------+
  |                |
 AZ-A             AZ-B
 EC2/ECS          EC2/ECS
  +--------+-------+
           |
      Aurora / RDS

Additional: ElastiCache, S3, SQS, CloudWatch
```

  Requirement               Service
  ------------------------- ------------------
  Global static content     CloudFront
  HTTP protection           WAF
  HTTP routing              ALB
  AZ resilience             Multi-AZ compute
  Variable demand           Auto Scaling
  Relational transactions   Aurora/RDS
  Hot data cache            ElastiCache
  Images                    S3

## Event-driven order processing

``` text
Order API
   |
Order DB + Transactional Outbox
   |
EventBridge / SNS
   |
   +-------------+--------------+
   |             |              |
Payment Q    Inventory Q    Notification Q
   |             |              |
Worker         Worker         Worker
```

-   SQS buffers each consumer's work.
-   DLQs isolate repeated failures.
-   Standard SQS is at-least-once; consumers must tolerate duplicates.
-   Use idempotency keys for payment processing.
-   Transactional outbox prevents losing an event between database
    commit and publication.
-   Saga coordinates distributed steps and compensations; Step Functions
    can orchestrate.

Trade-off: decoupling improves isolation and scalability but adds
eventual consistency and failure-handling complexity.

## Global multi-Region application

Route 53 or Global Accelerator can route users to healthy regional
endpoints. Each Region can have its own load balancer and compute fleet.

  -----------------------------------------------------------------------
  Route 53                            Global Accelerator
  ----------------------------------- -----------------------------------
  DNS-based routing                   Anycast static IPs/AWS global
                                      network

  Latency, weighted, failover,        Routes to healthy regional
  geolocation                         endpoints

  DNS caching affects failover        Traffic shift does not rely on DNS
  behaviour                           record changes
  -----------------------------------------------------------------------

  -----------------------------------------------------------------------
  Aurora Global Database              DynamoDB Global Tables
  ----------------------------------- -----------------------------------
  Relational SQL                      NoSQL

  One primary write Region in         Multi-Region multi-active writes
  standard configuration              

  Secondary Regions read-only until   Replicas accept writes
  promotion                           

  Asynchronous replication            Asynchronous replication
  -----------------------------------------------------------------------

Asynchronous replication can cause nonzero data loss during failover.

## Enterprise migration

  Workload                     Strategy/service
  ---------------------------- ---------------------------------
  Legacy application servers   Rehost with MGN → EC2
  Oracle database              SCT/DMS Schema Conversion + DMS
  NFS files                    DataSync → EFS/S3
  New microservices            Refactor to ECS/Lambda
  Unused application           Retire

Sequence: discover → establish landing zone/connectivity → group
migration waves → replicate → test → cut over → validate → decommission
after approval.

## Cost-optimized analytics

``` text
Logs/events
    |
Kinesis Data Streams or Data Firehose
    |
S3 data lake
    |
Glue catalog / ETL
    |
Athena for SQL queries
    |
S3 Lifecycle for archival
```

  Requirement                         Service
  ----------------------------------- ------------------------
  Stream ingestion/custom consumers   Kinesis Data Streams
  Managed delivery to S3              Data Firehose
  Durable data lake                   S3
  Catalog and ETL                     Glue
  Serverless SQL over S3              Athena
  Analytical warehouse                Redshift
  Archive                             S3 lifecycle + Glacier

Cost/performance: - Convert JSON/CSV to Parquet or ORC. - Partition by
useful query dimensions such as date. - Use Athena for intermittent SQL
queries when suitable. - Apply lifecycle policies to older data. -
Select ingestion service based on processing/replay needs.

Athena cannot directly query objects in archival classes that require
restoration. Restore them or keep queryable datasets in online storage
classes.

## Architecture decision framework

Evaluate: 1. Functional requirements. 2. Performance: latency,
throughput and concurrency. 3. Availability: instance, AZ or Region
failure tolerance. 4. Data: SQL/NoSQL, consistency and access patterns.
5. Scaling: predictable or variable demand. 6. Security: isolation,
encryption and access control. 7. Operations: managed vs self-managed.
8. Cost: baseline, peak and transfer costs. 9. Migration: preserve or
redesign existing systems.

Common trade-offs:

  -----------------------------------------------------------------------
  Trade-off               Option A                Option B
  ----------------------- ----------------------- -----------------------
  Availability vs cost    Single AZ               Multi-AZ

  Consistency vs latency  Strong consistency      Eventual consistency

  Operations vs control   Managed services        Self-managed

  Simplicity vs           Synchronous calls       Asynchronous events
  decoupling                                      

  Recovery speed vs cost  Backup & Restore        Warm Standby /
                                                  Active-Active
  -----------------------------------------------------------------------

No AWS service is universally best. Choose based on requirements and
explicit trade-offs.

------------------------------------------------------------------------

# Quick Reference --- Service Selection

  -----------------------------------------------------------------------
  Requirement                         Think of
  ----------------------------------- -----------------------------------
  HTTP path/host routing              ALB

  Static IP and TCP/UDP balancing     NLB

  Security appliance inspection       GWLB

  Maintain target metric              Auto Scaling target tracking

  Global static content               CloudFront

  Application data cache              ElastiCache

  Durable asynchronous buffer         SQS

  Pub/sub fan-out                     SNS

  Event filtering/routing             EventBridge

  Workflow orchestration              Step Functions

  Lift-and-shift servers              MGN

  Track migration                     Migration Hub

  Discover server dependencies        Application Discovery Service

  Database data migration             DMS

  Schema conversion                   SCT / DMS Schema Conversion

  File synchronization                DataSync

  Managed SFTP/FTPS/FTP/AS2           Transfer Family

  Accelerated distant S3 upload       S3 Transfer Acceleration

  S3-backed NFS/SMB                   S3 File Gateway

  Hybrid iSCSI block storage          Volume Gateway

  Virtual tape backup                 Tape Gateway

  Physical large-scale transfer       Snow Family

  Encrypted internet hybrid           Site-to-Site VPN
  connection                          

  Dedicated network                   Direct Connect

  Flexible compute commitment         Compute Savings Plans

  Stable EC2 family commitment        EC2 Instance Savings Plans

  Interruptible low-cost compute      Spot

  Guaranteed capacity in AZ           Capacity Reservation / applicable
                                      Zonal RI

  Analyze spending                    Cost Explorer

  Budget alerts                       AWS Budgets

  Detailed billing                    CUR / Data Exports

  Resource sizing                     Compute Optimizer

  Broad best-practice checks          Trusted Advisor
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Module 9 Completion Checklist

## 9.1 Architecture

-   [ ] Explain the six Well-Architected pillars.
-   [ ] Design for failure, decoupling and horizontal scaling.
-   [ ] Distinguish ALB, NLB and GWLB.
-   [ ] Explain Auto Scaling policies.
-   [ ] Distinguish CloudFront from ElastiCache.
-   [ ] Explain queues, DLQs, retries and idempotency.
-   [ ] Understand Multi-AZ, Multi-Region, RTO and RPO.
-   [ ] Compare disaster recovery strategies.

## 9.2 Migration

-   [ ] Understand the 6 Rs.
-   [ ] Know MGN, Migration Hub and Application Discovery Service.
-   [ ] Understand DMS full load and CDC.
-   [ ] Distinguish homogeneous and heterogeneous migration.
-   [ ] Distinguish SCT from DMS.
-   [ ] Know DataSync and Transfer Family.
-   [ ] Distinguish Storage Gateway types.
-   [ ] Understand Snow Family.
-   [ ] Distinguish Direct Connect from Site-to-Site VPN.

## 9.3 Cost Optimization

-   [ ] Compare On-Demand, RIs, Savings Plans and Spot.
-   [ ] Distinguish billing discounts from capacity reservations.
-   [ ] Select S3 storage classes and lifecycle policies.
-   [ ] Understand EBS, EFS and FSx cost considerations.
-   [ ] Identify NAT, cross-AZ and cross-Region costs.
-   [ ] Know Cost Explorer, Budgets, CUR, Compute Optimizer and Trusted
    Advisor.
-   [ ] Understand cost allocation tags and categories.

## 9.4 Architecture Scenarios

-   [ ] Design a highly available web application.
-   [ ] Design event-driven order processing.
-   [ ] Compare global routing and multi-Region data options.
-   [ ] Plan an enterprise migration.
-   [ ] Design a cost-conscious analytics platform.
-   [ ] Explain architectural trade-offs.

------------------------------------------------------------------------

**End of Module 9 --- Architecture, Migration & Cost Optimization.**

Dedicated exam preparation, cross-module revision and mock tests are
intentionally deferred to the final preparation phase.
