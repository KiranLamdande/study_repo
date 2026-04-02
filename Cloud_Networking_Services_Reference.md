# Cloud Networking Services: AWS, Azure & GCP
## Complete Reference for Cloud Architects — with Interview Questions & Scenarios

> **Audience:** Cloud Architects, Senior Engineers, Technical Leads
> **Scope:** Networking services across AWS, Azure, and GCP — deep-dive details, architect decision guides, and senior-level interview preparation

---

## Table of Contents

1. [AWS Networking Services](#1-aws-networking-services)
2. [Azure Networking Services](#2-azure-networking-services)
3. [GCP Networking Services](#3-gcp-networking-services)
4. [Cross-Cloud Comparison](#4-cross-cloud-comparison)
5. [Architect Decision Scenarios](#5-architect-decision-scenarios)
6. [Complex Interview Questions & Answers](#6-complex-interview-questions--answers)

---

---

# 1. AWS Networking Services

---

## 1.1 Amazon VPC — Virtual Private Cloud

### What It Is
Amazon VPC lets you provision a logically isolated section of the AWS Cloud where you launch resources in a virtual network you define. You control the IP address range (CIDR), subnet segmentation, route tables, gateways, and network ACLs. Every VPC is completely isolated from other VPCs by default — inter-VPC communication requires explicit configuration.

### Key Components

| Component | Purpose |
|-----------|---------|
| **Subnets** | Sub-segments of the VPC CIDR — public (internet-routable) or private |
| **Internet Gateway (IGW)** | Enables internet access for public subnets |
| **NAT Gateway** | Allows private subnet instances to initiate outbound internet connections |
| **Route Tables** | Define traffic routing paths per subnet |
| **Security Groups** | Stateful, instance-level firewall (allow rules only) |
| **Network ACLs** | Stateless, subnet-level firewall (allow and deny rules) |
| **VPC Endpoints** | Private connections to AWS services without internet traversal |
| **VPC Peering** | Direct network connection between two VPCs (same or different accounts/regions) |
| **Flow Logs** | Capture IP traffic metadata for security and troubleshooting |

### Technical Deep Dive

**CIDR Planning:** VPC supports /16 to /28 prefix. Subnets must be within the VPC CIDR. AWS reserves 5 IPs per subnet (network address, VPC router, DNS, future use, broadcast). A /24 subnet gives 251 usable IPs.

**Security Groups vs NACLs:**
- Security Groups: stateful (return traffic automatically allowed), applied to ENI, support allow rules only, evaluated as a union of all rules
- NACLs: stateless (return traffic needs explicit allow), applied to subnet, support allow and deny, rules evaluated in number order (lowest first wins)

**VPC Peering limitations:**
- Non-transitive: A↔B and B↔C does NOT give A↔C
- No overlapping CIDR blocks allowed between peered VPCs
- No edge-to-edge routing (peered VPC cannot use peer's IGW, VPN, or Direct Connect)

### When to Use

```
Use VPC when:
├── Every production workload in AWS (mandatory baseline)
├── Network isolation between environments (dev/staging/prod in separate VPCs)
├── Compliance requirements (HIPAA, PCI DSS) — isolate sensitive workloads
├── Multi-tier architecture (public ALB → private app servers → private RDS)
└── Hybrid cloud — extend on-premises networks to AWS via VPN or Direct Connect
```

### Real-World Use Case

**Scenario:** A healthcare company hosting an EHR system must ensure patient data never traverses the public internet, developers cannot reach production databases, and all traffic is logged for HIPAA compliance.

**Solution Architecture:**
```
VPC: 10.0.0.0/16
├── Public Subnet (10.0.1.0/24)  → ALB only, no EC2 instances
├── App Subnet (10.0.2.0/24)     → EC2 Auto Scaling Group (private)
├── DB Subnet (10.0.3.0/24)      → RDS Multi-AZ (private, no NAT)
├── VPC Endpoint (S3, Secrets Manager) → no internet for AWS API calls
└── VPC Flow Logs → CloudWatch Logs → HIPAA audit trail
```

---

## 1.2 AWS Transit Gateway (TGW)

### What It Is
AWS Transit Gateway acts as a regional network hub that connects multiple VPCs, AWS accounts, and on-premises networks through a single gateway. It replaces the complex mesh of VPC Peering connections with a hub-and-spoke topology, enabling transitive routing that VPC Peering cannot provide.

### Key Features

| Feature | Detail |
|---------|--------|
| **Attachments** | VPCs, VPN connections, Direct Connect gateways, peered TGWs |
| **Route Tables** | Multiple per TGW for traffic segmentation (e.g., shared-services vs. isolated VPCs) |
| **Multicast** | Native IP multicast support across VPCs |
| **Inter-region peering** | Connect TGWs in different regions over the AWS backbone |
| **Bandwidth** | Up to 50 Gbps per VPC attachment |
| **Equal-Cost Multipath** | ECMP for VPN connections to maximize throughput |

### Technical Deep Dive

**TGW Route Tables for Segmentation:**
```
TGW Route Tables:
├── "shared-services-rt" → Can reach all VPCs (DNS, monitoring, patch)
├── "prod-rt"           → Can reach shared-services, NOT staging/dev
├── "staging-rt"        → Can reach shared-services, NOT prod/dev
└── "dev-rt"            → Can reach shared-services, NOT prod/staging

Each VPC attachment associates with one route table
and propagates routes to relevant tables only.
```

**ECMP for VPN throughput:** A single VPN connection caps at 1.25 Gbps. With TGW, you can run multiple VPN connections and use ECMP to aggregate bandwidth: 4 VPN connections = up to 5 Gbps effective throughput.

### When to Use

```
Use Transit Gateway when:
├── Managing 5+ VPCs — peering mesh becomes unmanageable (N*(N-1)/2 connections)
├── Multi-account AWS Organizations architecture
├── Centralized egress inspection via a shared firewall VPC
├── Connecting multiple branch offices via VPN through a single hub
└── Need transitive routing — something VPC Peering explicitly cannot do
```

### Real-World Use Case

**Scenario:** A financial services firm has 40 VPCs across 8 AWS accounts (prod, dev, shared-services, security, logging, data, compliance, sandbox). They need to ensure prod cannot communicate with dev, both can reach shared-services (DNS, monitoring), and all traffic passes through a centralized firewall.

**Solution:** 
```
Transit Gateway with 4 route tables:
1. Prod RT     → routes to shared-services only
2. Dev RT      → routes to shared-services only  
3. SharedSvc RT→ routes to all VPCs
4. Security RT → routes to all VPCs (firewall inspection VPC)

All traffic to/from internet routes through Security VPC
(Gateway Load Balancer + network appliances) via TGW
```

---

## 1.3 AWS Direct Connect

### What It Is
AWS Direct Connect provides a dedicated, private network connection from on-premises facilities to AWS, bypassing the public internet. It offers consistent network performance (no internet congestion), reduced bandwidth costs for sustained high-volume data transfer, and the ability to access private VPC resources from on-premises without a VPN tunnel overhead.

### Connection Types

| Type | Bandwidth | Use Case |
|------|-----------|---------|
| **Dedicated Connection** | 1 Gbps, 10 Gbps, 100 Gbps | Large enterprises, colocation |
| **Hosted Connection** | 50 Mbps – 10 Gbps | Via APN partner, flexible bandwidth |
| **Direct Connect Gateway** | – | Connect one DX to multiple VPCs/regions |
| **SiteLink** | – | Route on-premises to on-premises via AWS backbone |

### Technical Deep Dive

**Resilience Patterns:**
```
Level 1 — Development (no resilience):
  On-prem → 1× Direct Connect → AWS

Level 2 — High Resilience (recommended):
  On-prem → DX Location A (primary)   → AWS
          → DX Location B (secondary) → AWS

Level 3 — Maximum Resilience:
  On-prem Site 1 → DX Location A → AWS
  On-prem Site 1 → DX Location B → AWS
  On-prem Site 2 → DX Location C → AWS
  On-prem Site 2 → DX Location D → AWS
```

**BGP Routing:** Direct Connect uses BGP (Border Gateway Protocol). You can influence path selection using BGP attributes: AS_PATH prepending (make a path look longer/less preferred) and MED (Multi-Exit Discriminator) values.

### When to Use

```
Use Direct Connect when:
├── Transferring >1 TB/day to/from AWS (cost savings vs internet data transfer)
├── Latency-sensitive workloads (trading, real-time data, VDI)
├── Compliance requires data to not traverse the public internet
├── Consistent, predictable network throughput (not subject to internet congestion)
└── Connecting to multiple AWS regions via Direct Connect Gateway
```

### Real-World Use Case

**Scenario:** An investment bank runs algorithmic trading systems that need sub-millisecond data feeds from AWS market data services to on-premises execution engines. VPN latency (~5ms jitter) is unacceptable. They transfer 10 TB daily — internet egress costs would be $900/day.

**Solution:** 10 Gbps Dedicated Direct Connect connection from their NYC data center to the nearest AWS Direct Connect location (Equinix NY5). Direct Connect eliminates internet jitter (consistent ~1.2ms RTT) and reduces data transfer cost to $0.02/GB vs $0.09/GB internet egress — saving $700/day.

---

## 1.4 Elastic Load Balancing (ELB)

### What It Is
AWS offers four load balancer types under the ELB umbrella. Each targets a different layer and use case — choosing the correct one is a common architect decision point.

### Load Balancer Types Compared

| Type | Layer | Protocol | Key Feature | Use Case |
|------|-------|---------|-------------|---------|
| **Application LB (ALB)** | 7 (HTTP) | HTTP, HTTPS, gRPC, WebSocket | URL/header/query routing, WAF integration | Web apps, microservices, APIs |
| **Network LB (NLB)** | 4 (TCP) | TCP, UDP, TLS | Ultra-low latency, static IPs, millions of req/s | High-throughput APIs, gaming, IoT |
| **Gateway LB (GWLB)** | 3 (IP) | All IP | Transparent traffic inspection | Network appliances (IDS/IPS, firewalls) |
| **Classic LB (CLB)** | 4 & 7 | TCP, HTTP | Legacy | Legacy EC2-Classic (avoid for new) |

### ALB Technical Deep Dive

**Routing Rules Priority:**
```
Rule 1 (priority 1): IF path = /api/v2/* → target-group-v2
Rule 2 (priority 2): IF header X-Canary = true → target-group-canary
Rule 3 (priority 3): IF query string version=beta → target-group-beta
Rule 4 (default):    → target-group-stable
```

**ALB Slow Start Mode:** Gradually increases traffic to newly registered targets over 30–900 seconds, preventing cold-start overload on new instances.

**NLB Key Properties:**
- Preserves client source IP (unlike ALB which uses X-Forwarded-For)
- Supports static Elastic IPs per AZ (useful for firewall whitelisting)
- Handles millions of requests per second at ~100µs latency
- Supports UDP (ALB does not)

### When to Use

```
ALB: Web applications, REST APIs, gRPC microservices, WebSocket, WAF required
NLB: Ultra-low latency TCP/UDP, static IP requirements, millions req/s, IoT/gaming
GWLB: Inserting 3rd-party network appliances (Palo Alto, Fortinet) transparently
```

### Real-World Use Case

**Scenario:** A gaming company needs to route WebSocket connections for real-time game state to game server pods in EKS, while HTTP lobby APIs go to a separate service. They also need UDP for voice chat.

**Solution:**
- **ALB** with WebSocket support for game state (HTTP Upgrade → WebSocket)
- **NLB** with UDP listener for voice chat (ALB does not support UDP)
- ALB listener rules route `/lobby/*` to lobby target group and `/game/*` to game-server target group

---

## 1.5 Amazon Route 53

### What It Is
Route 53 is AWS's highly available and scalable DNS and domain registration service. Beyond standard DNS, Route 53 provides health checking, traffic routing policies, and private hosted zones for DNS within VPCs. It is a critical component for multi-region architectures and disaster recovery.

### Routing Policies

| Policy | Behavior | Use Case |
|--------|----------|---------|
| **Simple** | Returns all values; client picks randomly | Single resource |
| **Weighted** | Split traffic by assigned weight (0–255) | A/B testing, canary deploys |
| **Latency** | Routes to region with lowest measured latency | Multi-region active-active |
| **Failover** | Primary/secondary with health checks | Active-passive DR |
| **Geolocation** | Routes based on user's geographic location | Content localization, compliance |
| **Geoproximity** | Routes based on location + bias factor | Traffic shifting between regions |
| **Multi-value** | Returns up to 8 healthy records | Simple load distribution |
| **IP-based** | Routes based on client CIDR block | ISP-level routing, network topology |

### Health Checks

Route 53 health checks can monitor:
- HTTP/HTTPS endpoints (evaluate response body for string match)
- TCP connections
- CloudWatch Alarms (composite health checks)
- Other Route 53 health checks (calculated health checks)

Health check intervals: 30s (standard) or 10s (fast, extra cost). Threshold: 3 consecutive failures = unhealthy.

### When to Use

```
Route 53 when:
├── Multi-region DNS failover (Failover policy + health checks)
├── Weighted routing for blue-green or canary deployments
├── Latency routing for global active-active architectures
├── Geolocation routing for data residency compliance (EU traffic → EU region)
├── Private hosted zones for internal service discovery in VPCs
└── DNS-level DDoS mitigation (Route 53 is anycast, 100% availability SLA)
```

### Real-World Use Case

**Scenario:** A global SaaS company needs EU users' data to stay in Europe (GDPR), APAC users to go to the Singapore region, and US users to the us-east-1 primary with failover to us-west-2.

**DNS Architecture:**
```
app.acme.com
├── Geolocation: EU countries → eu-west-1.acme.com (Failover: primary)
│   └── Failover: health check fails → eu-west-2.acme.com (secondary)
├── Geolocation: APAC countries → ap-southeast-1.acme.com
└── Geolocation: Default (Americas + others) → us-east-1.acme.com
    └── Failover: health check fails → us-west-2.acme.com
```

---

## 1.6 AWS WAF & AWS Shield

### What It Is

**AWS WAF** is a web application firewall operating at Layer 7 that protects applications against common web exploits. It inspects HTTP/HTTPS requests and applies rules to block or allow traffic based on conditions like IP addresses, HTTP headers, body content, URI strings, SQL injection patterns, and cross-site scripting.

**AWS Shield** provides DDoS protection:
- **Shield Standard:** Automatic, free, protects against common network/transport layer DDoS attacks
- **Shield Advanced:** $3,000/month, 24/7 DDoS response team, cost protection, enhanced detection, application-layer DDoS mitigation

### WAF Rule Types

| Rule Type | Example |
|-----------|---------|
| **IP set match** | Block traffic from known malicious IPs |
| **Geographic match** | Block all traffic from specific countries |
| **String match** | Block requests containing specific strings in URI |
| **Regex pattern** | Block requests matching regex patterns |
| **Rate-based rules** | Limit to 1,000 requests per 5 minutes per IP |
| **Managed rule groups** | AWS-managed OWASP Top 10, bot control, known bad inputs |
| **Bot Control** | Identify and manage bot traffic (scrapers, crawlers, credential stuffers) |

### When to Use

```
WAF: Web applications exposed to internet, OWASP Top 10 protection, rate limiting bots
Shield Standard: Always on, no configuration needed, all AWS customers get this
Shield Advanced: High-profile targets, financial services, gaming, DDoS cost protection
```

### Real-World Use Case

**Scenario:** An e-commerce company experiences credential stuffing attacks (bots testing 50,000 stolen username/password pairs per hour against their login endpoint) and SQL injection attempts on their product search API.

**WAF Rule Configuration:**
```
Rule 1 (Priority 0): AWS Managed — AWSManagedRulesKnownBadInputsRuleSet
Rule 2 (Priority 1): AWS Managed — AWSManagedRulesSQLiRuleSet
Rule 3 (Priority 2): Rate-based — Block IPs exceeding 100 req/5min to /login
Rule 4 (Priority 3): Bot Control — Challenge known bots, CAPTCHA suspicious bots
Rule 5 (Priority 4): Geo-block — Block countries with no legitimate customer base
```

---

## 1.7 AWS PrivateLink

### What It Is
AWS PrivateLink enables private connectivity between VPCs and AWS services, or your own services, without requiring internet access, VPC peering, or NAT devices. Traffic flows over the AWS network backbone using private IP addresses. It is the technology powering VPC Endpoints for AWS services and enables SaaS providers to offer their services privately to customers.

### Components

| Component | Description |
|-----------|-------------|
| **Interface Endpoint** | ENI with private IP; traffic to AWS service stays private |
| **Gateway Endpoint** | For S3 and DynamoDB only; route table entry (no ENI, no cost) |
| **Endpoint Service** | Your own service exposed via PrivateLink to other VPCs/accounts |

### When to Use

```
Use PrivateLink when:
├── Security requires AWS API calls (S3, KMS, Secrets Manager) to stay off internet
├── SaaS provider: offer your service to customers without VPC peering
├── Large-scale: PrivateLink scales to millions of requests/s (VPC Peering has limits)
├── Cross-account service sharing without exposing VPC CIDRs
└── Compliance: payment processors or healthcare services requiring private connectivity
```

### Real-World Use Case

**Scenario:** A payment processing company offers their payment API to 500 bank customers. Exposing a public endpoint is unacceptable for PCI compliance. VPC peering with 500 customers is operationally impossible.

**Solution:** Create an Endpoint Service backed by an NLB. Each of the 500 bank customers creates a VPC Interface Endpoint in their own VPC pointing to the payment processor's Endpoint Service. Traffic never leaves the AWS backbone. The payment processor whitelists each customer's AWS account ID to accept endpoint connection requests.

---

## 1.8 Amazon CloudFront

### What It Is
Amazon CloudFront is a globally distributed Content Delivery Network (CDN) with 450+ Points of Presence (PoPs) in 90+ cities across 47 countries. It caches and serves content from the edge closest to end users, reduces latency, and protects origins. CloudFront integrates natively with WAF, Shield Advanced, Lambda@Edge, and CloudFront Functions for edge compute.

### Caching Behaviors

```
CloudFront Distribution:
├── Default behavior: /* → S3 origin (static assets, cache 86400s TTL)
├── Behavior: /api/* → ALB origin (no cache, or short TTL)
├── Behavior: /images/* → S3 origin (long cache 604800s TTL)
└── Behavior: /auth/* → ALB origin (no cache, no cookies stripped)
```

### Edge Compute Options

| Option | Runtime | Max Duration | Use Case |
|--------|---------|-------------|---------|
| **Lambda@Edge** | Node.js, Python | 30s (viewer), 30s (origin) | Complex request manipulation, A/B testing, auth |
| **CloudFront Functions** | JavaScript (subset) | <1ms | Simple header manipulation, URL rewrites, cache key normalization |

### When to Use

```
CloudFront when:
├── Global user base needs low-latency access to static/dynamic content
├── DDoS protection at the edge (absorb attacks before origin)
├── Origin offloading — reduce load on application servers
├── HTTPS everywhere with custom certificates (ACM integration)
├── Video streaming (HLS/DASH — Progressive download or live streaming)
└── Edge authentication/authorization before requests reach origin
```

---

# 2. Azure Networking Services

---

## 2.1 Azure Virtual Network (VNet)

### What It Is
Azure Virtual Network is Azure's private networking fabric, providing isolated network environments for Azure resources. VNets are regional (a single VNet spans all AZs in a region) and support IPv4 and dual-stack IPv4/IPv6. Unlike AWS VPCs (which are regional but subnets are AZ-scoped), Azure subnets span the entire region.

### Key Components

| Component | Purpose |
|-----------|---------|
| **Subnets** | Sub-divisions of VNet address space — resources deployed into subnets |
| **Network Security Groups (NSG)** | Stateful L4 firewall — applied to subnet or NIC |
| **Application Security Groups (ASG)** | Tag-based NSG rules (group VMs by role, not IP) |
| **Route Tables (UDR)** | Override default Azure routing for custom traffic paths |
| **Service Endpoints** | Direct route to Azure PaaS (SQL, Storage) over Azure backbone |
| **Private Endpoints** | Private IP for Azure PaaS services inside your VNet |
| **VNet Peering** | Connect VNets (same or cross-region) — non-transitive |
| **VNet Gateway** | VPN or ExpressRoute gateway for hybrid connectivity |

### Azure vs AWS VNet Differences

| Aspect | Azure VNet | AWS VPC |
|--------|-----------|---------|
| **Subnet scope** | Regional (spans all AZs) | AZ-specific |
| **Transitive routing** | Via Azure Firewall or NVA | Via Transit Gateway |
| **Default DNS** | 168.63.129.16 (Azure DNS) | 169.254.169.253 (AmazonProvidedDNS) |
| **Security model** | NSG (stateful) + Azure Firewall | Security Groups + NACL |
| **Internet egress** | NAT Gateway or Azure Firewall | NAT Gateway |
| **Hub-and-spoke** | Azure Virtual WAN or hub VNet + peering | Transit Gateway |

### When to Use

```
Azure VNet when:
├── Every Azure workload requiring network isolation (mandatory baseline)
├── Hybrid connectivity — extend on-premises to Azure via VPN or ExpressRoute
├── Multi-tier application isolation (App Gateway subnet → App subnet → DB subnet)
├── Private connectivity to PaaS via Private Endpoints (SQL, Storage, Key Vault)
└── Zero-trust architecture — combine with Azure Firewall, NSG, and UDR
```

### Real-World Use Case

**Scenario:** A manufacturing company runs an SAP S/4HANA workload on Azure Large Instances, which must connect to Azure SQL Database, Azure Blob Storage, and their on-premises ERP over ExpressRoute — all without internet exposure.

**Architecture:**
```
VNet: 10.10.0.0/16
├── /24 Subnet: SAP Large Instances (specialized hardware)
├── /24 Subnet: App Gateway (WAF for external portal only)
├── /24 Subnet: Private Endpoints (SQL, Storage, Key Vault)
├── /27 Subnet: GatewaySubnet (ExpressRoute Gateway)
└── UDR: Force all traffic through Azure Firewall
    └── Azure Firewall: FQDN filtering, threat intelligence
```

---

## 2.2 Azure Application Gateway + WAF

### What It Is
Azure Application Gateway is a regional, layer-7 load balancer and application delivery controller. WAF v2 adds OWASP 3.2 protection with managed rule sets, custom rules, and bot protection. It is the standard entry point for web applications hosted in Azure.

### Features

| Feature | Detail |
|---------|--------|
| **SKU** | Standard v2 (LB only), WAF v2 (LB + firewall) |
| **Routing** | URL path, multi-site (multiple hostnames on one gateway) |
| **SSL** | End-to-end SSL, SSL offloading, certificate management (Key Vault integration) |
| **Autoscaling** | 0–125 Capacity Units automatically |
| **Zone redundancy** | Spans all AZs in a region |
| **Cookie affinity** | Session stickiness via ARRAffinity cookie |
| **Health probes** | Custom HTTP/HTTPS probes per backend pool |
| **Rewrite rules** | Modify request/response headers and URLs |

### Application Gateway vs Azure Front Door vs Azure Load Balancer

| Service | Layer | Scope | Primary Use |
|---------|-------|-------|-------------|
| **Azure Load Balancer** | 4 | Regional | TCP/UDP load balancing within a region |
| **Application Gateway** | 7 | Regional | HTTP(S) with WAF for a single region |
| **Azure Front Door** | 7 | Global | CDN + global HTTP(S) LB + WAF across regions |
| **Traffic Manager** | DNS | Global | DNS-level routing across regions |

### When to Use

```
Application Gateway when:
├── Single-region web application needing WAF (OWASP protection)
├── Multiple applications behind one entry point (multi-site hosting)
├── SSL termination with certificates managed in Azure Key Vault
├── AKS Ingress controller (AGIC — Application Gateway Ingress Controller)
└── URL path-based routing to different backend services/microservices
```

### Real-World Use Case

**Scenario:** A hospital system exposes a patient portal (HTTPS), an admin interface (/admin/*), and a FHIR API (/fhir/*) through a single entry point with OWASP protection, and connects all backends via Private Endpoints.

```
agw-hospital.azurewebsites.net (WAF v2, Prevention mode, OWASP 3.2)
├── /admin/*    → backend pool: admin-app-service (internal only)
├── /fhir/*     → backend pool: fhir-api-app-service
└── Default /*  → backend pool: patient-portal-app-service

All backend pools accessed via Private Endpoints (no public internet)
WAF custom rule: Block any request with Authorization header containing "Basic"
Rate limit: 500 req/min per IP to /fhir/*
```

---

## 2.3 Azure Front Door

### What It Is
Azure Front Door (AFD) is a global, scalable entry point for web applications using Microsoft's global edge network (190+ edge locations). It combines global HTTP/HTTPS load balancing, CDN caching, WAF, TLS termination, and URL routing into a single service. Unlike Application Gateway (regional), Front Door routes traffic globally to the nearest healthy origin.

### Tiers

| Tier | Features |
|------|---------|
| **Standard** | CDN, global LB, custom rules, 100 rules/policy |
| **Premium** | Standard + WAF managed rules, bot protection, Private Link origins, security analytics |
| **Classic** | Legacy tier — migrate to Standard/Premium |

### Key Differentiator: Private Link Origins (Premium)
Front Door Premium can connect to backends via Azure Private Link, meaning your App Service, Azure Functions, or internal Load Balancer never needs a public IP. Traffic from Front Door edge to origin is private.

### When to Use

```
Azure Front Door when:
├── Global application serving users across multiple regions
├── Active-active multi-region deployments with automatic failover
├── CDN + WAF + global LB without managing separate services
├── Origin servers must remain private (Private Link origins, Premium tier)
└── Need latency-based routing, weighted routing, and session affinity globally
```

### Real-World Use Case

**Scenario:** A global retailer has App Service deployments in West Europe, East US, and Southeast Asia. They need users routed to the nearest region, automatic failover if a region goes down, WAF protection, and CDN caching for product images — all while keeping App Services private (no public IPs).

**Solution:** Azure Front Door Premium:
- 3 origin groups (one per region) with health probes every 30s
- Latency-based routing routes users to nearest healthy origin
- Private Link origins: App Services have no public endpoints
- WAF with OWASP managed rules + bot protection
- Cache rules: product images cached for 7 days at edge; API responses not cached

---

## 2.4 Azure ExpressRoute

### What It Is
Azure ExpressRoute provides a dedicated, private connection from on-premises infrastructure to Azure datacenters, established through a connectivity provider's colocation facility. Traffic does not traverse the public internet, offering consistent latency, higher bandwidth, and stronger security guarantees than VPN.

### Circuit Types & Models

| Model | Description |
|-------|-------------|
| **ExpressRoute Circuit** | Dedicated circuit at a connectivity provider (50 Mbps – 100 Gbps) |
| **ExpressRoute Direct** | Direct 10 Gbps or 100 Gbps port into Microsoft's network |
| **ExpressRoute Global Reach** | On-prem site A ↔ Azure ↔ On-prem site B (site-to-site via Microsoft backbone) |
| **ExpressRoute FastPath** | Bypass ExpressRoute Gateway for ultra-low latency (Direct only) |

### Peering Types

```
Microsoft Peering:   On-premises → Microsoft 365, Azure PaaS (public IPs)
Private Peering:     On-premises → Azure VNet private address space
```

### Resilience: Active-Active vs Active-Passive

```
Active-Active (recommended):
  Site A → Provider A → ExpressRoute Circuit 1 → Azure
  Site B → Provider B → ExpressRoute Circuit 2 → Azure
  Both circuits carry traffic simultaneously (BGP load balancing)

Active-Passive (Failover):
  Primary: ExpressRoute Circuit
  Backup:  Site-to-Site VPN (lower cost, kicks in on DX failure)
```

### When to Use

```
ExpressRoute when:
├── >1 Gbps sustained bandwidth requirement between on-premises and Azure
├── Latency-sensitive workloads (< 10ms requirement, e.g., SAP, VDI, trading)
├── Compliance mandates private connectivity (no public internet data traversal)
├── Consistent, predictable throughput (not subject to internet variability)
└── Multi-site WAN: use Global Reach to route between global offices via Azure
```

---

## 2.5 Azure Virtual WAN (vWAN)

### What It Is
Azure Virtual WAN is a networking service that provides a unified wide-area network architecture — consolidating VNet connectivity, site-to-site VPN, point-to-site VPN, and ExpressRoute into a single managed hub. It is Microsoft's equivalent to AWS Transit Gateway, but with additional WAN orchestration capabilities and automated hub routing.

### Hub Types

| Hub Type | Included Services |
|----------|-----------------|
| **Basic** | Site-to-site VPN only |
| **Standard** | VPN + ExpressRoute + VNet peering + Azure Firewall |

### When to Use

```
Azure Virtual WAN when:
├── Large-scale networks: 100+ VNets or 50+ branch offices
├── Global SD-WAN integration (partner integrations: Cisco, Barracuda, VMware)
├── Automated hub routing (no manual route table management)
├── Any-to-any connectivity: branches ↔ VNets ↔ other branches
└── Replacing a complex manual hub-and-spoke peering architecture
```

### Real-World Use Case

**Scenario:** A multinational with 80 branch offices across 4 continents, 60 Azure VNets across 6 regions, and 3 large data centers connected via ExpressRoute needs unified WAN management.

**Solution:** Azure Virtual WAN with Standard Hubs in 6 regions. Branch offices connect via SD-WAN partner integration (automated provisioning). All-to-all routing via the vWAN hub eliminates manually peered spoke VNets. Azure Firewall deployed in each hub provides centralized security policy.

---

## 2.6 Azure Private Link & Private Endpoints

### What It Is
Azure Private Link enables private access to Azure PaaS services (Azure SQL, Storage, Key Vault, Cosmos DB, etc.) and your own services from within a VNet using Private Endpoints — a network interface with a private IP address in your subnet.

### Private Endpoint vs Service Endpoint

| Aspect | Service Endpoint | Private Endpoint |
|--------|----------------|----------------|
| **Network path** | Azure backbone (not internet) but uses public IP | Private IP in your VNet |
| **Accessibility** | Restricted to your VNet, but service still has public IP | Service has a private IP, truly private |
| **DNS** | No change | Requires private DNS zone configuration |
| **Cost** | Free | $0.01/hour per endpoint + $0.01/GB |
| **Cross-VNet access** | No (only the VNet where endpoint lives) | Yes (via VNet peering) |
| **On-premises access** | No | Yes (via ExpressRoute or VPN) |

### When to Use

```
Private Endpoint when:
├── PaaS services must be inaccessible from the internet (SQL, Key Vault, Storage)
├── On-premises applications need to access Azure PaaS privately (via ExpressRoute)
├── Cross-VNet PaaS access via peering
├── Regulatory/compliance requirement for complete network isolation
└── Replacing Service Endpoints for stronger isolation guarantees
```

---

## 2.7 Azure DNS & Azure Traffic Manager

### Azure DNS
A hosting service for DNS domains providing name resolution using Azure infrastructure. Supports public zones, private zones (resolution within VNets), alias records (pointing to Azure resources), and DNS-based health monitoring.

### Azure Traffic Manager
A DNS-based global traffic load balancer. Routes users to service endpoints based on routing method. Unlike Front Door (which uses anycast and proxies traffic), Traffic Manager is purely DNS-based — it returns the IP of the best endpoint, then the client connects directly.

### Traffic Manager Routing Methods

| Method | Logic | Use Case |
|--------|-------|---------|
| **Priority** | Route to primary; failover to secondary on failure | Active-passive DR |
| **Weighted** | Distribute traffic by weight (0–1000) | Canary, blue-green |
| **Performance** | Route to endpoint with lowest network latency | Global active-active |
| **Geographic** | Route based on user's DNS source location | Data residency |
| **Multivalue** | Return all healthy endpoints | Client-side load balancing |
| **Subnet** | Route based on client IP subnet | ISP routing, split-horizon |

### When to Use

```
Traffic Manager when:
├── DNS-based global load balancing (no reverse proxy overhead)
├── Non-HTTP workloads (TCP, SMTP, any protocol — Front Door is HTTP only)
├── Combining Azure + non-Azure endpoints in one routing policy
└── Active-passive DR with automatic DNS failover

Azure DNS when:
├── Hosting public or private DNS zones in Azure
├── DNS resolution for Private Endpoints (privatelink.* zones)
└── Alias records pointing to Azure resources (eliminates dangling DNS)
```

---

# 3. GCP Networking Services

---

## 3.1 Google Cloud VPC

### What It Is
GCP VPC is fundamentally different from AWS and Azure in one critical way: **GCP VPCs are global**. A single VPC spans all GCP regions worldwide. Subnets are regional (not zonal), and instances in different regions within the same VPC can communicate using internal IPs without VPN or peering — using Google's private global fiber network.

### GCP vs AWS vs Azure VPC Comparison

| Aspect | GCP VPC | AWS VPC | Azure VNet |
|--------|---------|---------|-----------|
| **Scope** | **Global** (one VPC, all regions) | Regional | Regional |
| **Subnet scope** | Regional | AZ-specific | Regional |
| **Cross-region routing** | Automatic within VPC | Requires TGW or peering | Requires global VNet peering |
| **Firewall rules** | Applied to VMs via tags/service accounts | Security Groups (per-ENI) | NSGs (per-subnet or NIC) |
| **Shared VPC** | Cross-project networking | Resource Access Manager | N/A (use peering) |

### Firewall Rules

GCP firewalls are VPC-scoped (not instance-scoped like AWS SGs) and use **network tags** or **service accounts** to target specific VMs:

```
Rule: Allow port 443 from 0.0.0.0/0 to VMs with tag "web-server"
Rule: Allow port 5432 from VMs with tag "app-server" to VMs with tag "db-server"
Rule: Deny all ingress (lowest priority default)
```

This tag-based model scales better than IP-based rules in dynamic environments where VM IPs change frequently.

### When to Use

```
GCP VPC when:
├── Multi-region applications — same VPC, no peering complexity
├── Global private backbone — GCP's Premium Tier network routes over Google fiber
├── Kubernetes workloads on GKE — native VPC-native cluster networking
└── Shared VPC — centralize network management across projects in an organization
```

---

## 3.2 Google Cloud Load Balancing

### What It Is
GCP Load Balancing is a global, fully distributed, software-defined service that operates at Google's edge. Unlike AWS ELB (which requires choosing between ALB/NLB/GWLB), GCP offers load balancers differentiated by scope (global vs regional) and protocol.

### Load Balancer Types

| Load Balancer | Layer | Scope | Protocol | Use Case |
|--------------|-------|-------|---------|---------|
| **Global External HTTPS** | 7 | Global | HTTP/HTTPS, HTTP/2, gRPC | Global web apps, CDN |
| **Regional External HTTPS** | 7 | Regional | HTTP/HTTPS | Regional web apps |
| **Global External TCP Proxy** | 4 | Global | TCP | Global TCP services |
| **Global External SSL Proxy** | 4 | Global | SSL/TLS | Global SSL with termination |
| **Regional External Network** | 4 | Regional | TCP/UDP | Regional TCP/UDP (preserves source IP) |
| **Internal HTTPS** | 7 | Regional | HTTP/HTTPS | Internal microservices, GKE |
| **Internal TCP/UDP** | 4 | Regional | TCP/UDP | Internal TCP/UDP (ILB) |
| **Internal TCP Proxy** | 4 | Regional | TCP | Internal managed proxy |

### Global External HTTPS Load Balancer — Technical Deep Dive

This is GCP's flagship LB. It terminates connections at Google's edge (anycast, same IP worldwide), so a user in Tokyo connecting to a US-based backend gets TCP termination in Tokyo, then uses Google's private network to reach the backend — dramatically reducing latency.

```
Client (Tokyo) → Google Edge PoP Tokyo → [Google Private Backbone]
              → Backend (us-central1) 

vs. Regional Load Balancer:
Client (Tokyo) → Internet → Backend (us-central1)
```

**URL Map routing:**
```
urlMap:
  - match: /api/v2/*   → backendService: api-v2-backend
  - match: /static/*   → backendBucket: static-assets-bucket (GCS)
  - default:           → backendService: main-app-backend
```

### When to Use

```
Global HTTPS LB: Global user base, CDN + LB in one, GCS backend support
Regional HTTPS LB: Single-region apps, compliance restricting global PoPs
Internal HTTPS LB: East-west microservice traffic within GKE/GCE
Regional Network LB: Need to preserve client source IP, UDP support
```

---

## 3.3 Google Cloud Interconnect

### What It Is
Cloud Interconnect provides high-bandwidth, low-latency connectivity between on-premises and GCP. Available in two variants:

| Type | Bandwidth | Setup | Use Case |
|------|-----------|-------|---------|
| **Dedicated Interconnect** | 10 Gbps or 100 Gbps per link | Direct physical connection at colocation | Large enterprises, consistent high throughput |
| **Partner Interconnect** | 50 Mbps – 50 Gbps | Via connectivity provider | Locations not near GCP colocation, flexible bandwidth |

### 99.99% SLA Architecture

```
For 99.99% SLA, GCP requires:
├── 2× Dedicated Interconnect circuits
├── In 2 different colocation facilities (metro areas)
├── 2× Cloud Routers (one per circuit)
└── ECMP BGP routing across both circuits
```

### Cloud Interconnect vs VPN

| Aspect | Cloud Interconnect | Cloud VPN |
|--------|------------------|----------|
| **Bandwidth** | 10–200 Gbps | Up to 3 Gbps per tunnel |
| **Latency** | Consistent, low | Variable (internet path) |
| **Cost** | Higher (port fee + per-GB) | Lower ($0.05–0.08/hour per tunnel) |
| **Use case** | Production, high-volume | Dev/test, backup, lower volume |
| **SLA** | 99.9% or 99.99% | 99.9% (HA VPN) |

---

## 3.4 Cloud CDN

### What It Is
Google Cloud CDN uses Google's globally distributed edge nodes to cache content served by GCP's Global External HTTPS Load Balancer. Because it integrates directly with the load balancer, no separate CDN distribution setup is required — you simply enable Cloud CDN on a backend service or backend bucket.

### Cache Modes

| Mode | Behavior |
|------|---------|
| **USE_ORIGIN_HEADERS** | Respect Cache-Control headers from origin |
| **CACHE_ALL_STATIC** | Cache all static content regardless of Cache-Control |
| **FORCE_CACHE_ALL** | Cache all responses (override origin headers) |

### Cache Key Configuration

```
Default cache key: Protocol + Host + URL path

Custom cache key examples:
- Include query parameters: ?version=2&color=blue cached separately
- Exclude User-Agent header: same content for all browsers
- Include custom header: X-Region for geo-personalized content
```

### When to Use

```
Cloud CDN when:
├── Serving static assets (images, JS, CSS) to global users
├── Reducing origin load and bandwidth costs
├── Video on demand (supports signed URLs for access control)
├── Dynamic content acceleration (cache at edge, shorter TTL)
└── Already using GCP Global HTTPS LB (CDN is just a toggle — enable on backend)
```

---

## 3.5 Cloud DNS

### What It Is
Google Cloud DNS is a high-performance, resilient, global DNS service hosted on Google's infrastructure. It provides 100% uptime SLA (Google operates anycast DNS globally). Supports public zones, private zones (resolution within VPCs), DNS peering between zones/projects, and response policies (DNS firewall — block resolution of malicious domains).

### Key Features

| Feature | Detail |
|---------|--------|
| **DNSSEC** | Cryptographic signing of DNS records |
| **Split-horizon DNS** | Same name, different response inside vs. outside VPC |
| **DNS Peering** | Delegate zone resolution to another project's Cloud DNS |
| **Response Policies** | Override DNS responses (block malicious domains, redirect) |
| **Inbound forwarding** | On-premises DNS servers can resolve GCP private zones |
| **Outbound forwarding** | GCP resolves on-premises zones via on-premises DNS servers |

### When to Use

```
Cloud DNS when:
├── Public DNS hosting for GCP-hosted domains
├── Private DNS zones for GKE service discovery and internal APIs
├── DNS Peering across projects in a Shared VPC organization
├── DNS Security: Response policies to block malware/phishing domains
└── Hybrid DNS: Bidirectional resolution between GCP and on-premises
```

---

## 3.6 Cloud NAT

### What It Is
Cloud NAT provides outbound internet access for VM instances without external IP addresses. Unlike traditional NAT, Cloud NAT is a fully distributed, software-defined service with no single point of failure — it runs on Google's Andromeda SDN fabric.

### NAT IP Allocation Modes

```
Automatic (default): GCP automatically allocates NAT IPs as needed
Manual:              You specify static IPs (for external firewall whitelisting)
```

### Cloud NAT vs AWS NAT Gateway

| Aspect | GCP Cloud NAT | AWS NAT Gateway |
|--------|-------------|----------------|
| **HA** | Distributed, no single point of failure | Per-AZ (deploy in each AZ for HA) |
| **Bandwidth** | Auto-scales | 45 Gbps (burst) per gateway |
| **Static IPs** | Optional | Elastic IP assigned |
| **Cost** | $0.044/hour + $0.045/GB | $0.045/hour + $0.045/GB |
| **Architecture** | Regional, no zonal concern | Must deploy per-AZ for HA |

### When to Use

```
Cloud NAT when:
├── Private VMs need outbound internet (package updates, API calls)
├── Security requirement: VMs must not have public IPs
├── Need predictable outbound IPs for external service whitelisting
└── High-availability NAT without per-zone management complexity
```

---

## 3.7 Google Cloud Armor

### What It Is
Google Cloud Armor is GCP's DDoS protection and Web Application Firewall service. It operates at Google's edge infrastructure, protecting global load balancers from DDoS attacks and application-layer threats. Cloud Armor's DDoS mitigation benefits from Google's massive edge capacity — the same infrastructure that absorbed the largest recorded DDoS attack (46 million requests per second, blocked in 2022).

### Security Policy Rule Types

| Rule Type | Example |
|-----------|---------|
| **IP/CIDR allow/deny** | Block traffic from specific IP ranges |
| **Geographic restrictions** | Block all traffic from specific countries |
| **Preconfigured WAF rules** | SQLi, XSS, LFI, RFI, RCE (ModSecurity CRS) |
| **Rate limiting** | Throttle IPs exceeding threshold per interval |
| **reCAPTCHA integration** | Challenge suspected bots with CAPTCHA |
| **Adaptive Protection** | ML-based DDoS detection and recommended rules |

### When to Use

```
Cloud Armor when:
├── Global HTTPS LB exposed to internet (Armor is the WAF layer)
├── Need DDoS protection at Google's edge scale
├── OWASP Top 10 protection for web applications
├── Geo-blocking for compliance or threat reduction
└── Bot management and rate limiting for API endpoints
```

---

## 3.8 VPC Network Peering & Shared VPC

### VPC Network Peering
Connects two GCP VPC networks (same or different projects/organizations) so resources can communicate using internal IPs. Non-transitive (same as AWS and Azure peering).

### Shared VPC
Allows an organization to connect resources from multiple GCP projects to a common VPC network in a Host Project. This enables centralized network management while allowing teams to manage their own compute resources independently.

```
Shared VPC Architecture:
Host Project (Network Team owns)
├── Shared VPC network and subnets
├── Firewall rules and routing
└── IAM: grants "Compute Network User" to service projects

Service Project A (Team A)
└── VMs deployed into Shared VPC subnets

Service Project B (Team B)
└── VMs deployed into Shared VPC subnets

All projects share the same VPC — no peering needed
```

### When to Use

```
Shared VPC when:
├── Organization needs centralized network control with team-level compute autonomy
├── Compliance: network team must own all networking, app teams deploy VMs
├── Cost efficiency: single set of VPN/Interconnect connections shared
└── GKE: cluster nodes in Service Project use Shared VPC subnet IP space

VPC Peering when:
├── Connecting two distinct networks (different teams, different IP ranges)
├── Connecting to a partner/vendor GCP project
└── Simpler than Shared VPC for fewer projects
```

---

---

# 4. Cross-Cloud Comparison

---

## 4.1 Core Networking Service Mapping

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| **Private Network** | VPC (regional) | VNet (regional) | VPC (**global**) |
| **Subnet Scope** | AZ-specific | Regional | Regional |
| **Transit Routing Hub** | Transit Gateway | Virtual WAN / Hub VNet | Cloud Router + VPC peering |
| **Dedicated Connectivity** | Direct Connect | ExpressRoute | Cloud Interconnect |
| **VPN** | AWS VPN (up to 1.25 Gbps) | Azure VPN Gateway (up to 10 Gbps) | Cloud VPN (up to 3 Gbps/tunnel) |
| **DNS** | Route 53 | Azure DNS | Cloud DNS |
| **CDN** | CloudFront (450+ PoPs) | Azure Front Door / CDN | Cloud CDN (Google's edge) |
| **Global L7 LB** | ALB + CloudFront | Azure Front Door | Global External HTTPS LB |
| **Regional L7 LB** | ALB | Application Gateway | Regional External HTTPS LB |
| **L4 LB** | NLB | Azure Load Balancer | Regional Network LB |
| **WAF** | AWS WAF | Azure WAF (App GW / AFD) | Cloud Armor |
| **DDoS Protection** | Shield Standard/Advanced | Azure DDoS Basic/Standard | Cloud Armor |
| **Private PaaS Access** | PrivateLink | Private Endpoint | Private Service Connect |
| **NAT** | NAT Gateway (per-AZ) | NAT Gateway | Cloud NAT (regional, distributed) |
| **Firewall (Network)** | Security Groups + NACL | NSG + Azure Firewall | VPC Firewall Rules |
| **Managed Firewall** | AWS Network Firewall | Azure Firewall | Cloud NGFW (via Palo Alto) |

## 4.2 Key Architectural Differences

### Network Scope
- **GCP wins for multi-region simplicity:** One global VPC means a VM in `us-central1` and a VM in `europe-west1` share the same VPC with no peering — they communicate over Google's private backbone using internal IPs.
- **AWS and Azure** require explicit cross-region connectivity (TGW inter-region peering, Global VNet peering).

### Load Balancing Architecture
- **GCP:** Anycast at the edge. The same IP address responds from 100+ edge PoPs. TCP terminated at the nearest PoP, then forwarded over Google's backbone. This is built into the Global HTTPS LB.
- **AWS CloudFront + ALB:** Two separate services. CloudFront is the CDN/edge; ALB is the regional LB behind it.
- **Azure Front Door:** Similar to GCP's approach — single service with edge termination.

### Firewall Model
- **AWS:** Security Groups (per-ENI, stateful), NACLs (per-subnet, stateless). Rules based on IP/CIDR.
- **Azure:** NSGs (per-subnet or per-NIC, stateful). Rules based on IP/CIDR. Application Security Groups for tag-based targeting.
- **GCP:** VPC Firewall Rules (per-VM via network tags or service accounts, stateful). Tag-based model is more dynamic-IP-friendly.

---

---

# 5. Architect Decision Scenarios

---

## Scenario 1: Multi-Region E-Commerce Platform

**Requirements:**
- 2 million daily active users in North America, Europe, APAC
- Sub-100ms page loads globally
- Zero downtime during deployments
- PCI DSS compliance — cardholder data must not traverse public internet

**Cloud: AWS**

```
Architecture Decision:

Global Routing:    Route 53 (Latency routing) → CloudFront (CDN + WAF)
Regional LBs:      ALB in us-east-1, eu-west-1, ap-southeast-1
Compute:           ECS Fargate behind ALB (per region)
Database:          Aurora Global (primary us-east-1, replicas EU + APAC)
Payment Service:   Isolated VPC, PrivateLink for payment processor API
Network:           Transit Gateway per region (connects Prod, PCI-scoped VPCs)
WAF:               AWS WAF on CloudFront (OWASP + rate limiting)
DDoS:              Shield Advanced (cost protection for peak sale periods)

PCI Compliance:
├── Payment VPC: separate CIDR, no peering to application VPC
├── PrivateLink: connect App VPC to Payment VPC without exposing IPs
├── VPC Flow Logs: all traffic logged, 12-month retention in S3 + Glacier
└── WAF: block all non-TLS 1.2+ traffic
```

**Why these choices:**
- Route 53 Latency routing ensures users hit the nearest region
- CloudFront + WAF at edge — DDoS absorbed before reaching origin
- PrivateLink for PCI scope — payment traffic never on public internet
- Aurora Global gives APAC read latency < 20ms from local replica

---

## Scenario 2: Financial Services Hybrid Cloud

**Requirements:**
- 10 Gbps sustained data transfer between on-premises trading systems and Azure
- < 2ms additional latency (trading algo execution)
- On-premises Active Directory must remain authoritative; Azure services use same identities
- All Azure resources must be unreachable from internet

**Cloud: Azure**

```
Architecture Decision:

Hybrid Connectivity: ExpressRoute Direct 100 Gbps (two circuits, two providers, 99.99% SLA)
On-prem to Azure:    ExpressRoute Private Peering → VNet Gateway (Ultra Performance SKU)
Identity:            Entra ID Connect — sync on-prem AD to Entra ID
                     Pass-through authentication (passwords validated on-prem)
Network:             Hub-and-spoke VNet topology
├── Hub VNet: Azure Firewall, ExpressRoute GW, Bastion
├── Spoke 1: Trading applications (no internet egress)
├── Spoke 2: Analytics (read-only replicas)
└── Spoke 3: Management/monitoring
PaaS Access:         All SQL, Key Vault, Storage via Private Endpoints only
Internet:            Completely blocked — UDR forces all traffic through Azure Firewall
                     Azure Firewall denies all outbound except approved FQDNs

Why ExpressRoute Direct:
├── ExpressRoute Circuit max: 10 Gbps
├── ExpressRoute Direct: 100 Gbps dedicated port into Microsoft's network
├── FastPath: bypass VNet Gateway for ultra-low latency (sub-1ms improvement)
└── MACsec encryption at the physical layer (layer 2 encryption)
```

---

## Scenario 3: Cloud-Native Startup — GCP

**Requirements:**
- Global microservices deployed on GKE across 3 regions
- No public IP on any VM or service (security requirement)
- Multi-tenant SaaS — customers must access their data only (isolation)
- Real-time analytics on event data (10 million events/day)

**Cloud: GCP**

```
Architecture Decision:

Network:         Single Global VPC — GKE clusters in us-central1, europe-west1, asia-east1
                 communicate over private network without peering
Kubernetes:      Shared VPC for centralized network control
                 VPC-native GKE clusters (pod IPs from VPC subnet range)
Ingress:         Global External HTTPS LB → GKE Ingress (one IP, global)
                 Cloud Armor WAF policy on HTTPS LB
No public IPs:   Private Google Access enabled on subnets
                 Cloud NAT for outbound-only internet access from nodes
Multi-tenancy:   GKE Namespace per tenant + Network Policies (Cilium)
                 GCP Projects per enterprise customer (Shared VPC)
Analytics:       Pub/Sub → Dataflow → BigQuery (fully serverless, no infra)
Service Access:  Private Service Connect for Cloud SQL, Memorystore

Why GCP global VPC is ideal here:
├── GKE in 3 regions shares one VPC — pods can reach cross-region services
│   using internal IPs without peering complexity
├── Global HTTPS LB routes users to nearest GKE cluster automatically
├── No TGW equivalent needed — routing is automatic within one global VPC
└── Shared VPC gives network team control while app teams own namespaces
```

---

## Scenario 4: Choosing the Right Load Balancer

**Decision Framework:**

```
Q1: Is traffic HTTP/HTTPS or TCP/UDP?
├── HTTP/HTTPS → Go to Q2
└── Pure TCP/UDP → NLB (AWS), Azure LB (Azure), Regional Network LB (GCP)

Q2: Is the user base global or regional?
├── Global → CloudFront + ALB (AWS), Azure Front Door (Azure), Global HTTPS LB (GCP)
└── Regional → ALB (AWS), Application Gateway (Azure), Regional HTTPS LB (GCP)

Q3: Is WAF/OWASP protection required?
├── Yes → AWS WAF on ALB/CloudFront, Azure WAF on AppGW/AFD, Cloud Armor on HTTPS LB
└── No → Standard LB without WAF

Q4: Static IP required? Source IP preservation needed?
├── Yes → NLB (AWS — static EIP per AZ), Azure LB, Regional Network LB (GCP)
└── No → ALB, Application Gateway (reverse proxy — client IP in X-Forwarded-For)

Q5: gRPC or HTTP/2 end-to-end?
├── AWS: ALB supports HTTP/2 to clients; gRPC requires ALB (listener)
├── Azure: Application Gateway + Front Door support gRPC
└── GCP: Global HTTPS LB natively supports gRPC end-to-end

Q6: UDP required?
├── AWS: NLB (only option supporting UDP)
├── Azure: Azure Load Balancer
└── GCP: Regional External Network LB
```

---

## Scenario 5: When to Use VPN vs Direct Connect/ExpressRoute/Interconnect

```
Use VPN when:
├── Bandwidth < 1 Gbps
├── Dev/test or non-production workloads
├── Budget constraint — VPN tunnels cost ~$0.05/hour vs thousands for dedicated
├── Temporary connectivity during migration
└── Backup path for dedicated circuit (VPN as fallback)

Use Dedicated Circuit when:
├── Sustained bandwidth > 1 Gbps required
├── Latency-sensitive workloads (VDI, trading, SAP, real-time sync)
├── Compliance requires no public internet traversal
├── Daily data transfer > 1 TB (dedicated circuits have lower per-GB cost)
└── Production workloads requiring consistent, predictable performance
```

---

---

# 6. Complex Interview Questions & Answers

---

## Category A: VPC & Networking Fundamentals

---

### Q1: Explain the difference between a Security Group and a NACL in AWS. When would you use both together, and what are the limitations of each?

**Answer:**

Security Groups (SGs) are **stateful**, instance-level (ENI-attached) firewalls that support **allow rules only**. "Stateful" means if you allow inbound traffic on port 80, the return traffic is automatically allowed regardless of outbound rules. SGs evaluate **all rules together** — there is no rule ordering or explicit deny.

NACLs (Network Access Control Lists) are **stateless**, subnet-level firewalls that support both **allow and deny rules**. "Stateless" means return traffic must be explicitly allowed (using ephemeral port ranges 1024–65535). NACL rules are evaluated **in numerical order** — the first matching rule wins.

**When to use both:**

```
Scenario: Block a compromised IP immediately without modifying SG rules on 500 instances.
Solution: Add DENY rule on NACL for the offending IP CIDR.
         NACLs apply to the entire subnet — instant effect on all instances.
         SGs would require updating 500 security group rules individually.

Layered security pattern:
├── NACL: Coarse-grained subnet-level deny list (known bad IPs, country blocks)
└── SG:   Fine-grained instance-level allow rules (specific ports, source SGs)
```

**Limitations:**
- SG: Cannot explicitly DENY — only allow. Cannot block a specific IP that's allowed by another rule.
- SG: Hard limit of 5 SGs per ENI, 60 inbound + 60 outbound rules per SG.
- NACL: Stateless — must allow ephemeral ports (1024–65535) for return traffic.
- NACL: Rules evaluated in order — rule 100 (allow) stops evaluation before rule 200 (deny for same traffic).
- Both: Cannot filter based on DNS name or FQDN (IP-only). Use AWS Network Firewall or a proxy for FQDN filtering.

---

### Q2: A company has 50 VPCs across 5 AWS accounts. They want all VPCs to communicate with each other AND route internet egress through a single centralized firewall. How would you architect this?

**Answer:**

This is a hub-and-spoke with centralized egress inspection pattern using Transit Gateway.

```
Architecture:

Hub Account — Shared Services
├── Inspection VPC (10.0.0.0/24)
│   ├── Gateway Load Balancer (GWLB)
│   ├── Network Appliances (Palo Alto, Fortinet, etc.)
│   └── GWLB Endpoint in each spoke VPC via PrivateLink
└── Transit Gateway (TGW)
    ├── Inspection VPC attachment
    ├── 50 Spoke VPC attachments (all accounts via TGW Resource Access Manager share)
    └── Route Tables:
        ├── Spoke RT: default route 0.0.0.0/0 → Inspection VPC attachment
        └── Inspection RT: specific VPC CIDRs → direct; 0.0.0.0/0 → NAT Gateway

Traffic flow — internet egress:
Spoke VPC VM → TGW (Spoke RT default → Inspection VPC)
→ GWLB → Firewall appliance (inspect/allow/deny)
→ GWLB → TGW (Inspection RT: 0.0.0.0/0 → NAT Gateway)
→ Internet

Traffic flow — east-west (VPC to VPC):
Spoke VPC A → TGW → Inspection VPC → GWLB → Firewall
→ Inspection RT: 10.1.0.0/16 → TGW → Spoke VPC B
```

**Key decisions:**
- Gateway LB is essential — it transparently inserts the firewall appliance without changing source/destination IPs
- TGW route tables enforce that all traffic, even east-west, passes through the inspection VPC
- Use TGW Resource Access Manager sharing to attach VPCs from all 5 accounts to one TGW
- Spoke VPCs have no IGW — all internet traffic must go through the hub

---

### Q3: Explain VPC Peering vs PrivateLink vs Transit Gateway. When is each appropriate?

**Answer:**

These three services solve different versions of the "private connectivity" problem:

**VPC Peering:**
- Direct, private, encrypted connection between exactly two VPCs
- Non-transitive (A↔B + B↔C ≠ A↔C)
- Both sides can initiate connections
- No bandwidth limitation, no additional hop latency
- **Best for:** Few VPCs (< 10), bidirectional communication, same or different accounts/regions

**PrivateLink:**
- One-way: consumer VPC accesses a service in provider VPC via Interface Endpoint (ENI with private IP)
- Provider VPC CIDR is never exposed to consumer
- Scalable: one Endpoint Service can serve thousands of consumer VPCs
- Transitive: consumer's on-premises (via VPN/DX) can reach the service through the consumer's VPC
- **Best for:** SaaS/service provider model, large-scale (100s of consumers), CIDR overlap OK, strict access control

**Transit Gateway:**
- Regional hub: connect any combination of VPCs, VPNs, DX connections
- Transitive routing IS possible (A→TGW→B, B→TGW→C, so A can reach C via TGW routing)
- Multiple route tables for network segmentation
- Per-attachment and per-GB pricing adds up
- **Best for:** Large networks (20+ VPCs), complex routing with segmentation, hybrid connectivity hub

```
Decision matrix:
├── 2-10 VPCs, bidirectional, no CIDR overlap → VPC Peering
├── Service offering to many consumers, CIDR overlap OK → PrivateLink
└── 20+ VPCs, complex routing, hybrid hub, segmentation → Transit Gateway
```

---

### Q4: How does BGP work in the context of AWS Direct Connect, and how would you influence path selection for active-active connections?

**Answer:**

BGP (Border Gateway Protocol) is the routing protocol used between on-premises routers and AWS over Direct Connect. Each Direct Connect connection runs eBGP sessions between the on-premises customer gateway and the AWS Direct Connect router.

**BGP attributes for path selection (in preference order AWS uses):**
1. **LOCAL_PREF** — Higher is preferred. Set on your on-premises router to prefer one DX over another.
2. **AS_PATH length** — Shorter is preferred. Prepend AS numbers to make a path appear longer/less preferred.
3. **MED (Multi-Exit Discriminator)** — Lower is preferred. Used when you have multiple circuits to the same AWS region.
4. **eBGP over iBGP** — eBGP preferred over iBGP (rarely relevant for this use case).

**Active-Active with traffic engineering:**

```
Scenario: Two DX connections (10 Gbps each) to us-east-1.
Goal: Normal traffic uses both circuits equally; on failure, all traffic moves to survivor.

Config on your BGP router:
Circuit A: Advertise prefixes with LOCAL_PREF 200, no AS_PATH prepend
Circuit B: Advertise prefixes with LOCAL_PREF 200, no AS_PATH prepend
→ ECMP: AWS will distribute traffic across both (equal LOCAL_PREF, equal AS_PATH)

For weighted split (70/30 traffic distribution):
Circuit A: LOCAL_PREF 200 (for traffic preferring circuit A)
Circuit B: LOCAL_PREF 100 + AS_PATH prepend (own AS prepended twice)
→ Circuit A gets majority; Circuit B gets remainder

Failover: if Circuit A fails, BGP withdraws routes via Circuit A,
all traffic automatically routes via Circuit B within BGP convergence time (~30s)
```

**AWS to on-premises path selection:**
AWS uses the same BGP attributes. To influence which DX circuit AWS uses for return traffic, advertise more specific prefixes (/25 vs /24) on the preferred circuit, or use AS_PATH prepending on the circuit you want to de-prefer.

---

## Category B: Load Balancing & DNS

---

### Q5: An ALB is returning 504 Gateway Timeout errors intermittently for 3% of requests. Walk me through your diagnostic approach.

**Answer:**

504 Gateway Timeout means the ALB made a request to the target (EC2/container) but the target didn't respond within the idle timeout (default 60 seconds).

**Systematic diagnostic process:**

```
Step 1: Establish the pattern
├── Check ALB access logs → which target IPs are involved?
├── Time-based pattern? Correlate with Auto Scaling events, deployments, high traffic
├── URL pattern? Specific API endpoints (slow DB queries, heavy processing)?
└── Target-specific? If same targets always → target health issue vs. all targets → LB config

Step 2: Check target health
├── ALB target group health checks → are 504s on "healthy" targets or "draining" targets?
├── EC2 metrics: CPU, memory, network connections → are targets overwhelmed?
├── Application logs on targets → are requests arriving but taking >60s?
└── Connection count per target → is one target getting too many connections?

Step 3: Check ALB configuration
├── Idle timeout (default 60s): is the app intentionally taking >60s? Increase timeout.
├── Connection draining period: deregistered targets in draining → may timeout
├── Slow start mode enabled? New targets getting too much traffic before warm-up?
└── Target group stickiness: sticky sessions routing all traffic to one overloaded target?

Step 4: Check network path
├── Security group on targets: allows traffic from ALB's SG?
├── Application listening on correct port? (app deployed on :8080, target group expects :80)
├── Response header: is target sending a response within 60s? Check with curl --max-time 65
└── Keep-alive: if target closes connection early, ALB may get 502 not 504

Root causes found in practice:
├── Database queries timing out (>60s) causing app thread to block
├── Downstream service call timeout cascading (app waits on slow third-party API)
├── Target not accepting new connections (connection backlog full)
├── Graceful shutdown too slow: target deregistered but still processing long requests
└── Memory pressure causing GC pauses >60s on JVM applications
```

---

### Q6: Explain how Route 53 handles a failover from a primary to secondary region. What are the failure modes?

**Answer:**

**Failover mechanism:**

```
Route 53 Failover routing works as follows:
1. Primary record: alias to ALB in us-east-1, associated with health check
2. Secondary record: alias to ALB in us-west-2, FAILOVER type "SECONDARY"
3. Health check: HTTP check to primary ALB /health endpoint, 30s interval, 3 failures threshold

Normal operation:
→ DNS query → Route 53 evaluates primary health check → healthy → returns primary ALB IP

Failover sequence:
T+0:    Primary ALB /health returns 503 (first failure)
T+30:   Second health check failure
T+60:   Third failure → Route 53 marks primary health check UNHEALTHY
T+60+:  DNS queries → Route 53 sees primary unhealthy → returns secondary ALB IP
T+60+:  New connections route to us-west-2 (secondary)

Recovery:
T+X:    Primary recovers, /health returns 200
T+X+30: Next health check succeeds (1 success threshold default)
T+X+30: Route 53 marks primary HEALTHY → returns primary IP again
```

**Failure modes and limitations:**

1. **DNS TTL lag:** Route 53 sets TTL (e.g., 60s). Clients caching the old IP continue hitting primary for up to TTL seconds after failover. For fastest failover: TTL = 60s (minimum for alias records), health check interval = 10s (fast), failure threshold = 1. Minimum RTO: ~30s.

2. **Health check target is the LB, not the app:** Health check passes if ALB responds 200, but what if the app behind the ALB is broken but returns 200? Use application-specific health checks (check `/health` which verifies DB connectivity).

3. **Calculated health checks for complex scenarios:** A single /health endpoint may not represent all app dependencies. Use a calculated health check (parent check passes only if all child checks pass: /health + DB reachability + cache availability).

4. **Route 53 health check locations:** Checks originate from AWS regions. If primary ALB has an NSG/SG blocking Route 53 health checker IPs, health checks will fail even if the app is healthy.

5. **Active-active failback:** When primary recovers, Route 53 immediately starts returning primary IP. Clients that had cached secondary IP (TTL not yet expired) still go to secondary. There's a brief period where some clients hit primary and some hit secondary.

---

### Q7: What is the difference between Azure Application Gateway and Azure Front Door? A company wants to host a web app globally with WAF protection. Which do you recommend?

**Answer:**

**Core architectural difference:**

- **Application Gateway** is a **regional** service. It runs within a specific Azure region (e.g., West Europe). It is a reverse proxy that terminates connections from clients, processes them, and forwards to backends in the same region.

- **Azure Front Door** is a **global** service running on Microsoft's edge PoPs worldwide (190+ locations). It terminates client connections at the nearest edge PoP, providing sub-second latency improvement for users far from backends.

**Decision: Front Door for global WAF + distribution**

```
Recommend Azure Front Door Premium when:
├── Users are globally distributed (different continents)
├── You want WAF + CDN + global LB in one service (avoids managing App GW + CDN separately)
├── Backends in multiple Azure regions (Front Door routes to nearest healthy region)
├── Private Link origins (Premium) — keep App Service endpoints private
└── Single global WAF policy across all regions

Recommend Application Gateway + WAF when:
├── Single-region deployment (all users and backends in one region)
├── AKS ingress (AGIC — Application Gateway Ingress Controller integrates natively)
├── SSL/TLS termination with certificates in Azure Key Vault (tighter integration)
├── URL path routing to multiple backend services in same region
└── Compliance restricts traffic to a single region (no edge PoPs outside region)

Common pattern: Use BOTH
├── Front Door Premium (global entry point, WAF, CDN, global routing)
├── Application Gateway (regional tier 2, OWASP rules, AKS ingress)
└── Front Door → App Gateway → AKS/App Service
    Front Door WAF: DDoS, geo-blocking, bot protection
    App Gateway WAF: OWASP rules, application-specific rules
```

---

## Category C: Advanced Architecture

---

### Q8: A GCP customer complains that their Global HTTPS Load Balancer is routing all traffic to a single region even though they have backends in 3 regions. What are the possible causes?

**Answer:**

GCP's Global HTTPS LB uses a combination of anycast routing and backend selection algorithms. Traffic should automatically route to the nearest healthy backend. If all traffic concentrates on one region:

```
Diagnosis checklist:

1. Health check status
   → Check all backend service health checks in all 3 regions
   → If backends in 2 regions show UNHEALTHY, LB routes to only healthy region
   → Fix: correct health check path, ensure firewall allows health check source IPs
          (GCP health checks originate from 35.191.0.0/16 and 130.211.0.0/22)

2. Backend capacity configuration
   → Check "Max utilization" or "Max RPS" per backend service
   → If us-central1 has maxUtilization=0.8, europe-west1=0.1, asia-east1=0.1
     → LB fills us-central1 to 80% before spilling to other regions
   → Fix: balance utilization targets across regions

3. Traffic balancing mode
   → UTILIZATION mode: LB fills backends proportionally to utilization threshold
   → RATE mode: LB distributes based on requests per second limit
   → CONNECTION mode: LB distributes based on connections
   → Misconfigured UTILIZATION target causes spill to only one region

4. Failover policy (failoverRatio)
   → failoverRatio: 0.1 means if primary backend is <10% healthy, failover to backup
   → If primary region has 15% healthy instances, no failover triggered — all goes there

5. Backend service scope
   → Verify all 3 regions' instance groups are added to the SAME backend service
   → A common mistake: 3 separate backend services, only one attached to URL map

6. URL map routing
   → Check if URL map routes /* to only one backend service
   → Other regions may be attached but not referenced by any URL map rule

7. Warm-up time
   → If backends were recently added, LB may not have started routing to them yet
   → Wait 5–10 minutes for backend registration to propagate to all PoPs
```

---

### Q9: Describe how you would design a zero-trust network architecture for a financial services company migrating to AWS. What network controls would you implement?

**Answer:**

Zero-trust means "never trust, always verify" — no implicit trust based on network location. Every request must be authenticated, authorized, and encrypted regardless of origin.

```
Zero-Trust Network Architecture on AWS:

Principle 1: No implicit trust based on VPC membership
├── Replace traditional "trusted internal network" assumption
├── Every service-to-service call requires mTLS authentication
└── Use AWS App Mesh or similar service mesh for mTLS between microservices

Principle 2: Micro-segmentation
├── Separate Security Groups per microservice (not per tier)
├── SG rules: allow only specific source SG → destination SG + port combinations
├── Example: payment-service-sg allows inbound 8443 from order-service-sg ONLY
└── No broad "all internal traffic allowed" rules

Principle 3: Private by default
├── All compute in private subnets (no public IPs on EC2/ECS)
├── All AWS API calls via VPC Endpoints (S3, SQS, SNS, DynamoDB, KMS, SM, etc.)
├── ALB in public subnet → TLS only → targets in private subnet
└── No SSH/RDP: use AWS Systems Manager Session Manager (no port 22 open)

Principle 4: Encryption everywhere
├── TLS 1.2+ for all inter-service communication (enforce via ALB security policy)
├── mTLS between microservices (App Mesh + ACM PCA for certificate management)
├── EBS volumes: encrypted with CMK
├── RDS: encrypted at rest + SSL in transit (require_secure_transport=ON)
└── S3: SSE-KMS, enforce via bucket policy (deny requests without aws:SecureTransport)

Principle 5: Network traffic inspection
├── AWS Network Firewall in dedicated inspection VPC
├── All east-west traffic routed through inspection VPC via TGW
├── Domain filtering: allow only known-good FQDNs (deny all others)
├── IDS signatures: block known malicious traffic patterns
└── TLS inspection for outbound (decrypt, inspect, re-encrypt)

Principle 6: Identity-based access
├── IMDSv2 only (instance metadata — prevent SSRF attacks from reaching IMDS)
├── EC2 Instance Profile → IAM Role (no long-lived access keys on instances)
├── Service accounts: least-privilege IAM roles per service
└── Attribute-based access control (ABAC) via IAM tags

Principle 7: Continuous monitoring
├── VPC Flow Logs → Security Lake → Athena queries
├── CloudTrail: all API calls logged, integrity validated
├── GuardDuty: ML-based threat detection (unusual API calls, network anomalies)
├── Security Hub: aggregated findings, compliance scoring
└── Config Rules: continuous compliance evaluation (SG not open to 0.0.0.0/0)

Principle 8: Just-in-time access
├── No persistent bastion hosts
├── SSM Session Manager: temporary, audited shell access
├── Requests via ServiceNow → auto-approved short-lived SG rule → access → revoke
└── All session activity recorded to CloudWatch Logs
```

---

### Q10: Compare GCP Premium Tier vs Standard Tier networking. When would an architect choose Standard Tier despite the lower quality of service?

**Answer:**

GCP offers two network service tiers that differ in how traffic is routed between users and GCP:

**Premium Tier:**
- Traffic enters Google's network at the PoP **closest to the user** (anycast)
- Travels across Google's private fiber backbone to the destination region
- Hot potato ingress, cold potato egress
- Single global anycast IP
- Better performance: lower latency, fewer hops, more reliable
- Required for: Global Load Balancing, Cloud CDN

**Standard Tier:**
- Traffic enters Google's network at the **Google region's PoP** (not closest to user)
- Travels across the public internet to the region first, then onto Google's network
- Regional IP addresses only
- Lower cost: ~25–30% cheaper for egress
- Regional load balancers only (no global)

```
Cost comparison example:
Egress to internet from us-central1:
├── Premium Tier: $0.08/GB (optimized path)
└── Standard Tier: $0.06/GB (standard internet routing)

When to choose Standard Tier:

1. Internal-only services
   → Service only accessed from within GCP (no external users)
   → Standard Tier has same performance within GCP
   → Example: internal microservices, data pipelines, batch processing

2. Cost-sensitive, non-latency-sensitive workloads
   → Large data exports where 25% egress savings outweigh performance
   → Batch jobs transferring TBs to partners (nightly, not real-time)

3. Regional workloads with regional users
   → Service serves only EU users → region is europe-west1
   → EU users' internet path to europe-west1 is already short
   → Premium Tier improvement is minimal; Standard Tier savings are real

4. Development/test environments
   → Performance SLA not required for non-production
   → Save costs on dev workloads

When Premium Tier is always required:
├── Global user base (multi-continent)
├── Real-time or latency-sensitive APIs (trading, gaming, VoIP)
├── Using Cloud CDN (requires Premium)
├── Global HTTPS Load Balancer (requires Premium)
└── SLA-backed production workloads with global reach
```

---

### Q11: A customer says their AWS Direct Connect is experiencing packet loss and high latency during business hours only. How would you diagnose and resolve this?

**Answer:**

Business-hours-only congestion is a specific pattern that points to specific root causes:

```
Phase 1: Isolate the layer

Layer 1 (Physical):
├── Check AWS Direct Connect console: port errors, CRC errors, light levels
├── Work with your connectivity provider — is their equipment showing errors?
└── Business hours rarely affect Layer 1 unless physical issue triggered by temperature/load

Layer 2 (BGP):
├── Check BGP session stability: flapping BGP causes brief packet loss
├── BGP timers: if holding time = 90s, session may be slow to detect loss
└── During high traffic: BGP control plane may be overwhelmed

Layer 3 (Traffic Engineering):
├── Most likely cause: bandwidth saturation on the DX link
├── Check AWS CloudWatch metric: ConnectionBpsEgress/Ingress
├── If link approaches capacity during business hours → congestion = packet loss/latency

Phase 2: Confirm bandwidth saturation

aws cloudwatch get-metric-statistics \
  --namespace AWS/DX \
  --metric-name ConnectionBpsIngress \
  --dimensions Name=ConnectionId,Value=dxcon-XXXXXX \
  --start-time 2025-01-20T08:00:00Z \
  --end-time 2025-01-20T18:00:00Z \
  --period 300 --statistics Average

If ingress/egress approaches link bandwidth (e.g., 9.5 Gbps on a 10 Gbps DX) → confirmed

Phase 3: Identify traffic sources

VPC Flow Logs query (Athena):
SELECT sourceaddress, destinationaddress, SUM(bytes) as total_bytes
FROM vpc_flow_logs
WHERE day = '2025-01-20' AND hour BETWEEN '09' AND '17'
GROUP BY sourceaddress, destinationaddress
ORDER BY total_bytes DESC
LIMIT 20;

→ Identifies which hosts/services are consuming bulk bandwidth during business hours

Phase 4: Resolution options

Short-term:
├── QoS/traffic shaping on on-premises router: prioritize latency-sensitive traffic
├── BGP communities: advertise some routes only via VPN to divert low-priority traffic
└── Route large-batch transfers to S3 via VPC Endpoint (internet path) instead of DX

Long-term:
├── Upgrade DX circuit bandwidth (10G → 100G Direct Connect Direct)
├── Add a second DX connection and use ECMP for 2× throughput
├── Separate DX connections by workload type (transactional vs. batch/backup)
└── For S3/DynamoDB heavy workloads: use Gateway Endpoints (no DX/internet needed)
```

---

### Q12: You need to deploy a service that multiple Azure customers will consume privately without VNET peering. Explain your approach and the DNS considerations.

**Answer:**

This is the Azure Private Link (Endpoint Service) pattern for SaaS providers.

**Architecture:**

```
Provider side (your Azure subscription):
├── Deploy service behind a Standard Load Balancer (internal, private IP)
├── Create Private Link Service
│   └── Points to your internal Standard LB
│   └── NAT IP configuration (SNAT to provider VNet range)
│   └── Visibility: list of approved Azure subscription IDs
│   └── Auto-approval or manual approval per request

Consumer side (customer's subscription):
├── Customer creates Private Endpoint in their VNet
│   └── Points to your Private Link Service resource ID or alias
│   └── Gets a private IP in the customer's VNet subnet
│   └── You approve the connection (or it auto-approves if subscription ID whitelisted)
└── Customer configures DNS to resolve service FQDN to the private endpoint IP
```

**DNS configuration (critical and often missed):**

```
Challenge: service.acme.com resolves publicly to your load balancer's public IP.
          Consumer needs service.acme.com to resolve to private endpoint IP (10.x.x.x).

Solution A: Azure Private DNS Zone (recommended)
Consumer creates Private DNS Zone: acme.com (or service.acme.com)
├── A record: service.acme.com → 10.0.5.4 (private endpoint IP)
├── Links DNS zone to consumer's VNet
└── Resolution: VMs in consumer VNet → 10.0.5.4 (private)
               Public DNS → your load balancer IP (public)

Solution B: Provider creates alias-based DNS
Provider gives customer a stable alias: abcde.privatelink.acme.com
Consumer creates CNAME: service.acme.com → abcde.privatelink.acme.com
Consumer creates Private DNS Zone for privatelink.acme.com
├── A record: abcde.privatelink.acme.com → 10.0.5.4
└── This lets provider update private endpoint IPs without consumer DNS changes

Common pitfall: On-premises DNS
├── On-premises DNS server forwards *.acme.com to on-premises resolvers
├── On-premises resolvers can't resolve Azure Private DNS zones
└── Fix: Configure DNS conditional forwarder for service.acme.com
         Forward to Azure DNS (168.63.129.16) only from on-premises resolvers
         Requires ExpressRoute or VPN for Azure DNS to be reachable

DNS resolution flow:
On-prem client → On-prem DNS → conditional forwarder → Azure DNS (168.63.129.16)
              → Private DNS Zone → returns 10.0.5.4 (private endpoint IP)
              → Client connects to 10.0.5.4 → Azure Private Link → your service
```

**SNAT and source IP consideration:**

Your service sees traffic from the Private Link Service's NAT IP (in your VNet) — not the customer's VNet IP. This affects logging, rate limiting, and geo-based logic. If customer IP visibility is needed, enable TCP proxy protocol on the internal load balancer and handle it in the application layer.

---

### Q13: Explain how GCP VPC-native (alias IP) GKE clusters differ from routes-based clusters, and why VPC-native is the recommended approach.

**Answer:**

**Routes-based clusters (legacy):**
```
├── Pod IPs allocated from a secondary CIDR range managed by GKE
├── Each node gets a /24 range for its pods (/24 = 256 pod IPs per node)
├── Routing: GCP VPC route table entry per node (podCIDR → node VM)
│   Example: route "10.48.0.0/24 via node-1", "10.48.1.0/24 via node-2"
├── Scale limit: GCP VPC has a default limit of 250 routes per VPC
│   100 nodes = 100 routes → approaching limit
│   500 nodes = 500 routes → requires quota increase + route propagation lag
└── External access: pods cannot be accessed from outside VPC (not in VPC's IP space)
```

**VPC-native (alias IP) clusters — recommended:**
```
├── Pods get IPs from a secondary IP range allocated to the subnet
│   (Not from routes — from the VPC subnet's alias IP ranges)
├── GCP understands pod IPs as part of the VPC natively
├── No routes created: pod IPs are routable within the VPC without routing entries
├── Scale: supports 1,500+ nodes without routing table explosion
└── External access: pods accessible from:
    ├── On-premises via VPN/Interconnect (pod IPs are in VPC range)
    ├── Peered VPCs (pod IPs propagate via peering)
    └── Google Cloud services (direct pod IP communication without SNAT)
```

**Additional advantages of VPC-native:**

1. **Private Google Access for pods:** Pods can access Google APIs (Cloud Storage, BigQuery, etc.) using internal IPs without NAT.

2. **Network policies via Dataplane v2 (Cilium):** VPC-native clusters can use eBPF-based network policies — faster, more scalable than iptables-based kube-proxy.

3. **Intranode visibility:** VPC-native enables Flow Logs per-pod, not just per-node. You can see pod-to-pod traffic within the same node.

4. **Container-native load balancing:** NEG (Network Endpoint Groups) target individual pod IPs directly from the GCP Load Balancer — bypassing kube-proxy and the node-level iptables rules. This reduces latency (one less hop) and improves health check accuracy (check the pod, not the node).

```
Routes-based LB path:    GLB → Node (kube-proxy) → Pod
VPC-native LB with NEG:  GLB → Pod (direct, container-native NEG)
                         Fewer hops, faster, accurate health checks
```

---

## Category D: Troubleshooting & Operations

---

### Q14: How would you troubleshoot a situation where on-premises systems cannot reach a new Azure SQL Database that was configured with a Private Endpoint?

**Answer:**

This is one of the most common Private Endpoint issues in hybrid environments. The root cause is almost always DNS resolution.

```
Symptom: On-premises application gets connection timeout to
         acme-sql.database.windows.net even though Private Endpoint exists.

Systematic troubleshooting:

Step 1: Verify DNS resolution
From on-premises server:
nslookup acme-sql.database.windows.net
→ If returns public IP (e.g., 20.x.x.x) → DNS not resolving to private endpoint
→ If returns private IP (10.0.3.5) → DNS OK, issue is routing/firewall

Step 2: If DNS returns public IP (most common cause)
The on-premises DNS resolver is hitting public DNS → returning Azure SQL's public IP.
Even if you created a Private DNS Zone in Azure, on-premises doesn't know about it.

Solution:
├── Create conditional forwarder on on-premises DNS server:
│   Zone: privatelink.database.windows.net
│   Forward to: 168.63.129.16 (Azure's internal DNS resolver)
│   (Requires ExpressRoute or VPN connectivity to Azure for 168.63.129.16 to be reachable)
└── Azure DNS Resolver (if using Azure DNS Private Resolver):
    Create inbound endpoint in Azure (gets a private IP, e.g., 10.0.10.4)
    On-premises conditional forwarder points to 10.0.10.4 instead of 168.63.129.16

Step 3: Verify Private DNS Zone configuration in Azure
├── Private DNS Zone: privatelink.database.windows.net must exist
├── A record: acme-sql → 10.0.3.5 (private endpoint NIC IP)
├── VNet link: DNS zone must be linked to the VNet containing the private endpoint
└── Verify: az network private-dns record-set a list --zone-name privatelink.database.windows.net

Step 4: Verify routing
From on-premises (after DNS resolves to 10.0.3.5):
traceroute 10.0.3.5
→ Should traverse ExpressRoute/VPN gateway into Azure VNet
→ If traceroute fails after entering Azure: check VNet route tables, NSG on PE subnet

Step 5: Verify NSG on Private Endpoint subnet
NSGs on the Private Endpoint subnet:
├── Inbound: allow port 1433 from on-premises IP range
├── Common mistake: NSG denies inbound because source is on-premises CIDR
│   but NSG rule only allows VNet CIDR
└── Fix: add inbound rule for on-premises CIDR → 10.0.3.0/24 → TCP 1433

Step 6: Verify Private Endpoint connection approval
az network private-endpoint show --name pe-sql --resource-group rg-prod
→ Check provisioningState = Succeeded
→ Check privateLinkServiceConnectionState.status = Approved
→ If Pending: approve the connection in Azure SQL Private Endpoint config
```

---

### Q15: Design a multi-cloud networking strategy for a company that wants to run on both AWS and Azure for resiliency, with the requirement that workloads in each cloud can communicate privately.

**Answer:**

This is a complex architecture involving cross-cloud private connectivity, which is not available natively — it requires explicit network tunnels or third-party SD-WAN.

```
Options for cross-cloud private connectivity:

Option 1: VPN over internet (simplest, lowest cost)
AWS: VPN Gateway (Customer Gateway in Azure)
Azure: VPN Gateway (connects to AWS VPN endpoint)
Tunnel: IPsec/IKEv2, encrypted, over internet
Bandwidth: up to 1.25 Gbps per tunnel pair
Latency: ~10–30ms (internet path, varies)
Use when: Dev/test, non-latency-sensitive, lower budget

Option 2: Direct private connectivity via colocation (recommended for prod)
Both AWS and Azure have presence in major colocation facilities
(Equinix, Digital Realty, CoreSite, etc.)

AWS: Direct Connect (from colo to AWS)
Azure: ExpressRoute (from colo to Azure)
Cross-connect: physical cable in colo between DX and ER circuits
→ Traffic: AWS VPC → Direct Connect → colo cross-connect → ExpressRoute → Azure VNet
→ No internet traversal, consistent latency (2–8ms depending on colo proximity)
→ This is the approach used by large financial institutions

Option 3: SD-WAN overlay (Aviatrix, Megaport, Alkira)
Third-party SD-WAN controller manages tunnels between AWS and Azure
Aviatrix: BGP-based encrypted tunnels, single control plane for multi-cloud routing
Megaport: Virtual cross-connect (VXC) between AWS and Azure via Megaport's fabric
→ Simpler operational model than managing raw VPN or dedicated circuits
→ Adds cost but reduces complexity

Recommended architecture for enterprise:

Network topology:
AWS:   Transit Gateway (hub for all AWS VPCs)
Azure: Virtual WAN Hub (hub for all Azure VNets)
Link:  AWS TGW ↔ [Direct Connect/ExpressRoute in colocation] ↔ Azure vWAN Hub

IP addressing (CRITICAL — no overlapping CIDRs across clouds):
AWS:   10.1.0.0/8 reserved for AWS VPCs (10.1.0.0/16, 10.2.0.0/16, etc.)
Azure: 10.2.0.0/8 reserved for Azure VNets (10.128.0.0/16, etc.)
On-prem: 192.168.0.0/16

DNS:
AWS Route 53 Private Hosted Zones: *.aws.internal
Azure Private DNS Zones: *.azure.internal
Cross-cloud DNS: Conditional forwarders in each cloud's DNS resolver
→ AWS resolver forwards azure.internal queries to Azure DNS Resolver IP
→ Azure resolver forwards aws.internal queries to Route 53 Resolver IP

Security:
├── All cross-cloud traffic encrypted (IPsec or MACsec)
├── Firewall in each cloud inspects cross-cloud traffic before reaching workloads
├── Separate security zone for cross-cloud traffic (different subnets/SGs/NSGs)
└── Zero-trust: just because traffic comes from the other cloud doesn't mean it's trusted
    → mTLS between cross-cloud services
    → Workload identity validation (not just IP-based trust)
```

---

---

## Quick Reference: Architect Cheat Sheet

### "Which networking service do I need?" Decision Tree

```
NEED: Private network foundation
└── Always: VPC (AWS) / VNet (Azure) / VPC (GCP) — mandatory for all workloads

NEED: Connect multiple VPCs/VNets
├── 2–10 VPCs, bidirectional → VPC Peering (all clouds)
├── 20+ VPCs or transitive routing → Transit Gateway (AWS) / Virtual WAN (Azure)
└── GCP: same VPC across regions, no peering needed

NEED: Connect on-premises to cloud
├── < 1 Gbps or dev/test → VPN (all clouds)
└── > 1 Gbps, latency-sensitive, compliance → Direct Connect / ExpressRoute / Cloud Interconnect

NEED: Load balance HTTP(S) traffic
├── Global users → CloudFront + ALB (AWS) / Front Door (Azure) / Global HTTPS LB (GCP)
└── Regional only → ALB (AWS) / Application Gateway (Azure) / Regional HTTPS LB (GCP)

NEED: WAF protection
├── AWS → AWS WAF (on ALB / CloudFront)
├── Azure → Azure WAF (on App Gateway / Front Door)
└── GCP → Cloud Armor (on Global/Regional HTTPS LB)

NEED: Private access to managed services (SQL, Storage)
├── AWS → PrivateLink / Interface VPC Endpoints / Gateway Endpoints (S3, DynamoDB)
├── Azure → Private Endpoints (all PaaS) / Service Endpoints (lighter alternative)
└── GCP → Private Service Connect / Private Google Access

NEED: DNS
├── Public DNS hosting → Route 53 / Azure DNS / Cloud DNS
├── DNS-based global routing → Route 53 Traffic Policy / Traffic Manager
└── Private DNS for hybrid → Route 53 Resolver / Azure DNS Private Resolver / Cloud DNS forwarding

NEED: DDoS protection
├── AWS → Shield Standard (free, automatic) / Shield Advanced ($3K/mo)
├── Azure → DDoS Basic (free) / DDoS Standard ($2.9K/mo)
└── GCP → Cloud Armor Standard / Managed Protection Plus ($3K/mo)
```

---

*Last Updated: 2025 | Sources: AWS Documentation, Azure Documentation, GCP Documentation*
*For the latest pricing, always verify at aws.amazon.com/pricing, azure.microsoft.com/pricing, cloud.google.com/pricing*
