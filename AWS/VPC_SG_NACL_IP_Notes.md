[AWS_VPC_SG_NACL_IP_Notes.md](https://github.com/user-attachments/files/29506125/AWS_VPC_SG_NACL_IP_Notes.md)
# AWS VPC — Security Groups, NACLs, IP Addressing & Console Guide

---

## Security Groups and NACLs — The Two Layers of Security in AWS

AWS gives you **two different layers of firewall protection** inside a VPC. They operate at different levels and serve different purposes — but together they are the final barriers that decide whether any network traffic reaches your resources or not.

- **Security Group** → Firewall at the **EC2 instance (resource) level**
- **NACL** → Firewall at the **Subnet level**

Think of it this way:

> NACL is the security guard at the entrance of your apartment building floor.
> Security Group is the lock on your individual apartment door.
> Traffic has to pass BOTH to reach your resource.

---

## Security Groups — Deep Dive

### What Is a Security Group?

A Security Group is a **virtual firewall attached directly to an EC2 instance** (or RDS, Lambda, etc.). It controls what traffic is allowed in (inbound) and what is allowed out (outbound) at the instance level.

You can think of it as a personal bouncer standing right outside your EC2. Every incoming and outgoing connection has to pass through this bouncer.

### Key Characteristics

**1. Allow Rules Only — No Deny**
Security Groups can only have ALLOW rules. You can never write a rule that says "block this IP." If you don't write an allow rule for something, it is automatically denied. Deny is the default state — silence = denial.

**2. Stateful**
This is the most important characteristic of Security Groups.

Stateful means: **if you allow an inbound connection, the response is automatically allowed back out — even if you have no explicit outbound rule for it.**

AWS tracks the connection state. It knows that a response to an allowed inbound request is legitimate traffic. You don't need to manage both sides.

Example: You allow inbound HTTP on port 80. A user's browser sends a request. AWS lets it in. The server sends a response back to the user's browser. AWS automatically allows that response out — even if you have no outbound rule at all. The connection state tracks it.

**3. Applied at the Instance Level**
Each EC2 instance gets one or more Security Groups assigned to it. Multiple EC2s can share the same Security Group. If you update a Security Group's rules, the change applies immediately to all instances using it.

---

### Inbound Traffic in Security Groups

**Inbound rules** control what traffic can COME IN to your EC2 instance.

By default (for a new custom Security Group):
- All inbound traffic is **DENIED** — nothing can reach your EC2 unless you explicitly allow it.

Common inbound rules you'd add:
| Port | Protocol | What It Allows |
|---|---|---|
| 22 | TCP | SSH — connect to your Linux EC2 remotely |
| 80 | TCP | HTTP — web traffic |
| 443 | TCP | HTTPS — encrypted web traffic |
| 3306 | TCP | MySQL database connections |
| 5432 | TCP | PostgreSQL database connections |
| 3000 | TCP | Node.js app (custom port) |

**Best practice for inbound:** Never open a port to `0.0.0.0/0` unless it absolutely needs to be public. For SSH (port 22), only allow YOUR specific IP address.

---

### Outbound Traffic in Security Groups

**Outbound rules** control what traffic can LEAVE your EC2 instance.

By default (for a new custom Security Group):
- All outbound traffic is **ALLOWED** — your EC2 can reach anywhere.

This default outbound rule looks like:
| Port | Protocol | Destination |
|---|---|---|
| All | All | 0.0.0.0/0 |

This means your EC2 can make any outbound connection by default — call any API, download from any server, etc.

**Exception — Port 25 (SMTP/Email) is always blocked outbound:**

AWS blocks outbound port 25 by default, even if you try to add an allow rule for it. This is a hard block at the infrastructure level.

**Why?**
Port 25 is used for sending emails. Spammers frequently abuse cloud servers to send millions of spam emails. AWS pre-emptively blocks it across all accounts to prevent this. If you legitimately need to send emails, AWS wants you to use **Amazon SES (Simple Email Service)** instead — which has its own spam controls built in. You can request AWS to lift the port 25 block, but it requires submitting a support request and justifying the need.

So to summarize the default outbound state:
- Port 25 outbound → **BLOCKED** (AWS infrastructure level, not Security Group)
- All other ports outbound → **ALLOWED** (default Security Group rule)
- All inbound → **NOT ALLOWED** (default for new Security Groups)

---

### Default Security Group in AWS

When you create a VPC, AWS automatically creates a **Default Security Group** for it. This is different from a custom Security Group you create yourself.

**Default Security Group rules:**

| Direction | Rule | What It Means |
|---|---|---|
| Inbound | Allow all traffic from the same Security Group | EC2 instances assigned to the same default SG can communicate freely with each other |
| Outbound | Allow all traffic to anywhere | All outbound traffic is allowed |

The key point: The default SG allows instances within the same SG to talk to each other, but blocks all external inbound traffic. This is designed so that when you launch an EC2 without thinking about Security Groups, it doesn't become completely open to the internet — it's still somewhat protected.

**Best practice:** Don't use the default Security Group for production. Create custom Security Groups with specific, minimal rules. This way you always know exactly what is allowed and what isn't.

---

### Security Group — Referencing Other Security Groups as Source

This is a powerful and often underused feature.

Instead of allowing traffic from a specific IP address in an inbound rule, you can allow traffic **from another Security Group**.

Example:
- Your database EC2 has a Security Group called `sg-database`
- Instead of allowing MySQL (port 3306) from `10.0.2.0/24` (the subnet), you allow it from `sg-appserver`
- Now ONLY EC2 instances that have `sg-appserver` attached can access the database
- If an attacker somehow gets into a different EC2 in the same subnet, they still can't reach the database

This is more secure than IP-based rules because it's identity-based, not network-location-based.

---

## NACL — Network Access Control List

### What Is a NACL?

A **NACL (Network Access Control List)** is a firewall that operates at the **subnet level** — it controls what traffic can enter or leave an entire subnet, regardless of what's inside the subnet.

Every subnet in AWS is automatically associated with a NACL. If you don't create a custom NACL, the subnet uses the **Default NACL** (which allows everything).

### Key Characteristics

**1. Both Allow and Deny Rules**
Unlike Security Groups, NACLs support both ALLOW and DENY rules. You can explicitly block specific IPs or IP ranges.

**2. Stateless**
This is the most critical difference from Security Groups.

Stateless means: **AWS does NOT track connection state. Every packet is evaluated independently against the rules — both the request AND the response.**

If you allow inbound HTTP on port 80, you also MUST explicitly allow the outbound response. If you don't add the outbound rule, the response will be blocked — even though the inbound request was allowed.

**Why does this matter?**
HTTP responses don't come back on port 80. When a client makes a request, the server sends the response to the client's **ephemeral port** — a random high-numbered port (typically 1024–65535). You must allow outbound traffic on these ephemeral ports in your NACL, or responses will be blocked.

**3. Rules Are Evaluated in Order (by Rule Number)**
Rules have a number (e.g., 100, 200, 300). AWS evaluates them from lowest to highest. The first rule that matches the traffic is applied, and AWS stops checking further rules.

This means rule ordering matters. If rule 100 allows all traffic and rule 200 denies a specific IP, the deny never executes — rule 100 matched first.

**Convention:** Use increments of 100 (100, 200, 300...) to leave room to insert rules later.

**The * rule (implicit deny):** At the bottom, there is always a `*` rule that denies all traffic that didn't match any other rule. You can't delete or change this rule.

**4. Applied at Subnet Level**
One NACL can be associated with multiple subnets, but one subnet can only have one NACL at a time.

---

### Default NACL vs Custom NACL

| | Default NACL | Custom NACL |
|---|---|---|
| Inbound rules | Allow ALL traffic | Deny ALL traffic by default |
| Outbound rules | Allow ALL traffic | Deny ALL traffic by default |
| When used | Automatically assigned to new subnets | You create and explicitly assign it |

When you create a custom NACL and assign it to a subnet, ALL traffic is blocked by default until you add allow rules. This is the opposite of the Default NACL.

---

### Why Do We Need NACL If We Already Have Security Groups?

This is a very common interview question. Here's the real reason:

**1. NACLs Can DENY — Security Groups Cannot**
Security Groups can only allow. If you want to explicitly BLOCK a specific IP address or range from even reaching your subnet — say, blocking a known attacker's IP — you cannot do that with a Security Group.

With a NACL, you add a DENY rule for that IP with a low rule number (say 50), and it gets blocked before anything else is evaluated. The traffic never even reaches the EC2 instance's Security Group.

**2. NACLs Are the Subnet-Level Defense**
Security Groups are attached to individual resources. If you have 50 EC2 instances in a subnet and need to block an IP from reaching all of them, you'd need to update 50 Security Groups. With NACL, you update one rule and it applies to the entire subnet instantly.

**3. Defense in Depth**
Security is never a single layer. If someone bypasses or misconfigures a Security Group, the NACL is still there as an additional barrier. Two independent layers of security are always better than one.

**4. NACLs Are Stateless — Different Use Case**
Because NACLs evaluate every packet independently, they're better for certain traffic inspection scenarios where you want to analyze each packet without state tracking.

**One-line summary for interviews:**
> "Security Groups allow traffic — NACLs also deny it. If you need to block a specific IP, Security Group can't do it but NACL can. Together they give you defense in depth."

---

### NACL vs Security Group — Full Comparison

| Feature | Security Group | NACL |
|---|---|---|
| Applied at | Instance / resource level | Subnet level |
| Stateful or Stateless | **Stateful** | **Stateless** |
| Rules | Allow ONLY | Allow AND Deny |
| Rule evaluation | All rules evaluated together | Evaluated in order by rule number — first match wins |
| Default (new custom) | All inbound denied, all outbound allowed | All inbound denied, all outbound denied |
| Default (AWS default) | Same-SG inbound allowed, all outbound allowed | All inbound allowed, all outbound allowed |
| Response traffic | Automatically allowed (stateful) | Must explicitly allow with outbound rule (stateless) |
| Blocking specific IPs | Cannot | Can — using Deny rules |
| Number of resources | One SG → many instances. One instance → up to 5 SGs | One NACL → many subnets. One subnet → one NACL |

---

## Understanding the AWS Console Image — Three Things Explained

The image shows the VPC creation screen in the AWS Console. It has three settings: **IPv4 CIDR Block**, **IPv6 CIDR Block**, and **Tenancy**.

---

### 1. IPv4 CIDR Block — `10.0.0.0/16` → 65,536 IPs

This is where you define the IP address range for your entire VPC.

**What Is IPv4?**
IPv4 (Internet Protocol version 4) uses 32-bit addresses, written as four numbers separated by dots: `10.0.0.0`. Each section (called an octet) is 8 bits, ranging from 0 to 255.

**How Is the IP Range Calculated?**

The CIDR notation `10.0.0.0/16` works like this:

An IPv4 address has **32 bits total**.

The number after the `/` (the prefix length) tells you how many bits are **fixed** (the network part). The remaining bits are **free** (the host part — for assigning to individual resources).

```
10.0.0.0/16

32 total bits
- 16 bits are fixed (the "10.0" part is locked — it defines your network)
- 16 bits are free (the ".0.0" part — for individual resource IPs)

Free bits = 32 - 16 = 16
Total IPs = 2^16 = 65,536
```

More examples:

| CIDR | Fixed Bits | Free Bits | Total IPs | Calculation |
|---|---|---|---|---|
| /8 | 8 | 24 | 16,777,216 | 2^24 |
| /16 | 16 | 16 | 65,536 | 2^16 |
| /24 | 24 | 8 | 256 | 2^8 |
| /28 | 28 | 4 | 16 | 2^4 |

**AWS Reserves 5 IPs from Every Subnet:**
From every subnet you create, AWS automatically reserves 5 IP addresses for its own use:
- First IP (e.g., `10.0.1.0`) → Network address — identifies the subnet itself
- Second IP (`10.0.1.1`) → AWS VPC router
- Third IP (`10.0.1.2`) → AWS DNS server
- Fourth IP (`10.0.1.3`) → Reserved by AWS for future use
- Last IP (`10.0.1.255`) → Broadcast address

So for a `/24` subnet with 256 IPs → you only get **251 usable IPs** (256 - 5 = 251).

**What CIDR to Choose for a VPC?**
AWS recommends `/16` for a VPC (65,536 IPs) — it gives you plenty of room to create subnets. The subnets then use smaller ranges like `/24` or `/28`.

**Private IP Ranges (RFC 1918):**
These are reserved ranges for private networks — they're not routable on the public internet:
- `10.0.0.0/8` → 16.7 million IPs (most commonly used in VPCs)
- `172.16.0.0/12` → 1 million IPs
- `192.168.0.0/16` → 65,536 IPs (commonly used in home routers)

---

### 2. IPv4 vs IPv6 — What's the Difference?

**Why Was IPv6 Even Created?**

IPv4 has 32 bits → 2^32 = only ~4.3 billion possible addresses. As of the early 2010s, we started running out. Every phone, laptop, IoT device, server, and smart TV needs an IP address. 4.3 billion isn't enough for the modern world.

IPv6 was created to solve this. It uses **128 bits** → 2^128 = 340 undecillion addresses (that's 340 followed by 36 zeros). Essentially unlimited.

**Format Differences:**

| | IPv4 | IPv6 |
|---|---|---|
| Bits | 32 bits | 128 bits |
| Total addresses | ~4.3 billion | ~340 undecillion |
| Format | Decimal, dot-separated: `192.168.1.1` | Hexadecimal, colon-separated: `2001:0db8:85a3:0000:0000:8a2e:0370:7334` |
| Private ranges | Yes (RFC 1918) | No — all addresses are globally unique |
| NAT required | Yes (due to address shortage) | No — every device can have a globally unique IP |
| Header size | 20 bytes | 40 bytes (but simpler — no checksum) |

**IPv6 Shorthand Rules:**
- Leading zeros in each group can be removed: `0db8` → `db8`
- One consecutive group of all-zero blocks can be replaced with `::`: `2001:db8::8a2e:370:7334`

**In AWS VPC:**
- IPv4 is mandatory for every VPC
- IPv6 is optional — you can add it, but it's not required
- IPv6 addresses in AWS are always **public** — there are no private IPv6 ranges
- If you use IPv6 inside your VPC, all resources with IPv6 addresses can potentially communicate directly with the internet (no NAT needed — though you still control this with Security Groups and NACLs)
- AWS provides IPv6 blocks in `/56` size for VPCs and `/64` for subnets

**In the console image:** The two options are:
- "No IPv6 CIDR block" → your VPC is IPv4-only (most common for beginners and internal apps)
- "Amazon-provided IPv6 CIDR block" → AWS assigns you an IPv6 range from their pool

For most learning and DevOps work, you use IPv4 only. IPv6 becomes relevant for compliance, ISPs, or when building services that need to be natively accessible to IPv6-only clients.

---

### 3. Tenancy — Default vs Dedicated

**Tenancy** controls whether your EC2 instances share physical hardware with other AWS customers.

**Default Tenancy:**
Your instances run on **shared physical hardware** — multiple AWS customers' virtual machines are co-located on the same physical server (isolated by the hypervisor, but sharing the same machine).

This is the standard option. It's cheaper and you don't need to worry about it — the hypervisor ensures complete isolation between customers.

**Dedicated Tenancy:**
Your instances run on **physical hardware reserved exclusively for your account** — no other customer's VMs are on the same physical machine.

This costs significantly more. It's used for:
- **Compliance requirements** — some regulations (HIPAA, PCI-DSS) require dedicated hardware
- **Software licensing** — some enterprise software licenses are per physical core, and shared tenancy makes this complicated
- **Security-sensitive workloads** — maximum isolation from other customers

**Dedicated Instance vs Dedicated Host:**
- **Dedicated Instance** → runs on dedicated hardware, but you don't control which specific physical machine
- **Dedicated Host** → gives you visibility and control over the specific physical server (you can see the socket and core count) — useful for complex licensing

**Important:** If you set VPC-level tenancy to "Dedicated," ALL instances launched in that VPC will automatically be Dedicated instances — even if you don't specify it at launch. You can't mix Default and Dedicated within a Dedicated-tenancy VPC.

For 99% of use cases including DevOps work: **always choose Default**.

---

## Important VPC Things to Know in the AWS Console

### Things You See When Creating a VPC

When you go to AWS Console → VPC → Create VPC, you'll see:

**VPC Settings:**
- **Name tag** → Just a label. Name it clearly (e.g., `prod-vpc`, `dev-vpc`)
- **IPv4 CIDR** → The IP range. Use `10.0.0.0/16` for most cases
- **IPv6 CIDR** → Leave as "No IPv6" unless you specifically need it
- **Tenancy** → Leave as Default

**"VPC and more" option:** AWS now offers a wizard that creates subnets, route tables, and an Internet Gateway for you automatically. Useful for quick setup, but understand what it creates.

---

### Things in the VPC Dashboard You Must Know

**1. Your VPCs**
Shows all VPCs in the current region. Each VPC has:
- VPC ID (e.g., `vpc-0a1b2c3d`)
- IPv4 CIDR
- State (Available/Pending)
- Default VPC — AWS creates one default VPC per region when you make an account. Never delete the default VPC unless you know exactly what you're doing.

**2. Subnets**
Each subnet shows:
- Subnet ID
- VPC it belongs to
- Availability Zone
- IPv4 CIDR
- Available IPs (total minus AWS's 5 reserved)
- "Auto-assign public IPv4 address" setting → if enabled, EC2 instances launched in this subnet automatically get a public IP. Turn this ON for public subnets, keep it OFF for private subnets.

**3. Route Tables**
Each route table shows:
- Routes (destination → target)
- Which subnets are associated with it
- Every VPC has a main route table by default. When you create a subnet and don't assign it a route table, it uses the main route table.

**Important:** The main route table should NOT have a route to the Internet Gateway. Keep the main route table as a "private by default" route table. Create a separate route table for public subnets.

**4. Internet Gateways**
Shows the IGW and which VPC it's attached to. State should be "Attached." An unattached IGW does nothing.

**5. NAT Gateways**
Shows:
- Which subnet it lives in (should always be a PUBLIC subnet)
- Elastic IP associated with it
- State (Available/Pending/Deleting)

**Important cost note:** NAT Gateways cost money — both per hour and per GB of data processed. Delete NAT Gateways when not in use (e.g., in dev environments). Leaving a NAT Gateway running when you don't need it is a common source of unexpected AWS bills.

**6. Security Groups**
Shows all Security Groups, their associated VPC, and their rules.

**Important:** Security Groups are VPC-specific. A Security Group created in VPC-A cannot be used in VPC-B.

**7. Network ACLs**
Shows all NACLs, which VPC they belong to, and which subnets are associated. Always check the rule numbers and ordering when troubleshooting connectivity issues.

**8. Elastic IPs**
Static public IPs you've allocated. If an Elastic IP is not associated with a running instance, **you're charged for it**. Always release unneeded Elastic IPs.

**9. VPC Peering**
Allows two VPCs to communicate privately — even across different AWS accounts or regions. Traffic travels through AWS's private network, not the internet.

For peering to work:
- CIDR ranges of the two VPCs must NOT overlap
- You must update route tables in BOTH VPCs to route traffic through the peering connection
- Security Groups in both VPCs must allow the traffic

**10. VPC Endpoints**
Allows EC2 instances in a private subnet to access AWS services (like S3, DynamoDB) WITHOUT going through the internet or NAT Gateway.

Two types:
- **Gateway Endpoint** → for S3 and DynamoDB. Free. Added as a route in your route table.
- **Interface Endpoint** → for other AWS services (SQS, SNS, ECR, etc.). Uses a private IP inside your VPC. Costs money.

Use case: An EC2 in a private subnet needs to upload files to S3. Instead of routing through the NAT Gateway (costs money per GB), use a VPC Gateway Endpoint — traffic stays within AWS's private network and is free.

**11. VPC Flow Logs**
Found under the VPC's details → Flow Logs tab. You can enable flow logs at VPC, subnet, or ENI level. Destination is CloudWatch Logs or S3.

**12. DHCP Option Sets**
Controls DNS settings for your VPC — which DNS servers your instances use, domain name, etc. Most of the time, leave this as the AWS default which points to the Amazon DNS server (`169.254.169.253`).

---

### Other Important VPC Concepts You Must Know

**Bastion Host / Jump Server**
A small EC2 instance in the public subnet that you SSH into first, and then SSH into private EC2 instances from there. The private instances only allow SSH from the bastion's Security Group.

Modern alternative: **AWS Systems Manager Session Manager** — connect to private EC2 without port 22 being open at all. No bastion needed. Uses AWS's private network.

**VPC CIDR Cannot Be Changed After Creation**
Once you create a VPC with `10.0.0.0/16`, you cannot change that CIDR block. You can add secondary CIDR blocks (up to 4 additional ones), but you can't replace the primary one. Plan your IP range carefully before creating.

**Subnet CIDR Must Be a Subset of VPC CIDR**
If your VPC is `10.0.0.0/16`, your subnets must be within that range — e.g., `10.0.1.0/24`, `10.0.2.0/24`, etc. You cannot use `192.168.1.0/24` in a VPC with `10.0.0.0/16`.

**Subnets Cannot Overlap**
Two subnets in the same VPC cannot have overlapping IP ranges. `10.0.1.0/24` and `10.0.1.128/25` would overlap — AWS won't let you create this.

**One Internet Gateway per VPC**
You cannot attach two Internet Gateways to one VPC.

**A Resource Can Have Multiple Security Groups**
An EC2 instance can have up to 5 Security Groups. The rules are combined — all Allow rules from all attached SGs are merged. If any SG allows a connection, it's allowed (there's no conflict between SGs on the same instance).

**ENI — Elastic Network Interface**
Every EC2 instance has at least one ENI — it's the virtual network card. The ENI has the private IP, public IP, MAC address, and Security Groups attached to it. You can move an ENI from one EC2 to another (useful for failover scenarios).

**Link-Local Address — 169.254.169.254**
This is a special IP that every EC2 instance can reach to query **instance metadata** — information about itself. Not a real network IP; it's always available locally.

Examples of what you can get:
```
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/public-ipv4
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

This endpoint is important for security — attackers who compromise an EC2 can use it to steal IAM role credentials. AWS now requires IMDSv2 (a more secure version) to prevent this.

**VPC Limits (Default Soft Limits)**
- 5 VPCs per region
- 200 subnets per VPC
- 500 Security Groups per VPC
- 60 inbound + 60 outbound rules per Security Group

These can be increased by requesting an AWS service quota increase.

---

---

# Interview Questions & Answers

---

**Q1. What is the difference between a Security Group and a NACL?**

A Security Group operates at the individual resource (EC2) level and is stateful — meaning if you allow inbound traffic, the response automatically goes out without any additional rule. It supports Allow rules only.

A NACL operates at the subnet level and is stateless — every packet, including responses, is evaluated independently against the rules. You must explicitly allow both directions. NACLs support both Allow and Deny rules. You use NACLs when you need to block a specific IP or range from an entire subnet — something a Security Group can't do.

---

**Q2. Why do we need NACL if Security Groups already exist?**

Security Groups can only allow traffic — they cannot deny. If you want to block a known malicious IP address from reaching any resource in your subnet, Security Group can't do it. NACL can. You add a Deny rule with a low rule number and that IP is blocked before it even gets to the EC2's Security Group.

NACL also operates at the subnet level — one rule affects all resources in that subnet. If you need to apply the same restriction to 50 EC2 instances, one NACL rule does it instead of updating 50 Security Groups. They work together as defense in depth — if one layer is misconfigured, the other is still there.

---

**Q3. What happens if NACL allows traffic but the Security Group does not?**

The traffic is blocked. Both layers must allow the traffic for it to reach the resource. NACL is evaluated first at the subnet boundary. If it allows the traffic, it then reaches the EC2's Security Group. If the Security Group denies it (or has no allow rule), the traffic is dropped. Both must say yes.

---

**Q4. What is the Default Security Group in AWS and what are its rules?**

Every VPC comes with a Default Security Group. It has two rules: inbound allows all traffic from instances that are also assigned to the same Default Security Group (so instances in the same SG can talk to each other freely), and outbound allows all traffic to anywhere. It blocks all inbound traffic from outside. Best practice is to never use the Default Security Group in production — create custom Security Groups with specific, minimal rules so you always know exactly what is allowed.

---

**Q5. Why is port 25 blocked on AWS EC2 by default?**

Port 25 is used for SMTP — sending emails directly. AWS blocks it by default at the infrastructure level to prevent cloud servers from being used to send spam. Spammers commonly abuse cheap cloud servers for mass email campaigns. If you legitimately need to send emails, AWS wants you to use Amazon SES which has spam controls, verified domains, and bounce management. You can request AWS support to unblock port 25, but it requires justification.

---

**Q6. What does CIDR notation mean? How do you calculate the number of IPs in 10.0.0.0/16?**

CIDR stands for Classless Inter-Domain Routing. The `/16` is the prefix length — it tells you how many bits of the 32-bit IP address are fixed (the network part). The remaining bits are free to assign to resources.

For `/16`: 32 - 16 = 16 free bits. 2^16 = 65,536 total IP addresses in the range.

AWS also reserves 5 IPs from each subnet — the first four and the last one — for network address, router, DNS, future use, and broadcast. So a `/24` subnet gives 256 IPs total minus 5 = 251 usable IPs.

---

**Q7. What is the difference between IPv4 and IPv6?**

IPv4 uses 32 bits, giving about 4.3 billion addresses, written in decimal dot notation like `192.168.1.1`. IPv6 uses 128 bits, giving 340 undecillion addresses, written in hexadecimal colon notation like `2001:db8::1`. IPv6 was created because the world is running out of IPv4 addresses.

In AWS, IPv4 is mandatory for every VPC. IPv6 is optional and all IPv6 addresses in AWS are globally routable — there are no private IPv6 ranges. IPv6 also eliminates the need for NAT since every device can have a unique global address.

---

**Q8. What is Tenancy in VPC and when would you use Dedicated?**

Tenancy controls whether EC2 instances share physical hardware with other AWS customers. Default tenancy means your VMs are co-located on shared physical servers (isolated by the hypervisor). Dedicated tenancy means your instances run on hardware reserved exclusively for your account.

You'd use Dedicated for compliance requirements like HIPAA or PCI-DSS that mandate physical isolation, or for software with per-core licensing that doesn't work well on shared hardware. It costs significantly more. For most DevOps and application workloads, Default tenancy is the right choice.

---

**Q9. Can you change the CIDR block of a VPC after it's created?**

No, you cannot replace the primary CIDR block of a VPC once it's created. This is why planning the IP range upfront is important. However, you can ADD secondary CIDR blocks to expand the address space (up to 4 additional blocks). This is why a common recommendation is to use `/16` for a VPC — it gives enough IPs to create many subnets without running out.

---

**Q10. What is a NACL rule number and why does it matter?**

NACL rules are evaluated in ascending numerical order — lowest number first. The first rule that matches incoming traffic is applied, and AWS stops evaluating further rules. So if rule 100 allows all traffic and rule 200 denies a specific IP, the deny never runs because rule 100 already matched.

This ordering is critical for security. If you want to block an IP, the deny rule must have a LOWER number than any allow rule that would otherwise permit that traffic. Convention is to use multiples of 100 (100, 200, 300...) to leave gaps for inserting rules later. There's also a default asterisk (*) rule at the bottom that denies all unmatched traffic — you can't delete it.

---

**Q11. What is the difference between a stateful and a stateless firewall in the context of AWS?**

A stateful firewall (Security Group) tracks the state of network connections. If you allow an inbound connection, it automatically allows the return traffic out without needing an explicit outbound rule. The firewall knows the response belongs to the original request.

A stateless firewall (NACL) treats every packet independently without any memory of previous packets. If you allow inbound HTTP on port 80, you must ALSO explicitly allow outbound traffic on ephemeral ports (1024-65535) so the HTTP response can leave. If you forget the outbound rule, the response is dropped. This is a common mistake when configuring NACLs.

---

**Q12. What is VPC Peering and what are its limitations?**

VPC Peering creates a private network connection between two VPCs so resources in each can communicate directly using private IPs without going through the internet. It works across different AWS accounts and regions.

The key limitation is that there is no transitive peering. If VPC-A is peered with VPC-B, and VPC-B is peered with VPC-C, VPC-A cannot communicate with VPC-C through VPC-B. Each peering connection is direct and independent. Also, the CIDR blocks of peered VPCs must not overlap — if both use `10.0.0.0/16`, peering won't work.

---

**Q13. What is a VPC Endpoint and why would you use it?**

A VPC Endpoint lets resources in a private subnet access AWS services like S3 or DynamoDB without going through the internet or NAT Gateway. Traffic stays entirely within AWS's private network.

You'd use it for cost and security. Without an endpoint, your private EC2 must go through a NAT Gateway to reach S3, and NAT Gateway charges per GB of data processed. With a Gateway Endpoint for S3, traffic bypasses the NAT Gateway entirely and is free. It's also more secure since the traffic never leaves AWS's network.

---

**Q14. What is the metadata IP address in EC2 and why is it important for security?**

Every EC2 instance can call `169.254.169.254` to get its own metadata — instance ID, public IP, IAM role credentials, and more. This is a link-local address only reachable from inside the EC2.

It's a security concern because if an attacker compromises an EC2 running with an IAM role, they can call this endpoint to get the IAM role's temporary credentials and use them to access other AWS services. AWS introduced IMDSv2 (Instance Metadata Service v2) to require a session token before responding, which prevents simple server-side request forgery (SSRF) attacks from stealing these credentials.

---

**Q15. How would you troubleshoot a situation where your EC2 in a private subnet cannot connect to a database in another private subnet?**

I'd check in this order:

First, verify the route table of the EC2's subnet has a local route for the VPC CIDR — it should (it's always there by default). Then check the Security Group of the database — it must have an inbound rule allowing the database port (e.g., 3306 for MySQL) from either the EC2's IP, its subnet CIDR, or its Security Group. Next, check the NACL on both subnets — since NACLs are stateless, both the request and the response traffic must be explicitly allowed in both directions. Finally, verify there's no explicit DENY in the NACL blocking the traffic. If VPC Flow Logs are enabled, I'd also check them — a REJECT entry would immediately tell me exactly which layer is blocking it.

---

*📌 The Security Group vs NACL distinction is one of the most frequently asked VPC questions in DevOps interviews. The key things to always mention: SG is stateful, NACL is stateless. SG is Allow only, NACL is Allow + Deny. SG is instance level, NACL is subnet level. And always say: NACLs are needed because SGs cannot deny.*

*🩷 Read all about my Cloud and DevOps Journey on hashnode: https://cloudgirllogs.hashnode.dev/*
