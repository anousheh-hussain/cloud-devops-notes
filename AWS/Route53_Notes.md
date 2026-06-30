# AWS Route 53 — Notes + Interview Questions

---

## What Is DNS? (Start Here — Everything Else Builds On This)

### The Problem DNS Solves

Every server on the internet has an IP address — a number like `54.239.28.85`. When you want to visit Amazon, you don't type that number into your browser. You type `amazon.com`. 

But computers don't understand `amazon.com`. They only understand IP addresses.

**DNS (Domain Name System)** is the system that translates human-readable domain names into machine-readable IP addresses. It is essentially the **phonebook of the internet**.

Without DNS, you'd have to memorize the IP address of every website you want to visit. DNS makes the internet usable for humans.

---

### How DNS Resolution Works — Step by Step

When you type `amazon.com` into your browser and press Enter, here is exactly what happens:

**Step 1 — Browser Cache**
Your browser first checks if it already knows the IP for `amazon.com` from a recent visit. If it's cached and still valid, it uses that directly. No DNS query needed.

**Step 2 — OS Cache**
If the browser doesn't have it, it asks the Operating System. Your OS has its own DNS cache too.

**Step 3 — Recursive Resolver (Your ISP or a Public DNS)**
If the OS doesn't have it either, the query goes to a **Recursive Resolver** — a DNS server configured on your network. This is usually:
- Your ISP's DNS server (assigned automatically)
- Or a public one like Google's `8.8.8.8` or Cloudflare's `1.1.1.1`

The Recursive Resolver is the one that does all the heavy lifting. It goes and finds the answer on your behalf.

**Step 4 — Root DNS Server**
The Recursive Resolver first asks a **Root DNS Server**. There are only 13 sets of root servers in the world (managed by ICANN). The Root Server doesn't know the IP of `amazon.com`, but it knows who manages `.com` domains. It responds: "Ask the `.com` TLD nameserver."

**Step 5 — TLD Nameserver**
TLD = Top Level Domain. The `.com` TLD nameserver knows which nameservers are responsible for `amazon.com`. It responds: "Ask Amazon's nameservers at `ns1.p31.dynect.net`."

**Step 6 — Authoritative Nameserver**
The Recursive Resolver now asks Amazon's own nameserver — the **Authoritative Nameserver**. This is the final authority. It has the actual DNS records for `amazon.com` and responds with the IP address: `54.239.28.85`.

**Step 7 — Response Back to Browser**
The Recursive Resolver returns the IP to your OS → OS returns it to your browser → your browser connects to `54.239.28.85`.

This entire process takes milliseconds. And once resolved, the result is **cached** for a period of time defined by the **TTL (Time To Live)** — so the next request is instant.

---

### Important DNS Terms

**TTL (Time To Live)**
Every DNS record has a TTL value in seconds. This tells resolvers how long to cache the answer before asking again.
- TTL of 300 = cache for 5 minutes
- TTL of 86400 = cache for 24 hours

Low TTL = changes propagate faster (but more DNS queries = slightly more cost)
High TTL = faster for users (cached) but changes take longer to propagate globally

**DNS Propagation**
When you update a DNS record, the change doesn't happen instantly everywhere. Every resolver that has the old record cached will keep using it until the TTL expires. This is why DNS changes can take minutes to hours to "propagate" globally.

---

## Your Statement — Correction and Proper Explanation

You said: *"We generally resolve the load balancer IP address using DNS of any website like amazon.com."*

This is slightly flipped. Here's the correction:

**What actually happens:**
When you visit `amazon.com`, DNS resolves the **domain name** (`amazon.com`) **TO** the Load Balancer's IP (or DNS name). You're resolving a name → getting an IP. Not the other way around.

**The full correct picture with Load Balancers:**

In AWS, when you create an Application Load Balancer (ALB), AWS gives it a long DNS name like:
`my-load-balancer-123456.us-east-1.elb.amazonaws.com`

You do NOT put the ALB's IP address in your DNS record. Here's why:

ALBs are elastic — their underlying IP addresses can change at any time as AWS scales them up or down. If you hardcode an IP address into your DNS A record, and AWS changes the ALB's IP, your website breaks.

Instead, you create a DNS record that points `amazon.com` → to the ALB's DNS name (`my-load-balancer-123456...elb.amazonaws.com`). Route53 handles this using a special record type called an **ALIAS record** (explained below). Route53 internally keeps the resolution up to date even as the ALB's IPs change.

**So the correct flow for a real AWS application is:**

```
User types: amazon.com
  ↓
DNS resolves amazon.com → ALB DNS name (via ALIAS record in Route53)
  ↓
ALB DNS name resolves → Current IP(s) of the ALB
  ↓
User's request reaches the ALB
  ↓
ALB routes to healthy EC2 instance
```

---

## What Is Route 53?

**Route 53** is AWS's fully managed DNS service. It's named Route 53 because **DNS operates on port 53** — and adding "Route" references the internet routing concept.

Route 53 does three distinct things:

**1. Domain Registration**
You can buy and register domain names directly through Route 53 (like `myapp.com`). AWS handles the ICANN registration.

**2. DNS Service (Authoritative DNS)**
Route 53 acts as the Authoritative Nameserver for your domain. When anyone in the world queries your domain, Route 53 answers with the correct IP or resource.

**3. Health Checking**
Route 53 can monitor your endpoints and automatically route traffic away from unhealthy ones.

**Key characteristic: Route 53 is a global service.** It doesn't belong to any single AWS region. DNS needs to be globally distributed for low latency, and Route 53 uses AWS's global network of Points of Presence (PoPs) worldwide — this is called **Anycast routing** (all Route 53 servers share the same IP addresses; your query automatically goes to the nearest one).

---

## Why Getting Your Own DNS Is Difficult

You could technically run your own DNS server — BIND is a free, popular DNS server software. So why does anyone pay for Route 53 or other DNS services?

**1. Global Infrastructure Required**
DNS needs to be fast for users everywhere in the world. A single DNS server in Mumbai is too slow for users in the US or Europe. You'd need servers on every continent, connected through high-speed networks. That's a massive infrastructure investment.

**2. 100% Uptime Is Non-Negotiable**
If your DNS server goes down, your entire website becomes unreachable — even if your web servers are perfectly healthy. DNS is the first step in any connection. You'd need redundant servers, failover, and constant monitoring. AWS guarantees Route 53 at 100% availability SLA — the only AWS service with this guarantee.

**3. DDoS Protection**
DNS servers are common targets for DDoS (Distributed Denial of Service) attacks. Route 53 is backed by AWS Shield and handles attacks automatically. A self-hosted DNS server getting DDoSed means your site goes down.

**4. Anycast Routing**
Route 53 uses a technique called Anycast — the same IP address is announced from multiple locations worldwide. Your DNS query automatically goes to the closest AWS location. Building Anycast yourself requires BGP networking expertise and agreements with ISPs globally.

**5. DNSSEC, Monitoring, Logging**
DNS Security Extensions (DNSSEC), query logging, health check integration, geo-routing — all of these are built into Route 53. Implementing them yourself from scratch takes months of engineering work.

**6. Cost vs Value**
Route 53 costs $0.50/month per hosted zone and $0.40 per million DNS queries. Running your own global DNS infrastructure with the same reliability would cost thousands of dollars per month.

**In one line:** Route 53 gives you enterprise-grade global DNS without you having to build and maintain any of the infrastructure yourself.

---

## DNS Records — Everything You Need to Know

A DNS record is a single entry in your DNS zone that maps a name to a value. Here are all the record types you'll encounter:

### A Record (Address Record)
Maps a domain name to an **IPv4 address**.

```
amazon.com → 54.239.28.85
```
Most basic record. Used when your resource has a static IP.

---

### AAAA Record (Quad-A Record)
Maps a domain name to an **IPv6 address**.

```
amazon.com → 2600:9000:201d:d800:...
```
Same as A record but for IPv6.

---

### CNAME Record (Canonical Name)
Maps a domain name to **another domain name** (not an IP).

```
www.amazon.com → amazon.com
blog.amazon.com → amazon.com
```

**Critical limitation: CNAME cannot be used at the Zone Apex (the root domain).**
You cannot create a CNAME for `amazon.com` itself — only for subdomains like `www.amazon.com`. This is a DNS standard rule. This is why AWS created the ALIAS record.

---

### ALIAS Record (AWS-Specific)
This is Route 53's custom record type. It's not a standard DNS record — it's AWS's solution to the CNAME-at-root-domain limitation.

An ALIAS record maps a domain name to an **AWS resource** — like an ALB, CloudFront distribution, S3 static website, or another Route 53 record.

```
amazon.com → my-load-balancer-123456.us-east-1.elb.amazonaws.com
```

**Why ALIAS is better than CNAME for AWS resources:**
- ALIAS can be used at the zone apex (`amazon.com`) — CNAME cannot
- ALIAS queries to AWS resources are **free** — CNAME queries are charged
- ALIAS automatically updates if the underlying AWS resource's IP changes
- ALIAS has lower latency because Route 53 resolves it internally

**When to use ALIAS vs A record:**
- Pointing to an ALB, CloudFront, S3, or other AWS service → use ALIAS
- Pointing to a specific static IP → use A record

---

### MX Record (Mail Exchange)
Specifies which mail servers handle email for your domain.

```
amazon.com → mail.amazon.com (priority 10)
```
The priority number matters — lower number = higher priority. Used so email providers know where to deliver `@amazon.com` emails.

---

### TXT Record (Text Record)
Stores arbitrary text data. Used for:
- **Domain verification** — Google, AWS, GitHub ask you to add a TXT record to prove you own the domain
- **SPF (Sender Policy Framework)** — tells email servers which IP addresses are allowed to send email on behalf of your domain
- **DKIM** — email signing verification

---

### NS Record (Nameserver Record)
Lists the authoritative nameservers for your domain. When you create a hosted zone in Route 53, it gives you 4 NS records like:
```
ns-123.awsdns-12.com
ns-456.awsdns-34.net
ns-789.awsdns-56.org
ns-012.awsdns-78.co.uk
```
These are what you put into your domain registrar (GoDaddy, Namecheap, etc.) to delegate DNS control to Route 53.

---

### SOA Record (Start of Authority)
Automatically created with every hosted zone. Contains administrative information about the zone — the primary nameserver, admin email, serial number, and TTL settings. You rarely need to modify this.

---

### PTR Record (Reverse DNS)
The opposite of an A record — maps an IP address back to a domain name. Used for email authentication and network troubleshooting. Requires you to request PTR record creation through your ISP or AWS support.

---

### SRV Record (Service Record)
Specifies the location of a service — hostname, port, and priority. Used for protocols like SIP (VoIP), XMPP (chat), and some game servers.

---

## Hosted Zones — What Are They?

A **Hosted Zone** is a container in Route 53 that holds all the DNS records for a specific domain.

Think of it as a folder labeled `amazon.com` that contains all the DNS records (A, CNAME, MX, TXT, etc.) for that domain and all its subdomains.

When you create a hosted zone for `amazon.com`, Route 53 automatically creates:
- An **NS record** with 4 nameservers assigned to your zone
- An **SOA record** with zone metadata

You then add your own records (A, CNAME, ALIAS, etc.) inside this hosted zone.

**Cost:** $0.50/month per hosted zone.

### Public Hosted Zone

A **Public Hosted Zone** is accessible from the **public internet**. Anyone in the world can query it.

Use case: Your website `amazon.com` — you want the whole world to be able to find it.

### Private Hosted Zone

A **Private Hosted Zone** is only accessible from **within a specific VPC** (or multiple VPCs you associate it with). It's invisible to the public internet.

Use case: Internal microservices that talk to each other using domain names like `payments.internal` or `db.internal`. You don't want these resolvable from the internet — only from within your VPC.

Example: Your app server queries `db.internal` → Route 53 Private Hosted Zone resolves it to `10.0.3.5` (the database's private IP) → connection is made privately without any internet exposure.

---

## Route 53 With Your Own Domain (GoDaddy / Namecheap / Any Registrar)

You do NOT have to register your domain through AWS to use Route 53 for DNS. You can register on GoDaddy, Namecheap, or anywhere else, and still use Route 53 as the DNS provider.

### How It Works

**Step 1 — Register domain on GoDaddy**
You buy `myapp.com` on GoDaddy for ₹1,000/year. GoDaddy is now your **domain registrar** (they handle ICANN registration).

**Step 2 — Create a Hosted Zone in Route 53**
Go to Route 53 → Hosted Zones → Create Hosted Zone → enter `myapp.com` → Public Hosted Zone.

Route 53 creates the zone and gives you 4 NS records, something like:
```
ns-245.awsdns-30.com
ns-1427.awsdns-50.org
ns-1963.awsdns-53.co.uk
ns-815.awsdns-38.net
```

**Step 3 — Update Nameservers in GoDaddy**
Go to GoDaddy → DNS Management for `myapp.com` → change the Nameservers from GoDaddy's defaults to the 4 NS records from Route 53.

This tells the world: "For any DNS queries about `myapp.com`, ask Route 53's nameservers."

**Step 4 — Add Your Records in Route 53**
Now add all DNS records (A, ALIAS, CNAME, MX, etc.) inside your Route 53 hosted zone. GoDaddy is now only the registrar (ownership management) — Route 53 handles all actual DNS.

**Propagation time:** After changing nameservers, it takes 24-48 hours to fully propagate globally as cached NS records expire.

### When Domain Is Registered Directly With Route 53

If you register through Route 53, the NS records are automatically configured — you don't have to manually update anything. Route 53 registers the domain AND manages DNS for it out of the box.

Either way works. Most people already have domains on GoDaddy or Namecheap — the nameserver delegation approach is the standard way to use Route 53 with an existing domain.

---

## Route 53 Routing Policies

This is where Route 53 becomes much more than just a DNS service. **Routing Policies** control HOW Route 53 responds to DNS queries.

### 1. Simple Routing
One domain → one resource. Basic DNS. If you add multiple IPs to the same record, Route 53 returns all of them in a random order (the client picks one). No health checks.

Use: basic single-server setup.

### 2. Weighted Routing
Split traffic across multiple resources by percentage.

```
myapp.com → EC2 in us-east-1 (weight 70)
myapp.com → EC2 in eu-west-1 (weight 30)
```

70% of users go to us-east-1, 30% to eu-west-1.

Use case: **A/B testing** a new version (send 10% of traffic to v2, 90% to v1). **Blue-green deployment** (gradually shift traffic from old to new version).

### 3. Latency-Based Routing
Route 53 measures which AWS region has the lowest latency for the user making the request, and routes them there.

```
myapp.com → EC2 in Mumbai (for Indian users — lowest latency)
myapp.com → EC2 in us-east-1 (for US users — lowest latency)
```

Use case: Global applications where you have servers in multiple regions and want users to automatically reach the fastest one.

### 4. Failover Routing
Active-Passive setup. You define a **Primary** resource and a **Secondary** resource.

- Route 53 sends all traffic to the Primary.
- Route 53 runs health checks on the Primary.
- If the Primary fails health checks → Route 53 automatically switches all traffic to the Secondary.
- When Primary recovers → traffic switches back.

Use case: Disaster recovery. Primary is your main server; secondary is a backup (could even be an S3 static page saying "we're under maintenance").

### 5. Geolocation Routing
Route users based on their **geographic location** (country or continent level).

```
Users from India → Indian server
Users from USA → US server
Users from Europe → EU server
Users from anywhere else → Default
```

Use case: Legal compliance (GDPR — European user data must stay in Europe), localization (show the right language/currency), content licensing restrictions.

### 6. Geoproximity Routing
Routes based on geographic location but with a **bias** — you can artificially expand or shrink the area served by each resource using a bias value (+/-).

Positive bias (+) → expand coverage, attract more users to this resource.
Negative bias (-) → shrink coverage, push users to other resources.

Use case: Gradually shifting users from one region to another. More granular than geolocation.

### 7. Multivalue Answer Routing
Returns up to 8 healthy records in response to each DNS query. Like Simple Routing with multiple IPs, but it integrates health checks — unhealthy endpoints are automatically removed from responses.

Use case: Basic DNS-level load balancing across multiple servers. Not a replacement for a real load balancer, but a lightweight alternative for simple setups.

### 8. IP-Based Routing
Route traffic based on the **user's IP address range** (CIDR). You define which IP ranges go to which endpoint.

Use case: You know your corporate users come from a specific IP range — route them to an internal-optimized endpoint. ISP-specific routing.

---

## Health Checks in Route 53

Route 53 Health Checks monitor the health of your endpoints and integrate with routing policies (especially Failover) to automatically route traffic away from unhealthy resources.

### Types of Health Checks

**1. Endpoint Health Check**
Route 53 sends HTTP, HTTPS, or TCP requests to your endpoint (IP or domain) from multiple AWS locations worldwide.

You configure:
- Protocol (HTTP/HTTPS/TCP)
- Hostname and port
- Path to check (e.g., `/health`)
- Failure threshold (how many consecutive failures = unhealthy)
- Interval (every 10 or 30 seconds)

Route 53 considers an endpoint healthy if more than 18% of its health checkers report it as healthy. This prevents a single regional outage from marking a global resource as failed.

**2. Calculated Health Check**
Combines the results of multiple health checks using boolean logic (AND, OR, NOT).

Example: Report "healthy" only if BOTH the web server health check AND the database health check pass.

Use case: Complex applications with multiple dependencies.

**3. CloudWatch Alarm Health Check**
Monitors a CloudWatch metric and marks the resource as unhealthy if the alarm fires.

Example: If CPU usage on your EC2 exceeds 90% for 5 minutes, trigger the CloudWatch alarm → Route 53 marks it unhealthy → traffic is routed away.

Use case: Resources that are technically reachable but performing poorly under load.

### Health Check + Failover Routing = Automatic Disaster Recovery

This is the key use case:

1. Set up Failover Routing: Primary (us-east-1) + Secondary (eu-west-1)
2. Attach a health check to the Primary
3. Route 53 checks the Primary every 30 seconds
4. Primary server crashes → health check fails → Route 53 automatically starts returning the Secondary's IP
5. Users are transparently redirected to the backup — no manual intervention needed
6. When Primary recovers → Route 53 switches back

**Health check for private resources:** Route 53 health checkers are on the public internet. They can't directly check private EC2 instances. The workaround: associate the health check with a CloudWatch alarm that monitors the private resource. The alarm can see VPC metrics; Route 53 monitors the alarm.

---

## Other Important Route 53 Information

### Route 53 Resolver (Hybrid Cloud DNS)

When you have a hybrid setup — some resources in AWS and some in your on-premise data center — DNS becomes complicated. On-premise servers don't know how to resolve AWS private DNS names, and VPC resources don't know how to resolve on-premise DNS.

**Route 53 Resolver** solves this:
- **Inbound Endpoints** → on-premise servers can forward DNS queries for AWS resources (like `db.internal`) into Route 53 Resolver in the VPC
- **Outbound Endpoints** → VPC resources can forward DNS queries for on-premise resources (like `corp.company.com`) to on-premise DNS servers

Use case: A company with servers in their own data center AND in AWS — all resources can find each other by name.

### Route 53 Resolver DNS Firewall

A managed firewall for DNS queries inside your VPC. You can:
- Block queries to known malicious domains (command-and-control servers used by malware)
- Allow queries only to approved domains
- Prevent data exfiltration via DNS tunneling

Works by filtering outbound DNS queries from your VPC before they leave AWS.

### DNSSEC (DNS Security Extensions)

DNS was built without security in mind. DNS responses can be forged — an attacker can intercept a DNS query and return a fake IP address, redirecting users to a malicious server. This is called **DNS spoofing** or **cache poisoning**.

DNSSEC solves this by digitally signing DNS responses. The recipient can cryptographically verify that the response came from the real authoritative nameserver and wasn't tampered with.

Route 53 supports DNSSEC for both domain registration and for signing hosted zones.

### Route 53 Pricing Summary

| Feature | Price |
|---|---|
| Hosted Zone | $0.50/month per zone |
| DNS Queries (first 1 billion/month) | $0.40 per million queries |
| Health Checks (basic) | $0.50/month per endpoint |
| Domain Registration | Varies by TLD ($13/year for .com) |
| Latency/Geo/Failover queries | $0.70 per million queries |

### Route 53 100% SLA

Route 53 is the **only AWS service with a 100% availability SLA**. AWS commits that Route 53 will be available at all times. If it goes down (which has essentially never happened), AWS provides service credits.

This is why Route 53 is trusted for critical production DNS — no other AWS service makes this guarantee.

### Route 53 vs Other DNS Services

| Feature | Route 53 | Cloudflare | GoDaddy DNS |
|---|---|---|---|
| Global Anycast Network | ✅ | ✅ | Limited |
| AWS Native Integration | ✅ Best | ❌ | ❌ |
| Health Checks + Failover | ✅ | ✅ | ❌ |
| Pricing | Pay per query | Free tier available | Free with domain |
| DNSSEC | ✅ | ✅ | Limited |
| Availability SLA | 100% | 100% | Not guaranteed |

### Alias Record Cannot Have TTL

Unlike regular DNS records where you set a TTL, AWS Alias records don't have a TTL — Route 53 manages it automatically based on the underlying AWS resource's own TTL. You can't set a custom TTL for an Alias record.

---

---

# Interview Questions & Answers

---

**Q1. What is DNS and how does it work?**

DNS stands for Domain Name System. It's the system that translates human-readable domain names like amazon.com into IP addresses that computers can use to make connections. Without DNS, you'd have to remember the IP address of every website.

When you type a domain name, your browser first checks its local cache. If not cached, it asks a Recursive Resolver — usually your ISP's DNS server. The Recursive Resolver asks the Root DNS server, which directs it to the TLD nameserver for the domain extension. The TLD server points to the domain's authoritative nameserver — the one that has the actual DNS records. The authoritative nameserver returns the IP address, which is then cached and returned to the browser.

---

**Q2. What is Route 53?**

Route 53 is AWS's fully managed DNS service, domain registrar, and health checking system. It's named after port 53, which is the standard port DNS uses. Route 53 acts as an authoritative nameserver for your domains, answering DNS queries from anywhere in the world using AWS's global network of Points of Presence. It's the only AWS service with a 100% availability SLA. Beyond basic DNS, it supports advanced routing policies like latency-based, geolocation, weighted, and failover routing — making it useful for traffic management, not just name resolution.

---

**Q3. What is the difference between an A record, CNAME, and ALIAS record?**

An A record maps a domain name directly to an IPv4 address. A CNAME maps a domain name to another domain name — it's an alias that eventually resolves to an IP. However, CNAME cannot be used at the zone apex — you can't create a CNAME for the root domain like `amazon.com`, only for subdomains.

The ALIAS record is AWS Route 53's custom solution for this limitation. Like a CNAME, it points to a hostname rather than an IP — specifically an AWS resource like an ALB or CloudFront. But unlike CNAME, it CAN be used at the zone apex. It's also free for queries to AWS resources and automatically stays in sync if the resource's IPs change.

---

**Q4. How would you point a domain registered on GoDaddy to an AWS resource using Route 53?**

First, create a Public Hosted Zone in Route 53 for the domain. Route 53 will assign four NS nameserver records. Then go to GoDaddy's DNS settings for the domain and replace GoDaddy's default nameservers with the four Route 53 nameservers. This delegates DNS authority to Route 53. After propagation — which takes up to 48 hours — any DNS queries for the domain will be answered by Route 53. You then create your actual DNS records (ALIAS pointing to an ALB, A records, etc.) inside the Route 53 hosted zone.

---

**Q5. What is a Hosted Zone in Route 53?**

A Hosted Zone is a container for all the DNS records belonging to a specific domain. It's like a folder for `myapp.com` that holds all its A records, CNAME records, MX records, and so on.

There are two types: a Public Hosted Zone is accessible from the internet — used for websites and public-facing resources. A Private Hosted Zone is only accessible from within a VPC — used for internal services where you want resources to find each other by name without exposing anything publicly.

---

**Q6. What is the difference between a Public and Private Hosted Zone?**

A Public Hosted Zone resolves DNS queries from anywhere on the internet. When someone types your domain, the public hosted zone returns the answer.

A Private Hosted Zone only resolves queries from within the associated VPC. It's invisible to the public internet. For example, you might create `payments.internal` in a private hosted zone, and only EC2 instances inside your VPC can resolve it. The public internet has no idea this domain exists. It's used for internal microservice communication, database hostnames, and any resource you don't want publicly discoverable.

---

**Q7. What is TTL in DNS and why does it matter?**

TTL — Time To Live — is a value in seconds attached to every DNS record. It tells DNS resolvers how long to cache the answer before querying again.

A high TTL (86400 = 24 hours) means clients cache the result and don't query frequently. This reduces DNS query costs and speeds up resolution for end users. But if you change the IP address, old clients keep using the cached old IP for up to 24 hours before they pick up the change.

A low TTL (60 = 1 minute) means changes propagate quickly, but resolvers query more frequently, which increases DNS query costs and adds slight latency.

Best practice: reduce TTL to 60–300 seconds before making a planned DNS change (like migrating to a new server), then increase it again after the change is confirmed.

---

**Q8. What are the different routing policies in Route 53?**

Route 53 supports several routing policies. Simple routing sends all traffic to one resource. Weighted routing splits traffic by percentage — useful for A/B testing or gradual deployments. Latency-based routing sends users to the AWS region with the lowest latency for them. Failover routing sends traffic to a primary resource and automatically switches to a secondary if the primary fails health checks. Geolocation routing routes users based on their country or continent. Geoproximity routing is similar but allows you to bias the routing by adjusting coverage areas. Multivalue answer routing returns multiple healthy records for basic DNS-level load distribution.

---

**Q9. What is Failover Routing and how does it work with Health Checks?**

Failover routing is an active-passive setup where Route 53 sends all traffic to a primary resource, but monitors it with a health check. If the health check detects the primary is down, Route 53 automatically starts returning the secondary resource's DNS record instead. Users are redirected without any manual intervention.

The health check sends periodic HTTP, HTTPS, or TCP requests to the endpoint. If it fails more than the configured threshold, Route 53 considers the resource unhealthy and activates failover. When the primary recovers, traffic automatically shifts back. This gives you automatic disaster recovery at the DNS level.

---

**Q10. Why would you use Route 53 instead of just running your own DNS server?**

Running your own DNS server sounds simple but becomes complex at scale. You'd need servers in multiple continents for low latency, 100% uptime with redundancy, DDoS protection, Anycast routing implementation, DNSSEC support, and 24/7 maintenance. This requires significant infrastructure investment and engineering expertise.

Route 53 provides all of this for $0.50 per hosted zone per month. It's backed by AWS's global network, has a 100% SLA — the only AWS service with that guarantee — and integrates natively with all other AWS services. The cost-benefit ratio makes self-hosted DNS economically irrational for any serious application.

---

**Q11. What is the difference between Geolocation and Latency-Based routing?**

Geolocation routing routes based on where the user is geographically — their country or continent. Even if a user in India would actually get lower latency from a Singapore server, if you've configured geolocation to send all Indian users to Mumbai, they go to Mumbai regardless.

Latency-based routing measures actual network latency between the user and each AWS region and routes to whichever region is fastest at that moment. It's based on real performance data, not just geography.

Use geolocation when you need strict geographic control — like GDPR compliance requiring European user data to stay in Europe. Use latency-based when you simply want the best performance for each user.

---

**Q12. Can Route 53 health checks monitor private EC2 instances in a VPC?**

Not directly, because Route 53's health checkers are on the public internet and can't reach private IPs. The workaround is to use CloudWatch Alarm-based health checks. You create a CloudWatch alarm that monitors a metric for your private resource — like the status check or a custom application metric. You then configure the Route 53 health check to monitor that CloudWatch alarm state rather than the endpoint directly. If the alarm fires, Route 53 treats the resource as unhealthy and triggers failover accordingly.

---

**Q13. What is Route 53 Resolver and when would you need it?**

Route 53 Resolver solves DNS for hybrid cloud environments where you have resources both in AWS VPCs and in on-premise data centers. By default, VPC resources can't resolve on-premise hostnames, and on-premise servers can't resolve private AWS hostnames.

Resolver provides Inbound Endpoints — on-premise servers can forward DNS queries for AWS private domains into Route 53 via these endpoints. And Outbound Endpoints — VPC resources can forward DNS queries for on-premise domains to on-premise DNS servers. This creates seamless name resolution across a hybrid environment without either side needing to know where the other's servers physically are.

---

*📌 Route 53 is tested in AWS Solutions Architect and DevOps interviews primarily around: ALIAS vs CNAME, routing policies and when to use each (especially Failover and Latency), health checks with private resources, and the hosted zone concept. The routing policy use cases — A/B testing with Weighted, disaster recovery with Failover, compliance with Geolocation — are exactly the kind of scenario questions interviewers ask.*

*🩷 Read about my Cloud/Devops Journey on Hashnode: https://cloudgirllogs.hashnode.dev/*
