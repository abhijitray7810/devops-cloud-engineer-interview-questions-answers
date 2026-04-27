# AWS EC2 — The Complete Guide
### Everything You Need to Know: From Console to Production
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html
---
  
## Table of Contents

1. [What is EC2?](#what-is-ec2)
2. [EC2 Console Overview](#ec2-console-overview)  
3. [Section 1 — Instances](#section-1--instances) 
   - [1.1 Instances](#11-instances)
   - [1.2 Instance Types](#12-instance-types)
   - [1.3 Launch Templates](#13-launch-templates)
   - [1.4 Spot Requests](#14-spot-requests)
   - [1.5 Savings Plans](#15-savings-plans)
   - [1.6 Reserved Instances](#16-reserved-instances) 
   - [1.7 Dedicated Hosts](#17-dedicated-hosts)
   - [1.8 Capacity Reservations](#18-capacity-reservations)
   - [1.9 Capacity Manager](#19-capacity-manager)
4. [Section 2 — Images (AMIs)](#section-2--images-amis)
   - [2.1 AMIs](#21-amis)
   - [2.2 AMI Catalog](#22-ami-catalog)
5. [Section 3 — Elastic Block Store (EBS)](#section-3--elastic-block-store-ebs)
   - [3.1 Volumes](#31-volumes)
   - [3.2 Snapshots](#32-snapshots)
   - [3.3 Lifecycle Manager](#33-lifecycle-manager)
6. [Section 4 — Network & Security](#section-4--network--security)
   - [4.1 Security Groups](#41-security-groups)
   - [4.2 Elastic IPs](#42-elastic-ips)
   - [4.3 Placement Groups](#43-placement-groups)
   - [4.4 Key Pairs](#44-key-pairs)
   - [4.5 Network Interfaces](#45-network-interfaces)
7. [Section 5 — Load Balancing](#section-5--load-balancing)
   - [5.1 Load Balancers](#51-load-balancers)
   - [5.2 Target Groups](#52-target-groups)
   - [5.3 Trust Stores (ELB)](#53-trust-stores-elb)
8. [Section 6 — Auto Scaling](#section-6--auto-scaling)
   - [6.1 Auto Scaling Groups](#61-auto-scaling-groups)
9. [Launching an Instance — Step by Step](#launching-an-instance--step-by-step)
   - [Step 1.1 — Name and Tags](#step-11--name-and-tags)
   - [Step 1.2 — Amazon Machine Image (AMI)](#step-12--amazon-machine-image-ami)
   - [Step 1.3 — Instance Type](#step-13--instance-type)
   - [Step 1.4 — Key Pair (Login)](#step-14--key-pair-login)
   - [Step 1.5 — Network Settings](#step-15--network-settings)
   - [Step 1.6 — Configure Storage](#step-16--configure-storage)
   - [Step 1.7 — Advanced Details](#step-17--advanced-details)
   - [Step 1.8 — Summary](#step-18--summary)
   - [Step 1.9 — Launch Instance](#step-19--launch-instance)
10. [What Happens After Launch — Production Flow](#what-happens-after-launch--production-flow)
11. [Real Production Architecture Example](#real-production-architecture-example)
12. [Common Mistakes and What Breaks If You Skip Something](#common-mistakes-and-what-breaks-if-you-skip-something)

---

## What is EC2?

**Amazon Elastic Compute Cloud (EC2)** is AWS's core virtual machine service. It lets you rent a computer in AWS's data center and run any workload on it — a web server, database, API backend, machine learning job, or anything else.

The word **"Elastic"** means you can scale up or down at any time. You don't buy hardware. You pay for what you use, by the hour or by the second.

Think of it this way:
- Your laptop is a physical computer with a fixed CPU, RAM, and disk.
- An EC2 instance is a virtual computer running inside a physical server at AWS.
- You can create it in 60 seconds, use it for an hour, and destroy it.
- Or run it 24/7 for years as your production server.

**EC2 sits at the center of almost every AWS architecture.** Even if you use serverless services like Lambda, at some level they run on EC2 infrastructure underneath.

---

## EC2 Console Overview

When you open the EC2 console, the left sidebar is organized into six major sections. Each section is a different domain of EC2 functionality. Here is what each one manages:

| Section | What It Controls |
|---|---|
| Instances | The virtual machines themselves |
| Images (AMIs) | The "blueprints" used to create instances |
| Elastic Block Store | The hard drives (storage) attached to instances |
| Network & Security | Firewalls, IP addresses, SSH keys, network cards |
| Load Balancing | Traffic distribution across multiple instances |
| Auto Scaling | Automatic creation/deletion of instances based on demand |

---

## Section 1 — Instances

### 1.1 Instances

This is the main list of all your running (or stopped) virtual machines. Each row in this table is one instance — one computer running in AWS.

**Each instance has:**
- An **Instance ID** (e.g., `i-0abc12345def67890`) — the unique identifier
- An **Instance State** — running, stopped, terminated, pending, stopping
- A **Public IP** — how the internet reaches it (changes every restart unless you use Elastic IP)
- A **Private IP** — how other AWS services reach it internally (never changes)
- An **Instance Type** — how powerful it is (CPU, RAM)
- An **Availability Zone** — which physical data center it lives in
- A **Key Pair Name** — what SSH key was used to create it

**What happens here in production:**
- You monitor whether instances are healthy or failing
- You stop/start instances (stop = machine off but data preserved; start = machine back on)
- You terminate instances (permanent deletion — data gone unless on separate EBS volume)
- You connect to instances via SSH (Linux) or RDP (Windows)
- You reboot instances when needed
- You view system logs if an instance won't boot

**What happens if an instance is "stopped" vs "terminated":**
- **Stopped:** The virtual machine is turned off. EBS data persists. You are NOT charged for compute time, but you ARE still charged for EBS storage. You can start it again.
- **Terminated:** The instance and its root EBS volume are destroyed permanently (unless you set the "Delete on termination" flag to false). You cannot recover a terminated instance.

**What "running" actually means:**
When an instance is "running," AWS has allocated a physical CPU core(s) and RAM on one of their hypervisor servers, loaded your AMI, and booted an operating system. Your OS is running right now, waiting for work.

---

### 1.2 Instance Types

Instance types define the hardware profile of your virtual machine — how many CPUs it has, how much RAM, what network speed, and what storage capabilities.

**The naming convention:**

Every instance type name follows a pattern: **[Family][Generation].[Size]**

Example: `t3.micro`
- `t` = family (T = burstable general purpose)
- `3` = generation 3 (higher = newer hardware)
- `micro` = size (nano < micro < small < medium < large < xlarge < 2xlarge < 4xlarge...)

**Instance families and when to use them:**

| Family | Letter | Best For | Example |
|---|---|---|---|
| General Purpose | t, m | Web servers, small apps, dev environments | t3.micro, m6i.large |
| Compute Optimized | c | CPU-heavy workloads, batch processing, gaming | c6i.xlarge |
| Memory Optimized | r, x | Databases, in-memory caches, big data | r6i.2xlarge |
| Storage Optimized | i, d | High IOPS databases, data warehousing | i3.xlarge |
| Accelerated Computing | p, g, inf | Machine learning, GPU rendering, AI inference | p3.2xlarge |
| HPC | hpc | Scientific computing, simulations | hpc6a.48xlarge |

**The "t" family — burstable instances:**

The `t2.micro` and `t3.micro` (free tier eligible) are special. They use a **CPU credit system:**
- When your CPU usage is low, you earn credits over time.
- When you need a burst of CPU (like during startup), you spend those credits.
- If you run out of credits and you are in "Standard" mode, your CPU is throttled to baseline.
- If you switch to "Unlimited" mode, you can burst freely but pay extra.

This is why a `t2.micro` can feel fast for occasional tasks but slow if you run CPU-intensive work for too long — it runs out of credits.

**Why choosing the right instance type matters:**
- Too small → your app is slow, crashes under load, or runs out of memory
- Too large → you overpay massively for idle capacity
- Wrong family → a memory-heavy database on a compute-optimized instance wastes money

**In production:** Teams benchmark their workload and right-size instances. A common pattern is to start with `m5.large`, monitor CPU and RAM with CloudWatch, then resize based on actual usage.

---

### 1.3 Launch Templates

A **Launch Template** is a saved configuration for launching EC2 instances. Instead of filling out every option (AMI, instance type, key pair, security group, storage, etc.) every time, you save it once as a template.

**Why this matters:**

Without launch templates:
- You manually configure every setting each time you launch an instance
- You might forget a security group, choose the wrong AMI, or set wrong storage
- Auto Scaling Groups cannot function without a template or launch configuration

With launch templates:
- One-click (or one API call) launches a perfectly configured instance
- Auto Scaling Groups use them to create new instances automatically
- You can version templates — `v1`, `v2`, `v3` — and roll back if needed
- CI/CD pipelines can programmatically launch instances with zero human input

**A launch template stores:**
- AMI ID
- Instance type
- Key pair
- Security groups
- Network settings (VPC, subnet)
- Storage volumes
- IAM instance profile
- User data (startup scripts)
- Tags
- Advanced options

**Versioning in production:**

Say your team uses `v3` of a launch template for all production servers. You update the AMI to patch a security vulnerability → create `v4`. You test `v4` on a staging server. It works. You update your Auto Scaling Group to use `v4`. New instances that Auto Scaling spins up will use the patched AMI. Old instances can be gradually replaced (instance refresh). You never touched a running server.

---

### 1.4 Spot Requests

**Spot Instances** are spare EC2 capacity that AWS sells at a massive discount — typically 70–90% cheaper than On-Demand pricing.

**How they work:**

AWS has physical servers sitting idle. Rather than waste that capacity, they auction it off cheaply. You bid up to a maximum price (or just accept the current Spot price). As long as the Spot price stays below your max, your instance runs. If AWS needs that capacity back (someone is paying full price for it), they give you a **2-minute warning**, then terminate your instance.

**The trade-off:**
- Huge cost savings
- Your instance can be terminated at any time by AWS with only 2 minutes notice

**When to use Spot:**
- Batch processing jobs (video encoding, data analysis, ML training)
- Stateless workloads where losing an instance doesn't lose data
- Any workload that can be checkpointed and resumed
- Development and test environments

**When NOT to use Spot:**
- Production web servers handling live traffic
- Databases with non-replicated data
- Any workload where a sudden 2-minute shutdown causes data loss or outage

**Spot Fleet:**
You can request a fleet of Spot Instances across multiple instance types and Availability Zones. If one type is unavailable or too expensive, the fleet automatically uses another. This dramatically reduces interruption risk.

**In production:** Large ML training jobs at companies like Netflix or Airbnb run on Spot. If an instance is interrupted, the training job checkpoints to S3 and resumes on a new Spot instance. They cut ML training costs by 70%.

---

### 1.5 Savings Plans

Savings Plans are a flexible pricing model where you commit to spending a certain dollar amount per hour for 1 or 3 years, and in return get up to 72% discount.

**Two types:**

**Compute Savings Plans** — Most flexible. Your committed spend applies to any EC2 instance regardless of family, size, region, or OS. Also applies to Fargate and Lambda.

**EC2 Instance Savings Plans** — Less flexible but bigger discount. Committed to a specific instance family in a specific region (e.g., all `m5` instances in `us-east-1`). Applies regardless of size or OS within that family.

**How they work in practice:**

Say you commit to $0.10/hr. Every hour, AWS applies the Savings Plan discount to your most expensive EC2 usage first, up to $0.10/hr worth. Usage beyond that is billed at On-Demand rates.

You don't get a specific instance. You just get cheaper rates on whatever you run.

**Savings Plans vs Reserved Instances:**
- Reserved Instances: You reserve a specific instance type. More restrictive, similar savings.
- Savings Plans: You commit to spend. More flexible. Generally preferred for modern architectures.

**What if you don't use a Savings Plan?**
You pay On-Demand pricing, which is the most expensive option. For a server that runs 24/7, the difference between On-Demand and a Savings Plan can be thousands of dollars per month.

---

### 1.6 Reserved Instances

Reserved Instances (RIs) are a billing construct where you commit to using a specific instance type for 1 or 3 years and get a significant discount.

**Three payment options:**
- **All Upfront:** Pay everything now. Largest discount.
- **Partial Upfront:** Pay some now, some monthly. Medium discount.
- **No Upfront:** Pay monthly. Smallest discount but no capital needed.

**Two types:**

**Standard RIs:** Locked to specific instance family, size, OS, and region. Cannot be changed. Biggest discount (up to 72%).

**Convertible RIs:** Can change the instance family, size, or OS during the term. Smaller discount (up to 54%) but flexibility if your needs change.

**Marketplace:**
If you have unused RIs, you can sell them on the AWS Reserved Instance Marketplace to other AWS customers. You can also buy RIs from others if you need them for a shorter term than 1 year.

**In production:**
Reserved Instances work well when you have predictable baseline capacity — e.g., "we always need at least 10 `m5.large` instances running 24/7." You RI those 10. Any additional instances above 10 that Auto Scaling spins up during traffic spikes run On-Demand or Spot.

---

### 1.7 Dedicated Hosts

A **Dedicated Host** is a physical EC2 server that is allocated entirely for your use. No other AWS customer's virtual machines share that physical hardware with you.

**Why would you need this?**

**Compliance and regulatory requirements:** Some industries (government, finance, healthcare) require that workloads do not share physical hardware with unknown third parties.

**Bring Your Own License (BYOL):** Microsoft Windows Server, SQL Server, Oracle, and some other software licenses are tied to physical cores or sockets. On a shared host, AWS manages the licensing. On a Dedicated Host, you can use your existing per-socket or per-core licenses, which can save tens of thousands of dollars per year.

**What you control on a Dedicated Host:**
- Which physical host your instances run on
- When instances are placed on which hosts
- Core/socket visibility for license management

**Cost:**
Dedicated Hosts are significantly more expensive than shared tenancy. You pay for the entire physical host regardless of how much of it you use.

**Dedicated Hosts vs Dedicated Instances:**
- **Dedicated Host:** You manage placement on a specific physical server. You see the server's physical details. Best for licensing.
- **Dedicated Instance:** Your instance runs on hardware dedicated to your account, but AWS manages placement. You don't see physical details. Slightly cheaper than Dedicated Hosts.

---

### 1.8 Capacity Reservations

A **Capacity Reservation** reserves EC2 capacity in a specific Availability Zone for any duration — even 1 hour. Unlike Reserved Instances, you are NOT committing to pricing. You are just ensuring capacity is available.

**The problem this solves:**

EC2 capacity in a given Availability Zone is not infinite. During regional events (natural disasters, huge demand spikes), available capacity can run out. If you need to quickly scale up and capacity isn't available, your Auto Scaling launch fails.

**How it works:**
You create a Capacity Reservation specifying:
- Instance type (e.g., `c5.2xlarge`)
- Number of instances
- Availability Zone
- Duration (open-ended or fixed end date)

AWS immediately reserves that capacity for you. You pay for it from the moment of reservation — whether you use those instances or not.

**When to use:**
- Before a planned event (product launch, Black Friday) when you know you'll need a sudden burst of capacity
- For disaster recovery — reserve capacity in another AZ so you can failover quickly
- Regulated workloads that must guarantee compute availability

**Combined with Savings Plans or RIs:**
You can attach Capacity Reservations to RIs or Savings Plans to both guarantee capacity AND get discounted pricing.

---

### 1.9 Capacity Manager (AWS Resource Access Manager for EC2)

**Capacity Manager** (formally part of AWS RAM) allows you to centrally manage and share Capacity Reservations across multiple AWS accounts within an AWS Organization.

**The problem it solves:**
Large enterprises have hundreds of AWS accounts (one per team, environment, or application). If each account manages capacity independently, there is waste — Account A has unused reserved capacity while Account B is capacity-constrained.

**How it works:**
- The management account creates a "Capacity Reservation" sharing policy
- Unused Capacity Reservations in one account can be used by other accounts in the organization
- Centralized visibility shows capacity utilization across all accounts

**In production:**
A company's platform team might reserve 200 `m5.xlarge` instances at the organization level. Individual teams can draw from that pool. Unutilized capacity automatically becomes available for other teams.

---

## Section 2 — Images (AMIs)

### 2.1 AMIs

An **Amazon Machine Image (AMI)** is a snapshot of a complete EC2 instance at a point in time. It contains:
- The operating system (Amazon Linux, Ubuntu, Windows, Red Hat, etc.)
- All installed software, libraries, and configurations
- The root volume contents
- Block device mapping (what volumes to attach and how)
- Launch permissions

**Think of an AMI as a "golden image" or a master template for a virtual machine.** When you launch an instance, AWS clones that AMI onto a new virtual machine in seconds.

**Why AMIs matter:**

Without AMIs, every new EC2 instance would be a blank OS. You'd have to manually install your web server, database, application code, configurations, security hardening, monitoring agents, etc. every single time. That could take 30 minutes to hours and is error-prone.

With a custom AMI:
- A new instance boots with everything pre-installed
- 100 instances are identical — no configuration drift
- Boot time to "ready for traffic" is minutes instead of hours

**Creating a custom AMI:**
1. Launch a base instance (e.g., Amazon Linux 2023)
2. SSH in and install/configure everything: nginx, Node.js, your app, CloudWatch agent, etc.
3. Harden the OS (disable root login, update packages, etc.)
4. "Create Image" from the running or stopped instance
5. AWS stops the instance, snapshots all EBS volumes, registers the AMI
6. You now have an AMI ID (e.g., `ami-0abc123def456789`)

**AMI regions:**
AMIs are region-specific. If you create an AMI in `ap-northeast-1` (Tokyo) and want to use it in `us-east-1` (Virginia), you must copy it to that region first.

**AMI ownership:**
- **Public AMIs:** Anyone can use them (e.g., Amazon's official Amazon Linux AMIs, community AMIs)
- **Private AMIs:** Only your account can use them
- **Shared AMIs:** You share specific AMIs with specific AWS account IDs

---

### 2.2 AMI Catalog

The **AMI Catalog** is a searchable directory of all AMIs available to you:
- **AWS-provided:** Amazon Linux 2023, Amazon Linux 2, Windows Server varieties, etc.
- **AWS Marketplace:** Commercial software pre-baked into AMIs (e.g., Palo Alto firewall, F5 load balancer, Splunk, hardened CIS-benchmark OS images). These often have per-hour licensing costs on top of EC2 costs.
- **Community AMIs:** AMIs shared publicly by other AWS customers. **Use with caution** — community AMIs are not vetted by AWS. Always verify the source before using in production.
- **My AMIs:** Your custom AMIs and AMIs shared with your account.

**Filters available:**
- Architecture (x86_64, arm64)
- Root device type (EBS, instance store)
- Virtualization type (HVM — hardware virtual machine, is the standard today)
- OS
- Region

---

## Section 3 — Elastic Block Store (EBS)

**EBS is the hard drive for your EC2 instance.** It is a network-attached block storage device. Unlike the instance itself, EBS volumes exist independently — if you terminate an instance, the EBS volume can survive (unless you set "delete on termination").

Think of EBS like a USB hard drive. You can plug it into an instance, unplug it, and plug it into another instance.

---

### 3.1 Volumes

An **EBS Volume** is a block storage device. It looks like a regular disk to the operating system inside your EC2 instance.

**Volume Types:**

| Type | Name | IOPS | Throughput | Best For |
|---|---|---|---|---|
| gp3 | General Purpose SSD v3 | Up to 16,000 | Up to 1,000 MB/s | Default for most workloads |
| gp2 | General Purpose SSD v2 | 3 IOPS/GB, up to 16,000 | Up to 250 MB/s | Legacy, prefer gp3 |
| io2 Block Express | Provisioned IOPS SSD | Up to 256,000 | Up to 4,000 MB/s | Critical production databases |
| io1 | Provisioned IOPS SSD | Up to 64,000 | Up to 1,000 MB/s | High performance databases |
| st1 | Throughput Optimized HDD | 500 IOPS | Up to 500 MB/s | Big data, log processing |
| sc1 | Cold HDD | 250 IOPS | Up to 250 MB/s | Infrequently accessed data, cheapest |

**gp3 vs gp2:**

`gp3` is the modern default. It decouples IOPS from size — with `gp2`, you got 3 IOPS per GB, so the only way to get more IOPS was to buy a bigger volume. With `gp3`, you configure IOPS and throughput independently of size. `gp3` is also ~20% cheaper than `gp2`.

**What IOPS means:**
IOPS = Input/Output Operations Per Second. A database might do thousands of tiny reads and writes per second. Low IOPS = slow database performance, high query latency.

**Root volume vs data volume:**
- The **root volume** (`/`) holds the OS and is created from the AMI. Usually 8–30 GB.
- **Data volumes** are additional disks you attach for databases, application data, logs, etc.
- Best practice: Keep data on separate volumes from the OS. If you need to resize the OS, you don't touch the data. If you need to replace the server, you detach and reattach data volumes.

**Multi-attach:**
With `io1` and `io2` volumes, you can attach the same volume to multiple instances simultaneously (within the same AZ). Used for clustered applications like Oracle RAC.

**What happens if an EBS volume fails?**
EBS volumes are replicated within their Availability Zone. Hardware failure of the underlying storage is automatically handled by AWS — your volume continues working. However, if the AZ itself has an outage, your volume is unavailable. For critical data, take snapshots (backups) to S3.

---

### 3.2 Snapshots

A **Snapshot** is a point-in-time backup of an EBS volume stored in S3 (though you access it through the EC2 console, not S3 directly).

**How snapshots work:**

The first snapshot copies all used blocks of the volume. Each subsequent snapshot is **incremental** — only blocks that changed since the last snapshot are copied. This makes snapshots space-efficient and fast.

However, to restore a volume from any snapshot, AWS reconstructs the full volume by reading that snapshot + all previous incremental changes. You don't need to manage this — AWS handles it automatically.

**Key snapshot capabilities:**

- **Restore a volume:** Create a new EBS volume from a snapshot. The volume can be in any AZ or region.
- **Copy to another region:** Snapshots can be copied cross-region for disaster recovery.
- **Share:** Share a snapshot with another AWS account.
- **AMI creation:** Creating an AMI automatically takes a snapshot of all attached volumes.
- **Recycle Bin:** Deleted snapshots can be protected with a retention rule so they aren't permanently deleted immediately.

**Consistency:**

For a database, if you snapshot while the database is writing, you might capture a partially written state. Solutions:
1. Freeze the filesystem or quiesce the database before snapshotting
2. Use AWS Backup with application-consistent snapshots
3. Use Multi-AZ databases (RDS) which handle this automatically

**In production:**
A common pattern is daily snapshots with 7-day retention for development volumes, and hourly snapshots with 30-day retention for production databases.

---

### 3.3 Lifecycle Manager

**Amazon Data Lifecycle Manager (DLM)** automates the creation, retention, and deletion of EBS snapshots and EBS-backed AMIs.

**Without DLM:**
Someone manually takes snapshots, remembers to delete old ones, and maintains the backup rotation. This is error-prone and often neglected.

**With DLM:**
You create a **Lifecycle Policy** that says:
- Take a snapshot of all volumes tagged `Environment=Production` every 24 hours at 3:00 AM UTC
- Retain the last 30 snapshots
- Copy the snapshot to `us-west-2` for disaster recovery
- Delete snapshots older than 30 days

This runs automatically, forever, without human intervention.

**Policy types:**
- **EBS Snapshot Policy:** Automates snapshots of EBS volumes
- **EBS-backed AMI Policy:** Automates creation and deregistration of AMIs (useful for baking new server images)
- **Cross-account copy event policy:** Automatically copy snapshots shared from another account

---

## Section 4 — Network & Security

### 4.1 Security Groups

A **Security Group** is a virtual firewall that controls inbound and outbound traffic for your EC2 instances. It operates at the instance level (not the subnet level — that's Network ACLs).

**Key characteristics:**

- **Stateful:** If you allow inbound traffic on port 80, the return traffic (the response) is automatically allowed outbound, even if there's no explicit outbound rule for it.
- **Allow-only:** Security groups only have ALLOW rules. There is no explicit DENY. Everything not allowed is implicitly denied.
- **Applied to the ENI:** Security groups are attached to Elastic Network Interfaces, not directly to instances. An instance can have up to 5 security groups attached.
- **Evaluated as a whole:** If any security group attached to an instance allows traffic, it is allowed. There is no ordering.

**Inbound rules define what traffic can reach your instance:**

Example rules for a web server:
- Allow TCP port 80 from `0.0.0.0/0` (HTTP from anywhere)
- Allow TCP port 443 from `0.0.0.0/0` (HTTPS from anywhere)
- Allow TCP port 22 from `10.0.0.0/8` (SSH only from internal VPN range)

**Outbound rules define what traffic your instance can send:**

By default, all outbound traffic is allowed. In high-security environments, you restrict outbound traffic too (e.g., only allow port 443 to specific S3 endpoints).

**Referencing security groups in rules:**

Instead of an IP range, you can use another security group as the source. This is powerful:

Example: Your `web-sg` security group allows inbound port 443 from the internet. Your `db-sg` allows inbound port 5432 (PostgreSQL) only from `web-sg`. This means only your web server instances can talk to the database — not any internet traffic, not even your laptop.

**What happens if you don't configure security groups correctly:**

- Too open: Your database is accessible from the internet → data breach
- Too restrictive: Your web server can't reach the internet to download updates → patching fails
- Missing egress rule: Your instance can't send logs to CloudWatch → no visibility into production

---

### 4.2 Elastic IPs

A **public IP address** assigned to an EC2 instance is dynamic by default — every time you stop and start the instance, it gets a new public IP.

**The problem this creates:**
- DNS records pointing to your old IP break
- Whitelists at firewalls/clients that use IP addresses break
- Monitoring and alerting tools lose track of the instance

An **Elastic IP (EIP)** is a static public IPv4 address that you own in your AWS account. It stays the same regardless of what instance it's attached to or whether the instance is stopped.

**How it works:**
1. Allocate an Elastic IP from AWS's pool (or bring your own)
2. Associate it with an EC2 instance or a Network Interface
3. The instance now has a persistent public IP

**Key behaviors:**
- An EIP not associated with a running instance **costs money** (~$0.005/hr). AWS charges for unused EIPs to prevent hoarding of IP addresses, which are a finite resource.
- You can quickly remap an EIP to a different instance — if your primary server fails, you associate the EIP to your standby server. DNS records don't change. Failover happens in seconds.
- You can have up to 5 EIPs per region (can request more via support).

**In production:**
For web servers behind a load balancer, you usually don't need EIPs — the load balancer has its own DNS name and IP management. EIPs are most useful for single-server setups, bastion hosts (jump boxes), NAT gateways, or any case where you need a stable IP for external whitelisting.

---

### 4.3 Placement Groups

**Placement Groups** control how EC2 instances are physically positioned across AWS's underlying hardware. This matters for network performance and fault tolerance.

**Three strategies:**

**Cluster Placement Group:**
All instances are placed on hardware within a single Availability Zone, as close together as possible — sometimes on the same physical rack.
- Result: Ultra-low latency (~microseconds), high bandwidth (up to 100 Gbps) between instances
- Risk: If the rack fails, all instances might go down together
- Use case: High-performance computing, big data jobs, distributed machine learning, anything needing maximum inter-node network speed

**Spread Placement Group:**
Instances are placed on distinct underlying hardware — different racks with separate power and networking.
- Result: Maximum fault isolation. Losing one rack doesn't affect other instances.
- Limitation: Maximum 7 instances per AZ per group
- Use case: Critical services where you need to guarantee individual instances don't share hardware fate. E.g., the 3 nodes of a Zookeeper quorum.

**Partition Placement Group:**
Instances are divided into "partitions." Each partition is on separate hardware (different racks). Instances within a partition may share hardware with each other.
- Result: Balance between scale and fault isolation. You know which partition an instance is in.
- Scale: Up to 7 partitions per AZ, hundreds of instances per partition
- Use case: Distributed databases (Cassandra, HDFS, Kafka) where you want rack-awareness. You place each replica in a different partition so a rack failure only loses one replica.

---

### 4.4 Key Pairs

A **Key Pair** consists of a public key and a private key used for SSH authentication to Linux instances (and password decryption for Windows instances).

**How it works:**

When you create a key pair:
1. AWS generates a public-private key pair
2. AWS stores the **public key** on their servers
3. You download the **private key** (`.pem` file) — **AWS never stores this. If you lose it, it's gone.**

When you launch a Linux instance with this key pair:
- AWS places the public key into the instance's `~/.ssh/authorized_keys` file during first boot
- To connect: `ssh -i my-key.pem ec2-user@<public-ip>`
- Your private key authenticates you without a password

**What if you lose the private key?**

You cannot recover it. Your options:
1. Stop the instance
2. Detach its EBS root volume
3. Attach the volume to another instance as a secondary volume
4. Mount it and replace the `authorized_keys` file with a new public key
5. Detach and reattach to the original instance

This is painful. In production, teams store private keys in a secrets manager (AWS Secrets Manager, HashiCorp Vault) and/or use AWS Systems Manager Session Manager (SSM), which doesn't require SSH or key pairs at all — you connect through the AWS console or CLI using IAM permissions.

**Key pair types supported:**
- RSA (2048-bit, classic, widely compatible)
- ED25519 (newer, faster, more secure — not supported by all older SSH clients)

**In production:**
Many mature teams stop using Key Pairs entirely in favor of AWS Systems Manager Session Manager. SSM uses IAM roles, provides full audit logs of every session, and doesn't expose SSH ports to the internet.

---

### 4.5 Network Interfaces

An **Elastic Network Interface (ENI)** is a virtual network card. Every EC2 instance has at least one ENI (the primary one, `eth0`). You can attach additional ENIs.

**What each ENI has:**
- A Primary Private IPv4 address
- One or more Secondary Private IPv4 addresses
- One Elastic IP per private IP (optional)
- One Public IPv4 address (optional)
- One or more IPv6 addresses
- A MAC address
- One or more Security Groups
- A Source/Destination check flag

**Why would you add a secondary ENI?**

- **Dual-homed instances:** An instance with one ENI in a public subnet and one ENI in a private subnet, acting as a gateway or security appliance
- **Management network:** A separate ENI dedicated to monitoring/management traffic so it doesn't mix with production traffic
- **License compliance:** Some software is licensed to a specific MAC address. By keeping the ENI separate, you can move it between instances and the MAC address (and thus the license) follows.
- **High availability failover:** Move an ENI from a failed instance to a replacement instance — private IPs and MAC addresses follow.

**Source/Destination Check:**

By default, EC2 drops packets where the instance isn't the source or destination. If you run a NAT instance or a software router/firewall on EC2, you must disable this check, because the instance is intentionally forwarding packets for others.

---

## Section 5 — Load Balancing

Load balancing distributes incoming traffic across multiple EC2 instances. This is fundamental to any highly available production architecture.

### 5.1 Load Balancers

AWS offers four types of Elastic Load Balancers (ELBs):

**Application Load Balancer (ALB) — Layer 7:**
- Routes traffic based on HTTP/HTTPS content: URL paths, hostnames, headers, query strings
- Example: `api.example.com/users` → sends to the "users" service; `api.example.com/payments` → sends to the "payments" service
- Supports WebSockets, HTTP/2, gRPC
- Built-in SSL termination
- The most common type for web applications and microservices

**Network Load Balancer (NLB) — Layer 4:**
- Routes based on TCP/UDP/TLS — it doesn't look inside the packet
- Extremely high performance: millions of requests per second, single-digit millisecond latency
- Preserves client IP address (ALB replaces source IP with load balancer IP)
- Use case: Gaming, real-time communications, TCP-based protocols, when you need ultra-low latency

**Gateway Load Balancer (GWLB) — Layer 3:**
- Specialized for transparent insertion of network appliances (firewalls, intrusion detection, deep packet inspection)
- Traffic passes through your third-party security appliance transparently
- Use case: Enterprise network security architectures

**Classic Load Balancer (CLB) — Legacy:**
- The original ELB, supports both Layer 4 and Layer 7 but with fewer features
- AWS recommends migrating to ALB or NLB
- Don't use for new projects

**How an ALB works step by step:**
1. DNS resolves `api.example.com` to the ALB's IP address(es)
2. Client sends HTTP request to the ALB
3. ALB receives the request and evaluates routing rules in order
4. Matching rule forwards to a Target Group (e.g., a group of EC2 instances running your API)
5. ALB picks a healthy instance using its load balancing algorithm (round-robin by default, or least outstanding requests)
6. ALB forwards the request to that instance, adding headers like `X-Forwarded-For` with the client's real IP
7. Instance processes the request and sends response back to ALB
8. ALB returns response to the client

**Health Checks:**
The load balancer periodically checks each registered instance. For ALB, it sends an HTTP request to a health check endpoint (e.g., `GET /health`). If the instance returns HTTP 200, it's healthy. If it fails X times in a row, the instance is marked unhealthy and removed from rotation. Traffic stops going to it automatically.

This means: **a broken instance stops receiving traffic without any human intervention.**

---

### 5.2 Target Groups

A **Target Group** is a collection of registered targets that a load balancer routes traffic to.

**Target types:**
- **Instance:** Route to EC2 instances by instance ID
- **IP:** Route to any IP address (can be outside AWS — useful for on-premises or containers)
- **Lambda:** Route HTTP traffic to a Lambda function

**Target Group settings:**

**Protocol and Port:** What protocol and port the load balancer uses to communicate with targets (e.g., HTTP:8080, HTTPS:443, TCP:3306)

**Health Check Configuration:**
- Path (for HTTP/HTTPS): `/health`, `/ping`, etc.
- Healthy threshold: How many consecutive successes to consider healthy
- Unhealthy threshold: How many consecutive failures to consider unhealthy
- Interval: How often to check
- Timeout: How long to wait for a response

**Stickiness (Session Affinity):**
When enabled, the same client is always routed to the same target (using a cookie). Useful for applications that store session state in memory. Better practice: store session state externally (Redis, DynamoDB) so stickiness isn't needed.

**Deregistration delay:**
When you remove an instance from a target group (e.g., during deployment or scale-down), the load balancer waits this long before stopping sending traffic to it. Default: 300 seconds. During this time, in-flight requests complete. Prevents dropping active connections.

---

### 5.3 Trust Stores (ELB)

**Trust Stores** are used for **mutual TLS (mTLS)** authentication on Application Load Balancers.

In normal HTTPS (one-way TLS):
- Client verifies the server's certificate
- Server doesn't verify the client's identity

In **mutual TLS**:
- Client verifies the server's certificate
- Server (in this case, the ALB) also verifies the client's certificate
- Only clients with a valid certificate from a trusted CA can connect

**A Trust Store** is a collection of Certificate Authority (CA) certificates that the ALB trusts. If a client presents a certificate signed by one of those CAs, the connection is allowed.

**When to use mTLS:**
- B2B APIs where you know exactly which clients should connect (partners, microservices)
- Zero-trust architectures
- Financial services, healthcare, or government APIs with strict authentication requirements

**How it works:**
1. You create a Trust Store in the ELB console and upload CA certificates
2. You configure an HTTPS listener on your ALB to use this Trust Store
3. When a client connects, the ALB requests the client's certificate
4. If the certificate is valid and signed by a trusted CA, the connection proceeds
5. The ALB can optionally pass client certificate details to your backend as HTTP headers

---

## Section 6 — Auto Scaling

### 6.1 Auto Scaling Groups

An **Auto Scaling Group (ASG)** automatically manages a fleet of EC2 instances — launching new instances when demand increases and terminating them when demand decreases.

**Core configuration:**

- **Minimum capacity:** Never go below this many instances (even if demand is zero)
- **Desired capacity:** The "target" number of instances right now
- **Maximum capacity:** Never exceed this many instances (cost protection)

**Scaling policies — how the ASG knows when to scale:**

**Target Tracking Scaling:** The simplest and most common. You set a target metric value, and the ASG continuously adjusts instance count to maintain it.
- Example: "Keep average CPU utilization at 50%." When CPU exceeds 50% across the fleet, add instances. When it drops below, remove instances.

**Step Scaling:** Define thresholds with different actions.
- Example: If CPU > 70% → add 2 instances. If CPU > 90% → add 5 instances.

**Scheduled Scaling:** Scale on a calendar schedule.
- Example: Every weekday at 8 AM, set desired capacity to 20 (office hours traffic). At 8 PM, set desired to 5.

**Predictive Scaling:** Uses ML to forecast traffic and pre-scales before the traffic arrives. Useful for workloads with predictable patterns (Monday morning traffic spikes, etc.).

**Instance refresh:**
When you update the launch template (e.g., new AMI), you trigger an "Instance Refresh." The ASG gradually replaces old instances with new ones:
1. Terminates a percentage of old instances
2. Launches new instances with the updated template
3. Waits for new instances to pass health checks
4. Proceeds to the next batch

This enables **zero-downtime deployments** of infrastructure updates.

**Lifecycle hooks:**
Before an instance is added to the load balancer (or before it's terminated), lifecycle hooks pause the process and wait for a signal. This allows you to:
- Run configuration scripts after launch before traffic hits the instance
- Drain connections and save state before termination

**Health checks:**
ASGs can use EC2 health checks (is the instance running?) or ELB health checks (is the application healthy?). If ELB health checks are enabled and your app is broken (returns 500s), the ASG terminates and replaces the unhealthy instance automatically.

**In production — a full Auto Scaling flow:**

1. Your e-commerce site runs 5 instances normally.
2. A sale begins. Traffic spikes 10x. Average CPU shoots to 85%.
3. ASG's target tracking policy detects CPU > 50% target.
4. ASG launches 10 new instances using the launch template.
5. New instances boot, pull application code (or it's pre-baked in the AMI), register with the ALB target group.
6. ALB health check passes. New instances receive traffic.
7. CPU drops back to 50%. Everyone is happy.
8. Sale ends. Traffic drops. CPU falls to 15%.
9. ASG scale-in begins. Instances are deregistered from ALB.
10. Deregistration delay (300s) passes. In-flight requests complete.
11. ASG terminates excess instances back to 5.
12. You paid for exactly the compute you needed, for exactly the time you needed it.

---

## Launching an Instance — Step by Step

This section walks through every option in the "Launch Instance" wizard, what each setting means, and what happens if you set it wrong.

---

### Step 1.1 — Name and Tags

**What it is:**
A name tag is just a tag with the key `Name`. Tags are key-value metadata pairs attached to AWS resources.

**Why tags matter:**
- **Cost allocation:** AWS Cost Explorer can break down costs by tag. Tag `Environment=Production` vs `Environment=Development` to see how much each costs.
- **Automation:** Lambda functions, Systems Manager, and other tools can target resources by tag. "Apply this patch to all instances tagged `Patch=True`."
- **Organization:** When you have 500 instances, a name like `prod-api-web-01` is the only way to know what something is.
- **Access control:** IAM policies can restrict which instances a user can start/stop based on tags.

**What happens if you don't tag:**
Your AWS bill becomes a mystery. You can't tell which instances serve which purpose. Compliance reporting becomes a manual nightmare. Cost optimization is impossible.

**Naming convention in production:**
`[environment]-[application]-[role]-[number]`
Example: `prod-payments-api-01`, `staging-auth-worker-03`

---

### Step 1.2 — Application and OS Images (Amazon Machine Image)

**What it is:**
The AMI is the operating system + pre-installed software that your instance boots from. This was covered in detail in [Section 2](#section-2--images-amis).

**At launch time you choose:**
- An AMI from the catalog (AWS, Marketplace, Community, or your own)
- The AMI must match your instance type's architecture (x86_64 or arm64)

**Amazon Linux 2023 (the default):**
- AWS's own Linux distribution, optimized for EC2
- Based on Fedora, similar to RHEL/CentOS
- Regular security updates, long support lifecycle
- Includes AWS tools pre-installed (SSM agent, CloudWatch agent)
- `dnf` package manager (not `yum` or `apt`)

**What happens if you choose a Community AMI from an untrusted source:**
The AMI could contain malware, crypto miners, backdoors, or data-exfiltration tools pre-installed. Always verify the publisher. For production: use AWS-official AMIs or build your own from a known base.

---

### Step 1.3 — Instance Type

The instance type defines CPU, RAM, network, and storage performance. Covered in detail in [Section 1.2](#12-instance-types).

**For the launch wizard:**
- `t2.micro` and `t3.micro` are free-tier eligible (750 hrs/month in the first year)
- The wizard shows the selected type's vCPUs, RAM, and on-demand price per hour
- You can change instance type after launch (requires stop → change → start)

**Common mistake:** Leaving everything on `t2.micro` in production. A database on a `t2.micro` with 1 GB RAM will fail under any real load.

---

### Step 1.4 — Key Pair (Login)

The SSH key pair for connecting to Linux instances. Covered in [Section 4.4](#44-key-pairs).

**What if you choose "Proceed without a key pair":**
- You cannot SSH into the instance using a key pair
- You can still connect using AWS Systems Manager Session Manager (if the SSM agent is running and an IAM profile is attached)
- For instances where you never need SSH (e.g., fully automated workloads, containers), this is fine
- If you need SSH access later and have no key pair, see the recovery procedure in Section 4.4

---

### Step 1.5 — Network Settings

This section determines where in the network your instance lives and what traffic can reach it.

#### Step 1.5.1 — Network (VPC)

**VPC = Virtual Private Cloud.** It's your private, isolated network within AWS. All EC2 instances must live inside a VPC.

Every AWS account comes with a **Default VPC** in each region — pre-configured with public subnets in each Availability Zone. This is fine for learning but suboptimal for production.

**In production:** Companies create custom VPCs with:
- Public subnets (for load balancers, bastion hosts)
- Private subnets (for app servers, databases)
- Multiple Availability Zones (for high availability)
- VPC peering or Transit Gateway (to connect multiple VPCs)

**What happens if you put everything in the default VPC:**
It works, but all subnets are public by default, meaning all instances get public IPs. Your database is technically reachable from the internet (only your security group saves you — defense in depth is better).

#### Step 1.5.2 — Subnet

A **subnet** is a range of IP addresses within your VPC, tied to a specific Availability Zone.

**Example VPC layout:**
```
VPC: 10.0.0.0/16 (65,536 IPs)
├── Public Subnet AZ-a:  10.0.1.0/24 (256 IPs) — has route to Internet Gateway
├── Public Subnet AZ-b:  10.0.2.0/24
├── Private Subnet AZ-a: 10.0.3.0/24 — no direct internet route
└── Private Subnet AZ-b: 10.0.4.0/24
```

**Public subnet:** Has a route to an Internet Gateway. Instances can have public IPs and be reached from the internet.

**Private subnet:** No route to the internet. Instances have only private IPs. They can reach the internet outbound through a NAT Gateway (for updates, API calls, etc.) but cannot be reached inbound from the internet.

**In production:** Web servers/load balancers go in public subnets. App servers and databases go in private subnets. This layered approach is defense in depth.

#### Step 1.5.3 — Auto-Assign Public IP

If enabled, AWS automatically assigns a public IPv4 address from its pool to the instance.

**Enable when:** Instance is in a public subnet and needs to be directly reachable from the internet, or needs to call internet APIs without a NAT Gateway.

**Disable when:** Instance is in a private subnet. Or you want to use an Elastic IP instead (for a stable IP). Or you're behind a load balancer and direct internet access isn't needed.

**What happens if you're in a private subnet with auto-assign public IP enabled:**
The instance gets an IP, but the subnet's route table has no route to the internet, so it's unreachable from outside anyway. The IP is wasted.

#### Step 1.5.4 — Firewall (Security Groups)

**1.5.4.1 Create Security Group:**
Define a new security group from scratch in this wizard. You specify a name, description, and inbound rules.

**1.5.4.2 Select Existing Security Group:**
Use a pre-created security group. In production, security groups are defined by your infrastructure team and you select the appropriate one (e.g., `web-sg`, `app-sg`, `db-sg`).

**1.5.5 Common Security Groups:**
If your organization has pre-defined security groups shared across projects, they appear here for quick selection.

**Critical rule:** Never attach a security group with `0.0.0.0/0` on port 22 (SSH) or port 3389 (RDP) in production. This exposes your instance to brute-force attacks from the entire internet. Attackers scan the entire IPv4 internet continuously for open SSH ports.

---

### Step 1.6 — Configure Storage

#### Step 1.6.1 — Root Volume (gp3, 8 GiB by default)

The root volume is the main disk that holds your OS. The wizard creates one automatically from the AMI.

**Key settings:**
- **Size:** Default is usually 8 GiB for Linux. Increase if your OS, logs, or application needs more space.
- **Volume type:** Default is `gp3`. Adjust IOPS and throughput if needed.
- **IOPS (gp3):** Default 3,000. Increase for I/O-heavy workloads up to 16,000.
- **Encrypted:** Encrypt the volume with a KMS key. Strongly recommended in production. Encryption is at rest — data on disk is encrypted. There's no measurable performance impact on modern instances.
- **Delete on termination:** If checked, this volume is deleted when the instance is terminated. Default: ON for root volumes. For root volumes, this is usually fine. For data volumes, you often want this OFF.

**What happens if the root volume fills up (100% disk usage):**
The OS cannot write anything — no new files, no logs, no temp files. The instance becomes unstable. Processes crash. You may not be able to SSH in. This is a very common production outage. Monitor disk usage with CloudWatch.

#### Step 1.6.2 — Add New Volume

Add additional EBS volumes. Common use cases:
- Separate volume for database data files (so you can resize the OS volume without touching data)
- Separate volume for logs (so log growth doesn't fill the OS volume)
- High-performance volumes with different settings than the root volume

#### Step 1.6.3 — File Systems

**1.6.3.1 S3 Files (new):**
Mount an S3 bucket as a filesystem using Mountpoint for Amazon S3. Your instances can read/write files to S3 using standard filesystem operations. Good for large datasets where you don't need the full speed of EBS.

**1.6.3.2 EFS (Elastic File System):**
A fully managed, scalable NFS filesystem. Unlike EBS (which can only be attached to one instance at a time in most cases), EFS can be mounted on **thousands of EC2 instances simultaneously**. All instances see the same files.

Use case: Shared file storage across multiple web servers (e.g., user-uploaded files, shared configuration, CMS media files).

**1.6.3.3 FSx:**
AWS-managed file systems for specific workloads:
- **FSx for Windows File Server:** SMB/CIFS shares, Active Directory integration. For Windows workloads.
- **FSx for Lustre:** High-performance parallel filesystem for HPC, ML training, video processing. Integrates with S3.
- **FSx for NetApp ONTAP:** Full NetApp filesystem with snapshots, cloning, multi-protocol.
- **FSx for OpenZFS:** ZFS filesystem features (snapshots, compression, clones) in AWS.

---

### Step 1.7 — Advanced Details

Advanced Details contains fine-grained control over instance behavior. Most of these have sensible defaults but are important to understand for production.

#### Domain Join Directory
Automatically join the instance to an AWS Managed Microsoft Active Directory (or Simple AD) domain on first boot. Used in Windows environments where servers must be domain-joined for authentication and Group Policy.

**If not set:** The instance is a standalone server. Windows authentication is local. Linux PAM/SSSD-based AD integration is manual.

#### IAM Instance Profile

An **IAM Instance Profile** gives the EC2 instance an IAM role. This allows applications running on the instance to call AWS APIs (S3, DynamoDB, CloudWatch, etc.) without storing AWS access keys on the server.

**This is critically important.** Storing AWS access keys (Access Key ID + Secret Access Key) on EC2 instances is a major security risk. If the instance is compromised, the attacker has your AWS credentials.

**With an IAM Instance Profile:**
- The instance has a role (e.g., `ec2-app-role`)
- That role has policies attached (e.g., `AmazonS3ReadOnlyAccess`, `CloudWatchAgentServerPolicy`)
- Code on the instance calls the AWS metadata endpoint (`169.254.169.254`) to get temporary credentials automatically rotated by AWS
- No hardcoded keys anywhere

**What happens if you don't set an IAM profile:**
Your application can't call AWS APIs without embedding credentials in code or config files. This is the #1 source of AWS credential leaks on GitHub.

#### Hostname Type
Controls whether the instance's hostname is based on its private IP or a resource-based name.
- **IP name:** `ip-10-0-1-5.ec2.internal`
- **Resource-based:** `i-0abc12345def.ec2.internal` — stays the same even if the IP changes

#### Instance Auto-Recovery

If the underlying hardware fails (host hardware issue, OS kernel panic detected by system status check), auto-recovery automatically moves the instance to healthy hardware. The instance gets the same instance ID, EBS volumes, Elastic IP, and private IP — as if nothing happened.

**If disabled:** A hardware failure that isn't self-recovering requires manual intervention.

#### Shutdown Behavior

Controls what happens when the OS issues a shutdown command (e.g., `sudo shutdown -h now`):
- **Stop:** Instance is stopped (disk preserved). You can start it again.
- **Terminate:** Instance is permanently deleted. No recovery.

**Default: Stop.** Change to "Terminate" only for ephemeral instances (like Spot workers) where you intend OS-level shutdown to destroy the instance.

#### Stop - Hibernate Behavior

When you stop an instance with hibernation:
1. The OS writes the entire contents of RAM to the EBS root volume
2. The instance stops
3. When you start it again, the RAM contents are restored from EBS
4. The OS resumes exactly where it left off — running processes continue

This is like closing your laptop lid vs shutting it down. Applications don't restart. Sessions don't reset.

**Requirements:** Instance must have an encrypted root volume, instance RAM must fit on the root volume, instance type must support hibernation (not all do).

**Use case:** Development environments where you want to pause work and resume exactly where you left off.

#### Termination Protection

A safety flag. When enabled, you cannot terminate the instance via the console or CLI until you explicitly disable termination protection.

**This prevents accidents** — misclicking "Terminate" on a production database server.

**In production:** Always enable on critical instances. CI/CD pipelines that need to terminate instances will disable it programmatically first.

#### Stop Protection

Similar to termination protection, but prevents stopping the instance. Use when an unintended stop would cause downtime or data loss.

#### Detailed CloudWatch Monitoring

By default, EC2 sends metrics to CloudWatch every **5 minutes**. With detailed monitoring, metrics are sent every **1 minute**.

**When you need it:**
- Auto Scaling policies need faster reaction times
- Debugging performance issues requires 1-minute granularity
- SLA compliance monitoring

**Cost:** Detailed monitoring has an additional CloudWatch charge.

**What you miss without it:** If a traffic spike lasts only 3 minutes, 5-minute metrics might not capture it. Your Auto Scaling group might not react fast enough.

#### Credit Specification (for T-type instances)

For `t2`, `t3`, `t3a`, `t4g` burstable instances:
- **Standard:** CPU is throttled when credits run out
- **Unlimited:** CPU can burst beyond credits; you pay for excess usage

**When to use Unlimited:** If your workload needs consistent CPU performance and you're running on a burstable instance. For a production web server, CPU throttling means slow responses — you don't want that. Either switch to a non-burstable instance type or use Unlimited mode.

#### Placement Group

Associate the instance with a Placement Group (Cluster, Spread, or Partition). Covered in [Section 4.3](#43-placement-groups).

**If not set:** AWS places the instance wherever it finds space. No guarantee about proximity to other instances or hardware isolation.

#### EBS-Optimized Instance

Provides dedicated bandwidth between the instance and EBS storage, separate from network bandwidth to other instances.

Most modern instance types have this enabled by default. For older instance types, enabling it prevents your disk I/O from competing with your network traffic.

**If disabled on an I/O-heavy workload:** Database queries are slow and unpredictable because disk I/O is contending with network traffic.

#### Purchasing Option

- **On-Demand (default):** Pay by the second/hour. Most expensive. Maximum flexibility.
- **Spot Instances:** Up to 90% discount. Can be interrupted with 2-minute notice.
- **Capacity Blocks:** Reserve a block of GPU instances for a fixed duration (ML training).
- **Interruptible Capacity Reservations:** Launch into a Capacity Reservation with spot-like pricing.

#### Capacity Reservation

Specify which Capacity Reservation to launch into, if you have one. Ensures your instance gets the reserved capacity.

#### Tenancy

- **Shared (default):** Your VM shares physical hardware with other AWS customers' VMs. You are fully isolated at the hypervisor level, but you're on shared hardware.
- **Dedicated Instance:** Your VM is on hardware dedicated to your account.
- **Dedicated Host:** Your VM is on a specific physical server you manage.

#### RAM Disk ID / Kernel ID

For specialized custom kernel configurations. Rarely used. Leave as default unless you have a specific kernel requirement (legacy systems, real-time kernels, etc.).

#### Nitro Enclave

Creates a highly isolated compute environment within the instance — no persistent storage, no external networking, no SSH access. Only the parent instance can communicate with it via a local channel.

**Use case:** Processing extremely sensitive data (encryption keys, patient data, financial PII). Code running inside the enclave is cryptographically attested — you can prove exactly what code is running inside.

**Requires:** At least 2 vCPUs (one must be dedicated to the enclave).

#### CPU Options

Fine-grained control over vCPUs:
- **Disable hyperthreading:** Some HPC applications or licensed software (where licenses are per-core) benefit from physical cores only, not hyperthreaded logical cores.
- **Specify fewer vCPUs:** If you want a large instance's memory but don't need all its CPUs (e.g., you need 256 GB RAM but only 8 CPUs), you can configure fewer vCPUs. This can also reduce licensing costs for per-vCPU software.

#### Metadata Options

The **Instance Metadata Service (IMDS)** is an endpoint at `http://169.254.169.254` that provides information about the running instance: its IAM credentials, AMI ID, region, AZ, tags, user data, and more.

**Metadata Version — IMDSv1 vs IMDSv2:**

- **IMDSv1 (legacy):** Simple HTTP GET. No authentication required. Vulnerable to SSRF (Server-Side Request Forgery) attacks — a web application vulnerability where an attacker tricks the server into fetching the metadata URL, exposing IAM credentials.

- **IMDSv2 (recommended):** Requires a session token. First make a PUT request to get a token, then use the token in subsequent metadata requests. SSRF attacks cannot get the token (the PUT method doesn't work through typical SSRF vulnerabilities).

**Best practice:** Set to "V2 only (token required)" for all production instances.

**Metadata response hop limit:**
HTTP TTL value. Default 2 for IMDSv2. If your instance runs containers (like Docker), the container needs to reach the metadata service through an extra network hop. Set to 2 to allow containers to access instance metadata. Set to 1 if you want to prevent containers from accessing instance metadata.

**Allow tags in metadata:**
When enabled, instance tags are available via the metadata endpoint. Useful for scripts that need to know the instance's environment or role without hardcoding values.

#### User Data

**User Data is a script (bash, PowerShell, cloud-init) that runs automatically on the first boot of the instance.**

This is how you automate server configuration at launch time.

**Example User Data for a simple web server:**
```bash
#!/bin/bash
yum update -y
yum install -y nginx
systemctl start nginx
systemctl enable nginx
echo "Hello from $(hostname)" > /usr/share/nginx/html/index.html
```

When this instance launches:
1. AWS cloud-init reads the user data script
2. Runs it as root during first boot
3. nginx is installed, started, and configured

**User data runs only on the first boot** (unless you configure cloud-init to run it on every boot). For subsequent configuration, use Systems Manager Run Command or Ansible.

**What happens if your user data script fails:**
The instance boots, but it's not correctly configured. It might look "running" in the console but not actually work. Always add error handling and logging. A common pattern: log user data execution to CloudWatch Logs so you can debug failures.

**Base64 encoding:**
User data must be base64-encoded when passed via the API. The console does this automatically. If you check "User data has already been base64 encoded," you provide pre-encoded data.

**Security:** Don't put secrets (passwords, API keys) in user data. It's accessible via the instance metadata endpoint to anyone who can SSH in. Use AWS Secrets Manager or Parameter Store instead.

---

### Step 1.8 — Summary

The summary panel shows what you've configured:
- Number of instances to launch simultaneously
- AMI name and ID
- Instance type
- Security groups
- Storage volumes and sizes
- Free tier eligibility notice (if applicable)

**Free tier:**
In your first 12 months:
- 750 hours/month of `t2.micro` or `t3.micro`
- 750 hours/month of public IPv4 address
- 30 GiB of EBS storage
- 2 million I/Os
- 1 GB of EBS snapshots
- 100 GB of bandwidth to the internet

Hours are shared across all your `t2.micro` instances. Running 1 instance for 30 days = 720 hours (within the 750 limit). Running 2 instances simultaneously for 15 days = 720 hours. Running 3 instances simultaneously burns through 750 hours in ~10 days.

---

### Step 1.9 — Launch Instance

When you click "Launch Instance," here is the complete sequence of what AWS does:

1. **Validates your configuration** — Checks that the AMI exists, the instance type is available in your chosen AZ, the subnet is valid, security groups exist, key pair exists, etc.

2. **Finds a physical host** — AWS's scheduler finds a hypervisor server in your chosen AZ with enough spare capacity for your instance type.

3. **Provisions the EBS volumes** — Creates your root volume (and any additional volumes) by copying the AMI's snapshot to new EBS volumes in the same AZ.

4. **Allocates an Elastic Network Interface** — Creates a virtual NIC, assigns a private IP from your subnet's range, assigns a public IP if requested, attaches your security groups.

5. **Places the instance on the hypervisor** — The hypervisor (AWS Nitro, based on KVM) creates the VM, connects the EBS volumes (over the network), connects the ENI, allocates CPU and RAM.

6. **Boots the OS** — The instance's BIOS/UEFI runs, the bootloader finds the root volume, the kernel loads, cloud-init starts.

7. **cloud-init runs** — Sets the hostname, configures the network, installs SSH authorized keys (from your key pair), copies user data, runs user data scripts, and performs other first-boot initialization tasks.

8. **Instance state transitions** — In the console: `pending` → `running` (usually within 30–60 seconds)

9. **Status checks pass** — Two checks: System Status Check (is the underlying hardware OK?) and Instance Status Check (has the OS booted and is it responding?). Both must pass for the instance to be fully ready.

10. **Your instance is ready** — You can SSH in, HTTP is reachable, the application is running.

---

## What Happens After Launch — Production Flow

**Day-to-day operation of a production EC2 fleet:**

**Monitoring:**
- CloudWatch Agent on each instance sends OS-level metrics (memory, disk usage, process counts) — these are NOT sent by default; you must install the agent.
- EC2 automatically sends instance-level metrics (CPU utilization, network in/out, disk read/write ops) — no agent needed.
- Set CloudWatch Alarms: If CPU > 80% for 5 minutes → send SNS notification → page on-call engineer.

**Patching:**
- AWS Systems Manager Patch Manager automatically identifies and applies OS patches on a schedule.
- Alternatively, use Auto Scaling instance refresh: build a new AMI with patches, trigger instance refresh. Old instances are replaced by new ones with zero downtime.

**Logging:**
- CloudWatch Logs Agent or Unified CloudWatch Agent ships application logs, system logs, nginx/apache access logs to CloudWatch Logs in real time.
- Never store important logs only on the instance. If the instance is terminated, logs are gone. Centralize everything.

**Backups:**
- AWS Backup or DLM takes scheduled snapshots of EBS volumes.
- For stateless instances (app servers), the AMI IS the backup — spin up a new instance from the same AMI.
- For stateful instances (databases), hourly EBS snapshots with multi-day retention.

**Security:**
- AWS Inspector continuously scans running instances for known CVEs in installed packages.
- AWS GuardDuty analyzes VPC flow logs for anomalous traffic patterns.
- Security groups are reviewed regularly. Unused open ports are removed.
- Systems Manager Session Manager replaces SSH. Port 22 is closed.

**Deployments:**
- New application versions are deployed by Auto Scaling instance refresh (new AMI), CodeDeploy agent (deploy code to running instances), or blue/green deployment (new ASG behind same ALB).

---

## Real Production Architecture Example

A typical 3-tier web application on EC2:

```
Internet
   |
   ↓
[Route 53] — DNS
   |
   ↓
[CloudFront] — CDN, DDoS protection (optional)
   |
   ↓
[Application Load Balancer] — in public subnet, AZ-a + AZ-b
   |  (HTTPS:443, health checks to /health)
   ↓
[Auto Scaling Group: Web/App Servers] — in private subnet, AZ-a + AZ-b
  - t3.medium instances
  - IAM profile with S3 read and SSM access
  - Security group: allows 443 from ALB only, 443 outbound
  - Min: 2, Desired: 4, Max: 20
  - Scale on ALBRequestCountPerTarget > 1000
   |
   ↓
[Amazon RDS Multi-AZ] — in private subnet, AZ-a + AZ-b
  (or a database EC2 instance with io2 EBS, daily snapshots)
   |
   ↓
[Amazon ElastiCache] — session store, caching
```

**Access pattern:**
- No direct SSH to any instance. All access via SSM Session Manager, logged to CloudTrail.
- Deployments via CodeDeploy or instance refresh.
- Patches via SSM Patch Manager weekly.
- All logs to CloudWatch Logs.
- All metrics in CloudWatch with alarms to PagerDuty.

---

## Common Mistakes and What Breaks If You Skip Something

| What You Skip | What Breaks |
|---|---|
| No IAM instance profile | App can't call AWS APIs without hardcoded credentials. Credentials leak. |
| Security group too open (0.0.0.0/0 on SSH) | Instance is brute-force attacked constantly. |
| No Elastic IP on standalone server | Every restart changes the public IP. DNS breaks. Whitelists break. |
| Root volume "delete on termination" off, accidentally terminate instance | Orphaned EBS volumes accumulate. Costs money. |
| No termination protection on critical servers | Accidental termination destroys production. |
| IMDSv1 enabled | SSRF vulnerabilities can steal IAM credentials. |
| User data runs as root, no secret management | Secrets in user data are visible to anyone with SSH access. |
| Single AZ deployment | One AZ outage takes down your entire application. |
| No health check endpoint on application | Load balancer can't determine if app is actually working. Broken instances receive traffic. |
| No EBS snapshots | Any data corruption or accidental deletion is permanent. |
| No CloudWatch monitoring or alarms | You find out about outages when users complain. |
| Instance type too small for workload | App crashes under load. OOM kills. Slow response times. |
| No Auto Scaling | Traffic spikes cause outages. You overpay at all other times. |
| Storing logs only on instance | Instance terminated → all logs gone forever. |
| No disk space monitoring | Disk fills up → OS freezes → instance becomes unresponsive. |

---

*This document covers the complete AWS EC2 service as presented in the AWS Management Console, from every sub-section of the console navigation to every configuration option during instance launch, through real-world production operation patterns.*

---
**Last Updated:** Based on AWS EC2 console as of 2025 | Region: ap-northeast-1 (Tokyo) referenced in console links
