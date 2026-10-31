# AWS SAA-C03 --- Module 7: Database Services

## Exam Revision Notes

**Scope:** Database fundamentals, RDS, Aurora, DynamoDB, ElastiCache,
specialized databases, architecture, migration, security and operations.

**Revision strategy:** Focus on AWS-specific features, service
selection, limits/behaviours and exam traps. Distributed-systems
fundamentals are intentionally summarized.

------------------------------------------------------------------------

# 7.1 Database Fundamentals & Selection

  -----------------------------------------------------------------------
  Database type           Best suited for         AWS service
  ----------------------- ----------------------- -----------------------
  Relational / SQL        Transactions, joins,    RDS, Aurora
                          structured data         

  Key-value / document    High-scale key-based    DynamoDB
                          access                  

  In-memory               Low-latency caching     ElastiCache

  Durable in-memory       Low-latency primary     MemoryDB
                          data store              

  Document                MongoDB-compatible      DocumentDB
                          documents               

  Graph                   Highly connected data   Neptune

  Wide-column             Cassandra-style         Keyspaces
                          workloads               

  Time-series             Timestamped             Timestream
                          metrics/events          

  Data warehouse          OLAP / analytical       Redshift
                          queries                 
  -----------------------------------------------------------------------

**OLTP vs OLAP:** RDS/Aurora are common OLTP choices; Redshift is a data
warehouse for analytical workloads. Avoid running heavy analytics
directly against a production transactional database when a separate
analytical workload is appropriate.

Choose a database based on access patterns, consistency/transaction
needs, scale, latency, compatibility and operational requirements.

------------------------------------------------------------------------

# 7.2 Amazon RDS

## Core facts

-   Managed relational database service; engines include PostgreSQL,
    MySQL, MariaDB, Oracle, SQL Server and Db2.
-   Runs in a VPC, normally in private subnets.
-   DB subnet group spans subnets in multiple AZs.
-   Use DNS endpoints, not hardcoded IP addresses; endpoints can resolve
    to a new address after failover.
-   Restrict DB security-group inbound traffic to the application
    security group.

## Instance and storage

  Family   Purpose
  -------- ------------------
  M        General purpose
  R        Memory optimized
  T        Burstable

Storage options include General Purpose SSD (gp2/gp3) and Provisioned
IOPS SSD (io1/io2, where supported).

-   IOPS = I/O operations per second.
-   Throughput = data transferred per second.
-   Latency = time for an I/O operation.
-   Storage autoscaling increases storage capacity; it does not fix
    CPU/memory bottlenecks.

## Multi-AZ vs Read Replicas

  Feature             Traditional Multi-AZ DB instance   Read replica
  ------------------- ---------------------------------- ----------------------------
  Purpose             High availability                  Read scaling
  Replication         Synchronous standby                Generally asynchronous
  Standby readable?   No, not for normal reads           Yes
  Failover            Automatic                          Promotion generally manual
  Endpoint            DB endpoint follows primary        Separate endpoint

Some engines support Multi-AZ DB clusters with readable instances; do
not confuse them with the traditional standby deployment.

Read replicas can lag. Multi-AZ and read replicas can be used together.

## Backups

-   Automated backups + transaction logs enable point-in-time recovery
    (PITR).
-   RDS automated backup retention: up to 35 days.
-   PITR restores to a **new** DB instance, not an in-place rollback.
-   Manual snapshots are retained until deleted.
-   Cross-Region snapshot copies support regional DR.
-   Backups do not replace HA.

## RDS security and operations

-   KMS encryption at rest; TLS in transit.
-   IAM DB authentication is available for supported
    engines/configurations.
-   IAM policy needs `rds-db:connect`; DB user must be configured; SQL
    grants remain separate.
-   IAM auth token is valid for 15 minutes to establish a new
    connection.
-   Secrets Manager stores credentials and supports rotation for
    supported configurations.

### RDS Proxy

Use for connection pooling/reuse, especially Lambda or connection
storms. It does not cache SQL queries or add database compute. Session
pinning can reduce multiplexing.

------------------------------------------------------------------------

# 7.3 Amazon Aurora

## Architecture

-   AWS-managed relational engine with MySQL-compatible and
    PostgreSQL-compatible editions.
-   Writer/readers have separate compute but share cluster storage.
-   Storage keeps six copies across three AZs and grows automatically.
-   Storage scaling is independent of compute scaling.
-   One writer handles writes at a time; readers provide read capacity
    and can be promoted.
-   The often-taught 4-of-6 write / 3-of-6 read quorum is a simplified
    conceptual model.

## Endpoints

  Endpoint            Purpose
  ------------------- --------------------------------------------
  Cluster endpoint    Current writer; use for writes
  Reader endpoint     Distributes new connections across readers
  Instance endpoint   Specific DB instance

**Exam trap:** Reader endpoint balances connections, not individual SQL
queries. A pooled connection generally stays on the selected reader.
Route read-after-write/freshness-critical reads to the writer if replica
lag matters.

## Availability and scaling

-   Readers can be promoted after writer failure, but connections can
    still break; applications need retries.
-   Promotion tiers influence reader selection.
-   Aurora Serverless v2 scales compute capacity in ACUs (approximately
    2 GiB memory per ACU, with proportional CPU/network).
-   Configure minimum and maximum ACUs.
-   Serverless v2 does not automatically create reader instances.
-   Eligible configurations support auto-pause; availability depends on
    engine/version/configuration.
-   Serverless is not automatically cheaper for steady high utilization.

## Global Database

-   One primary Region handles writes; secondary Regions provide local
    reads.
-   Cross-Region replication is asynchronous and storage-level.
-   Unplanned failover may lose recent transactions not yet replicated.
-   DB failover alone is not complete application DR; routing,
    credentials, KMS, networking and application services must also be
    addressed.

## Aurora backup/security

-   Automated backup retention generally 1--35 days; PITR restores a new
    cluster.
-   Manual snapshots persist until deleted.
-   Enable KMS encryption at cluster creation. To encrypt an existing
    unencrypted cluster: snapshot, copy with encryption, restore a new
    cluster.
-   Use TLS, private subnets and restrictive security groups.

## Aurora vs RDS

Choose Aurora when its distributed storage, reader architecture,
Serverless v2 or Global Database features fit. Choose standard RDS when
a conventional managed relational database is simpler or more
appropriate.

------------------------------------------------------------------------

# 7.4 Amazon DynamoDB

## Core data model

DynamoDB is a fully managed, serverless NoSQL key-value/document
database.

-   Table = collection; item = record; attribute = field.
-   Simple primary key = partition key only.
-   Composite primary key = partition key + sort key.
-   Partition key influences data distribution.
-   Sort key supports multiple related items and ordered/range access.
-   Partition key + sort key combination must be unique.

**Mindset:** Model around application access patterns, not normalized
entities and joins.

## Capacity modes

  Provisioned                   On-demand
  ----------------------------- -----------------------------
  Specify read/write capacity   Pay per request
  Can use auto scaling          Suits unpredictable traffic
  Suits predictable workloads   No manual capacity planning

### RCU/WCU essentials

For items up to 4 KB: - Strongly consistent read: 1 RCU per
read/second. - Eventually consistent read: 0.5 RCU per read/second. -
Transactional read: 2 RCUs per read/second.

For writes up to 1 KB: - Standard write: 1 WCU per write/second. -
Transactional write: 2 WCUs per write/second.

Larger items consume proportionally more units, rounded up. Total
capacity also depends on request rate and operation type.

## Consistency and access

-   Eventually consistent reads are the default.
-   Strongly consistent reads are supported on tables and LSIs.
-   GSIs support eventual consistency only.
-   Global Tables replicate asynchronously across Regions.

  -----------------------------------------------------------------------
  Query                               Scan
  ----------------------------------- -----------------------------------
  Uses partition key and optional     Reads items across table/index
  sort-key conditions                 

  Preferred for targeted access       Can be expensive at scale
  -----------------------------------------------------------------------

A FilterExpression is applied after items are read/evaluated; it does
not reduce read capacity consumed for evaluated items.

## GSI vs LSI

  -----------------------------------------------------------------------
  Feature                 GSI                     LSI
  ----------------------- ----------------------- -----------------------
  Partition key           Can differ from base    Same as base table
                          table                   

  Sort key                Optional                Different sort key

  Created after table     Yes                     No
  creation                                        

  Strongly consistent     No                      Yes
  reads                                           

  Capacity                Separate index capacity Shares base table
                          considerations          capacity
  -----------------------------------------------------------------------

Shortcuts: alternate partition key → GSI; same partition key with
alternate sort key → LSI; add index later → GSI.

## Additional features

  Feature            Purpose / exam clue
  ------------------ ----------------------------------------------------------------
  Streams            Item-level changes; trigger Lambda; 24-hour retention
  TTL                Asynchronous expiry using Unix epoch timestamp in seconds
  Transactions       ACID operations across supported items
  DAX                Managed in-memory cache for DynamoDB; microsecond read latency
  Global Tables      Multi-Region, multi-active replication
  PITR               Restore table to a point in time
  On-demand backup   Backup on request

Hot partitions result from uneven key distribution or concentrated
traffic. Use high-cardinality, well-distributed keys; investigate with
CloudWatch and Contributor Insights.

------------------------------------------------------------------------

# 7.5 ElastiCache & Other Database Services

## ElastiCache

Managed in-memory cache to reduce database load and read latency.

  -----------------------------------------------------------------------
  Feature                 Redis OSS / Valkey      Memcached
  ----------------------- ----------------------- -----------------------
  Data structures         Strings, hashes, lists, Simple key-value
                          sets, sorted sets,      
                          streams                 

  Persistence options     Yes                     No

  Replication             Yes                     No

  HA/failover             Supported               No replication-based
                          configurations          equivalent

  Pub/Sub                 Yes                     No

  Use                     Sessions, counters,     Simple object cache
                          leaderboards, cache     
  -----------------------------------------------------------------------

Valkey is open-source and Redis-compatible.

-   Cluster mode disabled: one shard with primary and optional replicas.
-   Cluster mode enabled: multiple shards, each with primary and
    optional replicas.
-   Replication is asynchronous; failover can lose recent writes.

### Caching patterns

  Pattern         Behaviour
  --------------- --------------------------------------------------------
  Cache-aside     App checks cache; on miss reads DB and populates cache
  Write-through   Update cache and DB in write flow
  Write-behind    Cache accepts write; DB updated asynchronously
  TTL             Entry expires after configured time

Watch for stale cache, cache stampedes, hot keys and invalidation
complexity.

## MemoryDB

Durable Redis OSS-compatible in-memory database, designed to act as a
durable primary data store. Uses a distributed Multi-AZ transaction log.

**Distinction:** ElastiCache is primarily a cache; MemoryDB is a durable
in-memory database.

## Specialized services

  -----------------------------------------------------------------------
  Service                             Use
  ----------------------------------- -----------------------------------
  DocumentDB                          MongoDB-compatible document
                                      database

  Neptune                             Graph database: fraud networks,
                                      recommendations, connected data

  Keyspaces                           Managed Apache Cassandra-compatible
                                      wide-column database; CQL

  Timestream                          Time-series: IoT telemetry,
                                      metrics, timestamped events

  Redshift                            Data warehouse / OLAP
  -----------------------------------------------------------------------

Compatibility does not imply every feature of the original engine is
identical.

------------------------------------------------------------------------

# 7.6 Database Architecture, Migration & Design

## Architecture choices

  Requirement                                Common choice
  ------------------------------------------ -------------------------------
  Relational transactions                    RDS / Aurora
  Relational read scaling                    Read replicas
  AZ-level HA                                Multi-AZ
  Repeated hot reads                         ElastiCache
  Repeated DynamoDB reads                    DAX
  Globally distributed relational reads/DR   Aurora Global Database
  Global key-value application               DynamoDB Global Tables
  Analytics separated from OLTP              Redshift / analytics pipeline
  Lambda connection storms                   RDS Proxy

## HA vs DR

-   HA keeps service available during local infrastructure failures.
-   DR recovers after a larger failure, such as a Region outage.

  DR strategy                Cost           Recovery time
  -------------------------- -------------- ---------------
  Backup and restore         Lowest         Highest
  Pilot light                Low--medium    Medium--high
  Warm standby               Medium--high   Low
  Multi-site active-active   Highest        Lowest

-   RPO = maximum acceptable data loss, expressed as time.
-   RTO = maximum acceptable recovery time.

## Database Migration Service (DMS)

DMS moves data between supported source and target databases.

-   Full load: migrates existing data.
-   CDC: captures ongoing changes.
-   Full load + CDC: migrates existing data and keeps target updated
    before cutover.

  -----------------------------------------------------------------------
  Tool                                Purpose
  ----------------------------------- -----------------------------------
  AWS DMS                             Data movement

  AWS Schema Conversion Tool (SCT)    Schema/code conversion where
                                      required and supported
  -----------------------------------------------------------------------

Example Oracle → PostgreSQL: assess/convert with SCT, migrate with DMS,
use CDC to capture ongoing changes, validate, then cut over.

**Exam trap:** DMS does not automatically convert all schema, stored
procedures and application SQL.

## Performance troubleshooting

  Symptom                   Investigate / consider
  ------------------------- ------------------------------------------
  Too many DB connections   RDS Proxy, connection pooling
  Repeated DB reads         ElastiCache
  Repeated DynamoDB reads   DAX
  Relational read load      Read replicas
  CPU bottleneck            Instance sizing / Serverless v2 capacity
  Storage capacity issue    Storage expansion/autoscaling
  I/O bottleneck            Storage type, IOPS, throughput
  Analytics affects OLTP    Separate analytics / Redshift

------------------------------------------------------------------------

# 7.7 Database Security & Operations

## Encryption

  Type         Protects                           Mechanism
  ------------ ---------------------------------- -----------------------
  At rest      Stored files, snapshots, backups   KMS-backed encryption
  In transit   Network traffic                    TLS

-   KMS manages keys; database service handles transparent
    encryption/decryption.
-   KMS is not called for every SQL query.
-   KMS at rest does not protect traffic in transit; use TLS.
-   For an existing unencrypted RDS/Aurora database, typical conversion
    is snapshot → encrypted copy → restore as new DB.
-   Cross-Region encrypted snapshot copies may need a destination-Region
    KMS key.

## Authentication and authorization

-   Authentication = who are you?
-   Authorization = what can you do?
-   IAM DB auth uses temporary tokens; token is valid for 15 minutes to
    establish a new connection.
-   IAM policy requires `rds-db:connect`; DB user must be configured for
    IAM auth.
-   SQL grants still control database privileges.

## Secrets Manager vs Parameter Store

  -----------------------------------------------------------------------
  Feature                 Secrets Manager         SSM Parameter Store
  ----------------------- ----------------------- -----------------------
  Store secrets           Yes                     Yes

  Built-in rotation       Supported secret types  Usually custom
  integrations                                    automation

  Encryption              KMS                     SecureString uses KMS

  Common use              DB credentials/API      Configuration and
                          secrets                 parameters
  -----------------------------------------------------------------------

Automatic DB credential rotation → Secrets Manager.

## Network security

-   Typical production DB: private subnet, not publicly accessible.
-   Restrict inbound DB port to application security group.
-   Security groups are stateful; NACLs are stateless.
-   Use least-privilege DB users and SQL grants.

## Monitoring and auditing

  Service                                    Purpose
  ------------------------------------------ -----------------------------------------------
  CloudWatch                                 Metrics, logs, alarms
  CloudTrail                                 AWS API activity / control-plane audit
  AWS Config                                 Resource configuration history and compliance
  Performance Insights / Database Insights   DB load and performance
  Enhanced Monitoring                        OS-level metrics
  Engine audit logs                          Database-level activity, when configured

Important metrics: `CPUUtilization`, `FreeableMemory`,
`DatabaseConnections`, `FreeStorageSpace`, `ReadLatency`,
`WriteLatency`, `ReadIOPS`, `WriteIOPS`, `AuroraReplicaLag`.

**Exam trap:** CloudTrail is not a substitute for SQL audit logs.

## Backup operations

-   RDS automated backup retention: up to 35 days.
-   Aurora automated backup retention: generally 1--35 days.
-   PITR restores to a new instance/cluster.
-   Manual snapshots remain until deleted.
-   Multi-AZ is HA, not backup.
-   AWS Backup centralizes backup policies for supported services.

------------------------------------------------------------------------

# High-Value SAA Comparison Tables

## Service selection

  Requirement                                            Choose
  ------------------------------------------------------ -------------
  Managed relational SQL                                 RDS
  AWS-optimized MySQL/PostgreSQL-compatible relational   Aurora
  Serverless key-value/document                          DynamoDB
  General-purpose cache                                  ElastiCache
  DynamoDB cache                                         DAX
  Durable Redis-compatible in-memory DB                  MemoryDB
  MongoDB-compatible documents                           DocumentDB
  Graph relationships                                    Neptune
  Cassandra-compatible wide-column                       Keyspaces
  Time-series                                            Timestream
  Data warehouse                                         Redshift

## Commonly confused pairs

  -----------------------------------------------------------------------
  Pair                                Key distinction
  ----------------------------------- -----------------------------------
  Multi-AZ vs read replica            HA vs read scaling

  Aurora cluster vs reader endpoint   Writer vs reader connections

  Reader endpoint                     Balances connections, not queries

  Automated backup vs snapshot        PITR window vs retained recovery
                                      point

  KMS vs TLS                          At rest vs in transit

  IAM DB auth vs SQL grants           Authentication vs authorization

  Secrets Manager vs Parameter Store  Managed secret rotation vs
                                      parameters

  CloudTrail vs CloudWatch            API audit vs metrics/alarms

  DMS vs SCT                          Move data vs convert schema/code

  GSI vs LSI                          Different partition key vs same
                                      partition key

  Query vs Scan                       Key-based access vs table/index
                                      scan

  DAX vs ElastiCache                  DynamoDB-specific cache vs general
                                      cache

  ElastiCache vs MemoryDB             Cache vs durable in-memory DB

  RPO vs RTO                          Data loss window vs recovery time
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Final Exam Checklist

## RDS / Aurora

-   [ ] Multi-AZ is HA; read replicas are for read scaling.
-   [ ] Traditional RDS Multi-AZ standby is not a normal read target.
-   [ ] Read replicas are generally asynchronous and can lag.
-   [ ] PITR restores to a new DB instance/cluster.
-   [ ] RDS Proxy manages connections; it does not cache queries.
-   [ ] Aurora has shared distributed storage and separate compute.
-   [ ] Aurora reader endpoint distributes connections, not queries.
-   [ ] Aurora Global Database replicates asynchronously across Regions.
-   [ ] Aurora Serverless v2 scales ACUs; it does not automatically add
    readers.

## DynamoDB

-   [ ] Partition key and sort key.
-   [ ] Provisioned vs on-demand.
-   [ ] Basic RCU/WCU calculations.
-   [ ] Eventually vs strongly consistent reads.
-   [ ] Query vs Scan; FilterExpression does not reduce evaluated read
    capacity.
-   [ ] GSI vs LSI.
-   [ ] Streams, TTL, DAX, Global Tables and PITR.
-   [ ] Hot partitions and key distribution.

## Other services

-   [ ] ElastiCache Redis OSS/Valkey vs Memcached.
-   [ ] ElastiCache vs MemoryDB.
-   [ ] DAX vs ElastiCache.
-   [ ] DocumentDB, Neptune, Keyspaces, Timestream and Redshift use
    cases.

## Architecture, security and operations

-   [ ] RPO vs RTO and four DR strategies.
-   [ ] DMS vs SCT; full load vs CDC.
-   [ ] KMS at rest vs TLS in transit.
-   [ ] IAM DB auth and `rds-db:connect`.
-   [ ] Secrets Manager vs Parameter Store.
-   [ ] CloudWatch vs CloudTrail vs AWS Config.
-   [ ] Multi-AZ vs backups and snapshots.

------------------------------------------------------------------------

# One-Page Memory Map

``` text
RELATIONAL
  RDS          -> Managed relational engines
  Aurora       -> AWS-optimized relational engine
  Multi-AZ     -> High availability
  Read replica -> Read scaling
  RDS Proxy    -> Connection pooling
  PITR         -> Restore to a point in time
  Global DB    -> Cross-Region Aurora

NOSQL
  DynamoDB       -> Key-value / document
  GSI            -> Alternate partition key
  LSI            -> Same partition key, alternate sort key
  DAX            -> DynamoDB cache
  Streams        -> Item changes
  TTL            -> Expire items
  Global Tables  -> Multi-Region DynamoDB

IN-MEMORY
  ElastiCache -> Cache
  MemoryDB    -> Durable in-memory database

SPECIALIZED
  DocumentDB -> MongoDB-compatible documents
  Neptune    -> Graph
  Keyspaces  -> Cassandra-compatible wide-column
  Timestream -> Time-series
  Redshift   -> Data warehouse

MIGRATION / OPERATIONS
  DMS          -> Move data
  SCT          -> Convert schema/code
  KMS          -> Encryption at rest
  TLS          -> Encryption in transit
  Secrets Mgr  -> Secret storage/rotation
  CloudWatch   -> Metrics and alarms
  CloudTrail   -> AWS API audit
  AWS Config   -> Configuration/compliance
  AWS Backup   -> Centralized backups
```

**Final takeaway:** Identify the workload first (relational, key-value,
cache, graph, time-series or analytics), then identify the requirement
(HA, read scaling, global access, migration, security or recovery). The
correct AWS service usually follows from those two decisions.
