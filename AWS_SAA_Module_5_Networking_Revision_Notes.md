# AWS SAA Revision Notes -- Module 5: Networking

## Topics Covered

-   VPC, CIDR, Subnets
-   Route Tables
-   Internet Gateway
-   NAT Gateway
-   VPC Peering
-   Transit Gateway
-   Site-to-Site VPN
-   Direct Connect
-   Route 53 (Hosted Zones, Records, Alias, Apex)
-   Routing Policies & Health Checks
-   CloudFront
-   Global Accelerator

## Key Decisions

-   DNS → Route53
-   CDN → CloudFront
-   Dynamic global traffic → Global Accelerator
-   Outbound Internet for private subnet → NAT Gateway
-   Multi-VPC → Transit Gateway
-   Dedicated connectivity → Direct Connect

## Common Traps

-   Public subnet ≠ Public EC2
-   Peering is non-transitive
-   Direct Connect is not encrypted by default
-   CloudFront caches; Global Accelerator does not
-   Apex cannot be a CNAME

## One Sentence

Networking is about securely connecting resources and directing users to
the optimal endpoint.
