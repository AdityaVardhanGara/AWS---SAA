# AWS SAA Revision Notes -- S3

## Basics

-   Object storage
-   Bucket = Regional logical container
-   Bucket names globally unique
-   Object = Data + Metadata + Key
-   Folders are prefixes

## Durability

-   11 nines
-   Multi-AZ within Region

## Storage Classes

-   Standard
-   Intelligent-Tiering
-   Standard-IA
-   One Zone-IA
-   Glacier Instant
-   Glacier Flexible
-   Glacier Deep Archive

## Versioning

-   Bucket-level
-   Version IDs
-   Delete Marker
-   Recover old versions

## Lifecycle

-   Transition storage classes
-   Expire objects
-   Delete old versions
-   Abort multipart uploads

## Security

-   IAM Policy
-   Bucket Policy
-   Public Access Block
-   Bucket Owner Enforced
-   Prefer no ACLs
-   Pre-Signed URLs

## Encryption

-   HTTPS/TLS = In transit
-   SSE-S3
-   SSE-KMS
-   SSE-C
-   Client-side

Envelope Encryption: - Data Key encrypts object - KMS Key encrypts Data
Key

## Performance

-   Strong consistency
-   Multipart Upload (\>100 MB recommended, \>5 GB required)
-   Range GET
-   Transfer Acceleration

## S3 vs EBS

  S3          EBS
  ----------- --------------
  Object      Block
  API         Mounted
  Unlimited   Fixed Volume

## Exam Nuggets

-   Strong consistency
-   Folder = Prefix
-   Versioning protects overwrite/delete
-   IAM + Bucket Policy preferred
-   SSE-KMS for compliance
-   Multipart for large files
