# 🌐 AWS VPC — Complete Guide: Concepts, Dashboard, Production Workflow & Interview Q&A

> **Everything you need to know about Amazon Virtual Private Cloud — from zero to production-ready.**

---

## 📚 Table of Contents

1. [What is a VPC?](#1-what-is-a-vpc)
2. [Why VPC Exists — The Problem It Solves](#2-why-vpc-exists)
3. [VPC Core Components — Deep Dive](#3-vpc-core-components)
4. [VPC Dashboard — Every Section Explained](#4-vpc-dashboard)
5. [Step-by-Step: Creating a Production VPC](#5-creating-a-production-vpc)
6. [How Traffic Flows — The Full Journey](#6-how-traffic-flows)
7. [VPC Peering & Advanced Networking](#7-vpc-peering--advanced-networking)
8. [Security in VPC — Layers Explained](#8-security-in-vpc)
9. [Real Production Architecture Example](#9-real-production-architecture)
10. [What Happens If You Skip Each Step](#10-what-happens-if-you-skip-each-step)
11. [Interview Questions & Answers](#11-interview-questions--answers)

---

## 1. What is a VPC?

### Definition

A **Virtual Private Cloud (VPC)** is a logically isolated section of the AWS cloud where you launch your AWS resources inside a virtual network that **you define and control**.

Think of it like this:

> 🏢 AWS is a giant building. A VPC is **your private floor** in that building — you decide the rooms (subnets), who can enter (security groups), what doors exist (gateways), and how people move around (route tables).

### Key Facts

- Every AWS account gets a **default VPC** in every region automatically.
- You can create **up to 5 VPCs per region** (soft limit, can be increased).
- A VPC lives in **one AWS Region** but spans **all Availability Zones** in that region.
- Resources in a VPC are **isolated from all other VPCs** by default — even within the same account.

### Default VPC vs Custom VPC

| Feature | Default VPC | Custom VPC |
|---|---|---|
| Created by | AWS automatically | You |
| CIDR Block | `172.31.0.0/16` | Your choice |
| Subnets | One public subnet per AZ | You decide |
| Internet Gateway | Pre-attached | You attach it |
| Best for | Quick testing/demos | Production workloads |
| Risk if deleted | Resources become unreachable | Controlled deletion |

> ⚠️ **Never delete your default VPC without understanding the consequences.** Many AWS services assume it exists. Recreating it requires a support ticket or CLI command.

---

## 2. Why VPC Exists

### Before VPC: EC2-Classic

Before VPCs existed (pre-2013), AWS had "EC2-Classic" — all instances shared a flat network. Every EC2 instance could potentially communicate with every other instance in AWS. This was:

- Insecure (no isolation)
- Uncontrollable (no private IPs you own)
- Not enterprise-grade

### VPC Solves These Problems

| Problem | VPC Solution |
|---|---|
| No network isolation | Each VPC is completely isolated by default |
| Can't have private subnets | Subnets with no route to internet |
| No control over IP ranges | You define your own CIDR |
| No firewall at network level | Network ACLs + Security Groups |
| Can't control routing | Custom Route Tables |

---

## 3. VPC Core Components

### 3.1 CIDR Block (IP Address Range)

**What it is:** A range of IP addresses assigned to your VPC, written in CIDR notation.

**Example:** `10.0.0.0/16` — this gives you 65,536 IP addresses (`10.0.0.0` to `10.0.255.255`).

**How CIDR math works:**

```
10.0.0.0/16  →  65,536 IPs  (large VPC)
10.0.0.0/24  →  256 IPs     (typical subnet)
10.0.0.0/28  →  16 IPs      (tiny subnet)

Formula: 2^(32 - prefix) = total IPs
/16 = 2^(32-16) = 2^16 = 65,536
/24 = 2^(32-24) = 2^8  = 256
```

**Rules:**
- Allowed range: `/16` (largest) to `/28` (smallest)
- Must use private IP ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- **Cannot change CIDR after creation** (you can only ADD secondary CIDRs)

**What happens if you pick a bad CIDR?**
- If it overlaps with on-premises network → VPN/Direct Connect peering fails
- If it overlaps with another VPC → VPC Peering fails
- If it's too small → you run out of IPs and can't add more subnets

---

### 3.2 Subnets

**What it is:** A subdivision of your VPC's IP range, locked to **one specific Availability Zone**.

**Why they exist:** To organize resources and control traffic flow. Public-facing resources go in public subnets; backend resources go in private subnets.

**Types of Subnets:**

```
VPC: 10.0.0.0/16
│
├── Public Subnet A  (10.0.1.0/24) — AZ: us-east-1a
│   └── Internet-accessible (load balancers, bastion hosts)
│
├── Public Subnet B  (10.0.2.0/24) — AZ: us-east-1b
│   └── Internet-accessible (redundancy)
│
├── Private Subnet A (10.0.3.0/24) — AZ: us-east-1a
│   └── App servers, backend APIs
│
├── Private Subnet B (10.0.4.0/24) — AZ: us-east-1b
│   └── App servers (redundancy)
│
├── DB Subnet A      (10.0.5.0/24) — AZ: us-east-1a
│   └── RDS, ElastiCache (most restricted)
│
└── DB Subnet B      (10.0.6.0/24) — AZ: us-east-1b
    └── RDS replicas (most restricted)
```

**Reserved IPs in every subnet:**
AWS reserves the first 4 and last 1 IP in every subnet:

```
10.0.1.0   → Network address (reserved)
10.0.1.1   → VPC router (reserved)
10.0.1.2   → DNS server (reserved)
10.0.1.3   → Future use (reserved)
10.0.1.255 → Broadcast (reserved)
```

So a `/24` subnet gives you **251 usable IPs**, not 256.

**What makes a subnet "public"?**
A subnet is public when:
1. It has a route to an Internet Gateway in its Route Table, AND
2. Resources in it have public IPs (or Elastic IPs)

> 🔑 **Critical:** A subnet is NOT public just because you call it public. The Route Table decides.

---

### 3.3 Internet Gateway (IGW)

**What it is:** A horizontally scaled, redundant, highly available VPC component that allows communication between your VPC and the internet.

**What it does:**
- Performs **NAT (Network Address Translation)** for instances with public IPs
- Acts as a **target in route tables** for internet-bound traffic

**How it works:**
```
EC2 Instance (10.0.1.10) sends request to google.com
    ↓
Route Table: 0.0.0.0/0 → igw-xxxxxxxx
    ↓
Internet Gateway translates: 10.0.1.10 → 54.23.45.67 (public IP)
    ↓
Packet goes to internet
    ↓
Response comes back, IGW translates back to 10.0.1.10
```

**One VPC = One IGW maximum.**

**What happens without an IGW?**
- Instances cannot reach the internet, even with public IPs
- No inbound traffic from the internet
- Things that break: package installs (apt/yum), API calls to external services, webhooks receiving traffic

---

### 3.4 Route Tables

**What it is:** A set of rules (routes) that determine where network traffic is directed.

**How it works:**
Every subnet must be associated with a route table. When a packet leaves a subnet, the route table decides where it goes.

**Route matching uses "most specific wins":**
```
Destination       Target
10.0.0.0/16   →  local          ← All VPC-internal traffic stays inside VPC
10.0.1.0/24   →  local          ← More specific routes take priority
0.0.0.0/0     →  igw-xxxxxxxx   ← Everything else goes to internet
```

**Types of Route Tables:**

| Type | Used For |
|---|---|
| Main Route Table | Default for subnets not explicitly associated |
| Custom Route Table | Assigned to specific subnets |
| Gateway Route Table | Associated with an IGW or VPN Gateway |

**Production best practice:**
- NEVER use the main route table for production subnets
- Create dedicated route tables for: public subnets, private subnets, DB subnets
- This prevents accidental exposure of private resources

**Example Production Route Tables:**

```
PUBLIC Route Table:
  10.0.0.0/16 → local
  0.0.0.0/0   → igw-xxxxxxxx   ← Can reach internet

PRIVATE Route Table:
  10.0.0.0/16 → local
  0.0.0.0/0   → nat-xxxxxxxx   ← Goes through NAT (one-way out only)

DB Route Table:
  10.0.0.0/16 → local          ← Only local VPC traffic, no internet at all
```

---

### 3.5 NAT Gateway

**What it is:** Network Address Translation Gateway — allows instances in **private subnets** to initiate outbound connections to the internet, while preventing the internet from initiating inbound connections to those instances.

**Why you need it:**
Private subnet EC2 instances (your app servers) need to:
- Download OS updates
- Pull Docker images
- Call external APIs (payment gateways, etc.)

But you don't want them directly reachable from the internet.

**NAT Gateway vs NAT Instance:**

| Feature | NAT Gateway | NAT Instance |
|---|---|---|
| Managed by | AWS | You |
| Availability | Highly available | Single point of failure |
| Bandwidth | Up to 100 Gbps | Limited by instance size |
| Cost | Higher | Lower (but your time isn't free) |
| Maintenance | None | Patching, monitoring, etc. |
| Use in production | ✅ Yes | ❌ No (legacy) |

**How it works:**
```
Private EC2 (10.0.3.10) → wants to reach apt.ubuntu.com
    ↓
Route Table: 0.0.0.0/0 → nat-xxxxxxxx
    ↓
NAT Gateway (in public subnet, has Elastic IP: 52.10.20.30)
    ↓
IGW: translates Elastic IP → internet
    ↓
Response comes back to NAT Gateway's Elastic IP
    ↓
NAT Gateway forwards back to 10.0.3.10
```

**Cost alert:** NAT Gateways cost ~$0.045/hour + data charges. In a large org, this can be $100s/month. One common optimization: use a single NAT Gateway per AZ (not per subnet) and use VPC Endpoints for AWS services.

---

### 3.6 Security Groups

**What it is:** A **stateful** virtual firewall that controls inbound and outbound traffic at the **instance level**.

**Stateful means:**
If you allow inbound traffic on port 80, the response is automatically allowed out — you don't need to write a rule for it.

**Rules syntax:**
```
Type      Protocol  Port Range  Source/Destination
HTTP      TCP       80          0.0.0.0/0      ← Allow all inbound HTTP
HTTPS     TCP       443         0.0.0.0/0      ← Allow all inbound HTTPS
SSH       TCP       22          10.0.1.0/24    ← SSH from bastion subnet only
Custom    TCP       3306        sg-app-server  ← MySQL only from app servers
```

**Key rules:**
- Default: **deny all inbound, allow all outbound**
- You can only ADD allow rules (no deny rules)
- Can reference other Security Groups as source/destination (very powerful)
- Applied to: EC2 instances, RDS, Load Balancers, Lambda in VPC, etc.

**Security Group chaining (production pattern):**
```
Internet → [ALB SG: allow 443] → [App SG: allow from ALB SG only] → [DB SG: allow from App SG only]
```

This means your DB can only receive traffic from your app servers, your app servers only from the load balancer. Clean and auditable.

---

### 3.7 Network ACLs (NACLs)

**What it is:** A **stateless** firewall at the **subnet level**. Controls traffic entering/leaving a subnet.

**Stateless means:**
You must explicitly allow both inbound AND outbound traffic. If port 80 inbound is allowed, the response (ephemeral ports 1024-65535 outbound) must also be explicitly allowed.

**NACL vs Security Group:**

| Feature | Security Group | Network ACL |
|---|---|---|
| Level | Instance | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow AND Deny |
| Rule evaluation | All rules evaluated | Rules in order (lowest number wins) |
| Default | Deny all in, allow all out | Allow all traffic |
| Use case | Primary security control | Extra defense layer / block specific IPs |

**When to use NACLs:**
- Blocking a specific IP that's attacking you
- Compliance requirements for subnet-level controls
- Extra layer before reaching security groups

**NACL Rule numbering:**
```
Rule #  Type    Protocol  Port    Source          Action
100     HTTP    TCP       80      0.0.0.0/0       ALLOW
200     HTTPS   TCP       443     0.0.0.0/0       ALLOW
300     SSH     TCP       22      203.0.113.5/32  ALLOW    ← Only from your IP
*       All     All       All     0.0.0.0/0       DENY     ← Default deny
```

Lower number = evaluated first. `*` rule always last.

---

### 3.8 VPC Endpoints

**What it is:** Allows your VPC to connect to AWS services (S3, DynamoDB, SNS, etc.) **without going through the internet, a NAT gateway, or an IGW**.

**Why it matters:**
Without VPC Endpoints, when your EC2 in a private subnet calls S3:
```
EC2 → NAT Gateway → Internet Gateway → Public S3 endpoint
```
This costs NAT data transfer fees and routes traffic outside AWS network.

With a VPC Endpoint:
```
EC2 → VPC Endpoint → S3 (stays within AWS network, no NAT needed)
```

**Types:**

| Type | Services | How it works |
|---|---|---|
| Gateway Endpoint | S3, DynamoDB | Adds route to route table (free) |
| Interface Endpoint | Most AWS services | Creates ENI in your subnet (costs money) |

**Production rule:** Always create S3 and DynamoDB Gateway Endpoints — they're free and save NAT costs.

---

### 3.9 Elastic IP (EIP)

**What it is:** A static, public IPv4 address associated with your AWS account that you can attach to instances or NAT Gateways.

**Why needed:**
Regular public IPs assigned to EC2 instances change every time the instance stops/starts. Elastic IPs are yours until you release them.

**Cost trap:**
- EIP attached to a running instance: **free**
- EIP not attached (idle): **$0.005/hour** ≈ $3.60/month — AWS charges you to stop hoarding IPs!

---

### 3.10 ENI (Elastic Network Interface)

**What it is:** A virtual network interface card. Every EC2 instance has at least one (eth0), but you can attach multiple.

**Use cases:**
- Dual-homed instances (connected to two subnets)
- Move a network interface (and its private IP/Security Groups) between instances during failover
- Dedicated management interfaces

---

## 4. VPC Dashboard

When you open the AWS VPC Console (`VPC > Your VPCs`), here's every section explained:

### 4.1 VPC Dashboard Home

The dashboard shows a summary card for your region:

```
VPCs: 3
Subnets: 12
Route Tables: 6
Internet Gateways: 2
Elastic IPs: 4
Endpoints: 2
NAT Gateways: 2
VPN Connections: 0
```

**What to check here:**
- Count of NAT Gateways (cost driver)
- Unattached Elastic IPs (unnecessary cost)
- Number of VPCs (approaching limit?)

---

### 4.2 Your VPCs

**Columns explained:**

| Column | Meaning |
|---|---|
| VPC ID | Unique identifier (e.g., `vpc-0abc1234`) |
| IPv4 CIDR | IP range of this VPC |
| State | Available / Pending |
| DHCP Options Set | DNS settings for instances |
| Main Route Table | Default route table ID |
| Main Network ACL | Default NACL ID |
| Tenancy | Default (shared hardware) or Dedicated |
| DNS hostnames | Whether instances get DNS names |
| DNS resolution | Whether DNS works inside VPC |

**DNS settings — why they matter:**
- `DNS hostnames: enabled` → EC2 gets names like `ec2-54-23-45-67.compute.amazonaws.com`
- `DNS resolution: enabled` → Route 53 Resolver works inside the VPC
- Both must be enabled for services like EKS, RDS endpoint resolution, and PrivateLink to work.

---

### 4.3 Subnets

**Columns explained:**

| Column | Meaning |
|---|---|
| Subnet ID | `subnet-0abc1234` |
| VPC | Which VPC it belongs to |
| State | Available / Pending |
| IPv4 CIDR | IP range of this subnet |
| Available IPs | Usable IPs remaining |
| Availability Zone | Which AZ (e.g., us-east-1a) |
| Route Table | Associated route table |
| Network ACL | Associated NACL |
| Default subnet | Is this in the default VPC? |
| Auto-assign public IPv4 | Does launching an EC2 here auto-give it a public IP? |

**Auto-assign public IPv4:**
- Public subnets: Enable this
- Private subnets: DISABLE this (critical! Accidentally leaving it on exposes instances)

---

### 4.4 Route Tables

**What you see:**
- Route table ID and name
- Associated VPC
- Associated subnets
- Routes tab: destination + target pairs
- Subnet associations tab: which subnets use this table

**Edge Association** (advanced):
You can associate a route table with a gateway (IGW or VGW) to inspect/redirect traffic entering your VPC — used with AWS Network Firewall or security appliances.

---

### 4.5 Internet Gateways

**States:**
- `Detached`: Created but not attached to a VPC
- `Attached`: Working and attached to a VPC

**Common mistake:** Creating an IGW but forgetting to attach it, then wondering why traffic doesn't flow.

---

### 4.6 Egress-Only Internet Gateways

**What it is:** Like a regular IGW but only for **IPv6 outbound** traffic. Prevents internet from initiating connections to your IPv6 instances.

It's the IPv6 equivalent of a NAT Gateway.

---

### 4.7 NAT Gateways

**Columns:**
| Column | Meaning |
|---|---|
| NAT Gateway ID | `nat-0abc1234` |
| Connectivity type | Public (for internet) or Private (for peered VPCs) |
| State | Available / Pending / Deleting |
| Elastic IP | Public IP it uses to talk to internet |
| Subnet | Which public subnet it lives in |
| Network Interface | Underlying ENI |

**State transitions:**
```
Creating → Pending → Available → Deleting → Deleted
```

**Deleting a NAT Gateway:**
- The Elastic IP is NOT released automatically — you must manually release it
- Existing connections will time out

---

### 4.8 Network ACLs

**What you see:**
- NACL ID and name
- Which VPC
- Associated subnets
- Inbound rules
- Outbound rules
- Each rule has: rule number, type, protocol, port range, source/destination, allow/deny

---

### 4.9 Security Groups

**What you see:**
- Security Group ID and name
- VPC it belongs to
- Inbound rules
- Outbound rules
- Each rule: type, protocol, port range, source/destination, description

**Security Groups belong to a VPC.** You cannot use a security group from VPC-A on a resource in VPC-B.

---

### 4.10 VPC Endpoints

**Types shown:**
- Gateway endpoints (S3, DynamoDB) — free, added to route table
- Interface endpoints (everything else) — creates ENIs, has hourly cost

**Policy:** Every endpoint has a resource policy controlling what it can access (e.g., restrict to only your specific S3 buckets).

---

### 4.11 VPC Peering

**What it is:** A networking connection between two VPCs allowing private routing.

**Shown in dashboard:**
- Peering connection ID
- Requester VPC
- Accepter VPC
- Status: Pending Acceptance / Active / Rejected

**Important limitations:**
- Non-transitive: VPC A ↔ B and B ↔ C does NOT mean A can reach C
- No overlapping CIDR blocks allowed
- Must add routes manually in both VPCs' route tables

---

### 4.12 Transit Gateways

**What it is:** A network hub that allows thousands of VPCs to connect to each other and to on-premises networks.

```
Traditional (VPC Peering):        Transit Gateway:
A ↔ B                             All VPCs → TGW hub → Any VPC
A ↔ C                             Transitive routing works
B ↔ C                             One connection per VPC
(n*(n-1)/2 peerings needed)       (n connections total)
```

**Use when:** You have more than 3-4 VPCs that need to communicate.

---

### 4.13 Site-to-Site VPN

**What it is:** Encrypted tunnel between your on-premises network and AWS VPC.

**Components:**
- **Virtual Private Gateway (VGW):** AWS side of the VPN, attached to your VPC
- **Customer Gateway:** Represents your on-premises VPN device
- **VPN Connection:** The tunnel between them (always 2 tunnels for redundancy)

---

### 4.14 Direct Connect

**What it is:** A dedicated physical network connection from your data center to AWS. Not over the internet.

**Why:** Consistent latency, higher bandwidth (1 Gbps to 100 Gbps), more predictable costs for high-volume data transfer.

---

### 4.15 DHCP Options Sets

**What it is:** Configuration for how DNS and hostname resolution work inside your VPC.

**Key settings:**
- `domain-name-servers`: Use `AmazonProvidedDNS` (default) or your own DNS
- `domain-name`: Domain suffix for hostnames
- `ntp-servers`: Time servers

**Custom use case:** Point to your own DNS servers for hybrid cloud environments where you need to resolve on-premises hostnames.

---

### 4.16 Managed Prefix Lists

**What it is:** A reusable list of CIDR blocks that you can reference in Security Groups and Route Tables.

**Example:** Create a prefix list of your corporate office IP addresses, then reference it in all security groups. When the IP changes, update one prefix list — all security groups update automatically.

---

### 4.17 Flow Logs

**What it is:** Captures information about IP traffic going to/from your VPC, subnets, or ENIs.

**Log format (each record):**
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
2       123456789  eni-abc1234  10.0.1.10 10.0.3.20 54321 3306 6 10 1000 1617000000 1617000060 ACCEPT OK
```

**Destinations:** CloudWatch Logs, S3, Kinesis Data Firehose

**Use cases:**
- Security: Detect port scanning, unusual outbound connections
- Troubleshooting: Why is my traffic being rejected?
- Compliance: Audit network activity
- Cost: Identify which resources generate most traffic

**What flow logs DON'T capture:**
- Traffic to/from 169.254.169.254 (instance metadata)
- DNS traffic to Route 53 Resolver
- DHCP traffic
- Traffic between instances and AWS services via VPC endpoints (partially)

---

## 5. Creating a Production VPC

### Step-by-Step Process

#### Step 1: Plan Your IP Space

Before touching the console, plan on paper:

```
Production VPC: 10.10.0.0/16

AZ-1 (us-east-1a):
  Public:  10.10.1.0/24  (251 IPs)
  Private: 10.10.3.0/24  (251 IPs)
  DB:      10.10.5.0/24  (251 IPs)

AZ-2 (us-east-1b):
  Public:  10.10.2.0/24  (251 IPs)
  Private: 10.10.4.0/24  (251 IPs)
  DB:      10.10.6.0/24  (251 IPs)
```

**Why plan first?** CIDR blocks cannot be changed after creation.

#### Step 2: Create the VPC

```
VPC Console → Create VPC
  Name: prod-vpc
  IPv4 CIDR: 10.10.0.0/16
  Tenancy: Default
```

At this point you have an empty VPC with:
- A main route table (auto-created) — do NOT use for resources
- A default NACL (allow all)
- A default Security Group (deny all inbound)

#### Step 3: Create Subnets

Create all 6 subnets, assigning each to the correct AZ:

```
subnet-pub-1a:   10.10.1.0/24  us-east-1a
subnet-pub-1b:   10.10.2.0/24  us-east-1b
subnet-priv-1a:  10.10.3.0/24  us-east-1a
subnet-priv-1b:  10.10.4.0/24  us-east-1b
subnet-db-1a:    10.10.5.0/24  us-east-1a
subnet-db-1b:    10.10.6.0/24  us-east-1b
```

Enable "Auto-assign public IPv4" **only** on public subnets.

#### Step 4: Create and Attach Internet Gateway

```
VPC Console → Internet Gateways → Create
  Name: prod-igw

Actions → Attach to VPC → prod-vpc
```

Without this, nothing leaves the VPC to the internet.

#### Step 5: Create NAT Gateways

One per AZ (high availability):

```
NAT Gateway 1:
  Subnet: subnet-pub-1a    ← Must be in PUBLIC subnet!
  Elastic IP: Allocate new
  Name: prod-nat-1a

NAT Gateway 2:
  Subnet: subnet-pub-1b
  Elastic IP: Allocate new
  Name: prod-nat-1b
```

Wait for status "Available" before proceeding (takes 1-2 minutes).

#### Step 6: Create Route Tables

```
Route Table: prod-rt-public
  VPC: prod-vpc
  Routes:
    10.10.0.0/16 → local
    0.0.0.0/0    → prod-igw
  Associate: subnet-pub-1a, subnet-pub-1b

Route Table: prod-rt-private-1a
  VPC: prod-vpc
  Routes:
    10.10.0.0/16 → local
    0.0.0.0/0    → prod-nat-1a    ← AZ-specific NAT
  Associate: subnet-priv-1a

Route Table: prod-rt-private-1b
  VPC: prod-vpc
  Routes:
    10.10.0.0/16 → local
    0.0.0.0/0    → prod-nat-1b    ← AZ-specific NAT
  Associate: subnet-priv-1b

Route Table: prod-rt-db
  VPC: prod-vpc
  Routes:
    10.10.0.0/16 → local          ← No internet access at all
  Associate: subnet-db-1a, subnet-db-1b
```

#### Step 7: Create Security Groups

```
SG: prod-sg-alb (Application Load Balancer)
  Inbound:
    HTTPS 443 from 0.0.0.0/0
    HTTP  80  from 0.0.0.0/0  (redirect to HTTPS)

SG: prod-sg-app (App Servers)
  Inbound:
    Custom TCP 8080 from prod-sg-alb  ← Only from ALB
    SSH 22 from prod-sg-bastion       ← Only from bastion

SG: prod-sg-db (Database)
  Inbound:
    PostgreSQL 5432 from prod-sg-app  ← Only from app servers

SG: prod-sg-bastion (Bastion/Jump Host)
  Inbound:
    SSH 22 from YOUR_OFFICE_IP/32    ← Only your IP
```

#### Step 8: Create VPC Endpoints

```
S3 Gateway Endpoint:
  Service: com.amazonaws.us-east-1.s3
  VPC: prod-vpc
  Route Tables: prod-rt-private-1a, prod-rt-private-1b, prod-rt-db

DynamoDB Gateway Endpoint:
  Service: com.amazonaws.us-east-1.dynamodb
  Same route tables
```

#### Step 9: Enable Flow Logs

```
Select prod-vpc → Actions → Create Flow Log
  Filter: All (Accept and Reject)
  Destination: CloudWatch Logs
  Log group: /vpc/prod-vpc-flowlogs
  IAM Role: Create new
```

#### Step 10: Tag Everything

```
VPC:              Name=prod-vpc, Env=production, Team=platform
Subnets:          Name=subnet-pub-1a, Tier=public, AZ=us-east-1a
Route Tables:     Name=prod-rt-public, Tier=public
NAT Gateways:     Name=prod-nat-1a, AZ=us-east-1a
Security Groups:  Name=prod-sg-alb, Service=load-balancer
```

**Why tag?** Cost allocation, automation, security audits, and sanity.

---

## 6. How Traffic Flows

### Scenario 1: User → HTTPS Website

```
1. User types https://myapp.com
2. DNS resolves to ALB's public IP (54.23.45.67)
3. Packet arrives at IGW
4. IGW routes to ALB in public subnet (10.10.1.50)
5. ALB Security Group: Port 443 from 0.0.0.0/0 → ALLOW
6. ALB terminates SSL, forwards HTTP to app server (10.10.3.20:8080)
7. App Server Security Group: Port 8080 from ALB SG → ALLOW
8. App server processes request, queries DB (10.10.5.10:5432)
9. DB Security Group: Port 5432 from App SG → ALLOW
10. DB returns data → App → ALB → User
```

### Scenario 2: App Server → Pull Docker Image (Outbound)

```
1. App Server (10.10.3.20) runs: docker pull ubuntu
2. Route Table: 0.0.0.0/0 → nat-1a
3. NAT Gateway receives packet from 10.10.3.20
4. NAT Gateway translates src IP to its Elastic IP (54.90.10.20)
5. Route Table on public subnet: 0.0.0.0/0 → prod-igw
6. IGW routes to internet (registry-1.docker.io)
7. Response returns to Elastic IP
8. NAT Gateway translates back to 10.10.3.20
9. Docker image downloaded successfully
```

### Scenario 3: App Server → S3 (With Endpoint)

```
1. App Server (10.10.3.20) calls s3.amazonaws.com
2. Route Table finds S3 prefix list → vpce-xxxxxxxx (Gateway Endpoint)
3. Traffic stays INSIDE AWS backbone network
4. No NAT Gateway, no IGW involved
5. Faster, cheaper, more secure
```

---

## 7. VPC Peering & Advanced Networking

### VPC Peering

**Setup steps:**
1. Requester VPC: Create Peering Connection → enter Accepter VPC ID
2. Accepter VPC: Accept the connection
3. In both VPCs: Add route to the other's CIDR → target = peering connection ID
4. Update Security Groups to allow traffic from the other VPC's CIDR

**Peering limitations:**
- No transitive routing (A↔B, B↔C ≠ A↔C)
- CIDRs must not overlap
- Works across accounts and regions (inter-region peering)
- Maximum 125 peering connections per VPC

### Transit Gateway

**When to use:** More than 5 VPCs, or need hub-and-spoke with on-premises.

```
Architecture:
  VPC-Prod    ──┐
  VPC-Dev     ──┤
  VPC-Shared  ──┼── Transit Gateway ── Direct Connect/VPN ── On-premises
  VPC-Security──┤
  VPC-Data    ──┘
```

**Transit Gateway features:**
- Route tables per TGW (isolate traffic between VPCs)
- Multicast support
- Network Manager for monitoring

### AWS PrivateLink

**What it is:** Expose your service to other VPCs without VPC Peering. Traffic never leaves AWS network.

**Use case:** You run a service other teams/customers use. Don't want full VPC peering — just expose one service endpoint.

---

## 8. Security in VPC

### Defense in Depth Model

```
Layer 1: AWS Shield / WAF       → DDoS protection, web exploits
Layer 2: Internet Gateway       → Only entry point for internet traffic
Layer 3: Network ACLs           → Subnet-level stateless firewall
Layer 4: Security Groups        → Instance-level stateful firewall
Layer 5: OS Firewall (iptables) → Instance-level OS firewall
Layer 6: Application Security   → Auth, input validation, etc.
```

### Security Best Practices

**1. Principle of Least Privilege for Security Groups**
- Never use `0.0.0.0/0` as a source except for public-facing load balancers
- Reference other Security Groups instead of CIDRs where possible
- Delete unused security groups regularly

**2. Bastion Host Pattern**
```
Internet → Bastion Host (public subnet) → SSH → Private instances
```
Or use AWS Systems Manager Session Manager (no bastion needed, no open port 22).

**3. No Direct Internet for Sensitive Resources**
- RDS, ElastiCache: always in private/DB subnets
- Never assign public IP to database instances
- Use VPC Endpoints instead of NAT for AWS service access

**4. Audit Regularly**
- Use AWS Config Rules to detect misconfigured security groups
- Use AWS Security Hub for automated compliance checks
- VPC Flow Logs to CloudWatch with metric filters for unusual activity

---

## 9. Real Production Architecture

### Three-Tier Web Application

```
┌─────────────────────────────────────────────────────────────┐
│  REGION: us-east-1                                          │
│  VPC: 10.10.0.0/16                                         │
│                                                             │
│  ┌──────── AZ-1a ─────────┐  ┌──────── AZ-1b ─────────┐   │
│  │                         │  │                         │   │
│  │  [Public 10.10.1.0/24]  │  │  [Public 10.10.2.0/24]  │   │
│  │   ALB Node  NAT GW 1a   │  │   ALB Node  NAT GW 1b   │   │
│  │                         │  │                         │   │
│  │  [Priv  10.10.3.0/24]   │  │  [Priv  10.10.4.0/24]   │   │
│  │   App Server 1          │  │   App Server 2          │   │
│  │   App Server 3          │  │   App Server 4          │   │
│  │                         │  │                         │   │
│  │  [DB    10.10.5.0/24]   │  │  [DB    10.10.6.0/24]   │   │
│  │   RDS Primary           │  │   RDS Replica           │   │
│  │   ElastiCache Node 1    │  │   ElastiCache Node 2    │   │
│  │                         │  │                         │   │
│  └─────────────────────────┘  └─────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
          ↕ (IGW above public subnets)
      Internet (Users access via Route 53 → ALB)
```

### What happens at each layer

**Internet to ALB:**
Users connect to ALB DNS → Route 53 → ALB in public subnet → Security Group allows 443

**ALB to App:**
ALB terminates TLS → forwards to target group of app servers in private subnets → App SG allows 8080 from ALB SG only

**App to DB:**
App calls RDS endpoint (DNS) → Routes to RDS Primary → DB SG allows 5432 from App SG only

**App to S3:**
App SDK calls S3 → VPC Gateway Endpoint → traffic stays on AWS backbone

**App to External API:**
App calls Stripe API → Route table → NAT Gateway → IGW → Internet → Stripe

---

## 10. What Happens If You Skip Each Step

| Step Skipped | Consequence |
|---|---|
| No Internet Gateway | No internet access for public subnets. Load balancer unreachable. |
| No NAT Gateway | Private subnet instances can't download updates, call external APIs, pull images. |
| Wrong Route Table | Traffic goes to wrong destination or gets dropped silently. Very hard to debug. |
| Auto-assign public IP on private subnet | Your app servers get public IPs and are reachable directly from internet — SECURITY RISK. |
| Security Group too open (0.0.0.0/0 on DB port) | Database exposed to entire internet — catastrophic. |
| No Multi-AZ subnets | Single AZ failure takes down entire application. |
| No VPC Flow Logs | No visibility into network traffic. Impossible to investigate security incidents. |
| Overlapping CIDRs | Cannot peer VPCs or connect to on-premises. Very painful to fix. |
| NAT Gateway in private subnet | NAT Gateway can't reach internet — won't work at all. |
| No tags | Cannot track costs, automate, or audit resources. Operations nightmare. |
| DNS hostnames disabled | RDS endpoints, EKS, PrivateLink DNS resolution fails. |
| No VPC Endpoints for S3 | All S3 traffic goes via NAT Gateway → higher costs, slower, less secure. |

---

## 11. Interview Questions & Answers

### Fundamentals

---

**Q1: What is a VPC and why is it needed?**

**A:** A VPC (Virtual Private Cloud) is a logically isolated network environment within AWS where you can launch resources. It's needed because without it, all resources would be on a shared, flat network with no isolation. VPC gives you full control over IP addressing, routing, security rules, and connectivity to other networks (internet, on-premises, other VPCs). It enables building secure, multi-tier architectures that mirror traditional data center networking but with cloud scale and flexibility.

---

**Q2: What is the difference between a public and a private subnet?**

**A:** A subnet is public when its associated route table has a route for `0.0.0.0/0` pointing to an Internet Gateway, AND resources in it have public IP addresses. A subnet is private when it has no direct route to an IGW — typically traffic either goes through a NAT Gateway for outbound-only internet access, or has no internet access at all (like DB subnets). The key insight: the subnet itself doesn't have a "public/private" flag — it's the route table that determines this.

---

**Q3: What is the difference between a Security Group and a Network ACL?**

**A:**

| | Security Group | NACL |
|---|---|---|
| Applies to | Instance/ENI | Subnet |
| Statefulness | Stateful (response auto-allowed) | Stateless (must allow both directions) |
| Rule types | Allow only | Allow and Deny |
| Rule order | All rules evaluated | Lowest number wins |
| Default | Deny all inbound | Allow all |

Use Security Groups as your primary security tool. Use NACLs for additional subnet-level control, especially for blocking specific IPs that are attacking you.

---

**Q4: What is a NAT Gateway and when would you use it?**

**A:** A NAT (Network Address Translation) Gateway allows instances in private subnets to initiate outbound connections to the internet while preventing the internet from initiating inbound connections. Use it when you have private subnet instances that need to: download OS patches, pull container images, call external APIs, or access any internet resource. Don't use NAT Gateway if you can use VPC Endpoints instead (for AWS services like S3 — it's cheaper and more secure). For production, deploy one NAT Gateway per Availability Zone to avoid cross-AZ traffic costs and single points of failure.

---

**Q5: Can two VPCs with the same CIDR be peered?**

**A:** No. VPC Peering requires non-overlapping CIDR blocks. If VPC-A is `10.0.0.0/16` and VPC-B is also `10.0.0.0/16`, peering is impossible because the route tables can't distinguish which VPC a `10.0.x.x` packet should go to. This is why IP address planning is critical before creating VPCs — choosing overlapping CIDRs is very painful to fix later.

---

**Q6: What is the difference between VPC Peering and Transit Gateway?**

**A:** VPC Peering is a direct, point-to-point connection between two VPCs. It's non-transitive (A↔B and B↔C doesn't give A access to C), requires a separate peering connection for every pair, and has no central management. Transit Gateway is a hub that VPCs connect to — it provides transitive routing, works across accounts and regions, has centralized route management, and supports thousands of VPCs. Use peering for simple 2-3 VPC connectivity; use Transit Gateway for complex multi-VPC architectures.

---

**Q7: What are VPC Endpoints? What types exist?**

**A:** VPC Endpoints allow your VPC to privately connect to supported AWS services without requiring internet, NAT, or VPN. Two types:
- **Gateway Endpoints** (S3 and DynamoDB): Free, work by adding a route to your route table that points to the endpoint.
- **Interface Endpoints** (most other AWS services — SNS, SQS, KMS, etc.): Create an ENI in your subnet with a private IP, charged hourly + data. Both types keep traffic within AWS's private network, improving security and potentially reducing costs.

---

**Q8: What is the difference between Elastic IP and a regular public IP?**

**A:** A regular public IP is dynamically assigned to an EC2 instance when it starts and is released when it stops — it changes with every restart. An Elastic IP is a static public IPv4 address that you own until you explicitly release it. Elastic IPs can be re-mapped to different instances (useful for failover), and they're billed if unattached ($0.005/hour). Use Elastic IPs when you need a stable IP address for DNS records, firewall whitelisting, or NAT Gateways.

---

**Q9: How does VPC Flow Logs work and what are its limitations?**

**A:** VPC Flow Logs capture metadata about IP traffic to/from network interfaces. Each record includes source/destination IP and port, protocol, packet count, byte count, action (ACCEPT/REJECT), and timestamps. They can be sent to CloudWatch Logs, S3, or Kinesis Firehose.

Limitations: They don't capture actual packet content, traffic to/from 169.254.169.254 (instance metadata), DNS queries to Route 53 Resolver, DHCP traffic, or traffic between instances and VPC endpoints for some services. There's also a delay — they're not real-time.

---

**Q10: A developer says their EC2 in a private subnet can't reach the internet. How do you troubleshoot?**

**A:** Work through the layers:

1. **Route Table:** Check if the private subnet's route table has `0.0.0.0/0 → NAT Gateway`
2. **NAT Gateway:** Verify it exists, is in `Available` state, and is in a **public** subnet
3. **NAT Gateway's subnet:** Verify that public subnet's route table has `0.0.0.0/0 → IGW`
4. **Internet Gateway:** Verify IGW is attached to the VPC
5. **Security Group:** Check outbound rules — default SG allows all outbound, but if custom, verify outbound 443/80 is allowed
6. **NACL:** Check both the private subnet's NACL (outbound) and public subnet's NACL (inbound for return traffic, must allow ephemeral ports 1024-65535)
7. **Instance:** Verify it's actually in the private subnet, OS-level firewall (iptables) isn't blocking

Flow Logs can confirm at which layer traffic is being dropped (ACCEPT vs REJECT).

---

**Q11: What is CIDR and how do you calculate subnet sizes?**

**A:** CIDR (Classless Inter-Domain Routing) notation represents an IP range as `base_address/prefix_length`. The prefix length indicates how many bits are fixed (network bits); the remaining bits are for hosts.

Formula: `2^(32 - prefix) = total IPs`. AWS reserves 5 IPs per subnet, so usable = total - 5.

Examples:
- `/16` = 65,536 IPs (used for VPCs)
- `/24` = 256 IPs, 251 usable (common subnet size)
- `/28` = 16 IPs, 11 usable (minimum subnet size in AWS)

For a production VPC, use `/16` for the VPC and `/24` for subnets — gives room for growth.

---

**Q12: What happens to traffic when a Security Group has no outbound rules?**

**A:** By default, a new Security Group allows all outbound traffic (`0.0.0.0/0`). If you delete all outbound rules, ALL outbound traffic is blocked — the instance can't initiate connections to anywhere, including AWS services. This is a common misconfiguration. For security-hardened environments, explicitly allow only required outbound ports (e.g., 443 to specific CIDRs, 5432 to DB subnet). Remember Security Groups are stateful — if inbound traffic is allowed, the response automatically goes out regardless of outbound rules.

---

**Q13: What is the difference between "default" VPC and a custom VPC?**

**A:** The default VPC is created automatically by AWS in every region. It has a `172.31.0.0/16` CIDR, one public subnet per AZ (all with auto-assign public IP enabled), an attached IGW, and a main route table pointing to the IGW. It's designed for easy getting-started. Custom VPCs have CIDRs you choose, subnets you design, no IGW by default, and private-by-default subnets. Use the default VPC for testing and demos; always use a custom VPC for production workloads to control IP space, ensure proper subnet separation, and prevent accidental public exposure.

---

**Q14: Can you change the CIDR of a VPC after creation?**

**A:** You cannot modify the primary CIDR block assigned to a VPC. However, you can add secondary CIDR blocks (up to 5 total, each up to `/16`). This allows expanding an IP range without rebuilding. But adding secondary CIDRs requires updating route tables and can complicate peering. The best practice is to plan your IP space thoroughly before creating the VPC. If you truly need a different CIDR, you must create a new VPC, migrate resources, and delete the old one.

---

**Q15: Explain the concept of "stateful" vs "stateless" in networking as it applies to AWS.**

**A:** Stateful means the firewall tracks connection state. When Security Groups allow an inbound connection on port 443, they automatically allow the response packets out — you don't write an explicit outbound rule for it. The connection is "remembered."

Stateless means no memory of previous packets. NACLs treat every packet independently. If you allow inbound TCP 443, you must explicitly allow outbound TCP ephemeral ports (1024-65535) for the responses, otherwise responses are blocked. This makes NACLs harder to configure correctly but gives more granular control.

In practice: use Security Groups as your primary control (easier, stateful), and layer NACLs only when you need subnet-level rules or explicit deny capabilities.

---

**Q16: A company is migrating from on-premises to AWS. What VPC design considerations should they make?**

**A:** Key considerations:

1. **IP Planning:** Choose VPC CIDRs that don't overlap with on-premises networks (critical for VPN/Direct Connect). Decide on a CIDR allocation strategy for future VPCs.

2. **Connectivity:** Choose between Site-to-Site VPN (quick, over internet, encrypted) or Direct Connect (dedicated line, consistent latency, higher bandwidth).

3. **Multi-Account Strategy:** Production, development, and shared services VPCs in separate AWS accounts connected via Transit Gateway.

4. **DNS Resolution:** Use Route 53 Resolver inbound/outbound endpoints to resolve both AWS and on-premises hostnames from either side.

5. **Security:** NACLs that mirror on-premises firewall policies. Security Groups mapped to server roles. Flow Logs for audit trail.

6. **High Availability:** Subnets in at least 2 AZs. NAT Gateways per AZ. DB subnets with Multi-AZ RDS.

7. **Egress Control:** Centralized egress through a shared services VPC with an inspection appliance or AWS Network Firewall.

---

**Q17: What is AWS PrivateLink?**

**A:** AWS PrivateLink allows you to expose a service to other VPCs (or AWS accounts) without peering the entire VPCs. The service provider creates a VPC Endpoint Service backed by a Network Load Balancer. Consumers create Interface Endpoints that appear as ENIs in their own subnets. Traffic flows privately through AWS's network — never over the internet, and the consumer only gets access to the specific service endpoint, not the entire VPC. This is the architecture behind most AWS service endpoints (ECS, SSM, Secrets Manager, etc.) when accessed via Interface Endpoints. Use it when you want to expose a microservice to other teams/customers without granting full network access.

---

**Q18: What are the costs associated with VPC components?**

**A:** 

| Component | Cost |
|---|---|
| VPC, Subnets, Route Tables, NACLs, SGs | Free |
| Internet Gateway | Free (pay for data transfer) |
| NAT Gateway | ~$0.045/hour + $0.045/GB processed |
| VPC Endpoints (Gateway: S3/DynamoDB) | Free |
| VPC Endpoints (Interface) | ~$0.01/hour per AZ + $0.01/GB |
| Elastic IP (unattached) | $0.005/hour |
| VPC Peering (same region) | Free (pay for data transfer) |
| Transit Gateway | $0.05/hour per attachment + $0.02/GB |
| Direct Connect | $0.03–$0.30/hour per port + data |
| VPC Flow Logs | CloudWatch/S3 ingestion/storage costs |

NAT Gateway and Transit Gateway are the major cost drivers in most architectures. Use VPC Endpoints for S3/DynamoDB to avoid NAT costs.

---

### Scenario-Based Questions

---

**Q19: You have a web app in a private subnet. Users can't reach it. Walk through the diagnosis.**

**A:**
1. Is there a public-facing load balancer (ALB/NLB) in a public subnet? Traffic must enter through it.
2. Does the ALB's Security Group allow 443/80 from `0.0.0.0/0`?
3. Does the ALB's target group have healthy targets (the private EC2s)?
4. Is the EC2's Security Group allowing traffic from the ALB's Security Group on the app port?
5. Is the EC2's public subnet route table pointing to the IGW? (No — it's private, so it should NOT have IGW — but the ALB should be in public subnets)
6. Is the EC2 actually running and the app process listening on the expected port?
7. Check NACL rules for both public (ALB) and private (EC2) subnets.
Use VPC Flow Logs to see ACCEPT/REJECT at each step.

---

**Q20: How would you set up a VPC for a multi-tier application with strict compliance requirements?**

**A:**
1. **Network Segmentation:** Three tiers — public (load balancers), private (app), restricted (DB) — each in dedicated subnets across 2+ AZs.
2. **No internet access for data layer:** DB subnets have route tables with only local routes.
3. **Private connectivity to AWS services:** VPC Endpoints for S3, DynamoDB, SSM, Secrets Manager — eliminating NAT for AWS API calls.
4. **Encrypted connectivity:** All inter-service traffic uses TLS. Security Groups reference other SGs, not CIDRs.
5. **Centralized logging:** VPC Flow Logs to S3 with CloudTrail. GuardDuty enabled for threat detection.
6. **No public IPs on app/DB instances:** All inbound traffic through ALB only.
7. **AWS Config Rules:** Detect SG rules with unrestricted access, public RDS instances, unencrypted EBS volumes.
8. **Bastion-less access:** Use AWS Systems Manager Session Manager — no open port 22 anywhere.

---

*End of AWS VPC Complete Guide*

---

> 📌 **Quick Reference Card**
>
> | Need to... | Use... |
> |---|---|
> | Connect VPC to internet | Internet Gateway |
> | Private instances reach internet | NAT Gateway |
> | Two VPCs communicate | VPC Peering |
> | Many VPCs communicate | Transit Gateway |
> | Block specific IP at subnet level | Network ACL (deny rule) |
> | Control traffic to/from instance | Security Group |
> | Connect to AWS services privately | VPC Endpoints |
> | Connect to on-premises (encrypted, internet) | Site-to-Site VPN |
> | Connect to on-premises (dedicated line) | Direct Connect |
> | Debug network traffic | VPC Flow Logs |
> | Expose service to other VPCs without full peering | AWS PrivateLink |
