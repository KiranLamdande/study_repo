# Cloud Compute & Serverless Services: AWS, Azure & GCP
## Complete Reference for Cloud Architects — with Interview Questions & Scenarios

> **Audience:** Cloud Architects, Senior Engineers, Technical Leads
> **Scope:** Compute and Serverless services across AWS, Azure, and GCP — deep-dive details, architect decision guides, and senior-level interview preparation
> **Companion to:** Cloud Networking Services Reference Guide

---

## Table of Contents

1. [AWS Compute & Serverless Services](#1-aws-compute--serverless-services)
2. [Azure Compute & Serverless Services](#2-azure-compute--serverless-services)
3. [GCP Compute & Serverless Services](#3-gcp-compute--serverless-services)
4. [Cross-Cloud Comparison](#4-cross-cloud-comparison)
5. [Architect Decision Scenarios](#5-architect-decision-scenarios)
6. [Complex Interview Questions & Answers](#6-complex-interview-questions--answers)

---

---

# 1. AWS Compute & Serverless Services

---

## 1.1 Amazon EC2 — Elastic Compute Cloud

### What It Is
Amazon EC2 is the foundational IaaS compute service in AWS, providing resizable virtual machines (instances) on demand. EC2 offers over 750 instance types across families optimized for different workload profiles. You have full control over the operating system, installed software, networking, storage, and IAM role assignment. EC2 is the workhorse of AWS — virtually every other AWS compute service (ECS, EKS, EMR, SageMaker) uses EC2 under the hood.

### Instance Families

| Family | Code | Optimized For | Example Workload |
|--------|------|--------------|-----------------|
| **General Purpose** | M, T | Balanced CPU/memory/network | Web servers, dev/test, small DBs |
| **Compute Optimized** | C | High CPU-to-memory ratio | HPC, batch, ML inference, game servers |
| **Memory Optimized** | R, X, z | High memory-to-CPU ratio | In-memory DBs (Redis, SAP HANA), analytics |
| **Storage Optimized** | I, D, H | High disk I/O | NoSQL databases, data warehousing, Hadoop |
| **Accelerated Computing** | P, G, Trn, Inf | GPU/Trainium/Inferentia | ML training, graphics rendering, video encoding |
| **HPC Optimized** | Hpc | High-performance networking | Tightly coupled simulations, CFD, weather modeling |
| **Mac** | mac1, mac2 | macOS | iOS/macOS app builds, Xcode CI/CD |

### Purchasing Options Deep Dive

```
On-Demand:
├── Full price, per-second billing (minimum 60 seconds)
├── No commitment, maximum flexibility
└── Use for: unpredictable workloads, dev/test, short-term experiments

Reserved Instances (RI):
├── 1-year or 3-year commitment
├── Standard RI: up to 72% savings — fixed instance type/size/region
├── Convertible RI: up to 66% savings — can exchange for different instance type
└── Use for: steady-state production workloads running 24/7

Savings Plans:
├── Compute Savings Plans: up to 66% — applies to ANY EC2, Fargate, Lambda
├── EC2 Instance Savings Plans: up to 72% — specific instance family in a region
└── More flexible than RIs — doesn't lock to a specific instance type

Spot Instances:
├── Up to 90% discount — spare AWS capacity
├── AWS can reclaim with 2-minute warning
├── Persistent spot requests, spot fleets, spot blocks (1-6 hour windows)
└── Use for: fault-tolerant batch jobs, stateless web servers, Hadoop/Spark

Dedicated Hosts:
├── Physical server dedicated to your use
├── Visibility into sockets and physical cores
├── Required for: BYOL (SQL Server, Windows Server per-socket licensing)
└── Use for: compliance requirements, software licensing constraints

Dedicated Instances:
├── Instance runs on hardware dedicated to one account
├── Unlike Dedicated Host: no visibility into physical host
└── Use for: compliance requiring hardware isolation, not BYOL
```

### Technical Deep Dive: Nitro System

Modern EC2 instances run on the AWS Nitro System — a combination of custom silicon (Nitro Cards), a lightweight hypervisor (KVM-based), and security chips. Key benefits:
- Near bare-metal performance: virtualization overhead < 1%
- EBS encryption handled by hardware (zero CPU cost)
- Network: ENA (Elastic Network Adapter) with up to 200 Gbps for some instances
- NVMe: local NVMe SSDs accessible with minimal overhead
- Nitro Enclaves: isolated compute environments for sensitive data processing (no persistent storage, no interactive access, cryptographic attestation)

### EC2 Auto Scaling Deep Dive

```
Auto Scaling Group (ASG) Components:
├── Launch Template: AMI, instance type, SG, IAM role, user data
├── Min/Max/Desired capacity
├── Scaling policies:
│   ├── Target Tracking: maintain CPU = 70% (simplest, recommended)
│   ├── Step Scaling: scale +2 if CPU > 70%, +4 if CPU > 90% (graduated response)
│   ├── Simple Scaling: cooldown period after each action (legacy)
│   └── Scheduled Scaling: pre-scale before known traffic spikes
├── Health checks: EC2 (default) or ELB (application-level)
└── Instance refresh: rolling replacement with configurable min healthy %

Warm Pools (for latency-sensitive scale-out):
├── Pre-initialized instances waiting in Stopped or Running state
├── When scale-out needed, warm pool instance starts immediately (vs. cold launch)
└── Dramatically reduces scale-out latency for slow-to-boot applications
```

### When to Use EC2

```
Use EC2 when:
├── Full OS control required (custom kernel modules, OS-level configurations)
├── Lift-and-shift migration of on-premises servers
├── Windows Server or SQL Server with Dedicated Host (BYOL licensing)
├── GPU-accelerated workloads (ML training, rendering, video transcoding)
├── Persistent, stateful workloads not suitable for containers/serverless
├── SAP HANA, Oracle DB — certified instance types required
└── Custom networking configurations not available in managed services

Avoid EC2 when:
├── Stateless web apps → prefer ECS Fargate or Lambda (less operational overhead)
├── Short-duration jobs → prefer Lambda or Fargate (no idle cost)
└── Development environments → consider Cloud9 or Fargate dev containers
```

### Real-World Use Case

**Scenario:** A genomics company runs computationally intensive DNA sequencing jobs that require 96 vCPUs and 768 GiB RAM per job, with jobs lasting 4–8 hours and running 200 jobs per day. Cost optimization is critical.

**Architecture Decision:**
```
Instance: r6i.24xlarge (96 vCPUs, 768 GiB RAM, memory-optimized)
Purchasing: Spot Instances (fault-tolerant batch — 80% cost savings)

ASG Configuration:
├── Mixed Instances Policy: r6i.24xlarge, r6a.24xlarge, r5.24xlarge (fallback types)
├── Capacity Rebalancing: proactively replace instances before Spot reclamation
├── Spot interruption handler: checkpoint job state to S3 on 2-min warning
└── Batch job framework: AWS Batch manages job queue + Spot fleet management

Cost comparison:
├── On-Demand r6i.24xlarge: $7.23/hour × 8h × 200 jobs = $11,568/day
└── Spot Instances (~80% discount): ~$2,314/day → $9,254/day savings
```

---

## 1.2 AWS Lambda

### What It Is
AWS Lambda is a serverless compute service that runs code in response to events without provisioning or managing servers. You upload code (or a container image up to 10 GB), configure a trigger, and AWS handles everything else: server provisioning, OS maintenance, capacity scaling, monitoring, and logging. Lambda scales from 0 to thousands of concurrent executions in seconds, and you pay only for actual compute time (billed in 1ms increments).

### Lambda Execution Model

```
Invocation Types:
├── Synchronous (RequestResponse): caller waits for response
│   Triggers: API Gateway, ALB, Lambda URL, SDK direct invoke
│   Timeout: up to 15 minutes; caller gets error on timeout
│
├── Asynchronous (Event): Lambda queues and retries automatically
│   Triggers: S3 events, SNS, EventBridge, SES, CodeCommit
│   Retry: 2 automatic retries, DLQ/destination for failures
│   At-least-once: same event may process twice (design for idempotency)
│
└── Poll-based (stream/queue): Lambda polls a source for records
    Triggers: SQS, Kinesis, DynamoDB Streams, Kafka (MSK/self-managed)
    Batch window + batch size control throughput vs latency trade-off
```

### Lambda Execution Environment

```
Cold Start (new execution environment initialized):
├── Container is created, runtime initialized, code loaded
├── Duration: 100ms–1s (depends on runtime and package size)
├── Java/.NET: longer cold starts (JVM/CLR initialization)
├── Python/Node.js: shorter cold starts

Mitigation strategies:
├── Provisioned Concurrency: pre-warm N execution environments
│   (eliminates cold starts for that concurrency level, adds cost)
├── Lambda SnapStart (Java only): restore from snapshot (ms vs seconds)
├── Keep packages small: separate layers for dependencies
└── ARM64 (Graviton2): 19% cheaper AND 10% faster cold starts

Warm Start: existing execution environment reused
├── Global scope (outside handler) is preserved between invocations
├── DB connections, SDK clients, cached config should be initialized globally
└── /tmp (512 MB–10 GB): persists within execution environment lifespan
```

### Lambda Concurrency Model

```
Account default: 1,000 concurrent executions (per region, soft limit)

Concurrency types:
├── Unreserved concurrency: shared pool across all functions
├── Reserved concurrency: guarantees N for a specific function
│   (Also LIMITS function to max N — useful for throttling downstream services)
└── Provisioned concurrency: pre-initialized environments (no cold start)

Throttling:
├── If concurrency limit hit → Lambda returns 429 TooManyRequests
├── Synchronous: error returned immediately to caller
├── Async: event queued, retried for up to 6 hours (exponential backoff)
└── SQS trigger: messages remain in queue until Lambda can process
```

### Lambda Power Tuning

Lambda memory settings (128 MB–10,008 MB) affect BOTH memory AND vCPU allocation. More memory = more CPU = faster execution. The optimal point is where cost (memory × duration) is minimized:

```
AWS Lambda Power Tuning tool (open-source Step Functions state machine):
Run function at 128MB, 256MB, 512MB, 1024MB, 2048MB, 3008MB
Measure duration and cost at each setting
Find the "inverted U" where cost is lowest (often not the minimum memory)

Common finding: 1024MB runs 4× faster than 128MB at 8× the memory
→ cost = memory × duration: 8× × 0.25× duration = 2× cost increase
                             vs. 1× × 1× duration = less work done
→ 512MB is often the sweet spot for most functions
```

### When to Use Lambda

```
Use Lambda when:
├── Event-driven processing: S3 uploads, DynamoDB streams, SNS/SQS messages
├── HTTP APIs: Lambda + API Gateway for REST/HTTP APIs (no server management)
├── Scheduled tasks: cron-style jobs via EventBridge Scheduler
├── ETL pipelines: file transformation, data enrichment between services
├── Real-time stream processing: Kinesis or DynamoDB Streams triggers
├── Webhook receivers: third-party service sends HTTP → Lambda processes
└── Microservices: stateless services that don't need persistent connections

Avoid Lambda when:
├── Execution > 15 minutes → use ECS Fargate, EC2, or Step Functions
├── Persistent TCP connections (databases, WebSockets) → ECS/EKS more suitable
├── Large binaries or files > 10 GB container limit → EC2 or ECS
└── Very high steady-state throughput (>10K TPS) → dedicated compute may be cheaper
```

### Real-World Use Case

**Scenario:** An e-commerce platform processes 500,000 order events per day. Each order triggers: fraud detection, inventory reservation, email confirmation, and analytics event emission. Processing must complete within 200ms, and the team wants zero server management.

**Architecture:**
```
Order Placed → SNS Topic (fan-out)
├── Lambda: FraudDetection (synchronous via SQS FIFO, 30s timeout)
│   └── Calls Fraud ML endpoint, writes result to DynamoDB
├── Lambda: InventoryReservation (SQS standard, reserved concurrency=100)
│   └── Prevents overwhelming inventory DB with too many concurrent writes
├── Lambda: EmailConfirmation (SQS, 60s visibility timeout)
│   └── Calls SES, idempotent (order ID as deduplication key)
└── Lambda: AnalyticsEmitter (Kinesis, batch=100, window=5s)
    └── Buffers events, writes batch to Kinesis Data Firehose → S3

Cost at 500K orders/day:
Lambda invocations: 500K × 4 functions = 2M invocations/day = $0.40/day
Lambda compute (avg 100ms × 256MB each):
= 2M × 0.1s × 0.256GB × $0.0000166667/GB-s = $0.85/day
Total Lambda cost: ~$1.25/day (vs. EC2 baseline ~$50-100/day)
```

---

## 1.3 Amazon ECS — Elastic Container Service

### What It Is
Amazon ECS is a fully managed container orchestration service that runs Docker containers in clusters. ECS manages container scheduling, placement, health monitoring, and integration with AWS services. Containers run on either EC2 instances you manage (ECS on EC2) or on AWS Fargate (serverless container execution where you pay per vCPU/memory consumed by your container, with no EC2 instances to manage).

### ECS Architecture Components

```
ECS Cluster
├── Task Definition (blueprint for containers)
│   ├── Container definitions (image, CPU, memory, ports, env vars, log config)
│   ├── Network mode: bridge, host, awsvpc (required for Fargate)
│   ├── IAM Task Role (permissions for container's application code)
│   ├── IAM Task Execution Role (ECS agent permissions: ECR pull, CW Logs)
│   └── Volumes: EFS, EBS (for Fargate), bind mounts
│
├── ECS Service (long-running workloads)
│   ├── Desired count: N replicas
│   ├── Load balancer integration (ALB target group)
│   ├── Service Auto Scaling (target tracking, step, scheduled)
│   ├── Rolling update or Blue/Green (via CodeDeploy)
│   └── Service Discovery (AWS Cloud Map → Route 53 private hosted zone)
│
└── ECS Tasks (one-off jobs: batch, migrations, data processing)
```

### ECS on EC2 vs Fargate

| Aspect | ECS on EC2 | ECS Fargate |
|--------|-----------|------------|
| **Infrastructure** | You manage EC2 instances | AWS manages compute |
| **Cost model** | EC2 instance pricing | Per vCPU-hour + per GB-hour |
| **Bin packing** | Multiple containers per EC2 (cost efficient at scale) | Per-task billing |
| **Control** | OS access, custom AMIs, GPU support | No OS access |
| **Cold start** | Near zero (container starts on running EC2) | ~5–30s (Fargate provisions micro-VM) |
| **Spot** | EC2 Spot + ECS (complex) | Fargate Spot (simple, 70% discount) |
| **Breakeven** | Scale: 10+ constantly running containers | Small-medium workloads |

### ECS Networking: awsvpc Mode

In awsvpc mode (required for Fargate, recommended for EC2), each task gets its own Elastic Network Interface (ENI) with a private IP in the VPC subnet. This means:
- Security Groups applied per-task (not per-EC2 instance)
- VPC Flow Logs capture per-task traffic
- No port-mapping conflicts between tasks
- Tasks accessible at their private IP directly

**ENI Trunking for EC2:** Without trunking, each EC2 instance supports a limited number of ENIs (e.g., m5.xlarge = 4 ENIs = 4 tasks). With ENI trunking enabled on the ECS container instance, a trunk ENI is created with branch ENIs virtualized per-task, allowing up to 120 tasks per instance regardless of instance type's ENI limit.

### When to Use ECS

```
Use ECS (Fargate) when:
├── Containerized apps without wanting to manage Kubernetes complexity
├── Microservices that need ALB integration, service discovery, auto-scaling
├── Batch processing containers (ECS Tasks with EventBridge Scheduler)
├── Migrating from on-premises Docker Compose to AWS
└── Team familiar with Docker but not Kubernetes

Use ECS on EC2 when:
├── GPU containers (ML inference, video processing) — Fargate doesn't support GPU
├── Custom AMIs required (specialized OS, kernel modules)
├── Large scale (500+ containers) where EC2 bin-packing reduces cost vs Fargate
└── Windows containers (Fargate supports Windows but with limitations)

Use EKS instead when:
├── Team has Kubernetes expertise and wants full Kubernetes API
├── Using Helm charts, CRDs, or Kubernetes-native tooling
└── Multi-cloud portability — Kubernetes runs everywhere
```

### Real-World Use Case

**Scenario:** A fintech startup has 15 microservices (Node.js, Python, Java) that need independent deployment, auto-scaling, and secure service-to-service communication. The team has Docker experience but no Kubernetes expertise.

```
Architecture: ECS Fargate + ALB + AWS Cloud Map

Per microservice:
├── ECS Service: 2–10 tasks (Fargate, awsvpc mode)
├── Task CPU/Memory: right-sized per service (256CPU/512MB to 4096CPU/8192MB)
├── Fargate Spot for non-critical services (payment-analytics, reporting)
├── ALB listener rules route: /api/payments/* → payment-service target group
└── Service Discovery: payment-service.prod.internal → Cloud Map → private IPs

Auto Scaling:
├── Target tracking: ALBRequestCountPerTarget = 1000 req/target
└── Scale from 2 to 20 tasks per service based on load

Security:
├── Each service has its own IAM Task Role (least privilege)
├── Security Group per service (payment-sg allows 8080 only from api-gateway-sg)
└── Secrets Manager: DB passwords injected as env vars via Task Definition secrets ref

Monthly cost estimate (15 services, average 3 tasks each):
45 Fargate tasks × 0.5 vCPU × $0.04048/vCPU-hour × 730h = $666/month
45 tasks × 1GB × $0.004445/GB-hour × 730h = $146/month
Total compute: ~$812/month (vs. EC2-based ~$1,500/month minimum for equivalent HA)
```

---

## 1.4 Amazon EKS — Elastic Kubernetes Service

### What It Is
Amazon EKS is a managed Kubernetes service that handles the Kubernetes control plane (API server, etcd, scheduler, controller manager) — running it across multiple AWS availability zones for high availability. AWS handles control plane upgrades, patching, and availability. You manage worker nodes (EC2 node groups, Fargate profiles, or Karpenter-managed nodes).

### EKS Architecture

```
EKS Control Plane (AWS Managed):
├── API Server (multi-AZ, behind NLB)
├── etcd (multi-AZ, encrypted)
├── Scheduler, Controller Manager
├── AWS integrations: IAM, VPC CNI, EBS/EFS CSI drivers
└── Accessible via kubectl → EKS endpoint (public or private)

Worker Nodes (You manage):
├── Managed Node Groups: ASG-backed, AWS manages AMI updates
├── Self-managed Node Groups: full control, custom AMIs
├── Fargate Profiles: serverless pods (matching namespace/label selector)
└── Karpenter (recommended): fast, flexible node provisioning

EKS Networking (VPC CNI):
├── Each pod gets a real VPC IP (from the node's ENI secondary IPs)
├── Security Groups for Pods: per-pod SG assignment
├── VPC-native: pods routable from anywhere in the VPC
└── Custom networking: pods in secondary CIDRs to avoid exhaustion
```

### EKS IAM Integration: IRSA vs Pod Identity

```
IRSA (IAM Roles for Service Accounts) — current approach:
├── Kubernetes Service Account annotated with IAM Role ARN
├── EKS OIDC provider issues JWT tokens
├── Pods assume IAM role via Web Identity Token
└── Each pod can have different IAM permissions

EKS Pod Identity — newer (2023), simpler:
├── No OIDC setup required per cluster
├── Pod Identity Agent (DaemonSet) handles token exchange
├── Supports cross-account role assumption
└── Simpler configuration than IRSA
```

### EKS Add-ons

| Add-on | Purpose |
|--------|---------|
| **VPC CNI** | Pod networking — allocates VPC IPs to pods |
| **CoreDNS** | Cluster DNS — service discovery |
| **kube-proxy** | Service networking — iptables rules |
| **AWS EBS CSI Driver** | Persistent volumes via EBS |
| **AWS EFS CSI Driver** | Shared persistent volumes via EFS |
| **AWS Load Balancer Controller** | Kubernetes Ingress → ALB, Service → NLB |
| **Karpenter** | Node autoprovisioning (replaces Cluster Autoscaler) |
| **GuardDuty for EKS** | Runtime threat detection |
| **Amazon VPC Lattice** | Service mesh for cross-cluster communication |

### Karpenter vs Cluster Autoscaler

```
Cluster Autoscaler (legacy):
├── Scales pre-defined node groups (ASGs)
├── Must pre-define instance types per node group
├── Scale-out: provision new node, wait for it to be Ready (~2–3min)
└── Scale-in: conservative (waits for pod scheduling to find other nodes)

Karpenter (recommended):
├── Provisions individual nodes (not ASGs) based on pending pod requirements
├── Chooses optimal instance type at time of provisioning (price/performance)
├── Uses EC2 Fleet for Spot diversification automatically
├── Scale-out: directly calls EC2 RunInstances (~45 seconds to Ready)
├── Consolidation: actively packs pods tighter, terminates underutilized nodes
└── Disruption budgets: controls which pods can be disrupted during consolidation
```

### When to Use EKS

```
Use EKS when:
├── Kubernetes expertise on the team
├── Helm charts, Operators, CRDs central to your tooling
├── Multi-cloud strategy (same Kubernetes manifests on EKS/AKS/GKE)
├── Complex service mesh requirements (Istio, Linkerd, App Mesh)
├── Large-scale (100+ microservices) needing fine-grained resource management
└── GitOps workflows (ArgoCD, Flux) as the deployment model

Use ECS instead when:
├── Team unfamiliar with Kubernetes — ECS has simpler mental model
├── Primarily AWS-native tooling (no multi-cloud requirement)
└── Simpler workloads that don't need full Kubernetes power
```

---

## 1.5 AWS Fargate

### What It Is
AWS Fargate is a serverless compute engine for containers that works with both ECS and EKS. With Fargate, you specify CPU and memory for your container task/pod, and AWS provisions, scales, and manages the underlying infrastructure — no EC2 instances to patch, right-size, or manage.

### Fargate Under the Hood

Fargate uses micro-VM technology (similar to Firecracker, AWS's open-source VMM) to provide strong task isolation. Each Fargate task runs in its own lightweight VM, providing security boundaries equivalent to EC2 instances but with container-level interface. This is why Fargate has slightly longer startup times than EC2-based containers.

### Fargate Resource Configurations

```
CPU/Memory combinations (valid pairs only):
CPU:    256   512   1024   2048   4096   8192   16384
Memory: 512   1024  2048   4096   8192   ...    ...   (MB)

Maximum: 16 vCPU, 120 GB memory (Fargate supports larger tasks than most containers need)

Ephemeral storage: 21 GB default, up to 200 GB additional (configurable per task)
```

### Fargate Spot

Fargate Spot provides up to 70% discount for fault-tolerant workloads. Like EC2 Spot, Fargate Spot capacity can be reclaimed with a 2-minute warning. ECS handles the interruption gracefully:

```
Fargate Spot strategy:
├── ECS Service with capacity provider strategy:
│   FARGATE_SPOT: weight=3 (75% of tasks on Spot)
│   FARGATE: weight=1 (25% of tasks on On-Demand for stability)
└── ECS drains interrupted Spot tasks, replaces with On-Demand or new Spot
```

### When to Use Fargate

```
Use Fargate when:
├── Want containers without managing EC2 instances (no patching, right-sizing)
├── Variable, unpredictable workloads (scales to zero possible via ECS)
├── Batch jobs / one-off tasks that start and stop (pay only when running)
├── Dev/test environments (spin up on-demand, shut down after use)
├── Compliance: strong task isolation without shared kernel
└── Small-to-medium scale (<500 containers) where per-task pricing is economical

Use ECS on EC2 when:
├── GPU workloads (Fargate doesn't support GPU)
├── Scale > 500 constantly running containers (EC2 bin-packing is cheaper)
└── Custom kernel modules or OS configurations needed
```

---

## 1.6 AWS Step Functions

### What It Is
AWS Step Functions is a serverless workflow orchestration service that coordinates multiple AWS services into workflows using a visual state machine. It manages state, handles errors and retries, runs parallel branches, and integrates with 200+ AWS services — all without writing glue code. Step Functions is the answer to "how do I orchestrate multiple Lambda functions and AWS services reliably?"

### Workflow Types

```
Standard Workflows:
├── Execution duration: up to 1 year
├── Exactly-once execution semantics
├── Full audit history (execution history stored)
├── Pricing: $0.025 per 1,000 state transitions
└── Use for: long-running business processes, human approval workflows

Express Workflows:
├── Execution duration: up to 5 minutes
├── At-least-once execution (idempotency required)
├── No execution history stored (CloudWatch Logs instead)
├── Pricing: $1 per 1M executions + $0.00001/GB-second duration
└── Use for: high-volume, short-duration event processing pipelines
```

### SDK Integration (Optimistic Execution)

Step Functions SDK integrations call AWS services directly (without Lambda as a middle layer), reducing cost and complexity:

```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::dynamodb:putItem",
  "Parameters": {
    "TableName": "Orders",
    "Item": {
      "OrderId": {"S.$": "$.orderId"},
      "Status": {"S": "PROCESSING"}
    }
  },
  "Next": "SendNotification"
}
```

This calls DynamoDB directly — no Lambda function needed. Supported: Lambda, DynamoDB, SQS, SNS, ECS, Fargate, Glue, Athena, SageMaker, and 190+ more.

### When to Use Step Functions

```
Use Step Functions when:
├── Multi-step workflows with branching, parallel execution, error handling
├── Human approval steps (wait for callback pattern)
├── Long-running processes that span minutes to days
├── Replacing complex Lambda orchestration code with a visual state machine
├── Saga pattern for distributed transactions (compensating transactions on failure)
└── ML pipelines (data prep → train → evaluate → deploy)

Avoid Step Functions when:
├── Simple linear pipeline with no branching → SNS/SQS is simpler
├── Sub-second latency required → Step Functions adds overhead (100ms+ per step)
└── Very high volume (>1M executions/day on Standard) → cost may exceed Lambda
```

### Real-World Use Case

**Scenario:** An insurance company processes claim submissions that require: document extraction (AI), fraud scoring (ML), human adjuster review (if fraud score > 0.7), payment calculation, and payment disbursement.

```yaml
# Step Functions State Machine (ASL)
States:
  ExtractDocuments:
    Type: Task
    Resource: arn:aws:states:::lambda:invoke
    Parameters: {FunctionName: document-extractor}
    Retry: [{ErrorEquals: [Lambda.ServiceException], MaxAttempts: 3}]
    Next: ScoreFraud

  ScoreFraud:
    Type: Task
    Resource: arn:aws:states:::sagemaker:invokeEndpoint
    Parameters: {EndpointName: fraud-model-v3, Body.$: $.extractedData}
    Next: FraudCheck

  FraudCheck:
    Type: Choice
    Choices:
      - Variable: $.fraudScore
        NumericGreaterThan: 0.7
        Next: HumanReview
    Default: CalculatePayment

  HumanReview:
    Type: Task
    Resource: arn:aws:states:::sqs:sendMessage.waitForTaskToken
    Parameters:
      QueueUrl: https://sqs.../adjuster-review-queue
      MessageBody:
        ClaimId.$: $.claimId
        TaskToken.$: $$.Task.Token  # Pause until human sends token back
    HeartbeatSeconds: 86400  # Wait up to 24 hours for human response
    Next: ReviewDecision
```

---

## 1.7 AWS Batch

### What It Is
AWS Batch is a fully managed batch computing service that efficiently runs large-scale batch and ML training jobs. It dynamically provisions the optimal quantity and type of compute resources based on the volume and requirements of submitted batch jobs — eliminating the need to manage batch computing infrastructure.

### Key Components

```
Job Definition:
├── Container image, vCPU, memory requirements
├── Retry strategy (max attempts, retry conditions)
├── Timeout (kill job after N seconds)
└── Mount points, environment variables

Job Queue:
├── Priority ordering between queues
└── Linked to one or more Compute Environments

Compute Environment:
├── Managed EC2: AWS scales EC2 capacity based on job demand
│   ├── Spot + On-Demand mix for cost optimization
│   ├── Instance type selection (optimal, or specific families)
│   └── min/max vCPUs control cost ceiling
└── Managed Fargate: for shorter jobs without EC2 management

Array Jobs:
├── Submit 1 job definition that spawns N child jobs (N up to 10,000)
├── Each child gets AWS_BATCH_JOB_ARRAY_INDEX env var (0 to N-1)
└── Parallelize: split a 1M row dataset into 1,000 × 1,000-row chunks
```

### When to Use AWS Batch

```
Use Batch when:
├── Large-scale parallel processing (genomics, financial risk simulations, rendering)
├── Jobs that need > 15 minutes (Lambda limit) but don't need persistent servers
├── Variable job load (Batch scales EC2 up and down based on queue depth)
├── Dependency ordering between jobs (job B runs after job A completes)
└── Cost-optimized compute — Batch natively manages Spot/On-Demand mix

vs. Lambda: Lambda is event-driven, <15min. Batch is for long-running batch jobs.
vs. Step Functions: Step Functions orchestrates workflow steps. Batch runs the heavy compute.
(Often used together: Step Functions orchestrates → Batch runs the computation)
```

---

## 1.8 AWS App Runner

### What It Is
AWS App Runner is a fully managed service for deploying containerized web applications and APIs directly from source code or a container image. It handles everything: container builds (from GitHub/CodeCommit), deployment, load balancing, TLS certificates, auto-scaling (including scaling to zero), and health monitoring. It is the simplest way to run a web application on AWS.

### App Runner vs ECS Fargate vs Lambda

| Aspect | App Runner | ECS Fargate | Lambda |
|--------|-----------|------------|--------|
| **Configuration** | Minimal (auto-detected) | Full task definition | Function config |
| **Source** | Git repo or ECR | ECR image | Code zip or ECR |
| **Build** | Automatic (CodeBuild inside) | Manual CI/CD needed | Deployment package |
| **Scale to zero** | Yes (with cold start ~5s) | No (minimum 1 task) | Yes (instant) |
| **VPC access** | Optional VPC connector | Native VPC | Optional VPC |
| **Concurrency** | Per-instance requests | Per-task | Per-function |
| **Max duration** | Unlimited | Unlimited | 15 minutes |

### When to Use App Runner

```
Use App Runner when:
├── Small team wanting to deploy a containerized API/web app in minutes
├── No Kubernetes or ECS expertise available
├── Proof of concept or MVP needing rapid deployment
├── Source-to-production pipeline without managing CI/CD infrastructure
└── Web apps needing auto-scaling including to zero (cost savings for low-traffic)

Not ideal when:
├── Complex networking (multiple services, inter-service communication)
├── Custom domain with advanced routing rules → App Runner has basic domain support
└── High control requirements (custom OS, GPU, specific instance types)
```

---


---

# 2. Azure Compute & Serverless Services

---

## 2.1 Azure Virtual Machines

### What It Is
Azure Virtual Machines provide on-demand, scalable compute in the Azure cloud. Like EC2, you choose the OS, size, storage, and network configuration. Azure VMs run on Microsoft's hypervisor (Hyper-V) and the Azure fabric controller. Azure differentiates through its Hybrid Benefit licensing model (bring existing Windows Server / SQL Server licenses), native Active Directory / Entra ID integration, and its rich SAP and Oracle certified VM families.

### VM Size Families

| Series | Optimized For | Examples |
|--------|--------------|---------|
| **B (Burstable)** | Variable CPU (baseline + burst credits) | Dev/test, small apps |
| **D/Ds v5** | General purpose (balanced) | Web apps, databases, enterprise |
| **E/Es v5** | Memory optimized | SAP HANA, in-memory DBs |
| **F/Fs** | Compute optimized (high CPU ratio) | Batch, gaming, rendering |
| **L** | Storage optimized (NVMe) | Cassandra, MongoDB, data warehousing |
| **N (NC, ND, NV)** | GPU | ML training (NC), inference (ND), visualization (NV) |
| **H** | HPC | MPI workloads, fluid dynamics, weather modeling |
| **M** | Very large memory (up to 11.5 TB RAM) | SAP HANA scale-up, in-memory analytics |

### Azure Hybrid Benefit — Cost Differentiator

```
Azure Hybrid Benefit (AHB): Use existing on-premises licenses in Azure

Windows Server:
├── With AHB: ~49% cost reduction on Windows VM licensing
└── Each Windows Server Datacenter license covers 2 Azure VMs

SQL Server:
├── With AHB: SQL Enterprise on Azure = SQL Standard pricing
└── Saves up to 55% on SQL Server VMs

Combined: Windows VM + SQL + AHB + Reserved Instance = up to 80% savings
vs. Linux + On-Demand pricing

Example:
Standard_D8s_v5 (8 vCPU, 32 GiB) in West Europe:
├── Linux On-Demand:    $0.384/hour
├── Windows On-Demand:  $0.768/hour (+100% Windows license)
├── Windows + AHB:      $0.384/hour (Windows license removed)
└── Windows + AHB + 3yr RI: $0.158/hour (59% total savings)
```

### VM Scale Sets (VMSS)

```
VMSS: Deploy and manage a set of identical VMs with auto-scaling

Orchestration modes:
├── Uniform: All VMs use the same model (classic, simpler)
└── Flexible: Mix VM sizes, supports individual VM management (recommended)

Scaling policies:
├── Metric-based: scale on CPU, memory, custom metrics
├── Schedule-based: pre-scale for known peak times  
└── Predictive scaling (preview): ML-based scale-out before demand peaks

Update policies:
├── Automatic: instances updated automatically when model changes
├── Rolling: batched update with configurable max unavailable %
└── Manual: operator controls when each instance updates

Health monitoring:
├── Application Health Extension: reports /health endpoint status to VMSS
└── Automatic Instance Repair: replaces unhealthy instances automatically
```

### Azure Spot VMs

Azure Spot VMs provide up to 90% discount on Azure VM pricing using surplus capacity. Unlike AWS Spot (2-minute warning), Azure Spot gives a 30-second eviction notice. Eviction policies:

```
Eviction Policy options:
├── Deallocate (recommended): VM is stopped, disk preserved, can restart later
└── Delete: VM and all disks deleted on eviction

Eviction types:
├── Capacity: Azure needs the capacity back
└── Price: spot price exceeded your max price bid

Spot VMs are NOT suitable for:
├── Production workloads requiring consistent availability
└── Any workload that cannot tolerate interruption
```

### When to Use Azure VMs

```
Use Azure VMs when:
├── Lift-and-shift from on-premises Hyper-V or VMware (Azure Migrate)
├── Windows Server + SQL Server with Hybrid Benefit (significant licensing savings)
├── SAP workloads (Azure SAP-certified M series VMs)
├── Active Directory domain controllers (require OS-level control)
├── Legacy applications requiring specific OS versions
└── GPU workloads: ML training (NC), inference (ND), VDI (NV)
```

---

## 2.2 Azure App Service

### What It Is
Azure App Service is a fully managed PaaS platform for hosting web applications, REST APIs, and mobile backends. It supports .NET, .NET Framework, Java, Python, Node.js, PHP, and Ruby — natively, without containers. App Service handles OS patching, capacity provisioning, load balancing, certificate management, and diagnostic logging. Its killer features for enterprise: deployment slots (zero-downtime blue-green), deep Azure DevOps/GitHub integration, and built-in authentication (EasyAuth).

### App Service Plan Tiers

```
Free (F1) and Shared (D1):
├── Shared compute (no SLA), limited to 60/240 minutes CPU/day
└── For: learning, very low traffic, demos

Basic (B1–B3):
├── Dedicated compute, no auto-scale, no deployment slots
└── For: dev/test environments, non-critical workloads

Standard (S1–S3):
├── Dedicated compute, auto-scale (up to 10 instances)
├── 5 deployment slots, custom domains + SSL
└── For: production workloads with moderate scale

Premium v3 (P0v3–P3v3):
├── Enhanced performance (Dv4-series VMs), VNet Integration
├── 20 deployment slots, up to 30 instances auto-scale
└── For: high-performance production workloads, VNet-connected apps

Isolated v2 (I1v2–I3v2):
├── App Service Environment (ASE) — dedicated infrastructure in your VNet
├── No shared infrastructure, maximum isolation
└── For: regulated industries, compliance (PCI, HIPAA), maximum performance
```

### Deployment Slots Deep Dive

```
Deployment Slots: separate, live environments sharing the same App Service Plan

Standard use: production + staging (2 slots minimum)
Premium:      up to 20 slots (dev, qa, uat, staging, canary, regional...)

Slot settings (sticky vs. swappable):
├── Swappable (default): app settings, connection strings → swap with code
└── Sticky (slot setting = true): stays with the slot regardless of swap
    Examples: ASPNETCORE_ENVIRONMENT, APP_SLOT_NAME, feature flags

Swap process:
1. Azure warms up staging slot (sends requests to /health endpoint)
2. Once warm, Azure atomically swaps the routing rules
3. Production is now the new code; old production moves to staging
4. Duration: 10–60 seconds (no downtime, no DNS change)
5. Rollback: swap again (old code is still in staging slot)

Progressive traffic routing (canary):
├── Route 5% of production traffic to staging slot
├── Monitor Application Insights metrics (error rate, latency)
└── When confident, complete the swap (100% to new code)
```

### App Service VNet Integration

```
Outbound connectivity (App Service → VNet resources):
├── VNet Integration: app routes outbound traffic into a VNet subnet
├── Requires: dedicated /28 subnet for integration
└── Allows: reaching private databases, APIs, internal services

Inbound connectivity (Internet → App Service, private):
├── Private Endpoint: App Service gets a private IP in your VNet
├── App Service public endpoint is disabled (accessRestrictions)
└── Traffic: Internet → Application Gateway/Front Door → Private Endpoint → App

Full network isolation pattern (Isolated tier, ASE):
├── App Service deployed inside your VNet (no public endpoints at all)
└── Requires: App Service Environment v3 (ASEv3)
```

### When to Use App Service

```
Use App Service when:
├── .NET/Java/Python/Node.js web apps or APIs — native runtime support
├── Team wants PaaS experience (no Docker/Kubernetes knowledge)
├── Zero-downtime deployments (deployment slots are simpler than Kubernetes rolling updates)
├── Built-in auth needed (Microsoft, Google, GitHub, Twitter SSO via EasyAuth)
├── WebJobs: background tasks without separate infrastructure
└── Hybrid connections: access on-premises resources from App Service

Not ideal when:
├── Polyglot containers (App Service supports containers but EKS/AKS is better for complex orchestration)
├── GPU workloads → use AKS with GPU node pool
└── Very low traffic with cost sensitivity → Azure Functions scales to zero
```

---

## 2.3 Azure Kubernetes Service (AKS)

### What It Is
AKS is Azure's managed Kubernetes service. Azure manages the control plane at no charge — you pay only for worker nodes. AKS provides deep integration with Azure services: Entra ID for authentication, Azure Monitor for observability, Azure Container Registry, Azure Policy (OPA Gatekeeper), Key Vault Secrets Store CSI Driver, and Workload Identity Federation.

### AKS Node Pool Types

```
System Node Pool (required):
├── Runs critical Kubernetes components (CoreDNS, metrics-server, tunnel-front)
├── Taint: CriticalAddonsOnly=true:NoSchedule (prevents user workloads)
└── Recommended: 3 nodes minimum, Standard_D4s_v5 or larger

User Node Pool(s):
├── Run application workloads
├── Multiple pools supported (e.g., linux-pool, windows-pool, gpu-pool, spot-pool)
└── Can be added/removed without cluster recreation

Spot Node Pool:
├── Uses Azure Spot VMs (up to 90% cost reduction)
├── Taints: kubernetes.azure.com/scalesetpriority=spot:NoSchedule
└── Use for: batch jobs, dev workloads, fault-tolerant services

Virtual Node (ACI Burst):
├── Deploys pods to Azure Container Instances (serverless)
├── Instantaneous scale-out (no node provisioning wait)
└── For: burst traffic, batch jobs, CI/CD job runners
```

### AKS Networking Options

```
Kubenet (basic networking):
├── Nodes get VNet IPs; pods get an overlay network (not VNet-routable)
├── NAT required for pod-to-external communication
└── Simpler setup, but pods not directly reachable from VNet

Azure CNI (recommended for production):
├── Each pod gets a real VNet IP (from the subnet's IP range)
├── Pods directly routable from VNet, on-premises (via ExpressRoute), peered VNets
├── IP planning critical: 30+ pods per node × 100 nodes = 3,000+ IPs needed
└── Azure CNI Overlay (preview): pods get overlay IPs but with CNI performance

Azure CNI Powered by Cilium (2023):
├── eBPF-based networking (replaces iptables — better performance at scale)
├── Network policies via Cilium (L3–L7 policy enforcement)
└── Hubble observability (real-time pod network traffic visualization)
```

### AKS Security Patterns

```
Workload Identity (replaces Pod Identity v1):
├── OIDC issuer: AKS issues tokens for Kubernetes Service Accounts
├── Federated credential: maps Kubernetes SA to Azure Managed Identity
├── Pod gets token, exchanges for Azure AD token → accesses Azure services
└── No secrets stored anywhere: no Managed Identity credentials in cluster

Key Vault Secrets Store CSI Driver:
├── Secrets fetched from Key Vault at pod startup
├── Mounted as files (not environment variables — less exposure)
└── Supports sync to Kubernetes Secrets (for apps expecting env vars)

Network Policies (Calico or Azure CNI):
├── Default: all pod-to-pod communication allowed (Kubernetes default)
└── Best practice: default-deny all ingress/egress, allow explicitly
    netpol: payment-service allows inbound only from api-gateway pods
```

### When to Use AKS

```
Use AKS when:
├── Microservices requiring Kubernetes orchestration (Helm, Operators, CRDs)
├── Team has Kubernetes expertise
├── Multi-cloud portability (same manifests work on AKS/EKS/GKE)
├── Complex workloads: mixed Linux/Windows, GPU, stateful apps
├── GitOps deployment model (ArgoCD, Flux)
└── Enterprise scale: 50+ services, complex networking, multi-team

vs. App Service: AKS for container orchestration complexity; App Service for simple web apps
vs. Azure Functions: AKS for long-running services; Functions for event-driven short tasks
```

---

## 2.4 Azure Functions

### What It Is
Azure Functions is Azure's serverless compute service — run event-triggered code without managing infrastructure. Functions integrate deeply with Azure services via Bindings: declarative input/output connectors that abstract away SDK boilerplate. Durable Functions extend the model with stateful orchestrations, fan-out/fan-in patterns, and long-running workflows.

### Hosting Plans

```
Consumption Plan:
├── True serverless — scale to 0, pay per execution + per GB-second
├── Cold starts: 1–3s (Python/Node), 2–5s (.NET), 3–10s (Java)
├── Max timeout: 5 minutes (configurable to 10)
└── Use for: infrequent or bursty event-driven workloads

Premium Plan (EP1, EP2, EP3):
├── Pre-warmed instances (no cold starts — perpetually warm)
├── VNet Integration (outbound), Private Endpoints (inbound)
├── Longer timeout: 60 minutes (unlimited with Durable Functions)
├── Max instance scale: 100 (vs 200 on Consumption)
└── Use for: latency-sensitive functions, VNet-connected workloads

Dedicated Plan (App Service Plan):
├── Functions run on existing App Service Plan
├── No cold starts (always running), predictable cost
└── Use for: Functions that run continuously (e.g., polling), cost predictability

Flex Consumption (preview, 2024):
├── Consumption pricing + faster cold starts + VNet support
└── Addresses the gap between Consumption and Premium
```

### Durable Functions Patterns

```
Pattern 1: Function Chaining
F1 output → F2 input → F3 input (sequential, stateful)
Use: multi-step data processing pipelines

Pattern 2: Fan-out / Fan-in
Orchestrator → [F1, F2, F3, ... FN] parallel → wait all → aggregate
Use: parallel processing, map-reduce, batch with aggregation

Pattern 3: Async HTTP API (polling)
Client → HTTP trigger → 202 Accepted + status URL
Client polls status URL → Orchestrator checks completion → 200 OK + result
Use: long-running operations exposed via HTTP (> 30s)

Pattern 4: Monitor
while not done:
  check condition
  if not met: wait interval (exponential backoff)
  if met: complete
Use: wait for external condition (job completion, status change)

Pattern 5: Human Interaction
Orchestrator → send approval email → waitForExternalEvent (timeout: 48h)
If approved: continue; If timeout: escalate or cancel
Use: approval workflows, manual checkpoints

Pattern 6: Aggregator (Entity Functions)
Stateful entity accumulates events over time
Use: counters, accumulators, session tracking
```

### When to Use Azure Functions

```
Use Azure Functions when:
├── Event-driven: Blob Storage triggers, Service Bus messages, HTTP webhooks
├── Scheduled jobs: Timer trigger (cron syntax) — daily reports, cleanup
├── Real-time stream processing: Event Hubs trigger (Kafka-compatible)
├── Lightweight APIs: HTTP trigger + Consumption plan (low cost for low traffic)
├── Orchestration: Durable Functions for complex multi-step workflows
└── Integration glue: transform data between Azure services

Use Azure Container Apps instead when:
├── Longer-running containers (>10 min)
├── Custom container runtimes
└── Microservices that are too complex for Functions but simpler than AKS

Use Logic Apps instead when:
├── Low-code/no-code integration
├── Non-developer team building workflows
└── Pre-built connectors to SaaS apps (Salesforce, SAP, ServiceNow)
```

---

## 2.5 Azure Container Apps

### What It Is
Azure Container Apps is a serverless container platform built on Kubernetes and KEDA (Kubernetes Event-Driven Autoscaling). It abstracts away Kubernetes completely — you deploy containers and define scaling rules without managing clusters or nodes. Container Apps automatically scales based on HTTP traffic, event messages (Service Bus, Event Hubs), CPU/memory, or custom metrics — and can scale to zero.

### Container Apps vs AKS vs Azure Functions

```
Azure Container Apps:
├── Abstracts Kubernetes — no kubectl, no node pools, no cluster management
├── Scale to 0 (like Functions), but for containerized long-running services
├── KEDA built-in: scale on Service Bus queue depth, Event Hubs lag, HTTP req/s
├── Dapr built-in: service discovery, pub/sub, secret store, state management
├── Max revision traffic splitting: A/B test at the container level
└── Between Functions (simple, event-driven) and AKS (complex, full control)

Use Container Apps when:
├── Containerized microservices that should scale to zero
├── Want Kubernetes capabilities without managing Kubernetes
├── KEDA-based event-driven scaling without setting up KEDA on AKS
└── Dapr for microservice communication patterns

Use AKS when:
├── Need full Kubernetes API, CRDs, custom Operators
├── Fine-grained node control (custom AMIs, GPU, specific instance types)
└── Complex multi-team cluster governance requirements
```

### Real-World Use Case

**Scenario:** A startup has 8 microservices. Some handle HTTP API traffic (need to scale with requests), others process Service Bus messages (need to scale with queue depth), and they want Dapr for service-to-service communication without implementing service discovery themselves.

```
Container Apps Environment (shared networking for all apps)
├── api-gateway:         HTTP scaling, min 1, max 50 replicas
│                        Dapr enabled (service invocation to internal services)
├── order-processor:     Service Bus scale rule, min 0, max 30 replicas
│                        Scales when queue depth > 10 messages
├── notification-sender: Event Hubs scale rule, min 0, max 20 replicas
│                        Scales with unprocessed event lag
├── product-catalog:     HTTP + min 2 replicas (no scale to zero — needs fast response)
└── report-generator:    CPU scale rule, min 0, max 5 replicas (batch, infrequent)
                         Scale to zero when no jobs running

Dapr service invocation:
order-processor calls product-catalog:
app.get("/order", async (req) => {
  const product = await daprClient.invoker.invoke(
    "product-catalog", "/products/" + req.productId, HttpMethod.GET);
});
// No service discovery code — Dapr handles it
```

---

## 2.6 Azure Container Instances (ACI)

### What It Is
Azure Container Instances (ACI) is the fastest and simplest way to run containers in Azure without managing virtual machines or orchestration platforms. Containers start within seconds and bill per second of CPU and memory used. ACI is designed for isolated, short-duration tasks — not for long-running production services.

### ACI vs Container Apps vs AKS

```
ACI:
├── Single container or container group (sidecar pattern)
├── Billed per-second when running
├── No orchestration, no auto-scaling
├── Start instantly (5–10 seconds to running)
└── Use for: one-off batch jobs, CI/CD build agents, dev testing, scheduled tasks

Container Apps:
├── Multiple containers with auto-scaling, revisions, Dapr
└── Use for: microservices that scale to zero

AKS:
├── Full Kubernetes for complex orchestration
└── Use for: enterprise scale, complex workloads
```

### When to Use ACI

```
Use ACI when:
├── Scheduled batch jobs (trigger via Azure Logic Apps or Event Grid cron)
├── CI/CD pipeline tasks (Azure DevOps agent as ACI container)
├── Dev/test spin-up: run a database container for integration tests, delete after
├── Data processing: one-off transformation job (start, process, stop, billed only for runtime)
└── AKS virtual nodes: burst to ACI when AKS cluster is at capacity

Real cost example:
ACI: 4 vCPU, 8 GiB, runs 2 hours daily for ETL
= $0.0000044/vCPU-second × 4 × 7200s + $0.00000049/GiB-s × 8 × 7200s
= $0.1267 + $0.0282 = $0.155/day = $4.65/month
vs. smallest always-on VM ($20-30/month)
```

---

## 2.7 Azure Logic Apps

### What It Is
Azure Logic Apps is a low-code/no-code integration Platform-as-a-Service for building automated workflows. It provides 400+ pre-built connectors (SAP, Salesforce, ServiceNow, SharePoint, Office 365, SQL, Oracle, etc.) and a visual designer for non-developers. Logic Apps competes with Azure Functions for automation but targets business analysts and integrators rather than developers.

### Logic Apps vs Azure Functions

| Aspect | Logic Apps | Azure Functions |
|--------|-----------|-----------------|
| **Audience** | Business analysts, integration specialists | Developers |
| **Development** | Visual designer, minimal code | Code-first (C#, Python, JS, Java) |
| **Connectors** | 400+ managed connectors | Custom code for each integration |
| **State** | Built-in stateful (Standard tier) | Durable Functions for state |
| **Long-running** | Yes (wait for human, external event) | Yes (Durable Functions) |
| **Pricing** | Per action execution | Per execution + GB-second |
| **Hosting** | Consumption (cloud) or Standard (container) | Multiple plan options |

### When to Use Logic Apps

```
Use Logic Apps when:
├── Business process automation (approval workflows, notifications)
├── SaaS integration (Salesforce → Azure SQL, ServiceNow → Teams notification)
├── EDI/B2B integrations (X12, EDIFACT message processing)
├── Non-developer team building/maintaining the automation
└── Rapid integration without writing code (pre-built connectors save weeks)
```

---


---

# 3. GCP Compute & Serverless Services

---

## 3.1 Google Compute Engine (GCE)

### What It Is
Google Compute Engine is GCP's IaaS offering — virtual machines running on Google's global infrastructure. GCE differentiates from AWS EC2 and Azure VMs in several ways: custom machine types (specify exact vCPU and memory), automatic Sustained Use Discounts (no commitment required), live migration (zero-downtime maintenance), and access to Google's premium global network for inter-region VM communication.

### Machine Type Families

| Family | Series | Optimized For | Example |
|--------|--------|--------------|---------|
| **General Purpose** | N2, N2D, N4, E2, T2D | Balanced workloads | Web servers, dev, small DBs |
| **Compute Optimized** | C2, C3 | High CPU performance | Gaming, HPC, video encoding |
| **Memory Optimized** | M2, M3 | Large in-memory workloads | SAP HANA, in-memory analytics |
| **Accelerator Optimized** | A2, A3 (H100), G2 | ML training/inference | PyTorch, TensorFlow, LLM serving |
| **Storage Optimized** | Z3 | High storage IOPS | NoSQL, large databases |
| **Arm-based** | T2A | Cost-efficient | Scale-out web, microservices |

### Custom Machine Types — GCP Differentiator

```
Standard machine types: fixed CPU/memory ratios
N2: 2 vCPU/8GB, 4/16, 8/32, 16/64, 32/128, 64/256, 80/320

Custom machine types: specify exact vCPU + memory independently
n2-custom-6-40960: 6 vCPU, 40 GB RAM
(Between n2-standard-4 at 16GB and n2-standard-8 at 32GB — but you need 40GB)

Rules:
├── Memory: 0.9 GB to 6.5 GB per vCPU (flexible range)
├── Extended memory: up to 24 GB per vCPU (with extended memory option)
└── Cost: exact CPU + memory you need, no overprovisioning

When custom types help:
├── Application needs 6 vCPU + 48 GB (between standard sizes)
├── Memory-intensive app needs 2 vCPU + 13 GB (not a standard combo)
└── Cost savings: only pay for exactly what's needed
```

### GCE Pricing: Sustained Use Discounts (SUDs)

```
SUDs apply automatically — no reservation, no commitment:
├── VM runs 25% of the month: base price (0% discount)
├── VM runs 50% of the month: ~10% discount (automatic)
├── VM runs 75% of the month: ~20% discount (automatic)
└── VM runs 100% of the month: ~30% discount (automatic)

Plus Committed Use Discounts (CUDs) on top:
├── 1-year CUD: up to 37% savings (on top of SUD)
└── 3-year CUD: up to 55% savings (on top of SUD)

Compare to AWS:
├── AWS On-Demand: full price always (no automatic discount)
├── AWS Reserved: up to 72% BUT requires 1–3 year commitment upfront
└── GCE SUD: 30% automatic discount just for running VMs consistently
```

### Live Migration

GCE's live migration is a significant operational differentiator — during host maintenance, GCE migrates running VMs to different physical hosts transparently while the VM continues running. AWS and Azure typically stop/restart VMs during maintenance (unless you opt into dedicated hosts).

```
GCE Live Migration:
├── VM continues running during physical host maintenance
├── Performance may drop briefly (~10ms latency spike during migration)
├── No reboot required, no user notification needed
└── Exceptions: VMs with local SSDs, GPU VMs → scheduled maintenance (with notification)
```

### Preemptible and Spot VMs

```
Preemptible VMs (legacy):
├── Fixed 24-hour maximum runtime
├── May be preempted at any time (30-second shutdown warning)
└── Up to 80% discount

Spot VMs (replaces Preemptible, current):
├── No maximum runtime (runs until GCP needs the capacity)
├── Still preemptible with 30-second warning
└── Up to 91% discount in some regions

GCE Spot vs AWS Spot vs Azure Spot:
├── AWS Spot: 2-minute warning, capacity-driven interruption
├── Azure Spot: 30-second warning, can set max price bid
└── GCE Spot: 30-second warning, no bidding (pure capacity-driven)
```

### When to Use GCE

```
Use GCE when:
├── Custom CPU/memory ratios avoid overprovisioning costs
├── Consistent workloads benefit from Sustained Use Discounts (no commitment)
├── Live migration requirement (zero-downtime maintenance)
├── GPU workloads: A3 (H100), A2 (A100) for ML training at massive scale
├── SAP HANA (M2 series, certified, up to 12 TB RAM)
└── Tightly integrated with GCP services (BigQuery, GCS, Pub/Sub)
```

---

## 3.2 Google Kubernetes Engine (GKE)

### What It Is
GKE is the most mature managed Kubernetes service — Google invented Kubernetes and runs the largest Kubernetes deployments on Earth (Gmail, YouTube, Google Search all run on Borg, Kubernetes' predecessor). GKE provides the most advanced Kubernetes features, the most stable releases, and the most seamless integration between Kubernetes and the underlying cloud infrastructure.

### GKE Modes

```
GKE Standard (full control):
├── You manage node pools (VM types, sizes, autoscaling config)
├── Full access to all Kubernetes features
├── Pricing: control plane ($0.10/hour for 2+ nodes) + node VMs
└── Use when: need custom node configs, GPU nodes, specific instance types

GKE Autopilot (fully managed nodes):
├── Google manages nodes — you only define pod specs with resource requests
├── No node pools, no node management, no wasted capacity
├── Pricing: per pod (vCPU + memory requested) when pod is running
├── Enforces pod security standards by default
└── Use when: want Kubernetes without managing node infrastructure

Cost comparison example (10 pods, 0.5 vCPU, 1 GB each):
├── GKE Standard: 3 × e2-standard-4 nodes ($0.134/hr) = $0.402/hr
│   Utilization: 5 vCPU used out of 12 = 42% efficiency
└── GKE Autopilot: 5 vCPU × $0.0445/hr + 10GB × $0.00487/hr = $0.271/hr
    100% efficient — pay only for requested resources
```

### GKE Autopilot Security Posture

GKE Autopilot enforces security baselines that are optional in Standard mode:
- Workload Identity required (no GCE service account on nodes)
- No privileged containers allowed
- Host namespace sharing disabled
- Container image from approved registries only
- Pod Security Standards enforced

### GKE Cluster Networking

```
GKE VPC-native clusters (required for Autopilot):
├── Pod IPs are VPC alias IPs (real VPC addresses, not overlay)
├── Pods directly reachable from VPC, on-premises, peered networks
├── No routing overhead — pod IPs are in the VPC routing table natively
└── Container-native load balancing: GCP LB routes directly to pod IPs (no kube-proxy hop)

Dataplane V2 (Cilium eBPF):
├── Replaces iptables-based kube-proxy with eBPF
├── ~30% better network throughput at scale
├── L7 network policies (HTTP path, method-based rules)
├── Hubble observability: real-time flow visualization for every pod connection
└── Available in both Standard and Autopilot

Multi-cluster networking:
├── GKE Gateway API: multi-cluster load balancing
├── Fleet: logical grouping of GKE clusters across regions/projects
└── Traffic Director: GCP-managed service mesh for multi-cluster service discovery
```

### GKE Node Auto-Provisioning (NAP)

Similar to Karpenter on AWS, NAP automatically creates and manages node pools:

```
GKE NAP:
├── Creates new node pools of the optimal machine type for pending pods
├── Scales down and deletes node pools when no longer needed
├── Respects pod resource requests, node affinities, taints
└── Works alongside manual node pools
```

### When to Use GKE

```
Use GKE when:
├── Kubernetes is the deployment standard (most mature managed K8s)
├── Autopilot for hands-off node management (ideal for teams focused on apps)
├── Large-scale ML workloads: TPU node pools, A100/H100 GPU nodes
├── Multi-cluster deployments across regions (Fleet + GKE Gateway API)
└── eBPF networking (Dataplane V2) for high-performance microservices
```

---

## 3.3 Cloud Run

### What It Is
Cloud Run is GCP's fully managed serverless container platform. Deploy any container (any language, any library, any binary) and Cloud Run handles everything: provisioning, scaling from 0 to thousands of instances, TLS termination, load balancing, and monitoring. Cloud Run is conceptually similar to Azure Container Apps, but Cloud Run was released earlier (2019) and is more mature.

### Cloud Run Execution Models

```
Cloud Run Services (HTTP-based, long-running):
├── Receives HTTP requests, scales based on concurrent requests
├── Scale to 0 when no traffic (no idle cost)
├── Min instances: set minimum to avoid cold starts on latency-sensitive paths
├── Max instances: set ceiling to prevent runaway cost
└── Concurrency: 1–1000 concurrent requests per instance (default 80)

Cloud Run Jobs (non-HTTP, batch):
├── Runs container to completion (exit 0 = success, non-zero = failure)
├── Parallelism: run multiple tasks in parallel (--parallelism flag)
├── Retry on failure: configurable retry count per task
└── Trigger: manually, Cloud Scheduler, Eventarc, Workflows
```

### Cloud Run Concurrency Model

```
Cloud Run uses concurrency-based scaling (vs Lambda's per-request scaling):

Lambda: 1 request = 1 execution environment (concurrent requests = concurrent instances)
Cloud Run: 1 instance handles N concurrent requests (default N=80)

Impact:
Lambda processing 1,000 RPS: 1,000 Lambda containers (each handles 1 request)
Cloud Run (concurrency=80): ~13 Cloud Run instances (each handles up to 80 requests)

When to set concurrency=1 (Lambda-like):
├── CPU-bound: each request uses full CPU (no benefit from sharing)
└── Stateful: request creates in-memory state that would conflict

When to keep concurrency=80 (default):
├── I/O-bound: many requests can share while waiting on DB/network
└── Cost efficiency: fewer instances, lower cost
```

### Cloud Run VPC Connectivity

```
Outbound (Cloud Run → VPC resources):
├── Serverless VPC Access: route outbound traffic into a VPC subnet
├── Direct VPC Egress (preferred, newer): direct ENI into VPC, better performance
└── Allows: Cloud Run connecting to private Cloud SQL, Memorystore, GKE services

Inbound (private ingress):
├── Internal ingress: only VPC traffic can trigger Cloud Run (no public internet)
└── Cloud Armor: attach WAF policy to Cloud Run service via load balancer
```

### Cloud Run Min Instances and Cold Starts

```
Cold start breakdown:
├── Instance startup: Cloud Run provisions a container (micro-VM, ~100ms)
├── Container pull: if image not cached at edge (~200–500ms first time)
├── Application startup: your app's initialization code (most variable: 10ms–5s)
└── Total cold start: 200ms–10s depending on runtime and app

Mitigation:
├── Min instances: always keep N warm instances (eliminates cold starts)
│   Cost: N × CPU cost/second even when idle
├── CPU boost at startup: 2× CPU during container initialization
├── Startup probes: don't route traffic until app signals readiness
└── Optimize app startup: lazy initialization, avoid loading all data at startup
```

### Cloud Run vs Cloud Functions vs GKE

```
Cloud Functions:
├── Code-only (no Dockerfile), simplest deployment model
├── Single function per deployment
└── Use for: simple event handlers, webhooks, single-purpose transformations

Cloud Run:
├── Container-based, any runtime, any library
├── HTTP services, background jobs (Cloud Run Jobs)
└── Use for: APIs, microservices, containers that scale to zero

GKE:
├── Full Kubernetes — orchestrate many services, stateful workloads
└── Use for: complex multi-service apps needing full K8s API

Decision rule:
Simple event handler → Cloud Functions
HTTP service / microservice → Cloud Run
Complex orchestration / stateful → GKE
```

### Real-World Use Case

**Scenario:** A startup builds a document processing API. Users upload PDFs → API extracts text + tables using a Python ML library (60 MB dependencies, 3rd party binary). Processing takes 2–30 seconds per document. Traffic: 0–500 req/hour (unpredictable).

```
Deployment: Cloud Run

Why Cloud Run (not Lambda):
├── ML library (60 MB): Lambda deployment limit is 50 MB (unzipped 250 MB) — borderline
├── 3rd party binary: requires custom Docker container (Cloud Run supports this natively)
├── Processing up to 30 seconds: Lambda would be fine (15 min limit), but
└── Concurrency=1: each request uses GPU/CPU heavily, can't share instance

Configuration:
├── Min instances: 0 (scale to zero for cost savings)
├── Max instances: 20 (limit concurrency, protect downstream DB)
├── CPU: 4 vCPU (--cpu=4, needed for ML inference)
├── Memory: 8 GiB (PDF parsing requires large memory)
├── Request timeout: 300s (up to 5 min per document)
├── Concurrency: 1 (CPU-bound, no benefit from concurrent requests)
└── CPU always allocated: false (CPU throttled when not processing — cost saving)

Cost at 500 req/hour average, 10 seconds each:
500 × 10s × 4 vCPU × $0.0000240/vCPU-s = $0.48/hour
500 × 10s × 8 GiB × $0.0000025/GiB-s  = $0.10/hour
Total: ~$0.58/hour = ~$14/day (vs. always-on VM ~$100/day minimum)
```

---

## 3.4 Cloud Functions

### What It Is
Google Cloud Functions is GCP's Functions-as-a-Service (FaaS) offering. Like AWS Lambda and Azure Functions, it runs event-triggered code without server management. Cloud Functions (2nd gen) is built on Cloud Run — functions are packaged as containers internally, giving them Cloud Run's scalability and all Cloud Run features (VPC, min instances, longer timeouts).

### Cloud Functions Generations

```
Cloud Functions 1st gen (legacy):
├── Max 9 minutes timeout, max 8 GB memory
├── No VPC direct egress, no min instances below 1
└── Still supported but not recommended for new deployments

Cloud Functions 2nd gen (current, built on Cloud Run):
├── Max 60 minutes timeout (HTTP), max 9 minutes (event-triggered)
├── Max 32 GiB memory, 8 vCPU
├── Min instances, concurrency, direct VPC egress
└── All Cloud Run features available
```

### Cloud Functions Triggers

```
HTTP Trigger: invoked via HTTPS URL
├── Synchronous: caller waits for response
└── Protected by IAM or allow unauthenticated

Event Triggers (via Eventarc):
├── Google Cloud Storage: object created/deleted/updated
├── Pub/Sub: message published to topic
├── Cloud Firestore: document created/updated/deleted
├── BigQuery: job completed, row inserted (via Pub/Sub)
├── Artifact Registry: new image pushed
└── Any Cloud Audit Log event (any GCP API call)
```

### When to Use Cloud Functions

```
Use Cloud Functions when:
├── Simple event handlers: resize image on GCS upload, parse CSV and write to BigQuery
├── Webhooks: GitHub webhook → Cloud Function → trigger Cloud Build
├── Real-time Firestore/Realtime DB triggers: update aggregation on data write
├── HTTP microservices where simplicity > flexibility
└── Team wants simplest possible deployment (no Dockerfile needed)

Use Cloud Run when:
├── Need Docker container (custom dependencies, binaries)
├── Batch jobs that run to completion (Cloud Run Jobs)
└── More control over concurrency, scaling parameters
```

---

## 3.5 Google Cloud Batch

### What It Is
Google Cloud Batch is a fully managed batch job scheduler for running large-scale batch workloads on GCE VMs. Like AWS Batch, it provisions VMs dynamically based on job requirements, schedules and runs jobs across a fleet of machines, and tears down the fleet when jobs complete.

### Cloud Batch vs GKE Jobs vs Dataflow

```
Cloud Batch:
├── VM-based batch (containers or scripts directly on VMs)
├── Array jobs: same job run N times in parallel with different inputs
├── Spot VMs natively for cost optimization
└── Use for: genomics pipelines, financial risk simulations, rendering farms

GKE Jobs + Workflows:
├── Kubernetes-native batch using Job and CronJob resources
└── Use when: already running GKE and batch is part of the same cluster

Cloud Dataflow:
├── Apache Beam-based stream and batch data processing
└── Use for: ETL, data transformation, real-time analytics (different paradigm)
```

---

## 3.6 Cloud Spanner (Compute Perspective)

### (Brief: Serverless compute tier for a globally distributed DB)

Cloud Spanner introduced a **serverless compute** model that is architecturally relevant for architects. Spanner Serverless:
- Scales processing nodes automatically (0 to thousands) based on query load
- No idle capacity charges — pay per processing unit consumed
- Relevant when designing systems that need global SQL with serverless economics

---

## 3.7 Google App Engine

### What It Is
Google App Engine (GAE) is GCP's original PaaS service (2008) — one of the first cloud PaaS platforms ever. It supports Standard Environment (fast cold starts, sandboxed runtime) and Flexible Environment (custom Docker containers). GAE is largely superseded by Cloud Run for new workloads, but remains in production at many organizations.

### App Engine Environments

```
Standard Environment:
├── Language-specific runtimes: Python, Java, Node.js, PHP, Ruby, Go
├── Scales to 0, extremely fast cold starts (~100ms)
├── Sandboxed: no arbitrary system calls, no writing to disk
└── Legacy but still supported; prefer Cloud Run for new apps

Flexible Environment:
├── Docker container-based
├── Custom runtimes, no restrictions
├── Minimum 1 instance always running (no scale to zero)
└── Replaced by Cloud Run for most use cases
```

### When to Consider App Engine

```
Consider App Engine when:
├── Existing GAE applications (migration cost outweighs new tech benefits)
├── Standard environment for extremely fast cold starts (Python/Go)
└── Otherwise: prefer Cloud Run for new containerized applications
```

---


---

# 4. Cross-Cloud Comparison

---

## 4.1 Compute Service Mapping

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| **Virtual Machines** | EC2 (750+ instance types) | Azure VMs (VMSS for scale) | Compute Engine (custom types) |
| **VM Scale Sets** | EC2 Auto Scaling Groups | VM Scale Sets | Managed Instance Groups |
| **Managed Kubernetes** | EKS | AKS | GKE (most mature) |
| **Serverless Kubernetes** | EKS Fargate Profiles | AKS Virtual Nodes (ACI) | GKE Autopilot |
| **Containers (Serverless)** | ECS Fargate | Azure Container Apps | Cloud Run |
| **Functions (FaaS)** | Lambda | Azure Functions | Cloud Functions (2nd gen) |
| **Container Registry** | ECR | Azure Container Registry | Artifact Registry |
| **Batch Processing** | AWS Batch | Azure Batch | Cloud Batch |
| **Workflow Orchestration** | Step Functions | Logic Apps / Durable Functions | Workflows / Eventarc |
| **Simple Container Deploy** | App Runner | Azure Container Instances | Cloud Run (simplest deploy) |
| **PaaS Web Apps** | Elastic Beanstalk | Azure App Service | App Engine |
| **Node Auto-Provisioning** | Karpenter | N/A (node pool scaling) | GKE NAP |
| **Mac Compute** | EC2 Mac | No equivalent | No equivalent |
| **Bare Metal** | EC2 Bare Metal instances | Azure Bare Metal | Bare Metal Solution |

## 4.2 Serverless Functions Comparison

| Feature | AWS Lambda | Azure Functions | Cloud Functions (2nd gen) |
|---------|-----------|----------------|--------------------------|
| **Max timeout** | 15 minutes | 60 min (Premium) | 60 minutes |
| **Max memory** | 10,008 MB | 14 GB (Premium EP3) | 32 GiB |
| **Max vCPU** | ~6 vCPU (at 10GB) | 4 vCPU (Premium) | 8 vCPU |
| **Scale to zero** | Yes | Yes (Consumption) | Yes |
| **Cold start** | 100ms–1s (Node/Python) | 1–3s (Consumption) | ~100ms–1s |
| **Concurrency model** | 1 request per instance | 1 per instance (default) | Configurable (1–1000) |
| **VNet access** | VPC (adds cold start latency) | Premium Plan | Direct VPC Egress |
| **Container support** | Yes (up to 10 GB image) | Yes (Premium) | Yes (built on Cloud Run) |
| **Warm instances** | Provisioned Concurrency | Pre-warmed (Premium) | Min instances |
| **Account limit** | 1,000 concurrent (default) | 200 instances | Configurable |
| **Pricing (compute)** | $0.0000166667/GB-s | $0.000016/GB-s | $0.0000025/GiB-s (vCPU) |

## 4.3 Container Platform Decision Matrix

```
Scale: Small (< 20 containers) → Medium (20–200) → Large (200+)

Small Scale:
├── AWS:   ECS Fargate or App Runner
├── Azure: Azure Container Apps or Container Instances
└── GCP:   Cloud Run

Medium Scale:
├── AWS:   ECS Fargate (with potential ECS on EC2 for cost)
├── Azure: Azure Container Apps or AKS (if K8s expertise exists)
└── GCP:   Cloud Run or GKE Autopilot

Large Scale / Complex Orchestration:
├── AWS:   EKS (with Karpenter for node management)
├── Azure: AKS
└── GCP:   GKE Standard (most mature)

Multi-Cloud Portability Requirement:
└── All clouds: EKS / AKS / GKE with shared Kubernetes manifests
    (Avoid cloud-specific services: ECS, Container Apps, Cloud Run in this case)
```

## 4.4 Cost Model Comparison

```
Virtual Machine Sustained Cost (comparable general purpose):
4 vCPU, 16 GB, us-east/eastus/us-central, Linux:

AWS:
├── On-Demand m6i.xlarge: $0.192/hour = $140/month
├── 1-year Reserved: $0.121/hour = $88/month (37% savings)
└── 3-year Reserved: $0.076/hour = $55/month (60% savings)

Azure:
├── On-Demand Standard_D4s_v5: $0.192/hour = $140/month
├── 1-year Reserved: $0.119/hour = $87/month (38% savings)
├── 3-year Reserved: $0.074/hour = $54/month (61% savings)
└── With Hybrid Benefit (Windows): -49% additional

GCP:
├── On-Demand n2-standard-4: $0.194/hour = $142/month
├── Sustained Use Discount (full month): $0.136/hour = $99/month (30% auto)
├── 1-year CUD: $0.129/hour = $94/month (34% savings)
└── 3-year CUD: $0.102/hour = $74/month (47% savings)
```

---

---

# 5. Architect Decision Scenarios

---

## Scenario 1: Choosing Between Lambda, Fargate, and EKS

**Business Context:** A financial services company is migrating a monolithic Java application to microservices. The team has 20 services: some are core transaction processors (high throughput, < 100ms SLA), some are reporting services (batch, run daily), and some are event processors (variable load, triggered by Kafka messages).

**Decision Framework:**

```
Service Analysis:

Transaction Processors (8 services):
├── Characteristics: persistent TCP connections to PostgreSQL, < 100ms P99 latency,
│   always-running, 500–2000 TPS per service
├── Lambda: No — persistent DB connections not suitable; cold starts unacceptable
├── ECS Fargate: Maybe — no OS management, but per-task DB connection overhead
└── Decision: ECS Fargate with RDS Proxy (connection pooling) or EKS
   └── EKS chosen: team has growing K8s expertise, Helm charts for consistency

Reporting Services (5 services):
├── Characteristics: run once daily, 2–4 hours duration, 16 vCPU + 32 GB RAM
├── Lambda: No — 15-minute timeout exceeded
├── ECS Fargate: Yes — ECS Tasks (not Services), Fargate Spot, EventBridge Scheduler
└── Decision: ECS Fargate Task (Fargate Spot + On-Demand fallback, ~70% savings)

Event Processors (7 services):
├── Characteristics: Kafka messages, variable 0–5000 msg/s, processing < 5 seconds each
├── Lambda: Yes — Lambda MSK trigger, batchSize=100, auto-scales with partition count
├── ECS Fargate: Possible but Lambda is simpler and cheaper at this scale
└── Decision: Lambda with MSK trigger + reserved concurrency to protect downstream DB

Final Architecture:
├── Core services:      EKS (n2.xlarge nodes, 3 AZs, Karpenter)
├── Batch reports:      ECS Fargate Spot Tasks via EventBridge Scheduler
└── Event processors:   Lambda + MSK trigger
```

---

## Scenario 2: Startup — Fastest Path to Production

**Context:** A 5-person startup has a Python Flask API, a React frontend, and a PostgreSQL database. They need to go from code to production in one week, scale automatically, and minimize DevOps overhead. Budget: < $500/month initial.

**Cloud: GCP (or AWS equivalent)**

```
GCP Stack (fastest path):
├── Frontend: Cloud Storage (static hosting) + Cloud CDN (~$5/month for low traffic)
├── API: Cloud Run (no infrastructure management, scale to 0)
│   ├── Deploy: gcloud run deploy (from GitHub Actions on push to main)
│   ├── Scale: 0 to 100 instances automatically
│   └── Cost: ~$10-30/month at startup traffic levels
├── Database: Cloud SQL PostgreSQL (db-n1-standard-2, smallest with HA)
│   └── ~$70/month with automated backups and HA
└── DNS/LB: Cloud Run built-in HTTPS domain (first step), custom domain later

Total: ~$85-115/month initial

AWS equivalent:
├── Frontend: S3 + CloudFront (~$5/month)
├── API: App Runner (deploy from ECR, auto-scales, no Docker/K8s expertise needed)
│   └── ~$20-40/month at startup traffic
├── Database: RDS PostgreSQL t3.small Multi-AZ (~$55/month)
└── Total: ~$80-100/month

Azure equivalent:
├── Frontend: Azure Static Web Apps (free tier) + Azure CDN
├── API: Azure Container Apps or Azure Functions
└── Database: Azure Database for PostgreSQL Flexible Server (Burstable B1ms) ~$15-20/month

Key insight: At startup scale, all three clouds cost similarly ($80-120/month).
Choose based on: team familiarity, startup credits (GCP/AWS/Azure all offer startup programs),
and which managed services you'll use later (BigQuery → GCP, .NET stack → Azure, etc.)
```

---

## Scenario 3: Multi-Region High Availability — EKS vs GKE

**Context:** A global SaaS company needs applications running in 3 regions (US, EU, APAC). Services must remain available if one region fails. They have 50 microservices. Team is experienced in Kubernetes.

**AWS: Multi-Region EKS**

```
Architecture:
├── 3 EKS clusters (us-east-1, eu-west-1, ap-southeast-1)
├── Global routing: Route 53 Latency policy → ALB per region
├── Data: Aurora Global Database (primary us-east-1, replicas EU + APAC)
├── Container images: ECR replication to all 3 regions (faster image pulls)
├── GitOps: ArgoCD per cluster, pulling from shared Git repo
└── Service mesh: AWS App Mesh or Istio for cross-service mTLS

Challenge: 3 separate clusters = 3× operational overhead
├── Separate cluster upgrades
├── Separate add-on management (metrics, logging, policy)
└── No native cross-cluster service discovery

Solution: EKS with Fleet Manager + GitHub Actions matrix deployments
```

**GCP: Multi-Region GKE (Fleet)**

```
Architecture:
├── 3 GKE Autopilot clusters (us-central1, europe-west1, asia-east1)
├── Fleet: logical grouping of all 3 clusters
│   ├── Fleet-level policy enforcement (Config Management)
│   ├── Fleet-level service mesh (Traffic Director)
│   └── Single pane of glass in Google Cloud Console
├── Global routing: GCP Global External HTTPS LB (single anycast IP, all 3 regions)
├── Multi-cluster Ingress: routes to nearest healthy cluster automatically
├── GKE Gateway API: cross-cluster service discovery via multi-cluster Services
└── Data: Cloud Spanner (globally distributed, multi-region by default)

Advantage: GCP Fleet makes multi-cluster feel like one logical system
Global LB + Multi-cluster Ingress = automatic regional failover without Route 53 config
```

**Decision:**
- **AWS EKS** if already invested in AWS ecosystem, using Aurora, need Graviton nodes
- **GCP GKE** if multi-cluster operations are complex, want Fleet management, or using BigQuery/Spanner

---

## Scenario 4: Serverless Architecture for an Event-Driven System

**Context:** An IoT platform ingests 10,000 device readings per second. Each reading requires: data validation, anomaly detection (ML model), storage to time-series DB, and alerting if threshold exceeded. Average processing time: 50ms per reading.

**Cloud: AWS**

```
Design:
Device → IoT Core → Kinesis Data Streams (10 shards, 1,000 records/s/shard)
                  ↓
          Lambda (Kinesis trigger)
          ├── batchSize: 500 records
          ├── bisectBatchOnFunctionError: true
          ├── startingPosition: LATEST
          ├── parallelizationFactor: 10 (10 Lambda per shard = 100 concurrent)
          └── Processing per batch:
              ├── Validate records (schema check, range validation)
              ├── Call SageMaker endpoint (anomaly detection)
              ├── Write validated readings to Timestream
              └── If anomaly: publish to SNS → Lambda → PagerDuty/Slack

Concurrency math:
10,000 records/s ÷ 500 records/batch = 20 batches/s
20 batches/s × 50ms/batch = 1 concurrent Lambda execution per shard
10 shards × 10 parallelizationFactor = 100 max concurrent Lambdas
→ Well within Lambda limits, auto-scales with Kinesis partition count

Cost:
Lambda: 10,000 records/s × 86,400s/day ÷ 500 (batchSize) = 1.73M invocations/day
= 1.73M × $0.0000002 = $0.35/day invocation cost
Compute: 1.73M × 0.05s × 512MB × $0.0000000166 = ~$0.74/day
Total Lambda: ~$1.10/day = $33/month (not including Kinesis/Timestream costs)

Azure equivalent: Event Hubs → Functions (Event Hubs trigger) → same pattern
GCP equivalent: Pub/Sub → Cloud Functions (Pub/Sub trigger) → same pattern
```

---

## Scenario 5: When to Use Step Functions vs Direct Lambda Chaining

**Anti-pattern:** Three Lambda functions where Lambda A calls Lambda B synchronously, which calls Lambda C synchronously.

```
Problems with Lambda → Lambda direct calls:
├── Error handling: if C fails, B's error handling must know about it explicitly
├── Retries: each function must implement retry logic (duplicated code)
├── Observability: tracing cross-function requires manual X-Ray instrumentation
├── Timeouts: A's 15-min timeout includes B's execution + C's execution (cascading)
└── State: if B succeeds but C fails, rolling back B requires compensating logic

Step Functions Solution:
└── Orchestrator owns the flow; Lambda functions are pure business logic

State Machine:
  ValidateOrder → ReserveInventory → ChargPayment → SendConfirmation

If ChargePayment fails:
├── Step Functions automatically triggers compensating transaction
├── ReleaseInventory (compensate ReserveInventory)
└── SendFailureNotification (compensate ValidateOrder result)

Additional benefits:
├── Visual debugging: see exactly which step failed, with input/output
├── Wait states: pause workflow for hours (polling external system)
├── Parallel branches: ValidateOrder + CheckFraud + CheckInventory in parallel
└── Human approval: pause at ChargePayment, wait for manager approval token

Use Step Functions when:
├── 3+ sequential or parallel Lambda functions
├── Error compensation (saga pattern) needed
├── Human approval or external system polling needed
└── Audit trail of each step's execution required (compliance)
```

---

## Scenario 6: Architect Decision — VM vs Container vs Serverless

**Framework for making the decision:**

```
Step 1: Assess execution characteristics
├── Duration: < 15min → Lambda/Functions; 15min-unlimited → containers/VMs
├── Stateful: persistent TCP connections, local files → containers or VMs
├── Burst pattern: 0 to peak instantly → serverless; predictable → VMs
└── Resource: GPU required → VMs; high memory > 10GB → containers or VMs

Step 2: Assess team and operational factors
├── Kubernetes expertise: yes → EKS/AKS/GKE; no → ECS Fargate / Container Apps / Cloud Run
├── Operational overhead tolerance: low → serverless > PaaS > containers > VMs
└── Compliance: isolated compute → Dedicated VMs or Fargate (micro-VM isolation)

Step 3: Assess cost profile
├── Consistent 24/7 load → VMs with Reserved Instances (cheapest at scale)
├── Variable/bursty load → serverless or containers with auto-scaling
└── Scale to zero required → Lambda/Functions/Cloud Run (zero idle cost)

Step 4: Assess integration
├── Event-driven (S3, SQS, Kafka, Pub/Sub) → Functions/Lambda (native triggers)
├── HTTP microservices → containers (ECS/Cloud Run/Container Apps)
└── Monolith migration → VMs (EC2/GCE/Azure VM) for lift-and-shift first

Decision matrix:
                    | Short (<15min) | Long (any)
Stateless, bursty   | Lambda/Func    | Cloud Run / Fargate
Stateless, steady   | Lambda/Func    | Fargate / Container Apps / Cloud Run
Stateful, persistent| ECS Fargate    | ECS/EKS/AKS/GKE
Full OS control     | N/A            | EC2 / Azure VM / GCE
GPU / ML training   | N/A            | EC2 P/G / Azure NC/ND / GCE A2/A3
```

---

---

# 6. Complex Interview Questions & Answers

---

## Category A: EC2, VMs & Compute Fundamentals

---

### Q1: Explain the difference between EC2 Reserved Instances and Savings Plans. When would you recommend one over the other?

**Answer:**

Both Reserved Instances (RIs) and Savings Plans reduce EC2 costs by committing to a certain usage level for 1 or 3 years in exchange for a discount. The key difference is **flexibility**.

**Reserved Instances (RIs):**
```
Standard RI:
├── Up to 72% savings
├── Locked to: specific instance family, size, region, OS, tenancy
├── Example: m6i.2xlarge, Linux, us-east-1, 1-year, no upfront
└── Cannot change anything — if you switch to m6i.4xlarge, RI doesn't apply

Convertible RI:
├── Up to 66% savings (less than Standard)
├── Can exchange: instance family, size, region, OS (within same region)
└── Flexibility costs ~6% in discount vs Standard RI
```

**Savings Plans:**
```
Compute Savings Plan:
├── Up to 66% savings
├── Applies automatically to: ANY EC2 instance family/size/region/OS + Fargate + Lambda
└── Committed to: a dollar amount per hour (e.g., $10/hour of compute)

EC2 Instance Savings Plan:
├── Up to 72% savings (same as Standard RI)
├── Applies to: specific instance family in a region (e.g., M family in us-east-1)
└── Flexible: any size, any OS within that family/region
```

**When to recommend which:**

```
Savings Plans (Compute) — recommended for most organizations:
├── Your fleet evolves (changing instance types over 1–3 years)
├── Mix of EC2, Fargate, Lambda in your environment
├── You want simplicity: one commitment covers everything
└── Commitment: a dollar/hour floor, not a specific instance

Standard RI:
├── Very stable fleet: same instance type, size, region for 1–3 years
├── Database servers (RDS also uses RIs) that won't change
└── Maximum savings priority over flexibility

Real recommendation for a startup:
"Use Compute Savings Plans. Your architecture will evolve over 3 years —
you'll switch instance families, adopt Fargate, grow Lambda usage.
Compute Savings Plans cover all of this with up to 66% savings and zero lock-in
to a specific instance type. Only use RIs for very stable workloads like
production RDS databases that definitely won't change instance size."
```

---

### Q2: A customer has a Java microservice on EC2 with 3-second cold start times after Auto Scaling events. How would you architect to minimize this impact?

**Answer:**

This is a multi-layered problem combining application startup optimization, infrastructure warm-up patterns, and health check tuning.

```
Root cause analysis:
3-second cold start = JVM startup + Spring context initialization + connection pool warmup
├── JVM startup: ~500ms (HotSpot), ~100ms (GraalVM native)
├── Spring context: load beans, scan packages, configure → 1–2s for large apps
├── DB connection pool: establish N connections to RDS → 500ms–1s
└── Cache warming: load hot data into in-memory cache → variable

Layer 1: Application-level optimizations
├── Spring Boot: enable lazy initialization (spring.main.lazy-initialization=true)
│   Only initializes beans when first needed, not all at startup
├── GraalVM Native Image: compile to native binary (50–100ms startup)
│   Trade-off: slower compilation, no dynamic class loading
├── Tiered compilation: -XX:+TieredCompilation reduces JVM startup by ~30%
├── Pre-warm connection pool: configure minimum pool size = 1 (not 0)
└── Application readiness probe: Spring Actuator /health/readiness
    Returns 503 until context loaded → ALB won't send traffic prematurely

Layer 2: Infrastructure warm-up patterns

Option A: ASG Lifecycle Hook
├── Hook triggers BEFORE instance enters InService state
├── Run pre-warming script (start app, wait for /health 200, then complete lifecycle action)
└── Delay: N seconds while app warms up (remove from traffic until ready)

Option B: Warm Pool (EC2 Auto Scaling)
├── Maintain a pool of pre-initialized (but not in-service) instances
├── When ASG scale-out triggered:
│   ├── Instance from warm pool moves to InService (already running JVM)
│   └── New instance added to warm pool to replace (background initialization)
├── Scale-out time: seconds (not minutes) for warm pool instances
└── Cost: warm pool instances incur EC2 cost (stopped state = just EBS, running = full)

Option C: Predictive Scaling
├── CloudWatch ML model predicts traffic increase
├── Scale-out 5–10 minutes BEFORE predicted peak
└── App initialized and warm before traffic arrives

Layer 3: ALB health check tuning
├── Health check interval: 30s (default) → 10s (faster detection of readiness)
├── Healthy threshold: 2 (default) → 2 consecutive successes before traffic
├── Deregistration delay: 300s (default) → 30s (faster scale-in)
└── Readiness vs liveness: use two separate endpoints
    /health/liveness: is app running? (restart if fails)
    /health/readiness: can app serve traffic? (remove from LB if fails)

Recommended combined approach:
1. Implement Spring Boot readiness probe → ALB only routes when truly ready
2. Add Warm Pool with 2 pre-initialized instances → scale-out time < 5 seconds
3. Enable Predictive Scaling → warm pool instances ready before peak
4. Target GraalVM Native Image for critical path services (accept build complexity)
```

---

### Q3: Explain how Lambda handles concurrency and what the implications are for a downstream database. How would you protect the database from Lambda thundering herd?

**Answer:**

```
Lambda concurrency model:
├── Each Lambda invocation = one execution environment
├── 1,000 concurrent Lambda executions = 1,000 instances simultaneously
├── Burst limit: +500 concurrent executions per minute (regional)
└── Each execution: opens DB connection if not already open

Problem: Lambda to RDS thundering herd
├── SQS queue has 100,000 messages backed up
├── Lambda scales to maximum concurrency (e.g., 1,000 concurrent)
├── Each Lambda opens a new RDS connection on cold start
├── RDS PostgreSQL max_connections (db.r6g.2xlarge) = 5,000
└── 1,000 Lambdas × 5 connections/function = 5,000 connections = max connections!
    ANY additional Lambda invocation or connection attempt fails

Solutions:

Solution 1: RDS Proxy (recommended)
├── RDS Proxy pools and reuses database connections
├── Lambda connects to Proxy endpoint (not directly to RDS)
├── RDS Proxy maintains connection pool to RDS: 20–100 connections (config)
├── 1,000 Lambda invocations → 1,000 Proxy connections → 50 RDS connections
└── RDS Proxy uses IAM auth — no password in Lambda environment variables

Solution 2: Reserved Concurrency Limit
├── Set Lambda reserved concurrency = 50 (hard limit)
├── SQS: messages stay in queue; Lambda processes 50 at a time
├── Effective connection count: 50 × 1 connection = 50 (manageable)
└── Trade-off: reduced throughput; queue processes slower

Solution 3: Connection Pool in Lambda (global scope)
import { Pool } from 'pg';
// Initialized ONCE per execution environment (warm start reuses)
const pool = new Pool({ max: 2, idleTimeoutMillis: 60000 });
// Each Lambda instance uses max 2 connections
// 500 concurrent Lambdas × 2 = 1,000 connections

Solution 4: Amazon Aurora Serverless v2
├── Aurora Serverless v2 supports Data API (HTTP-based, no persistent connections)
└── Lambda calls Data API → no connection count concern at all

Best practice combination:
├── RDS Proxy (primary protection — connection pooling)
├── Reserved concurrency (secondary — prevent unbounded scaling)
└── Connection health monitoring: CloudWatch RDS DatabaseConnections alarm
```

---

## Category B: Kubernetes & Container Deep Dives

---

### Q4: Explain the difference between a Kubernetes Deployment, StatefulSet, and DaemonSet. When would you use each?

**Answer:**

```
Deployment:
├── Manages stateless pods (each pod is interchangeable)
├── Rolling updates: create new pods before killing old (zero downtime)
├── Scale out/in freely: pods can be rescheduled to any node
├── No stable identity: Pod-0 gets replaced by Pod-xyz (random name)
└── Use for: web servers, APIs, microservices, any stateless application

StatefulSet:
├── Manages stateful pods (each pod has a persistent, stable identity)
├── Stable network identity: pod-0, pod-1, pod-2 (predictable names)
├── Stable storage: each pod gets its own PersistentVolumeClaim (doesn't share)
├── Ordered deployment: pod-0 starts and becomes Ready before pod-1 starts
├── Ordered deletion: pod-2 deleted before pod-1 before pod-0
└── Use for: databases (PostgreSQL primary at pod-0), message brokers (Kafka),
           distributed systems requiring leader election (ZooKeeper, etcd)

DaemonSet:
├── Runs exactly ONE pod on EVERY node in the cluster
├── New node added → DaemonSet pod automatically scheduled on it
├── Node removed → DaemonSet pod garbage collected
└── Use for: node-level infrastructure agents:
    ├── Log collectors (Fluent Bit: forward node logs to Loki/CloudWatch)
    ├── Monitoring agents (Datadog Agent, Prometheus Node Exporter)
    ├── Network plugins (CNI agents, kube-proxy)
    └── Security agents (Falco: runtime security monitoring per node)

Job:
├── Runs N pods to completion (exit 0), then terminates
├── Parallel jobs: run N pods concurrently for faster completion
└── Use for: database migrations, batch data processing, report generation

CronJob:
├── Job that runs on a cron schedule
└── Use for: daily backups, scheduled reports, periodic cleanup tasks

Decision examples:
"Deploy a web API with 3 replicas, rolling updates" → Deployment
"Deploy PostgreSQL with data persistence, pod-0 as primary" → StatefulSet
"Install a log forwarding agent on every cluster node" → DaemonSet
"Run a database schema migration before app deployment" → Job (init container or Job)
"Archive audit logs to S3 every night at midnight" → CronJob
```

---

### Q5: What is the difference between Kubernetes resource Requests and Limits? What happens when a pod exceeds its memory limit?

**Answer:**

```
Resource Requests:
├── The amount of CPU/memory Kubernetes GUARANTEES for the pod
├── Scheduler uses requests to decide which node to place pod on
│   (Node must have available capacity >= pod's requests)
├── Under resource pressure: pods with requests get their guaranteed allocation
└── Example: requests: {cpu: "500m", memory: "256Mi"}
   → Scheduler finds a node with 500m free CPU and 256Mi free memory

Resource Limits:
├── The MAXIMUM CPU/memory a pod can use (hard ceiling)
├── CPU limit: container throttled when exceeding limit (CPU shares, not killed)
├── Memory limit: container KILLED (OOMKilled) when exceeding limit
│   OOMKilled = exit code 137, triggers pod restart (depending on restartPolicy)
└── Example: limits: {cpu: "1000m", memory: "512Mi"}

QoS Classes (based on requests/limits):
├── Guaranteed: requests == limits for all containers
│   → Highest priority, last to be evicted under node memory pressure
├── Burstable: requests < limits (or only limits set)
│   → Middle priority, evicted if node memory pressure and pod exceeds requests
└── BestEffort: neither requests nor limits set
   → Lowest priority, first to be evicted under any pressure

What happens when memory limit exceeded:
1. Container exceeds its memory limit (e.g., uses 600Mi with limit 512Mi)
2. Linux OOM Killer kills the container (exit code 137)
3. Kubernetes detects container exit, checks restartPolicy
4. restartPolicy: Always (default) → Kubernetes restarts the container
5. If OOMKill happens repeatedly: pod enters CrashLoopBackOff state
   (exponential backoff: 10s, 20s, 40s... up to 5 minutes between restarts)

Practical implications:
├── Never deploy to production without resource requests (scheduler can't make decisions)
├── Set limits carefully: too low → OOMKills; too high → noisy neighbor
├── Vertical Pod Autoscaler (VPA): automatically adjusts requests/limits based on usage
└── Memory leak detection: sudden OOMKills in production often indicate memory leak

Common mistake: Setting only limits, no requests
├── QoS becomes Burstable (not Guaranteed)
└── Scheduler assumes 0 requests → overschedules nodes → all pods fight for resources
```

---

### Q6: A Kubernetes pod is stuck in "Pending" state. Walk me through your complete diagnostic process.

**Answer:**

```
Pending means: scheduler cannot find a suitable node for the pod.

Step 1: Get immediate reason
kubectl describe pod <pod-name> -n <namespace>
# Look at "Events" section at the bottom

Common events and their meaning:
├── "0/5 nodes are available: 3 Insufficient cpu, 2 node(s) had untolerated taint"
│   → Not enough CPU on 3 nodes; 2 nodes have taint that pod doesn't tolerate
├── "0/5 nodes are available: 5 Insufficient memory"
│   → No node has enough free memory (pods may have large memory requests)
├── "0/5 nodes are available: 5 node(s) didn't match pod affinity"
│   → Pod has nodeAffinity rules requiring specific labels, no matching nodes
└── "no nodes available to schedule pods" → all nodes NotReady (cluster-level issue)

Step 2: Diagnose resource insufficiency
kubectl top nodes  # Check current node utilization
kubectl get nodes  # Check node status (Ready / NotReady / SchedulingDisabled)
kubectl describe nodes | grep -A5 "Allocated resources"
# See: how much CPU/memory requested vs. available on each node

If all nodes are at capacity:
├── Cluster Autoscaler / Karpenter should add nodes
├── Check: is auto-scaling configured and working?
│   kubectl logs -n kube-system deployment/cluster-autoscaler
└── Check: pod resource requests (are they too high?)
   kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources}'

Step 3: Diagnose taint/toleration issues
kubectl get nodes -o json | jq '.items[].spec.taints'
# Check what taints exist on nodes

kubectl get pod <pod> -o jsonpath='{.spec.tolerations}'
# Check what tolerations the pod has

Common taint scenarios:
├── Production nodes tainted: node=prod:NoSchedule
│   Pod needs: tolerations: [{key: "node", value: "prod", effect: "NoSchedule"}]
├── Spot nodes: kubernetes.azure.com/scalesetpriority=spot:NoSchedule (AKS)
└── GPU nodes tainted to prevent non-GPU pods

Step 4: Diagnose affinity/anti-affinity issues
kubectl get pod <pod> -o yaml | grep -A20 "affinity"
# Check required vs preferred affinity rules

├── requiredDuringSchedulingIgnoredDuringExecution → HARD requirement
│   If no nodes match: pod stays Pending forever
└── preferredDuringSchedulingIgnoredDuringExecution → SOFT preference
   If no nodes match: pod still schedules (on any eligible node)

Step 5: Diagnose PVC binding issues
kubectl get pvc -n <namespace>
# If STATUS = Pending, the PVC is not bound to a PV

├── StorageClass not found → check storageClassName in PVC spec
├── No PV available with matching StorageClass → dynamic provisioning failed?
├── Availability zone mismatch: PV in us-east-1a, pod scheduled in us-east-1b
└── Fix for AZ mismatch: use WaitForFirstConsumer volumeBindingMode
   → PV provisioned in same AZ as the pod (not before scheduling)

Step 6: Check node selector / node name
kubectl get pod <pod> -o jsonpath='{.spec.nodeSelector}'
# Pod may require specific node label that doesn't exist
kubectl get nodes --show-labels | grep <required-label>

Full diagnostic flow:
Events say "Insufficient cpu/memory" → check node capacity, scale cluster
Events say "taint"                   → add toleration to pod spec
Events say "affinity"                → relax affinity rules or add matching nodes
Events say "Unschedulable"           → check for cordon: kubectl uncordon <node>
PVC Pending                          → fix storage class or AZ binding mode
```

---

### Q7: How does Kubernetes handle rolling updates? What configuration options control the rollout behavior, and how do you implement a canary deployment?

**Answer:**

```
Rolling Update Mechanism:
kubectl set image deployment/api api=acmecr/api:v2.0

Kubernetes update process:
1. Deployment controller creates new ReplicaSet for v2
2. Scales up new RS by 1 pod (v2 pod starts, passes readiness probe)
3. Scales down old RS by 1 pod (v1 pod removed from LB, terminates)
4. Repeat until: all pods = v2

Configuration in Deployment spec:
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # Max pods unavailable during update (number or %)
      maxSurge: 1          # Max pods above desired count during update

Example (desired=5, maxUnavailable=1, maxSurge=1):
├── Start: 5 × v1 pods
├── Step 1: create 1 × v2 (now 6 pods total), remove 1 × v1 (back to 5)
├── Step 2: create 1 × v2, remove 1 × v1 (continuing...)
└── End: 5 × v2 pods

maxUnavailable=0, maxSurge=1: zero-downtime (always have desired pods serving)
maxUnavailable=25%, maxSurge=25%: faster rollout (default Kubernetes values)

Controlling readiness:
├── readinessProbe: pod must pass before added to LB (critical for rolling update)
└── minReadySeconds: pod must be ready for N seconds before marked available
    (prevents briefly-healthy pods from being counted as ready)

Canary Deployment (native Kubernetes, without service mesh):

Approach 1: Two Deployments, split traffic by replica ratio
# Stable: 9 replicas (90% traffic)
# Canary: 1 replica (10% traffic — same selector as stable!)
Both deployments use labels: app: api
Service selector: app: api → routes to BOTH deployments proportionally
→ 9 stable pods + 1 canary pod = ~10% canary traffic (rough approximation)

Approach 2: Argo Rollouts (recommended for precise canary)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10    # 10% to canary
      - pause: {duration: 5m}  # wait 5 minutes
      - setWeight: 25    # 25% to canary
      - pause: {duration: 10m}
      - analysis:        # run automated analysis
          templates: [{templateName: success-rate}]
      - setWeight: 100   # full rollout

Analysis Template (automatic promotion/abort):
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
spec:
  metrics:
  - name: success-rate
    successCondition: result[0] >= 0.95  # >95% success rate
    provider:
      prometheus:
        query: |
          sum(rate(http_requests_total{status!~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))

Approach 3: Istio (service mesh) for traffic splitting by percentage
# VirtualService splits traffic 90/10 regardless of replica count
spec:
  http:
  - route:
    - destination: {host: api-stable}
      weight: 90
    - destination: {host: api-canary}
      weight: 10
```

---

## Category C: Serverless Architecture Deep Dives

---

### Q8: A Lambda function is experiencing intermittent timeouts processing SQS messages. Describe your complete troubleshooting and resolution approach.

**Answer:**

```
Initial data collection:

1. Check Lambda Insights / CloudWatch metrics:
   ├── Duration: P50, P95, P99 (identify timeout percentile)
   ├── ConcurrentExecutions: are we near reserved concurrency limit?
   ├── Throttles: any throttling events?
   └── Iterator Age (for SQS): how far behind is Lambda on the queue?

2. Identify the timeout pattern:
   ├── All timeouts? → code always slow or network issue
   ├── Specific messages? → message content causing issue (large payloads?)
   ├── At specific times? → downstream service slowness (DB peak load)
   └── After deployments? → code regression

3. Enable Lambda X-Ray tracing:
   ├── Identifies where time is spent: DB calls, external HTTP, Lambda overhead
   └── Sample output: {DB: 12s, HTTP: 0.3s, Code: 0.1s} → DB is the problem

Common causes and solutions:

A. Database connection exhaustion:
├── Symptom: timeouts correlate with high concurrency, long DB connect times in X-Ray
├── Root cause: Lambda opens new DB connection per invocation, RDS max_connections hit
└── Solution: RDS Proxy (connection pooling) or reserved concurrency limit

B. VPC cold start adding latency:
├── Symptom: first invocation after idle period takes 3+ seconds
├── Root cause: ENI attachment for VPC-connected Lambda adds 3–10s cold start
└── Solution: Provisioned Concurrency (pre-warmed instances avoid this)

C. Downstream service timeout propagation:
├── Symptom: Lambda times out, X-Ray shows HTTP call to external API is slow
├── Root cause: external API occasionally slow (> Lambda timeout)
└── Solutions:
    ├── Increase Lambda timeout to match realistic worst-case (but check cost)
    ├── Add circuit breaker (reject calls when external service is failing)
    └── Make processing async: Lambda acknowledges SQS message, sends to DLQ
        for retry rather than timing out and returning message to queue

D. Large SQS message payloads:
├── Symptom: timeouts correlate with large messages
├── Root cause: Lambda spending time downloading 250KB+ payloads per message
└── Solution: SQS Extended Client Library (stores payload in S3, SQS has pointer)

E. Missing SQS timeout configuration:
├── Problem: SQS visibility timeout < Lambda execution timeout
│   Message becomes visible again while Lambda is still processing it
│   → Another Lambda picks it up → duplicate processing
└── Rule: SQS visibility timeout = Lambda timeout × 6
   If Lambda timeout = 30s → SQS visibility timeout = 180s

F. Batch size too large:
├── Problem: batchSize=10, each message takes 3s → 30s total > 15min timeout?
├── No but: if 1 of 10 messages fails, all 10 are retried (no partial success by default)
└── Solution: 
    ├── batchItemFailures: return only failed message IDs (Lambda partial batch response)
    └── bisectBatchOnFunctionError: split batch in half on error (find the bad message)

Resolution checklist:
1. ✅ Set SQS visibility timeout = Lambda timeout × 6
2. ✅ Enable batchItemFailures partial success reporting
3. ✅ Add X-Ray tracing to identify slow component
4. ✅ Add DLQ on SQS source queue (messages that fail N times go to DLQ for investigation)
5. ✅ Set CloudWatch alarm on SQS ApproximateAgeOfOldestMessage (queue falling behind)
6. ✅ Add RDS Proxy if DB connections are the issue
7. ✅ Set reserved concurrency to protect downstream systems
```

---

### Q9: What is the Lambda Execution Environment lifecycle? How does this affect application design?

**Answer:**

```
Lambda Execution Environment lifecycle:

Phase 1: INIT
├── AWS creates a micro-VM (Firecracker) and downloads your code package/container
├── Runtime initializes (Node.js, Python, JVM startup, etc.)
├── Static initializers run (code OUTSIDE the handler function)
│   ├── SDK client initialization
│   ├── Database connection establishment  
│   └── Configuration loading
└── Duration: 100ms–10s (contributes to cold start, NOT billed to you)

Phase 2: INVOKE (repeated, potentially many times)
├── Handler function called with event and context
├── Environment reused between invocations (warm execution)
├── Global state PERSISTS between invocations in the same environment
└── Billed: from handler start to handler return (1ms granularity)

Phase 3: SHUTDOWN (eventual)
├── AWS reclaims environment after period of inactivity (~10–45 minutes typical)
├── SIGTERM sent to runtime → cleanup possible via shutdown extension
└── State in /tmp and global scope is lost

Design implications:

1. Initialize expensive resources in global scope (OUTSIDE handler):
// ✅ CORRECT: DB pool created once per execution environment
const pool = new Pool({ connectionString: process.env.DB_URL });

exports.handler = async (event) => {
  // Pool already exists — no setup time on warm invocations
  const result = await pool.query('SELECT * FROM orders WHERE id = $1', [event.orderId]);
  return result.rows[0];
};

// ❌ WRONG: New connection on every invocation
exports.handler = async (event) => {
  const client = new Client({ connectionString: process.env.DB_URL });
  await client.connect();  // 100–300ms on every invocation!
  ...
};

2. /tmp is NOT shared between invocations in different environments:
└── Cannot use /tmp as cross-invocation cache reliably (different environments get different /tmp)
    Use ElastiCache Redis or DynamoDB for shared caching across Lambda instances

3. Execution environments can run IN PARALLEL:
├── 100 concurrent requests → 100 separate execution environments
└── Global state is NOT shared between environments
    (No thread-safety concerns but also no shared state)

4. Idempotency is required:
├── Same event may be processed multiple times (async invocations retry on failure)
├── SQS + Lambda: at-least-once delivery guarantee
└── Design: check if order/event already processed before processing again
   Use DynamoDB conditional writes or an idempotency key table

5. Extension API (cleanup on shutdown):
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // Flush pending metrics, close DB connections gracefully
    metricPublisher.flush();
    connectionPool.close();
}));

6. Lambda Layers for shared code:
├── Layer: zip file mounted at /opt/ in execution environment
├── Shared across multiple functions (SDK wrappers, utilities, models)
└── Reduce deployment package size; layer can be versioned independently
```

---

### Q10: Compare Azure Durable Functions vs AWS Step Functions for implementing a long-running order processing workflow with human approval. Which would you recommend and why?

**Answer:**

```
Scenario: Order submitted → Validate → Credit Check → If credit OK: auto-approve;
          if borderline: human review (max 48h wait) → Fulfill → Notify

Azure Durable Functions Implementation:
[orchestrator.cs]
[FunctionName("OrderOrchestrator")]
public static async Task<OrderResult> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var order = context.GetInput<Order>();

    // Step 1: Validate (activity function)
    var validResult = await context.CallActivityAsync<bool>("ValidateOrder", order);
    if (!validResult) return OrderResult.Rejected("Validation failed");

    // Step 2: Credit check (activity function calling external API)
    var creditScore = await context.CallActivityAsync<CreditScore>("CheckCredit", order);

    if (creditScore.Score < 600) {
        // Step 3: Human review — pause and wait up to 48 hours
        using (var cts = new CancellationTokenSource(TimeSpan.FromHours(48))) {
            var approvalEvent = await context.WaitForExternalEvent<bool>(
                "ApprovalDecision", cts.Token);
            if (!approvalEvent) return OrderResult.Rejected("Human rejected");
        }
    }

    // Step 4: Fulfill
    await context.CallActivityAsync("FulfillOrder", order);

    // Step 5: Notify
    await context.CallActivityAsync("SendConfirmation", order);
    return OrderResult.Completed();
}

// Human approval: REST call to Durable Functions HTTP endpoint
POST /runtime/webhooks/durabletask/instances/{instanceId}/raiseEvent/ApprovalDecision
Body: true

AWS Step Functions Implementation:
{
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:validate-order",
      "Next": "CreditCheck"
    },
    "CreditCheck": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "credit-check",
        "Payload": {
          "order.$": "$",
          "taskToken.$": "$$.Task.Token"
        }
      },
      "HeartbeatSeconds": 172800,  // 48 hours
      "Next": "CreditDecision"
    },
    "CreditDecision": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.creditScore",
          "NumericLessThan": 600,
          "Next": "HumanReview" }
      ],
      "Default": "FulfillOrder"
    },
    "HumanReview": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "https://sqs.amazonaws.com/.../approval-queue",
        "MessageBody": {
          "taskToken.$": "$$.Task.Token",
          "orderId.$": "$.orderId"
        }
      },
      "HeartbeatSeconds": 172800,
      "Next": "ReviewDecision"
    },
    ...
  }
}

// Human approval: Lambda sends back task token
await sfn.sendTaskSuccess({ taskToken, output: JSON.stringify({approved: true}) });

COMPARISON:

Durable Functions pros:
├── Code-first: workflow is regular C#/Python code with async/await — intuitive for developers
├── Stateful entities: built-in support for aggregators, event sourcing
├── Lower cost: billed per activity function execution (cheaper for simple flows)
└── Replay transparency: understand state by reading code, not JSON state machines

Durable Functions cons:
├── Replay rules: orchestrator code MUST be deterministic (no DateTime.Now, no random)
│   Violations cause non-determinism bugs that are hard to debug
├── Less visual: no native visual workflow editor (unlike Step Functions console)
└── Azure-specific: no cross-cloud portability

Step Functions pros:
├── Visual workflow designer: drag-and-drop state machine, visible to business stakeholders
├── Rich integration: 200+ SDK integrations (call DynamoDB, SQS, SNS directly without Lambda)
├── Audit trail: full execution history with input/output per state (compliance-friendly)
├── Error handling: retry/catch built into each state declaratively
└── Standard Workflows: up to 1-year execution, exactly-once semantics

Step Functions cons:
├── JSON/YAML state machines: complex flows become verbose (1000+ line JSON)
├── Cost: $0.025 per 1,000 state transitions (Standard) — adds up for high-volume flows
└── Debugging: harder to understand complex flows from JSON vs. code

RECOMMENDATION:

"For this order processing scenario with human approval, I'd recommend AWS Step Functions:

1. The visual workflow provides business stakeholder visibility — the 48-hour approval
   deadline, the credit score branching, and the fulfillment sequence are all readable
   without understanding code.

2. Audit trail: Step Functions stores full execution history — who approved, when,
   what the credit score was. Essential for financial compliance.

3. The waitForTaskToken pattern is built specifically for human approval scenarios —
   it pauses execution and resumes when the approver sends the token back.

4. SDK integrations reduce Lambda functions needed — instead of a Lambda that calls SQS,
   Step Functions calls SQS directly.

I'd recommend Durable Functions if the team is C# developers deeply comfortable with
async/await patterns, or if the workflow requires complex stateful aggregation
(Durable Entities) that Step Functions doesn't natively support."
```

---

### Q11: Explain GKE Autopilot vs GKE Standard and when the cost calculation favors each.

**Answer:**

```
GKE Standard — Node-based billing:
├── You define node pools (machine type, size, count)
├── Pay for VMs whether pods use all resources or not
└── Pricing: node VMs + control plane ($0.10/hour for 2+ nodes)

GKE Autopilot — Pod-based billing:
├── Google manages nodes — you only deploy pods
├── Pay for vCPU + memory REQUESTED by running pods
└── Pricing: per-pod resource consumed + control plane ($0.10/hour)

Cost calculation example:

Scenario: 20 pods × 0.5 vCPU × 1 GiB memory, running 24/7

GKE Standard:
Minimum viable HA: 3 nodes (multi-AZ), e2-standard-4 (4 vCPU, 16 GiB) each
= 12 vCPU available, 48 GiB available (for 20 pods needing 10 vCPU, 20 GiB)
Utilization: 10/12 vCPU = 83% (good), 20/48 GiB = 42% (wasted memory)
Cost: 3 × e2-standard-4 × $0.134/hr = $0.402/hr = $293/month

GKE Autopilot:
Pay for exactly: 20 pods × 0.5 vCPU × $0.0445/hr + 20 pods × 1 GiB × $0.00487/hr
= $0.445/hr (vCPU) + $0.0974/hr (memory) = $0.5424/hr = $395/month

Standard WINS here: lower cost because memory utilization is high (packing is good)

Now change scenario: 100 pods × 0.1 vCPU × 0.2 GiB memory

GKE Standard (3 nodes minimum):
Cost: $293/month (same — minimum 3 nodes regardless)
Utilization: 10/12 vCPU = 83% (CPU good), 20/48 GiB = 42% (memory wasted)

GKE Autopilot:
100 pods × 0.1 vCPU × $0.0445/hr + 100 pods × 0.2 GiB × $0.00487/hr
= $0.445/hr + $0.0974/hr = $0.5424/hr = $395/month
(Same cost! Because request profiles similar)

Now: low traffic, 5 pods × 0.5 vCPU × 1 GiB

GKE Standard: still $293/month (minimum 3 nodes)
GKE Autopilot: 5 × 0.5 × $0.0445 + 5 × 1 × $0.00487 = $0.124/hr = $90/month
AUTOPILOT WINS for low replica count workloads

Key insight: 
├── Autopilot wins when: small pod counts, variable workloads, dev/test
├── Standard wins when: high pod density per node (bin-packing efficiency > cost)
│   and when GPU nodes, custom machine types, or specific OS required
└── Always true: Autopilot eliminates node management operational cost
   (patching, right-sizing, node pool management) — factor this in too

Additional Autopilot advantages beyond cost:
├── Spot pods (Autopilot equivalent of Spot VMs — 60-91% discount)
├── Automatic security hardening (no privileged containers, workload identity required)
└── Scale to zero: node removed when no pods scheduled (Standard: manual or CAS)
```

---

## Category D: Architecture Trade-offs

---

### Q12: A company is deciding between running their ML inference workloads on Lambda, ECS Fargate, or EC2. The model is 2 GB, inference takes 500ms, and they expect 50–500 requests per second with variable throughput. What do you recommend?

**Answer:**

```
Let's evaluate each option against the requirements:

Requirements analysis:
├── Model: 2 GB
├── Inference time: 500ms per request
├── Throughput: 50–500 RPS (10× variability)
└── Constraint: no GPU mentioned (CPU inference assumed)

Lambda evaluation:
├── Container image limit: 10 GB ✅ (2 GB model fits)
├── Memory: need 3–4 GB to load 2 GB model + overhead
│   Lambda max memory: 10,008 MB ✅
├── Cold start with 2 GB model image: 5–30s ❌
│   (Container pull + JVM/Python + 2 GB model load = very slow cold start)
├── Concurrency: 50–500 RPS × 500ms = 25–250 concurrent Lambdas ✅
│   (Within the 1,000 concurrent default limit)
├── Cost at 500 RPS: 500 × 0.5s × 4GB × $0.0000000166/ms-GB
│   = 1,000 GB-s/s × $0.0000000166 = $0.0000166/s = $1.43/day
└── PROBLEM: cold start. Loading a 2 GB model on each cold start is unacceptable.

   Mitigation: Provisioned Concurrency (25–50 pre-warmed instances)
   But: Provisioned Concurrency cost = $0.000004646/GB-s × 4GB × 50 warm instances
   = 50 × 4 × 3600 × $0.000004646 = $3,356/month just for warm instances
   → Lambda becomes expensive AND complex for this use case

ECS Fargate evaluation:
├── Container size: 2 GB model + inference code ✅
├── CPU: 4 vCPU per task (500ms per inference, CPU-bound) ✅
├── Memory: 8 GB (2 GB model + 6 GB headroom) ✅
├── Scale: 10–100 tasks based on ALB request count ✅
├── Model loading: once per task startup (not per request) ✅
│   Task startup: ~30–60s (Fargate cold start + model load)
│   But: model is IN MEMORY for the task lifetime (fast subsequent requests)
├── No GPU support in Fargate ⚠️ (CPU inference only)
└── Cost: 25 tasks (for 50 RPS) × 4vCPU × $0.04048 + 25 × 8GB × $0.004445
   = $4.048/hr + $0.889/hr = $4.94/hr = ~$3,593/month
   Scale to 100 tasks for 500 RPS: ~$14,371/month peak

EC2 evaluation (g4dn.xlarge for GPU acceleration):
├── 4 vCPU, 16 GB RAM, 1× T4 GPU (16 GB GPU memory)
├── GPU inference: 500ms → potentially 10–50ms with GPU (10–50× faster)
│   → 1 GPU instance handles 20–100 RPS at 10ms each
├── Auto Scaling: 1–5 g4dn.xlarge instances for 50–500 RPS
└── Cost: g4dn.xlarge On-Demand: $0.526/hr = $383/month per instance
   3 instances (for 500 RPS with headroom): $1,148/month (vs. $14K Fargate)

RECOMMENDATION:

For CPU inference (no GPU):
ECS Fargate is the right answer:
├── Model loaded once per task (not per request) = fast inference after warm-up
├── Auto-scales 10–100 tasks for 10× traffic variability
├── Fargate Spot for 30–50% cost savings (with On-Demand fallback)
└── No infrastructure management vs EC2

For GPU inference (if model can use CUDA):
EC2 g4dn.xlarge instances with Auto Scaling Group:
├── 10–50× faster inference with T4 GPU
├── Significantly cheaper than CPU-only Fargate at this scale
├── Accept: some infrastructure management (but manageable with SSM, patch automation)
└── Use: EC2 Auto Scaling with Target Tracking on ALBRequestCountPerTarget

General principle for ML inference:
< 100 RPS, model < 500MB, latency < 1s: Lambda (with Provisioned Concurrency)
100-10K RPS, 1-5 GB model, latency < 500ms: ECS Fargate or EC2
> 1K RPS or GPU required: EC2 GPU instances (g4dn, p3, inf2)
Production inference at scale: Amazon SageMaker Real-time Inference (managed ML serving)
```

---

### Q13: How would you implement auto-scaling for a Kubernetes deployment that needs to scale based on SQS queue depth (not CPU/memory)?

**Answer:**

```
Standard Kubernetes HPA only scales on CPU/memory metrics (built-in).
For SQS queue depth, we need custom metrics scaling.

Option 1: KEDA (Kubernetes Event-Driven Autoscaling) — recommended

KEDA is an open-source CNCF project that extends Kubernetes with event-driven scaling.
It supports 60+ scalers including SQS, Kafka, Azure Service Bus, Pub/Sub, etc.

Installation:
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace

KEDA requires an IAM role for reading SQS metrics (on EKS use IRSA):
apiVersion: v1
kind: ServiceAccount
metadata:
  name: keda-sqs-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/keda-sqs-reader

ScaledObject definition:
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor  # The Deployment to scale
  minReplicaCount: 0       # Scale to zero when queue empty
  maxReplicaCount: 50      # Never exceed 50 pods
  cooldownPeriod: 60       # Wait 60s before scaling down
  pollingInterval: 15      # Check SQS every 15 seconds

  triggers:
  - type: aws-sqs-queue
    authenticationRef:
      name: keda-aws-credentials  # TriggerAuthentication with IRSA
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/order-events
      queueLength: "10"      # Target: 1 pod per 10 messages
      awsRegion: us-east-1
      identityOwner: operator  # Use KEDA's own IRSA (not pod's SA)

TriggerAuthentication (IRSA):
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-aws-credentials
  namespace: production
spec:
  podIdentity:
    provider: aws  # Uses KEDA's service account IAM role

Behavior: 
├── 0 messages in SQS: order-processor scales to 0 pods
├── 10 messages:   1 pod
├── 100 messages: 10 pods (1 pod per 10 messages)
├── 500 messages: 50 pods (capped at maxReplicaCount)
└── Messages consumed, queue drains: scale back to 0

Option 2: Custom Metrics API + HPA

CloudWatch → External Metrics API (adapter) → Kubernetes HPA

1. Deploy k8s-cloudwatch-adapter (or KEDA external metrics)
2. HPA reads custom metric from adapter
3. HPA scales based on the metric value

HPA with external metric:
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicas: 1  # HPA cannot scale to zero (KEDA can)
  maxReplicas: 50
  metrics:
  - type: External
    external:
      metric:
        name: sqs_messages_visible
        selector:
          matchLabels:
            queue: order-events
      target:
        type: AverageValue
        averageValue: "10"  # 1 replica per 10 messages

Comparison:
KEDA:
├── Scale to ZERO (HPA cannot scale below 1)
├── 60+ supported event sources (SQS, Kafka, Service Bus, Pub/Sub, Redis, cron)
├── CNCF project, actively maintained, production-proven
├── Simpler configuration than Custom Metrics API approach
└── STRONGLY RECOMMENDED for event-driven Kubernetes scaling

Custom Metrics API (HPA only):
├── Cannot scale to zero
├── More complex setup (metrics adapter + HPA configuration)
└── Only reasonable if KEDA cannot be installed for policy reasons

Additional consideration: Effective message processing
With 50 pods processing SQS messages:
├── SQS visibility timeout must exceed pod processing time × retry multiplier
│   Processing: 30s per message → visibility timeout: 300s (safe margin)
├── Message deduplication: same message may appear twice (SQS at-least-once)
│   Design pods for idempotency (check DynamoDB if processed before acting)
└── Dead-letter queue: after 5 failed processing attempts → DLQ for investigation
```

---

## Quick Reference: Architect Cheat Sheet

### Compute Service Selection Guide

```
I need to run code that responds to an HTTP request:
├── Runs < 15 minutes, stateless, variable traffic → Lambda / Cloud Functions / Azure Functions
├── Runs any duration, containerized, scale to zero → Cloud Run / Container Apps / App Runner
└── Long-running service, complex routing → ECS Fargate / AKS / GKE + ALB/AGIC

I need to run a container fleet:
├── No K8s expertise, AWS → ECS Fargate
├── No K8s expertise, Azure → Azure Container Apps
├── No K8s expertise, GCP → Cloud Run or GKE Autopilot
├── K8s expertise → EKS / AKS / GKE Standard
└── GPU workloads → EC2 GPU / Azure NC/ND / GCE A2/A3 with K8s or direct

I need to process messages from a queue/stream:
├── Simple, < 15min → Lambda (SQS), Azure Functions (Service Bus), Cloud Functions (Pub/Sub)
├── Complex, long-running → ECS / Container Apps / Cloud Run Jobs
└── Kubernetes-based → KEDA ScaledObject (all clouds)

I need to orchestrate multi-step workflows:
├── Code-first, Azure → Durable Functions
├── Visual, AWS → Step Functions
└── GCP → Cloud Workflows (simple) or Vertex AI Pipelines (ML)

I need virtual machines:
├── Windows Server + SQL license → Azure VMs (Hybrid Benefit)
├── Custom CPU:memory ratio → GCE (custom machine types)
├── Maximum instance type variety → AWS EC2 (750+ types)
└── Zero-commitment discount → GCE (Sustained Use Discounts)

I need to run batch jobs:
├── AWS → AWS Batch (managed Spot fleet)
├── Azure → Azure Batch (managed VM pool)
├── GCP → Cloud Batch or GKE Jobs
└── All clouds: serverless batch → Lambda/Functions for < 15min jobs
```

### Cold Start Mitigation — All Clouds

```
AWS Lambda:
├── Provisioned Concurrency: pre-warm N instances (eliminates cold starts)
├── SnapStart (Java): save initialized environment snapshot (~10× faster)
└── Minimize package size: strip unused dependencies, use Lambda layers

Azure Functions (Consumption):
├── Premium Plan: pre-warmed instances (always-hot)
├── Always Ready instances: keep N specific function hosts warm
└── Minimize startup: lazy bean initialization, async initialization

Cloud Functions (GCP):
├── Min instances: keep N instances alive
├── Startup probe + CPU boost at startup
└── Optimize imports: move heavy imports inside handler for cold-path code

All platforms:
├── Use lightweight runtimes: Node.js/Python cold starts << Java/.NET
├── Reduce deployment package size (remove unused dependencies)
├── Connection pooling: initialize DB/HTTP clients in global scope
└── If cold starts unacceptable: use containers (Cloud Run/ECS/Container Apps)
   with min instances = 1 (always at least one warm instance)
```

---

*Last Updated: 2025 | Sources: AWS Documentation, Azure Documentation, GCP Documentation*
*For the latest pricing, always verify at aws.amazon.com/pricing, azure.microsoft.com/pricing, cloud.google.com/pricing*
