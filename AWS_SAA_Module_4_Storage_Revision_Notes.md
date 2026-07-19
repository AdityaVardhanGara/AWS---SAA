# AWS SAA Revision Notes -- Module 4: Storage Services

## EFS

-   Managed NFS for Linux
-   Shared across EC2
-   Regional with Mount Targets
-   POSIX compliant
-   Elastic capacity

Performance: General Purpose, Max I/O Throughput: Bursting, Provisioned,
Elastic

## FSx

-   Windows: SMB + AD
-   Lustre: HPC/ML
-   ONTAP: NetApp workloads
-   OpenZFS: ZFS workloads

## Decision Tree

-   Linux shared FS → EFS
-   Windows → FSx Windows
-   HPC → Lustre
-   NetApp → ONTAP
-   ZFS → OpenZFS

## Exam Tips

-   EFS is Linux only.
-   EFS is regional.
-   Choose FSx variant based on workload.

## One Sentence

EFS is the default managed shared Linux filesystem; FSx provides
specialized enterprise filesystems.
