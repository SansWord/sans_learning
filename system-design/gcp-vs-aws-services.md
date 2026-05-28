# GCP vs AWS: Major Services at a Glance

A quick-reference table of core cloud services and their GCP / AWS equivalents, grouped by category. This is the "what's the AWS version of X?" lookup that comes up constantly when reading docs or architecture diagrams written for the other cloud.

The **Notes** column is reserved for actual gotchas — naming traps, scope differences, deprecated-but-still-referenced services. Most rows don't need one.

---

## Compute

| Concept | GCP | AWS | Notes |
|---|---|---|---|
| Virtual Machine | Compute Engine | EC2 | |
| Managed Kubernetes | GKE | EKS | |
| Serverless Functions | Cloud Run functions (Cloud Functions) | Lambda | GCP rebranded 2nd-gen Cloud Functions as "Cloud Run functions" — same product, new name |
| PaaS / Container hosting | Cloud Run, App Engine | App Runner, Elastic Beanstalk | Cloud Run is the modern container-based default; App Engine is the older PaaS |

## Storage

| Concept | GCP | AWS | Notes |
|---|---|---|---|
| Object Storage | Cloud Storage (GCS) | S3 | |
| Block Storage (VM disks) | Persistent Disk | EBS | |
| Network File System | Filestore | EFS | |

## Databases

| Concept | GCP | AWS | Notes |
|---|---|---|---|
| Managed Relational | Cloud SQL, AlloyDB, Spanner | RDS, Aurora | Spanner is globally distributed with strong consistency; Aurora is regional |
| NoSQL Document | Firestore | DynamoDB | DynamoDB is key-value first; Firestore is document-first with realtime sync |
| In-Memory Cache | Memorystore | ElastiCache | Both wrap Redis / Memcached |
| Data Warehouse | BigQuery | Redshift | BigQuery is serverless by default; Redshift defaults to provisioned clusters (Serverless mode exists) |

## Networking

| Concept | GCP | AWS | Notes |
|---|---|---|---|
| Load Balancer | Cloud Load Balancing | ELB family (ALB / NLB / GWLB) | GCP has one product with L4/L7 modes; AWS splits by layer |
| CDN | Cloud CDN | CloudFront | |
| DNS | Cloud DNS | Route 53 | Route 53 also does domain registration; Cloud DNS does not |
| Private Network | VPC | VPC | Same name. GCP VPCs are global by default; AWS VPCs are regional |

## Messaging

| Concept | GCP | AWS | Notes |
|---|---|---|---|
| Pub/Sub (topic fan-out) | Pub/Sub | SNS | |
| Queue (point-to-point) | Pub/Sub (pull subscription) | SQS | AWS splits the two patterns into separate products; GCP Pub/Sub covers both via subscription type |
| Managed Kafka | Managed Service for Kafka | MSK | |

## Identity & Security

| Concept | GCP | AWS | Notes |
|---|---|---|---|
| IAM | Cloud IAM | IAM | AWS IAM is account-scoped; GCP IAM is project/folder/org-scoped via the resource hierarchy |
| Secrets Manager | Secret Manager | Secrets Manager | Naming differs by one letter — easy typo |

## Observability

| Concept | GCP | AWS | Notes |
|---|---|---|---|
| Logging | Cloud Logging | CloudWatch Logs | |
| Metrics & Monitoring | Cloud Monitoring | CloudWatch Metrics | |
| Distributed Tracing | Cloud Trace | X-Ray | |

## Containers

| Concept | GCP | AWS | Notes |
|---|---|---|---|
| Container Registry | Artifact Registry | ECR | GCP's older "Container Registry (GCR)" is deprecated — Artifact Registry is the successor and also handles non-container artifacts |

---

← [Back to Index](../README.md)
