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
