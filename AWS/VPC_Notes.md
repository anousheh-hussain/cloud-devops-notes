[AWS_VPC_Notes.md](https://github.com/user-attachments/files/28252199/AWS_VPC_Notes.md)
# AWS VPC — Notes + Interview Questions

---

## What Is a VPC? And Why Did It Even Need to Exist?

### The Problem Before VPC — What Was Happening in 2013-14

Before VPC became the default, AWS had a networking model called **EC2-Classic**.

When you launched an EC2 instance back then, it was placed into a **single giant flat network shared by ALL AWS customers**. Think of it like a massive open office floor — thousands of companies, all working in the same room, with no walls, no partitions, and no doors between them.

This meant:
- Any EC2 instance could potentially see and communicate with any other EC2 instance across completely different customer accounts
- There was zero isolation — your server and a stranger's server were on the same network
- You had no control over IP ranges, routing, or network topology
- Banks, hospitals, and government agencies could NOT use AWS — their data legally cannot sit on a shared open network

As AWS grew from a startup tool into enterprise-grade infrastructure, this became a critical problem. AWS introduced VPC in 2009 as an optional feature, but in **December 2013 made VPC the default for all new accounts**. EC2-Classic was permanently retired in 2022.

---

### So What Is a VPC?

**VPC = Virtual Private Cloud**

A VPC is your **own private, isolated network inside AWS**. It's like drawing walls around a section of AWS and saying: "This is MY network. Only my resources live here. Nobody from outside can see inside."

Think of it this way:

> AWS's entire infrastructure is like a massive apartment building with thousands of apartments.
> A VPC is YOUR apartment — private, locked, with your own rooms, keys, and rules.
> You decide who can come in, who can go out, and which rooms can talk to each other.

When you create a VPC, you define:
- What IP address range your network uses
- How the network is divided into sections (subnets)
- What can access the internet and what cannot
- What traffic is allowed in and out (security groups, route tables)

---

## IP Address Range — What Is It and Why Does It Matter?

When you create a VPC, the first thing you define is the **IP address range** — written in **CIDR notation** (Classless Inter-Domain Routing).

### What Is an IP Address?

An IP address is a unique number for a device on a network — like a home address for your server. It looks like: `192.168.1.45` — four numbers separated by dots, each between 0 and 255.

### What Is a CIDR Block?

A CIDR block defines a **range of IP addresses** available in your network.

It looks like: `10.0.0.0/16`

The number after the `/` tells you the size — the smaller the number, the more IPs you have.

| CIDR | Total IPs | Typical Use |
|---|---|---|
| `/16` | 65,536 IPs | VPC level — large network |
| `/24` | 256 IPs | Subnet level — medium |
| `/28` | 16 IPs | Very small subnet |

**Example:**
You create a VPC with CIDR `10.0.0.0/16` → 65,536 IPs to work with.

Then you carve it into subnets:
- Public Subnet: `10.0.1.0/24` → 256 IPs
- Private Subnet: `10.0.2.0/24` → 256 IPs

### Why Does It Matter?

- Every resource inside your VPC (EC2, RDS, Lambda) gets an IP from this range
- Resources use these private IPs to talk to each other inside the VPC
- You need a non-overlapping range if you connect two VPCs together or connect to your company's on-premise network

**AWS recommends using private IP ranges (RFC 1918):**
- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

These are reserved for private networks and are NOT routable on the public internet — so no conflict with real internet addresses.

---

## Subnets — Dividing Your VPC Into Sections

### What Is a Subnet?

A **Subnet (Sub-network)** is a smaller, divided section of your VPC.

If your VPC is an apartment building, subnets are individual floors or wings.

You divide your VPC into subnets to:
- Separate different types of resources (web servers vs databases)
- Control which resources can reach the internet and which cannot
- Place resources across multiple Availability Zones for high availability

Each subnet lives in **one specific Availability Zone**.

---

### Public Subnet

A **Public Subnet** is a subnet where resources CAN communicate directly with the internet.

For a subnet to be public, three things are required:
1. It must be connected to an **Internet Gateway** (the door to the internet)
2. Its **Route Table** must have a route sending internet traffic to that Internet Gateway
3. Resources inside it must have a **public IP address**

**What goes in a Public Subnet:**
- Load Balancers (receive traffic from users)
- NAT Gateways (let private resources go outbound)
- Bastion Hosts (a secure SSH jump server)

---

### Private Subnet

A **Private Subnet** is a subnet where resources CANNOT directly reach the internet, and the internet cannot directly reach them.

Resources here only have private IP addresses. They are completely hidden from the outside world.

**What goes in a Private Subnet:**
- Application servers (backend logic, APIs)
- Databases (RDS, Aurora) — your DB should NEVER be exposed to the internet
- Caching servers (ElastiCache)
- Internal microservices

But wait — if a private EC2 can't reach the internet, how does it download software updates? That's where the NAT Gateway comes in (explained below).

---

## Internet Gateway — The Door Between Your VPC and the Internet

### What Is an Internet Gateway?

An **Internet Gateway (IGW)** is the component that allows communication between resources in your VPC and the public internet.

It acts as the main **entrance and exit gate** of your VPC.

Without an Internet Gateway, nothing in your VPC can reach the internet, and the internet cannot reach anything in your VPC.

### How It Works

1. You create an Internet Gateway
2. You attach it to your VPC — one IGW per VPC
3. You update the Route Table of your public subnet to send internet-bound traffic to the IGW
4. EC2 instances in the public subnet with public IPs can now send and receive internet traffic

### Key Points
- One IGW per VPC
- AWS manages it — it's fully redundant and highly available, you don't worry about it failing
- It does bidirectional communication — traffic goes both in and out
- Private subnets have NO route to the IGW — that's what keeps them private

---

## Route Tables — The Traffic Director

### What Is a Route Table?

A **Route Table** is a set of rules (called routes) that determine **where network traffic is directed** inside a VPC.

Think of it as a GPS system for your network. Every time a packet of data needs to go somewhere, the route table says: "For this destination, go THIS way."

### How It Works

Every subnet is associated with a route table. When traffic leaves a resource, the route table checks the destination IP and decides where to send it.

**Example Route Table for a Public Subnet:**

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | local | All VPC-internal traffic stays inside |
| `0.0.0.0/0` | igw-xxxxx | Everything else goes to the Internet Gateway |

`0.0.0.0/0` means "any IP address that isn't local" — basically "send internet traffic to the IGW."

**Example Route Table for a Private Subnet:**

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | local | All VPC-internal traffic stays inside |
| `0.0.0.0/0` | nat-xxxxx | Outbound internet traffic goes to NAT Gateway |

The private subnet's internet route goes to a NAT Gateway — NOT the Internet Gateway directly. This is how private resources can download updates without being exposed.

### Key Points
- Each subnet is associated with one route table
- One route table can be shared by multiple subnets
- The **most specific route wins** — `10.0.0.0/16` beats `0.0.0.0/0` for local traffic
- The route table is what makes a subnet "public" or "private" — it's not a label, it's the routing rule

---

## Security Groups — The Firewall at the Resource Level

### What Is a Security Group?

A Security Group is a **virtual firewall at the instance/resource level** that controls what traffic is allowed in and out.

While the Route Table handles *where* traffic goes, the Security Group handles *whether* that traffic is allowed to enter or leave a specific resource.

Security Groups work on **Allow rules only** — there is no "Deny" rule. If you don't explicitly allow something, it's denied by default.

### Stateful Behavior

Security Groups are **stateful** — if you allow inbound traffic on port 80, the response automatically goes back out without needing a separate outbound rule. The connection is tracked.

### Common Security Group Setup for a Web Server

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Your IP only |
| HTTP | TCP | 80 | Anywhere (0.0.0.0/0) |
| HTTPS | TCP | 443 | Anywhere (0.0.0.0/0) |

**For a database (RDS), the Security Group would say:**
- Allow port 3306 (MySQL) ONLY from the web server's Security Group
- Nothing else — the internet has zero access

### Security Group vs Network ACL (NACL)

These are two different layers of security in a VPC:

| Feature | Security Group | Network ACL |
|---|---|---|
| Applied at | Instance / resource level | Subnet level |
| Stateful or Stateless | Stateful | Stateless |
| Rules | Allow only | Allow and Deny both |
| Default behavior | All inbound denied, all outbound allowed | All inbound and outbound allowed |

Stateless (NACL) means you must explicitly write both inbound AND outbound rules. If you allow inbound on port 80, the response won't automatically go out — you have to allow that too.

**In practice:** Security Groups are your primary tool. NACLs are used for broader subnet-level rules, like blocking a specific IP range from an entire subnet.

---

## Load Balancer and Elastic Load Balancer (ELB)

### What Is a Load Balancer?

A **Load Balancer** sits in front of multiple servers and **distributes incoming traffic across them** so no single server is overwhelmed.

Without a Load Balancer: all users hit one EC2 → it gets overloaded → your app crashes.
With a Load Balancer: traffic is spread across 5 EC2s → all healthy → app runs fine.

It also does **health checks** — if one EC2 becomes unresponsive, the Load Balancer stops sending traffic to it automatically and only sends to healthy instances.

### What Is ELB (Elastic Load Balancer)?

**ELB** is AWS's fully managed Load Balancer service. "Elastic" means it automatically scales its own capacity as traffic grows. AWS handles maintenance, redundancy, and scaling — you just configure it.

### Load Balancer vs ELB — What's the Difference?

A Load Balancer is the general concept — any system that distributes traffic. You could build one yourself with Nginx or HAProxy on an EC2.

ELB is AWS's managed, auto-scaling, highly available version of that concept. The key advantages:
- No single point of failure (AWS runs it across multiple AZs)
- Scales automatically during traffic spikes
- Integrated with Auto Scaling, SSL certificates (ACM), WAF, and more

### Types of ELB

| Type | Layer | Best For |
|---|---|---|
| **ALB** (Application Load Balancer) | Layer 7 — HTTP/HTTPS | Web apps, APIs, microservices — routes based on URL path or hostname |
| **NLB** (Network Load Balancer) | Layer 4 — TCP/UDP | Real-time apps, gaming — ultra-fast, handles millions of requests/sec |
| **GLB** (Gateway Load Balancer) | Layer 3 — IP | Traffic inspection, firewalls, intrusion detection |
| **CLB** (Classic Load Balancer) | Layer 4 & 7 | Old, being deprecated — don't use for new apps |

**ALB Path-Based Routing Example:**
- `myapp.com/api/users` → routes to EC2 running API service
- `myapp.com/images` → routes to EC2 running image service
- `myapp.com/` → routes to EC2 running the frontend

Same domain, different paths, different backend servers. This is the key ALB feature for microservices.

---

## NAT Gateway — Letting Private Subnets Access the Internet Safely

### The Problem

Resources in a private subnet have no public IP. They cannot reach the internet directly. But they still need to:
- Download OS updates (`apt-get`, `yum`)
- Pull Docker images
- Call external third-party APIs

How can a private server reach the internet WITHOUT being reachable FROM the internet?

---

### Why Mask the IP Address? (The Core Idea)

When a private server (IP: `10.0.2.5`) wants to reach Google, it sends a packet. But Google doesn't know how to respond back to `10.0.2.5` — that's a private IP, not routable on the public internet. Google has no idea where `10.0.2.5` is.

So the private server's request needs to go out with a PUBLIC IP attached to it — one that Google can actually respond to.

That's where **masking** comes in. The NAT Gateway **replaces (masks)** the private IP with its own public IP before sending the packet out. Google responds to the NAT Gateway's public IP. The NAT Gateway receives the response, reverses the translation, and delivers it to the private server.

The private server never needs a public IP. The outside world never sees the private server's real IP. They only see the NAT Gateway's public IP. This is both a security measure and a practical networking solution.

---

### What Is NAT (Network Address Translation)?

**NAT** is the process of **rewriting the source IP address of a packet** before it leaves your network.

**NAT Gateway flow:**
1. Private EC2 (`10.0.2.5`) wants to reach Google
2. Request goes to Route Table → Route Table says: send internet traffic to NAT Gateway
3. NAT Gateway replaces source IP `10.0.2.5` with its own public IP (`52.10.20.30`)
4. Request goes through the Internet Gateway to Google
5. Google responds to `52.10.20.30` (the NAT Gateway's public IP)
6. NAT Gateway receives the response, reverses the translation, delivers to `10.0.2.5`

**Why is NAT Gateway One-Way (Inbound Protected)?**
- Private subnet CAN initiate outbound connections (to internet) via NAT Gateway ✅
- Internet CANNOT initiate inbound connections to private subnet via NAT Gateway ❌

This is what makes it safe. You can go out, but nobody can come in.

---

### SNAT — Source Network Address Translation

**SNAT = Source Network Address Translation**

SNAT is the specific technique of **changing the source IP address** of an outgoing packet.

AWS NAT Gateway performs SNAT under the hood. When your private EC2 sends a packet out:
1. NAT Gateway receives it with source IP `10.0.2.5`
2. **Replaces (translates) the source IP** with its own public IP `52.10.20.30`
3. Records this translation in a connection table (so it knows how to reverse it)
4. Forwards the packet to the internet
5. When the response comes back, reverses the translation and delivers to `10.0.2.5`

---

### NAT Gateway vs SNAT — What's the Difference?

| | NAT Gateway | SNAT |
|---|---|---|
| What is it? | AWS managed service | A specific networking technique/mechanism |
| Relationship | NAT Gateway **performs** SNAT | SNAT is the **act** that happens inside NAT Gateway |
| Where used | Inside AWS VPC | Any networking context — Linux, on-prem routers, firewalls |
| Managed by | AWS fully manages it | You configure it manually (e.g., Linux `iptables`) |

**Analogy:** SNAT is like "address replacement" — it's the act. NAT Gateway is the post office where this act happens. AWS NAT Gateway is a fully managed version of that post office.

On-premise or Linux networking: you'd configure SNAT manually using `iptables` rules. On AWS: NAT Gateway handles everything automatically.

**There's also DNAT (Destination NAT):**
- SNAT → changes the **source** IP (outbound — private to internet)
- DNAT → changes the **destination** IP (inbound — port forwarding, exposing internal services)

NAT Gateway in AWS only performs SNAT. For DNAT use cases, you'd use a Load Balancer.

---

## VPC Flow Logs — Monitoring Network Traffic

### What Are VPC Flow Logs?

**VPC Flow Logs** is a feature that **captures metadata about all IP traffic** going through your VPC, subnets, or individual network interfaces.

Think of it as CCTV footage for your network — it records who talked to whom, on which port, at what time, and whether the traffic was accepted or rejected.

It does NOT capture the actual content of traffic — just the metadata (like how a call log shows who you called and for how long, but not what was said).

### What Information Gets Logged?

Each log entry captures:
- Source IP address
- Destination IP address
- Source port and destination port
- Protocol (TCP/UDP)
- Number of packets and bytes
- Start and end time
- **Action: ACCEPT or REJECT**

### Sample Flow Log Entry

```
2 123456789 eni-abc123 10.0.1.5 10.0.2.8 443 49203 6 10 840 ACCEPT OK
```

This reads as: Traffic from `10.0.1.5` to `10.0.2.8` on port 443, TCP, 10 packets, 840 bytes — was **ACCEPTED**.

### Where Are Flow Logs Stored?

- **Amazon CloudWatch Logs** — for real-time monitoring and alerts
- **Amazon S3** — for long-term storage and cost-effective analysis with Athena
- **Amazon Kinesis Data Firehose** — for streaming to third-party SIEM tools

### Why Are Flow Logs Useful?

1. **Security** — Detect suspicious traffic. Thousands of REJECT entries from one IP = someone is attacking you.
2. **Troubleshooting** — App can't connect to the database? Check flow logs to see if traffic is being REJECTED by a security group rule.
3. **Compliance** — Finance and healthcare industries require network traffic logging for audits.
4. **Cost analysis** — See which resources generate the most data transfer (AWS charges for outbound data).

### Flow Logs Can Be Enabled at Three Levels

| Level | Captures |
|---|---|
| VPC level | All traffic in the entire VPC |
| Subnet level | All traffic in a specific subnet |
| ENI (Network Interface) level | Traffic to/from one specific EC2 instance |

---

## Putting It ALL Together — A Real-World Example

Let's say you're building **"ShopKaro"** — an e-commerce platform. Here's how each VPC component plays a role.

### Architecture Setup

- VPC: `10.0.0.0/16` in Mumbai Region (`ap-south-1`)
- Public Subnet: `10.0.1.0/24` in AZ-1a
- Private App Subnet: `10.0.2.0/24` in AZ-1a
- Private DB Subnet: `10.0.3.0/24` in AZ-1b

### Tracing a Customer Request — "Buy Now" Button

**Step 1 — DNS + Internet Gateway**
A customer clicks "Buy Now" on ShopKaro.com. The DNS resolves to the Load Balancer's IP. The request hits the **Internet Gateway** — the only entry point into the VPC from the internet.

**Step 2 — Internet Gateway → Application Load Balancer (Public Subnet)**
The request lands on the ALB, which lives in the public subnet and has a public IP. The ALB's **Security Group** only allows HTTPS (port 443) from anywhere. The ALB checks which EC2 app servers are healthy via health checks.

**Step 3 — ALB → Private App Subnet (App Servers)**
The ALB forwards the "Buy Now" request to one of the healthy EC2 web servers — say `10.0.2.5`. This EC2 has NO public IP. It's completely hidden. The **Route Table** of the private subnet says: local VPC traffic stays local, internet-bound traffic goes to NAT Gateway.

**Step 4 — App Server → Database (Private DB Subnet)**
The web server connects to RDS MySQL at `10.0.3.10` to process the order. The RDS **Security Group** only allows port 3306 from the web server's Security Group — nothing else. The internet has zero access to this database.

**Step 5 — Private EC2 Downloads a Security Patch (Outbound Only)**
The EC2 at `10.0.2.5` sends a request to download an update. The **Route Table** says: internet traffic → NAT Gateway. The NAT Gateway performs **SNAT** — it replaces the source IP `10.0.2.5` with its own public IP `52.10.20.30` — and sends the request through the Internet Gateway to the internet. The update arrives, NAT Gateway delivers it to `10.0.2.5`. At no point was `10.0.2.5` exposed to the internet.

**Step 6 — VPC Flow Logs Records Everything**
Every connection — ALB to EC2, EC2 to RDS, NAT Gateway outbound — is logged. ACCEPT or REJECT, source and destination IPs, timestamps. If something breaks or an attack happens, this is where you look.

### How Each Component Contributed

| Component | Role in ShopKaro |
|---|---|
| VPC | The private network boundary — nobody outside can see inside |
| CIDR Block (`10.0.0.0/16`) | Defines the IP address pool for all resources |
| Public Subnet | Houses the Load Balancer and NAT Gateway — things that need internet access |
| Private App Subnet | Houses EC2 servers — isolated from direct internet access |
| Private DB Subnet | Houses the database — deepest layer, zero internet connectivity |
| Internet Gateway | The only door between VPC and internet — bidirectional |
| Route Table (Public) | Internet traffic → Internet Gateway |
| Route Table (Private) | Internet traffic → NAT Gateway (not directly to IGW) |
| Security Groups | Fine-grained firewall: only allow specific ports from specific sources |
| Application Load Balancer | Distributes traffic across EC2s, health checks, path-based routing |
| NAT Gateway | Lets private EC2s go outbound without exposing their IPs |
| SNAT | The mechanism NAT Gateway uses to replace private IPs with its public IP |
| VPC Flow Logs | Records all traffic for debugging, security auditing, compliance |

---

---

# Interview Questions & Answers

---

**Q1. What is a VPC and why was it introduced?**

VPC stands for Virtual Private Cloud. It's your own logically isolated private network inside AWS. Before VPC, AWS used EC2-Classic where all customer instances shared one flat network — there was no isolation between one company's servers and another's. This was a security and compliance problem. Companies in banking, healthcare, and government couldn't use AWS. VPC was introduced to let customers create their own isolated network with full control over IP ranges, subnets, routing, and firewall rules. AWS made it the default in late 2013.

---

**Q2. What is a CIDR block and why do you define it when creating a VPC?**

CIDR stands for Classless Inter-Domain Routing. A CIDR block like `10.0.0.0/16` defines the range of private IP addresses available inside your VPC. The `/16` tells you the size — a `/16` gives 65,536 IPs. Every resource in the VPC gets a private IP from this pool. You define it upfront because it determines how many resources you can run and how you'll divide the network into subnets. AWS recommends using private IP ranges that aren't routable on the public internet to avoid any conflicts.

---

**Q3. What is the difference between a Public and a Private Subnet?**

A public subnet is one where resources can communicate directly with the internet. This requires the subnet's route table to have a route pointing to an Internet Gateway, and resources inside must have public IPs.

A private subnet has no direct route to the internet. Resources only have private IPs and are hidden from the outside world. If a private resource needs outbound internet access — like downloading updates — it goes through a NAT Gateway, not the Internet Gateway directly. Private subnets are used for databases and app servers — anything that shouldn't be publicly accessible.

---

**Q4. What is an Internet Gateway and what does it do?**

An Internet Gateway is the component that connects a VPC to the public internet. It enables two-way communication — resources in your public subnet can reach the internet, and internet traffic can reach those resources. Without an IGW, the VPC is completely isolated from the internet. You can only attach one Internet Gateway to a VPC at a time. AWS manages it as a fully scalable and highly available service — it won't become a bottleneck.

---

**Q5. What is a Route Table in VPC?**

A Route Table is a set of rules that tells network traffic where to go. Every subnet is associated with a route table. When traffic leaves a resource, the route table checks the destination IP and decides the next hop. For a public subnet, the route for `0.0.0.0/0` points to the Internet Gateway. For a private subnet, that same route points to a NAT Gateway. Internal VPC traffic is always handled by the local route. The route table is actually what defines whether a subnet is "public" or "private" — not just a label.

---

**Q6. What is a Security Group and how does it work?**

A Security Group is a stateful firewall at the resource level. It controls inbound and outbound traffic for a specific EC2 instance or other resource using Allow rules only — there are no Deny rules. If something isn't explicitly allowed, it's denied by default. Stateful means if you allow inbound traffic, the response is automatically allowed back out without a separate rule. Security Groups are the primary layer of access control in AWS. You always attach at least one Security Group to any EC2 instance.

---

**Q7. What is the difference between a Security Group and a Network ACL?**

A Security Group operates at the resource level and is stateful — responses to allowed inbound traffic automatically go out. It only has Allow rules.

A Network ACL operates at the subnet level and is stateless — you must explicitly define both inbound and outbound rules. It supports both Allow and Deny rules, which is useful for blocking specific IPs at the subnet level.

In most architectures, Security Groups do the heavy lifting. NACLs are used for broader, coarser controls like blocking a malicious IP range from entering a whole subnet.

---

**Q8. What is a NAT Gateway and why is it needed?**

NAT stands for Network Address Translation. A NAT Gateway is placed in the public subnet and allows resources in private subnets to initiate outbound internet connections — like downloading OS updates — without exposing their private IPs to the internet and without being reachable from the internet.

It works by replacing the source IP of outgoing packets with its own public IP — this is called SNAT. The outside world only ever sees the NAT Gateway's IP. When the response comes back, NAT Gateway translates it and delivers it to the private resource. It's strictly one-directional for inbound — nothing from the internet can initiate a connection through NAT Gateway into the private subnet.

---

**Q9. What is SNAT and how does it differ from NAT Gateway?**

SNAT — Source Network Address Translation — is the specific technique of replacing the source IP address of an outgoing packet with a different IP. It's a fundamental networking concept that exists in any network environment — Linux servers, on-premise routers, firewalls.

NAT Gateway is AWS's fully managed service that implements SNAT automatically for your VPC. So SNAT is the mechanism or technique, and NAT Gateway is the AWS service that performs that technique on your behalf. On Linux, you'd configure SNAT manually using iptables rules. On AWS, the NAT Gateway handles all of this without you touching anything.

---

**Q10. What is the difference between an Internet Gateway and a NAT Gateway?**

Internet Gateway enables two-way communication — resources in the public subnet use it for both inbound and outbound traffic. The internet can initiate connections to those resources.

NAT Gateway enables one-way outbound communication only — private resources can go out to the internet through it, but nothing from the internet can come in through it. NAT Gateway itself sits in the public subnet and uses the Internet Gateway for its own outbound access. They work together — NAT Gateway routes through the Internet Gateway to reach the internet.

---

**Q11. What is the difference between a Load Balancer and ELB?**

A Load Balancer is a general concept — any system that distributes traffic across multiple servers. You can build one yourself using Nginx or HAProxy.

ELB — Elastic Load Balancer — is AWS's fully managed, auto-scaling implementation of that concept. AWS handles its availability, patching, and scaling. It integrates natively with Auto Scaling, ACM for SSL, WAF for security, and CloudWatch for monitoring. The "Elastic" in ELB means it scales its own capacity automatically — it won't be a bottleneck as traffic grows.

---

**Q12. What is the difference between an ALB and an NLB?**

An Application Load Balancer works at Layer 7 — the HTTP/HTTPS level. It can inspect request content and route traffic based on URL paths, hostnames, or headers. It's used for web apps and microservices where different endpoints go to different backend services.

A Network Load Balancer works at Layer 4 — the TCP/UDP level. It doesn't inspect content; it routes based on IP and port only. It's extremely fast — capable of handling millions of requests per second with very low latency. It's used for real-time applications, gaming servers, or anything requiring ultra-high performance where even a few milliseconds of extra latency matters.

---

**Q13. What is VPC Flow Logs and when would you use it?**

VPC Flow Logs is an AWS feature that captures metadata about IP traffic flowing through your VPC, subnets, or individual network interfaces. It records source and destination IPs, ports, protocol, whether traffic was accepted or rejected, and packet counts. It doesn't capture the actual content of traffic — just the metadata.

You'd use it for security analysis (spotting thousands of REJECT entries from one IP might indicate an attack), network troubleshooting (figuring out why a connection is being blocked — the logs will show a REJECT entry with the reason), and compliance auditing where regulated industries need to prove they're monitoring network traffic. Logs go to CloudWatch or S3.

---

**Q14. What is Availability Zone and why does it matter in a VPC?**

An Availability Zone is a physically separate data center within a region, with its own power, cooling, and networking — isolated from other AZs but connected via high-speed private links. If one AZ goes down, others are unaffected.

In a VPC, subnets are created within specific AZs. For high availability, you create subnets in at least two AZs and deploy your resources across them. If one AZ has an outage, your application keeps running in the other. A Load Balancer automatically routes traffic to the healthy AZ. This is the standard pattern for production workloads.

---

**Q15. Walk me through how a user request reaches a database in a properly designed VPC.**

The user's request hits the Internet Gateway — the entry point into the VPC. The IGW forwards it to the Application Load Balancer in the public subnet. The ALB's Security Group allows only HTTPS from anywhere. The ALB performs health checks and routes the request to a healthy EC2 in the private app subnet. The private EC2's Security Group only allows traffic from the ALB's Security Group on the app port.

The app server then connects to the database in the private DB subnet. The database's Security Group only allows port 3306 from the app server's Security Group — nothing else. The internet has no path to this database at all. Every connection throughout this entire flow is recorded in VPC Flow Logs as ACCEPT entries.

---

*📌 VPC is one of the most important AWS topics for DevOps and Cloud interviews. The end-to-end flow question — tracing a request from user to database through all VPC components — is extremely common. If you can describe that flow confidently, mentioning each component and why it exists, you've covered most of what interviewers ask about VPC.*
