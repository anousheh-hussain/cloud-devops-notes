[Networking_for_DevOps_Cloud.md](https://github.com/user-attachments/files/30292865/Networking_for_DevOps_Cloud.md)
# Networking for DevOps & Cloud Engineers

> Everything you need to know about networking to work confidently in cloud infrastructure and DevOps roles. No extra fluff.

---

## Table of Contents

1. [The OSI and TCP/IP Models](#1-the-osi-and-tcpip-models)
2. [IP Addressing](#2-ip-addressing)
3. [Subnetting and CIDR](#3-subnetting-and-cidr)
4. [DNS — Domain Name System](#4-dns--domain-name-system)
5. [TCP vs UDP](#5-tcp-vs-udp)
6. [Common Ports and Protocols](#6-common-ports-and-protocols)
7. [HTTP and HTTPS](#7-http-and-https)
8. [TLS/SSL](#8-tlsssl)
9. [Firewalls](#9-firewalls)
10. [NAT — Network Address Translation](#10-nat--network-address-translation)
11. [Load Balancing](#11-load-balancing)
12. [VPN](#12-vpn)
13. [Routing Basics](#13-routing-basics)
14. [SSH](#14-ssh)
15. [Network Security Concepts](#15-network-security-concepts)
16. [Cloud Networking Concepts](#16-cloud-networking-concepts)

---

## 1. The OSI and TCP/IP Models

Understanding layered models is how you diagnose where a network problem is occurring.

### OSI Model (7 Layers)

| Layer | Name | What it does | Examples |
|---|---|---|---|
| 7 | Application | User-facing protocols | HTTP, FTP, DNS, SMTP |
| 6 | Presentation | Encoding, encryption, compression | TLS, JPEG, ASCII |
| 5 | Session | Manages sessions between apps | NetBIOS, RPC |
| 4 | Transport | End-to-end delivery, port numbers | TCP, UDP |
| 3 | Network | Routing, logical addressing | IP, ICMP, routing protocols |
| 2 | Data Link | Node-to-node delivery, MAC addresses | Ethernet, ARP, switches |
| 1 | Physical | Raw bits over physical medium | Cables, NICs, hubs |

### TCP/IP Model (4 Layers) — What the Internet Actually Uses

| TCP/IP Layer | OSI Equivalent | Protocols |
|---|---|---|
| Application | Layers 5, 6, 7 | HTTP, DNS, SSH, FTP, SMTP |
| Transport | Layer 4 | TCP, UDP |
| Internet | Layer 3 | IP, ICMP, ARP |
| Network Access | Layers 1, 2 | Ethernet, WiFi |

### Why This Matters for DevOps

When debugging:
- Can't reach a server? Start at Layer 3 (is the IP reachable? `ping`)
- IP reachable but service not responding? Layer 4 (is the port open? `telnet`, `nc`)
- Port open but getting errors? Layer 7 (application-level — check logs)

---

## 2. IP Addressing

### IPv4

32-bit address written as four octets separated by dots.

```
192.168.1.100
```

Each octet is 0–255. Total ~4.3 billion addresses. We've run out — hence IPv6 and NAT.

### Address Classes (historical, but still tested)

| Class | Range | Default Subnet | Use |
|---|---|---|---|
| A | 1.0.0.0 – 126.255.255.255 | /8 | Large networks |
| B | 128.0.0.0 – 191.255.255.255 | /16 | Medium networks |
| C | 192.0.0.0 – 223.255.255.255 | /24 | Small networks |

### Private IP Ranges (RFC 1918)

These are not routable on the public internet. Used inside private networks (your home, VPC, office).

| Range | CIDR | Common Use |
|---|---|---|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | Large corporate/cloud networks |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | Medium networks |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | Home networks |

AWS VPCs use private ranges — EC2 instances get private IPs from these ranges.

### Special Addresses

| Address | Meaning |
|---|---|
| 127.0.0.1 | Loopback — "this machine itself" |
| 0.0.0.0 | All interfaces / default route |
| 255.255.255.255 | Broadcast to entire local network |

### IPv6

128-bit address in hexadecimal, grouped in 8 blocks of 4 hex digits.

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

Consecutive blocks of zeros can be compressed with `::` (once per address):

```
2001:db8:85a3::8a2e:370:7334
```

Key differences from IPv4:
- No broadcast — uses multicast instead
- Built-in IPSec
- No NAT needed — enough addresses for everything
- AWS supports IPv6 in VPCs

---

## 3. Subnetting and CIDR

### What is Subnetting?

Dividing a large network into smaller sub-networks. Improves security (isolate traffic), efficiency (reduce broadcast domains), and organization.

### CIDR Notation

CIDR (Classless Inter-Domain Routing) specifies a network using IP/prefix.

```
192.168.1.0/24
```

The `/24` means the first 24 bits are the network portion — leaving 8 bits for hosts.

### Quick Reference Table

| CIDR | Subnet Mask | Usable Hosts | Common Use |
|---|---|---|---|
| /8 | 255.0.0.0 | 16,777,214 | Large enterprise |
| /16 | 255.255.0.0 | 65,534 | AWS VPC default |
| /24 | 255.255.255.0 | 254 | Standard subnet |
| /28 | 255.255.255.240 | 14 | Small subnet |
| /32 | 255.255.255.255 | 1 | Single host (security group rules) |

### How to Calculate Hosts

Hosts in a subnet = 2^(32 - prefix) - 2

The -2 is for network address (first IP) and broadcast address (last IP), both reserved.

```
/24 = 2^(32-24) - 2 = 2^8 - 2 = 256 - 2 = 254 usable hosts
/28 = 2^(32-28) - 2 = 2^4 - 2 = 16 - 2 = 14 usable hosts
```

### AWS Subnets

AWS reserves 5 IPs in every subnet (not 2 like standard subnetting):

- First IP: network address
- Second IP: VPC router
- Third IP: DNS
- Fourth IP: reserved for future use
- Last IP: broadcast

So a `/28` in AWS gives you 16 - 5 = 11 usable IPs.

### Public vs Private Subnets

- **Public subnet** — has a route to an Internet Gateway. Resources here can be reached from the internet
- **Private subnet** — no route to Internet Gateway. Resources here cannot be directly reached from outside. Use NAT Gateway for outbound internet access

---

## 4. DNS — Domain Name System

### What DNS Does

Translates human-readable domain names into IP addresses.

```
google.com → 142.250.180.14
```

### DNS Resolution Process

1. Browser checks its own cache
2. OS checks its cache and `/etc/hosts`
3. Query goes to **Recursive Resolver** (usually your ISP or 8.8.8.8)
4. Resolver asks a **Root Nameserver** — "who handles .com?"
5. Root points to **TLD Nameserver** — "who handles google.com?"
6. TLD points to **Authoritative Nameserver** — has the actual record
7. Answer returned to browser, cached with TTL

### DNS Record Types

| Record | Purpose | Example |
|---|---|---|
| A | Maps hostname to IPv4 address | `api.example.com → 1.2.3.4` |
| AAAA | Maps hostname to IPv6 address | `api.example.com → 2001:db8::1` |
| CNAME | Alias to another hostname | `www.example.com → example.com` |
| MX | Mail server for domain | `example.com → mail.example.com` |
| TXT | Text data, used for verification | SPF, DKIM records |
| NS | Nameservers for domain | `example.com → ns1.awsdns.com` |
| PTR | Reverse DNS — IP to hostname | Used in email, logging |
| SOA | Start of Authority — zone metadata | Serial number, refresh interval |

### TTL — Time to Live

How long a DNS answer gets cached before being looked up again.

- Low TTL (60s) — changes propagate fast, more DNS queries, more cost
- High TTL (86400s = 1 day) — faster lookups, but slow propagation when you change records

Always lower TTL before making DNS changes, then raise it again after.

### CNAME vs ALIAS (AWS Specific)

Standard CNAME cannot be used on a root/apex domain (`example.com`) — only on subdomains (`www.example.com`).

AWS Route 53 ALIAS records solve this — they function like CNAME but work on root domains, and point to AWS resources (load balancers, CloudFront, S3) rather than a fixed IP. The IP is resolved internally by AWS.

### `/etc/hosts` File

Local override for DNS — checked before any DNS query.

```bash
cat /etc/hosts
# 127.0.0.1   localhost
# 192.168.1.10  myserver.local
```

Useful for local development and testing without actual DNS changes.

### `/etc/resolv.conf`

Specifies which DNS servers to use.

```bash
cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 8.8.4.4
```

### Useful DNS Commands

```bash
nslookup google.com              # basic DNS lookup
dig google.com                   # detailed DNS lookup
dig google.com MX                # look up specific record type
dig +short google.com            # just the IP
dig @8.8.8.8 google.com         # query specific DNS server
host google.com                  # simple lookup
```

---

## 5. TCP vs UDP

### TCP — Transmission Control Protocol

Connection-oriented, reliable, ordered delivery.

**Three-way handshake before data transfer:**

```
Client → SYN       → Server
Client ← SYN-ACK  ← Server
Client → ACK       → Server
[Connection established, data transfer begins]
```

**Four-way handshake to close:**

```
Client → FIN       → Server
Client ← ACK       ← Server
Client ← FIN       ← Server
Client → ACK       → Server
```

**Key features:**
- Acknowledgment for every segment
- Retransmission of lost packets
- Ordered delivery — receiver reorders out-of-sequence packets
- Flow control — receiver tells sender how much buffer space it has
- Congestion control — TCP backs off when network is congested

**Use when:** accuracy matters more than speed — HTTP/HTTPS, SSH, email, database connections, file transfers.

### UDP — User Datagram Protocol

Connectionless, no reliability guarantees, no ordering.

Just sends packets. No handshake, no ACKs, no retransmission.

**Key features:**
- Very low overhead
- Much faster than TCP
- Packets can be lost, duplicated, or arrive out of order
- Application layer handles any reliability if needed

**Use when:** speed matters more than perfection — DNS, video streaming, VoIP, gaming, NTP, DHCP.

### Side-by-Side

| | TCP | UDP |
|---|---|---|
| Connection | Handshake required | No connection |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | In-order delivery | No ordering |
| Speed | Slower (overhead) | Faster |
| Header size | 20 bytes | 8 bytes |
| Use case | HTTP, SSH, databases | DNS, streaming, gaming |

---

## 6. Common Ports and Protocols

Memorize these — they come up in security groups, firewall rules, and troubleshooting constantly.

| Port | Protocol | Service |
|---|---|---|
| 20 | TCP | FTP data transfer |
| 21 | TCP | FTP control |
| 22 | TCP | SSH |
| 23 | TCP | Telnet (unencrypted — never use) |
| 25 | TCP | SMTP (email sending) |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 (email receiving) |
| 143 | TCP | IMAP (email) |
| 443 | TCP | HTTPS |
| 465 | TCP | SMTP over TLS |
| 587 | TCP | SMTP submission |
| 993 | TCP | IMAP over TLS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 27017 | TCP | MongoDB |
| 8080 | TCP | HTTP alternate / app servers |
| 8443 | TCP | HTTPS alternate |

### Ephemeral Ports

When a client connects to a server, the client uses a temporary high-numbered port (1024–65535) for the return traffic. This is why NACL outbound rules need to allow this range for return traffic to work.

---

## 7. HTTP and HTTPS

### HTTP Methods

| Method | Use |
|---|---|
| GET | Retrieve data |
| POST | Send data to create/update |
| PUT | Replace a resource entirely |
| PATCH | Partially update a resource |
| DELETE | Delete a resource |
| HEAD | Like GET but headers only, no body |
| OPTIONS | What methods does this endpoint support? |

### HTTP Status Codes

| Range | Meaning | Common Examples |
|---|---|---|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Permanent, 302 Temporary, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

### Important for DevOps:

- **502 Bad Gateway** — load balancer can't reach the upstream server. Check if app is running, target group health, security groups
- **503 Service Unavailable** — server is up but overloaded, or no healthy targets in target group
- **504 Gateway Timeout** — upstream server responded too slowly. Check application performance, timeouts

### HTTP Headers You'll See

```
Host: example.com
Content-Type: application/json
Authorization: Bearer <token>
X-Forwarded-For: client-ip        # original client IP after passing through load balancer
X-Real-IP: client-ip
Cache-Control: no-cache
```

### HTTP/1.1 vs HTTP/2 vs HTTP/3

- **HTTP/1.1** — one request per TCP connection (or persistent connection but sequential). Still widely used
- **HTTP/2** — multiplexing — multiple requests over one TCP connection simultaneously. Faster
- **HTTP/3** — uses QUIC (UDP-based) instead of TCP. Even faster, especially on unreliable connections

---

## 8. TLS/SSL

### What It Does

TLS (Transport Layer Security) encrypts data in transit. HTTPS = HTTP + TLS. When you see the padlock in a browser, TLS is working.

SSL is the older, deprecated version. When people say SSL today they usually mean TLS.

### TLS Handshake (Simplified)

1. Client sends "ClientHello" — supported TLS versions, cipher suites
2. Server responds "ServerHello" — chosen TLS version and cipher, sends certificate
3. Client verifies certificate against trusted Certificate Authorities (CAs)
4. Key exchange — both sides derive the same session key without transmitting it
5. Encrypted communication begins

### TLS Certificates

A certificate contains:
- The domain name it's valid for
- The public key
- The issuer (Certificate Authority)
- Expiry date
- Digital signature from the CA

**Certificate types:**
- **DV (Domain Validated)** — just proves you own the domain. Fast, automated, used by Let's Encrypt
- **OV (Organization Validated)** — also verifies organization identity
- **EV (Extended Validation)** — highest level, shows company name in browser (rare now)

**Wildcard certificates** — `*.example.com` — valid for all subdomains one level deep.

**SAN certificates** — covers multiple specific domains in one cert.

### Where Certificates Live in Linux

```bash
/etc/ssl/certs/          # trusted CA certificates
/etc/ssl/private/        # private keys (root-only readable)
/etc/nginx/ssl/          # nginx cert location (common)
/etc/letsencrypt/live/   # Let's Encrypt certs
```

### Useful Commands

```bash
# check certificate expiry
openssl s_client -connect example.com:443 -servername example.com | openssl x509 -noout -dates

# view certificate details
openssl x509 -in cert.pem -text -noout

# check what TLS versions a server supports
nmap --script ssl-enum-ciphers -p 443 example.com
```

---

## 9. Firewalls

### What a Firewall Does

Filters network traffic based on defined rules — allow or deny based on IP, port, protocol, direction.

### Stateless vs Stateful

**Stateless firewall** — evaluates each packet independently, no memory of previous packets. Must explicitly define rules for both directions. NACLs in AWS are stateless.

**Stateful firewall** — tracks connection state. If you allow inbound traffic on a connection, return traffic is automatically allowed. Security Groups in AWS are stateful.

### AWS Equivalents

| Concept | AWS Implementation |
|---|---|
| Stateful firewall (instance level) | Security Groups |
| Stateless firewall (subnet level) | Network ACLs (NACLs) |
| Web Application Firewall | AWS WAF |
| Network-level firewall | AWS Network Firewall |

### Security Groups vs NACLs

| | Security Group | NACL |
|---|---|---|
| Level | Instance/resource | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and Deny |
| Rule evaluation | All rules | In order by number, stops at first match |
| Default | Deny all inbound, allow all outbound | Allow all inbound and outbound |

---

## 10. NAT — Network Address Translation

### What NAT Does

Translates private IP addresses to a public IP for outbound internet traffic. Allows many devices with private IPs to share one public IP.

```
Private: 10.0.1.50  →  NAT  →  Public: 54.12.34.56 (internet sees this)
```

### Why It Exists

IPv4 addresses are limited. Private subnets can't reach the internet directly. NAT solves both — conserves public IPs and allows private resources outbound internet access.

### Types of NAT

**SNAT (Source NAT)** — changes the source IP. Most common. Used when private servers need to reach the internet.

**DNAT (Destination NAT)** — changes the destination IP. Used in port forwarding.

**PAT (Port Address Translation / NAT Overload)** — multiple private IPs map to one public IP, differentiated by port number. What your home router does.

### AWS NAT Gateway

- Sits in a public subnet
- Allows private subnet instances to reach the internet (software updates, API calls)
- Does NOT allow inbound connections from the internet — outbound only
- Managed, highly available within an AZ
- Costs money — one NAT Gateway per AZ for HA

```
Private EC2 → NAT Gateway (public subnet) → Internet Gateway → Internet
```

---

## 11. Load Balancing

### What a Load Balancer Does

Distributes incoming traffic across multiple backend servers to prevent any single server from being overwhelmed.

Also provides:
- Health checks — stops sending traffic to unhealthy instances
- SSL termination — decrypts HTTPS at the load balancer, sends plain HTTP to backends
- High availability — if one server dies, traffic goes to others

### Load Balancing Algorithms

| Algorithm | How it works | Use case |
|---|---|---|
| Round Robin | Each server in turn | Equal capacity servers |
| Least Connections | Server with fewest active connections | Long-lived connections |
| IP Hash | Same client always hits same server | Session persistence |
| Weighted | Servers get traffic proportional to weight | Different capacity servers |

### AWS Load Balancers

**ALB (Application Load Balancer) — Layer 7:**

- Routes based on URL path, hostname, headers, query strings
- `/api/*` → backend service A, `/static/*` → backend service B
- Supports WebSockets
- Best for HTTP/HTTPS applications and microservices

**NLB (Network Load Balancer) — Layer 4:**

- Routes based on IP and port only, no HTTP awareness
- Extremely high performance, ultra-low latency
- Handles millions of requests per second
- Best for TCP/UDP applications, gaming, real-time

**CLB (Classic Load Balancer):**

- Old generation, don't use for new applications

### Health Checks

Load balancer sends periodic requests to each target. If a target fails a defined number of checks, it's marked unhealthy and removed from rotation. Restored automatically when it passes checks again.

### SSL Termination

HTTPS terminates at the load balancer — it decrypts traffic and forwards HTTP to backends. Backends don't need certificates. Reduces CPU load on application servers.

### Sticky Sessions

Ensures the same user always hits the same backend instance (using a cookie). Useful when session state is stored locally on the server. Better long-term solution: store session in a shared cache like Redis instead.

---

## 12. VPN

### What a VPN Does

Creates an encrypted tunnel between two networks (or a device and a network) over the public internet. Traffic inside the tunnel is encrypted and appears to come from the VPN endpoint, not the original source.

### Site-to-Site VPN

Connects two entire networks — for example, your on-premises data center to your AWS VPC.

```
On-prem Network ←→ [encrypted tunnel over internet] ←→ AWS VPC
```

AWS components:
- **Virtual Private Gateway (VGW)** — AWS-side endpoint
- **Customer Gateway** — your on-prem VPN device
- Two tunnels created for redundancy

### Client VPN

Individual users connect to a network from anywhere — remote workers accessing company resources.

### VPN vs Direct Connect (AWS)

| | VPN | AWS Direct Connect |
|---|---|---|
| Path | Over public internet | Dedicated private line |
| Setup time | Minutes | Weeks/months |
| Reliability | Depends on internet | Very high, SLA-backed |
| Cost | Low | High |
| Bandwidth | Limited by internet | Up to 100 Gbps |
| Use case | Quick setup, less critical | Production, consistent performance |

---

## 13. Routing Basics

### What a Router Does

Decides where to forward packets based on their destination IP address. Uses a routing table to make this decision.

### Routing Table

Every device has one. It maps destination networks to next hops.

```bash
route -n          # view routing table (Linux)
ip route show     # modern equivalent
```

Example output:

```
Destination     Gateway         Genmask         Iface
0.0.0.0         192.168.1.1     0.0.0.0         eth0    ← default route
192.168.1.0     0.0.0.0         255.255.255.0   eth0    ← local network
10.0.0.0        10.0.0.1        255.255.0.0     eth1    ← internal route
```

The **default route** (`0.0.0.0/0`) is where all traffic goes if no more specific route matches. Usually points to the gateway/router.

### Static vs Dynamic Routing

**Static routing** — routes manually configured by an admin. Simple, predictable, doesn't adapt to network changes.

**Dynamic routing** — routers discover routes automatically and adapt when the network changes. Used in large networks.

Key dynamic routing protocols:

| Protocol | Type | Algorithm | Use |
|---|---|---|---|
| RIP | Distance Vector | Hop count | Small, old networks |
| OSPF | Link State | Dijkstra | Enterprise LANs |
| BGP | Path Vector | Best path | Internet backbone, connects AS |

**BGP** is what ISPs use to exchange routing information and is what makes the internet work. AWS also uses BGP for Direct Connect and VPN connections.

### AWS Route Tables

Every VPC has a main route table. Every subnet is associated with one route table.

Example route table for a public subnet:

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | igw-xxxxxx (Internet Gateway) |

Example route table for a private subnet:

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | nat-xxxxxx (NAT Gateway) |

---

## 14. SSH

### How SSH Works

SSH (Secure Shell) provides an encrypted channel to remotely access and manage a Linux system.

Uses asymmetric cryptography:
- You have a **private key** (kept secret, never share)
- Server has your **public key** (stored in `~/.ssh/authorized_keys`)
- When you connect, the server challenges you with something only your private key can decrypt. If it works, you're authenticated — no password needed

### Basic Usage

```bash
ssh username@ip-address
ssh -i keyfile.pem username@ip-address    # specify private key
ssh -p 2222 username@ip-address           # non-default port
ssh -v username@ip-address                # verbose output for debugging
```

### Key Management

```bash
# generate a key pair
ssh-keygen -t ed25519 -C "your-email"         # modern, recommended
ssh-keygen -t rsa -b 4096 -C "your-email"    # RSA alternative

# copy public key to remote server
ssh-copy-id username@ip-address

# manually add public key
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
```

### Important Files

```bash
~/.ssh/id_ed25519          # private key — never share, 600 permissions
~/.ssh/id_ed25519.pub      # public key — safe to share
~/.ssh/authorized_keys     # list of public keys allowed to log in
~/.ssh/known_hosts         # fingerprints of servers you've connected to
~/.ssh/config              # SSH client config (aliases, options)
```

### SSH Config File

Saves typing for frequent connections:

```
Host myserver
    HostName 192.168.1.100
    User anousheh
    IdentityFile ~/.ssh/mykey.pem
    Port 22
```

Then just: `ssh myserver`

### SSH Agent Forwarding

Allows you to SSH from one server to another using your local private key — the key never touches the intermediate server.

```bash
ssh -A username@bastion     # connect to bastion with agent forwarding
ssh username@private-server # from bastion, key comes from your local machine
```

Important for Bastion Host patterns.

### SCP — Secure Copy

Transfer files over SSH:

```bash
scp file.txt username@ip:/remote/path/           # local to remote
scp username@ip:/remote/file.txt ./local/path/   # remote to local
scp -r folder/ username@ip:/remote/path/         # recursive (directory)
scp -i keyfile.pem file.txt username@ip:/path/   # with key file
```

### SFTP

Interactive file transfer over SSH:

```bash
sftp username@ip-address
# then: get file, put file, ls, cd, mkdir, exit
```

### SSH Server Config

```bash
/etc/ssh/sshd_config    # server configuration

# Important settings:
Port 22                         # change to reduce bot scanning
PermitRootLogin no              # never allow root SSH login
PasswordAuthentication no       # key-only authentication
PubkeyAuthentication yes
AllowUsers anousheh             # whitelist specific users
```

After editing:

```bash
systemctl restart sshd
```

---

## 15. Network Security Concepts

### Defense in Depth

No single security layer is enough. Apply multiple overlapping controls:

```
Internet → WAF → Load Balancer → Security Group → Instance → Application
```

If one layer is bypassed, the next catches it.

### Principle of Least Privilege (Network)

- Open only the ports actually needed
- Restrict source IPs where possible (allow SSH only from your IP, not 0.0.0.0/0)
- Private subnets for anything that doesn't need inbound internet access

### Common Attack Types Relevant to Infrastructure

**DDoS (Distributed Denial of Service)** — flood a server with traffic until it can't respond to legitimate requests. Mitigation: AWS Shield, CloudFront, rate limiting.

**Port Scanning** — attacker probes which ports are open. Mitigation: close unnecessary ports, use non-standard ports for admin access.

**Man-in-the-Middle (MITM)** — attacker intercepts traffic between two parties. Mitigation: TLS everywhere, certificate pinning.

**Brute Force** — repeatedly trying passwords or keys. Mitigation: key-based auth only, fail2ban, account lockout policies.

**DNS Spoofing** — attacker poisons DNS cache with fake records. Mitigation: DNSSEC, use trusted resolvers.

### ICMP and `ping`

ICMP is used by `ping` and `traceroute`. Not TCP or UDP — its own protocol.

```bash
ping 8.8.8.8                     # basic connectivity check
ping -c 4 8.8.8.8                # send exactly 4 packets
traceroute 8.8.8.8               # trace path to destination
mtr 8.8.8.8                      # live traceroute, better than traceroute
```

AWS security groups block ICMP by default — you must explicitly allow it to ping an EC2 instance.

### `netstat` and `ss`

Check what ports are open and what's listening:

```bash
ss -tuln                          # listening ports (modern)
netstat -tuln                     # same, older tool
ss -tp                            # show established connections with process
lsof -i :80                       # what process is using port 80
```

### `curl` and `wget` for Network Testing

```bash
curl -I https://example.com               # headers only
curl -v https://example.com               # verbose, shows TLS handshake
curl -o /dev/null -w "%{http_code}" URL   # just the status code
curl --resolve example.com:443:1.2.3.4 https://example.com  # test specific IP
wget https://example.com/file.zip         # download file
```

---

## 16. Cloud Networking Concepts

### VPC — Virtual Private Cloud

Your private network inside AWS. Completely isolated from other accounts.

Key components:

| Component | Role |
|---|---|
| VPC | The isolated network boundary |
| Subnet | Subdivides the VPC. Public or private |
| Internet Gateway | Allows internet traffic in/out of public subnets |
| NAT Gateway | Allows private subnets outbound internet access only |
| Route Table | Rules for where traffic is directed |
| Security Group | Stateful instance-level firewall |
| NACL | Stateless subnet-level firewall |
| VPC Peering | Connect two VPCs to route traffic between them |
| Transit Gateway | Hub connecting multiple VPCs and on-prem networks |
| Endpoints | Private connection to AWS services without internet |

### VPC Peering

Connects two VPCs so they can communicate using private IPs. Traffic stays on AWS backbone, doesn't go over the internet.

- Works within an account or across accounts
- Works across regions (inter-region peering)
- Not transitive — if VPC A peers with B, and B peers with C, A cannot reach C through B

### VPC Endpoints

Allow your VPC to connect to AWS services (S3, DynamoDB, etc.) without going through the internet.

- **Interface Endpoint** — ENI with private IP in your subnet. For most services
- **Gateway Endpoint** — entry in route table. Only for S3 and DynamoDB, free

### Transit Gateway

Connects multiple VPCs and on-premises networks through a central hub. Solves the peering complexity problem at scale.

### CDN — Content Delivery Network

Caches content at edge locations geographically close to users.

AWS equivalent: **CloudFront**. Reduces latency, offloads traffic from origin servers, provides DDoS protection.

### DNS in Cloud

Always use private hosted zones in your VPC for internal service discovery. Public hosted zones for external-facing domains.

In Route 53:
- **Public hosted zone** — resolves from anywhere on internet
- **Private hosted zone** — resolves only within associated VPCs

---

## Quick Reference — Troubleshooting Checklist

When something can't connect:

```
1. Can you ping the IP?
   No → routing issue or ICMP blocked (security group/NACL)

2. Can you reach the port?
   telnet ip port  OR  nc -zv ip port
   No → security group, NACL, or service not running

3. Is the service running?
   systemctl status servicename
   ss -tuln | grep port

4. Check security group — inbound rules allow the port?

5. Check NACL — both inbound and outbound rules?
   (Remember NACLs are stateless — need both directions)

6. Check route table — correct route for the destination?

7. Check DNS — resolves to correct IP?
   dig hostname

8. Check logs
   /var/log/nginx/error.log
   /var/log/messages
   journalctl -u servicename
```

---

*Notes maintained as part of Cloud + Linux learning journey. GitHub: [your repo link]*
