# AWS SAA Revision Notes -- EC2

## EC2

-   VM service using AMIs.
-   Nitro System handles virtualization and hardware offloading.

## Instance Families

-   M: General Purpose
-   C: Compute
-   R: Memory
-   I: Storage
-   P/G/Inf/Trn: GPU & AI

## EBS

-   Block storage
-   Persistent
-   AZ scoped
-   Snapshots stored in S3 (incremental)
-   AMI = Metadata + Snapshot references

### Types

-   gp3
-   io2
-   st1
-   sc1

## Instance Store

-   Local NVMe
-   Ephemeral
-   High performance

## ENI

-   MAC
-   Private IP
-   Public IP mapping
-   Elastic IP
-   Security Groups

## IPs

-   Private IP: Internal
-   Public IP: Internet
-   Elastic IP: Static public IPv4

## Security Groups

-   Stateful
-   Allow only
-   Attached to ENIs

## NACL

-   Stateless
-   Allow + Deny
-   Attached to subnet

## Route Tables

-   Decide next hop
-   Local route
-   0.0.0.0/0 -\> IGW or NAT

## Internet Gateway

-   Public internet access

## NAT Gateway

-   Outbound internet for private subnets
-   Place in public subnet

## Load Balancers

### ALB

-   Layer 7
-   HTTP routing

### NLB

-   Layer 4
-   TCP/UDP
-   Static IP

### GWLB

-   Firewalls / IDS / IPS

## Auto Scaling Groups

-   Launch Templates
-   Desired / Min / Max
-   Self-healing
-   Target Tracking
-   Step Scaling
-   Scheduled Scaling

## Quick Comparison

### SG vs NACL

  SG           NACL
  ------------ --------------
  Stateful     Stateless
  Allow only   Allow & Deny
  ENI          Subnet

### EBS vs Instance Store

  EBS          Instance Store
  ------------ ----------------
  Persistent   Ephemeral
  Network      Local

### ALB vs NLB vs GWLB

  ALB       NLB       GWLB
  --------- --------- --------------------
  Layer 7   Layer 4   Security Appliance

## Exam Nuggets

-   SG -\> ENI
-   NACL -\> Subnet
-   Snapshot incremental
-   NAT = outbound
-   ALB = HTTP
-   NLB = Performance
-   ASG = Elasticity + Self-healing
