# AWS — Auto Scaling, Target Groups, Bastion Host, SSH, SCP + Production Architecture Guide

---

## Auto Scaling Groups (ASG)

### What Is an Auto Scaling Group?

An **Auto Scaling Group** is an AWS service that **automatically manages a group of EC2 instances** — adding more when traffic increases and removing them when traffic drops.

Without ASG: You manually decide how many EC2 instances to run. During a traffic spike your servers crash. During off-hours you're paying for idle servers.

With ASG: AWS watches your traffic/CPU/load and automatically scales the number of EC2 instances to match demand. You always have the right number of servers — not too many, not too few.

### The Core Concept — Three Numbers

Every Auto Scaling Group is defined by three numbers:

| Setting | What It Means |
|---|---|
| **Minimum** | Always keep at least this many instances running. Never go below this. |
| **Maximum** | Never create more than this many instances. Your hard ceiling. |
| **Desired** | The target number right now. ASG will always try to maintain this count. |

Example: Min=2, Desired=2, Max=6.
- Normally 2 instances run.
- Traffic spikes → CPU hits 80% → ASG launches more until Desired=6.
- Traffic drops → ASG terminates excess → back to Desired=2, never below Min=2.

---

### How Does ASG Know When to Scale? — Scaling Policies

ASG uses **Scaling Policies** to decide when to add or remove instances.

**1. Target Tracking Scaling (Most Common)**
You pick a metric and a target value. ASG automatically adds/removes instances to keep that metric at the target.

Example: "Keep average CPU utilization at 50%."
- CPU goes to 80% → ASG adds instances until CPU comes down to 50%
- CPU drops to 20% → ASG removes instances until CPU comes back to 50%

Other common metrics: ALB request count per target, average network in/out.

**2. Step Scaling**
Define specific actions at specific thresholds.

Example:
- CPU > 70% → add 2 instances
- CPU > 90% → add 4 instances
- CPU < 30% → remove 1 instance

**3. Scheduled Scaling**
Scale at specific times you know in advance.

Example: Your app gets heavy traffic every weekday at 9 AM. Schedule ASG to increase to 5 instances at 8:45 AM and scale back at 6 PM.

**4. Predictive Scaling**
AWS uses machine learning to analyze historical traffic patterns and scales proactively before traffic arrives. AWS predicts the spike and launches instances in advance.

---

### Launch Template — The Blueprint for New Instances

When ASG needs to launch a new instance, it follows a **Launch Template** — a saved configuration that defines exactly what every new instance looks like.

A Launch Template contains:
- Which AMI (operating system) to use
- Instance type (t3.micro, m5.large, etc.)
- Key pair for SSH
- Security Groups to attach
- IAM Role
- User Data script (commands to run on first boot — install app, pull from GitHub, etc.)
- EBS volume settings

This is how ASG creates 5 identical instances at once — it copies the exact Launch Template settings for each one.

**User Data Script example** (runs once when instance boots):
```bash
#!/bin/bash
sudo apt update -y
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
echo "<h1>Server $(hostname)</h1>" > /var/www/html/index.html
```

Every new instance ASG creates will automatically install nginx and start serving, without you having to manually SSH in.

---

### ASG and the Load Balancer — How They Work Together

ASG and ALB are always used together in production.

- ASG creates/destroys EC2 instances based on load
- ALB distributes traffic across those instances
- ASG automatically **registers new instances with the ALB** when it creates them
- ASG automatically **deregisters instances from the ALB** before terminating them (waits for in-flight requests to finish — this is called **connection draining**)

The ALB sends traffic only to healthy, registered instances. If ASG creates 3 more instances during a spike, the ALB starts routing to all 5 (original 2 + 3 new) within seconds.

---

### Health Checks in ASG

ASG performs health checks on every instance. If an instance becomes unhealthy, ASG **automatically terminates it and launches a replacement**.

Two types:
- **EC2 Health Check (default):** Checks if the EC2 instance itself is running (system status check)
- **ELB Health Check:** Uses the Load Balancer's health check — if the ALB's HTTP check to the app fails, ASG considers it unhealthy

ELB health check is stricter and better for production — it catches cases where the instance is running but the application itself has crashed.

---

### ASG Cooldown Period

After ASG launches or terminates instances, it waits for a **cooldown period** (default 300 seconds) before taking any more scaling actions.

Why: After launching new instances, it takes a minute for them to start and register. Without cooldown, ASG might see CPU still high and launch more instances unnecessarily. Cooldown gives things time to stabilize before reacting again.

---

### Termination Policy — Which Instance Gets Killed First?

When ASG scales in (removes instances), it follows a termination policy. The default:
1. Remove from the AZ with the most instances (for balance)
2. Among those, terminate the instance with the oldest Launch Template version
3. If tied, terminate the one closest to its next billing hour

This keeps your infrastructure balanced across AZs and running the latest configuration.

---

## Target Groups

### What Is a Target Group?

A **Target Group** is a logical grouping of resources (EC2 instances, IPs, or Lambda functions) that a Load Balancer routes traffic to.

The ALB doesn't directly know about your EC2 instances. Instead:
- You create a Target Group
- You register your EC2 instances in that Target Group
- You tell the ALB: "Send traffic to THIS Target Group"

Think of it as a contact list for the ALB. The ALB calls the list, the list knows who's available and healthy.

---

### What Goes Inside a Target Group

**Target Type:**
- **Instance** → EC2 instances (most common)
- **IP** → Specific private IP addresses (for containers, on-premise servers)
- **Lambda** → A Lambda function as the backend

**Protocol and Port:**
The port your application is listening on — e.g., port 80 for HTTP, port 3000 for a Node.js app, port 8080 for a Java app.

**Health Check Settings:**
- Protocol and path: HTTP GET `/health` (a route in your app that returns 200 OK if healthy)
- Healthy threshold: 2 consecutive successes = healthy
- Unhealthy threshold: 3 consecutive failures = unhealthy
- Interval: Check every 30 seconds
- Timeout: Wait 5 seconds for a response

The Load Balancer **only sends traffic to healthy targets**. Unhealthy ones are automatically excluded until they recover.

---

### Target Group + ALB + ASG — How All Three Connect

```
Internet
  ↓
[ALB — Listener on port 443]
  ↓  (Listener Rule: forward to Target Group)
[Target Group: port 80, health check /health]
  ↓  (Routes to registered healthy targets)
[EC2 Instance 1]  [EC2 Instance 2]  [EC2 Instance 3]
       ↑ Auto Scaling Group manages these ↑
```

The ALB has a **Listener** (port 443) with a **Rule** (forward all traffic to Target Group X).
The Target Group knows which EC2s are registered and healthy.
ASG registers/deregisters EC2s in the Target Group as it scales.

---

### Why Use a Target Group Instead of Direct ALB-to-EC2?

Target Groups let you:
- **Decouple the ALB from individual instances** — you can replace all instances without changing ALB config
- **Use path-based routing** — `/api/*` → Target Group A (API servers), `/images/*` → Target Group B (image servers)
- **Use host-based routing** — `api.myapp.com` → Target Group A, `admin.myapp.com` → Target Group B
- **Run multiple apps behind one ALB** — each app gets its own Target Group and port

---

## Bastion Host / Jump Server

### The Problem

Your application EC2 instances are in a **private subnet** — they have no public IP, no direct internet access. This is correct for security. But how do you SSH into them to deploy code, check logs, or troubleshoot?

You can't SSH directly because there's no public route to them. This is where a **Bastion Host** solves the problem.

---

### What Is a Bastion Host?

A **Bastion Host** (also called a Jump Server) is a **small, hardened EC2 instance in the public subnet** that acts as the ONLY SSH entry point into your private infrastructure.

You SSH into the Bastion first. From the Bastion, you SSH into the private EC2 instances. The private instances only allow SSH from the Bastion's IP — nobody else.

```
Your Laptop
    ↓  SSH (port 22, your IP allowed)
[Bastion Host — Public Subnet — has Public IP]
    ↓  SSH (port 22, only Bastion's IP allowed)
[Private EC2 — Private Subnet — no public IP]
```

---

### Security Group Setup for Bastion

**Bastion Host Security Group:**
| Direction | Port | Protocol | Source |
|---|---|---|---|
| Inbound | 22 | TCP | YOUR specific IP only (not 0.0.0.0/0!) |
| Outbound | All | All | Anywhere |

**Private EC2 Security Group:**
| Direction | Port | Protocol | Source |
|---|---|---|---|
| Inbound | 22 | TCP | Bastion Host's Security Group (not IP — SG reference) |
| Inbound | 80/443 | TCP | ALB's Security Group |
| Outbound | All | All | Anywhere |

The private EC2 only accepts SSH from the Bastion's SG — not from the whole internet.

---

### How to Set Up a Bastion Host in AWS

**Step 1: Launch a small EC2 in the public subnet**
- AMI: Amazon Linux 2 or Ubuntu
- Instance type: t2.micro (free tier — bastion doesn't need power)
- Subnet: Public subnet
- Auto-assign Public IP: Enabled
- Key pair: Same key pair as your private instances (simpler) or a different one

**Step 2: Assign the Bastion its Security Group**
Allow only port 22 from your IP.

**Step 3: Copy your private key to the Bastion (or use SSH agent forwarding)**

There are two methods:

**Method A — Copy the .pem key to the Bastion (simpler but less secure):**
```bash
# From your laptop, copy the key to the Bastion
scp -i "mykey.pem" mykey.pem ubuntu@<BASTION-PUBLIC-IP>:~/.ssh/

# SSH into Bastion
ssh -i "mykey.pem" ubuntu@<BASTION-PUBLIC-IP>

# From Bastion, SSH into private EC2
ssh -i "~/.ssh/mykey.pem" ubuntu@<PRIVATE-EC2-IP>
```

**Method B — SSH Agent Forwarding (more secure — key never leaves your laptop):**
```bash
# Add key to SSH agent on your laptop
ssh-add mykey.pem

# SSH into Bastion with agent forwarding enabled (-A flag)
ssh -A -i "mykey.pem" ubuntu@<BASTION-PUBLIC-IP>

# From Bastion, SSH into private EC2 — uses your laptop's key via the agent
ssh ubuntu@<PRIVATE-EC2-IP>
```

Agent forwarding means your private key never gets copied to the Bastion. The Bastion just relays the authentication challenge back to your laptop. More secure.

---

### Modern Alternative — AWS Systems Manager Session Manager

AWS now offers **Session Manager** — you can connect to private EC2 instances directly from the AWS Console or CLI, with NO bastion host, NO public IP, and NO port 22 open anywhere.

How: Attach an IAM Role with `AmazonSSMManagedInstanceCore` policy to your EC2. SSM Agent (pre-installed on Amazon Linux 2) connects to the SSM service over HTTPS. You open a browser-based shell session through AWS Console.

For production DevOps work at companies, Session Manager is replacing Bastion Hosts because it's more secure (no SSH keys to manage, all sessions are logged in CloudWatch).

For learning and interviews: Know both. Bastion host is still very commonly asked.

---

## SSH — Secure Shell

### What Is SSH?

**SSH (Secure Shell)** is a protocol that allows you to **remotely control a Linux server over an encrypted connection**. When you SSH into an EC2 instance, you get a terminal — as if you were sitting physically in front of that server.

Everything you type in the SSH session executes on the remote server, not your laptop.

---

### SSH Syntax

```bash
ssh -i "path/to/key.pem" username@hostname-or-ip
```

**Breaking it down:**
- `ssh` → the command
- `-i "path/to/key.pem"` → identity file — your private key for authentication
- `username` → the default Linux user for the AMI:
  - Ubuntu AMI → `ubuntu`
  - Amazon Linux 2 AMI → `ec2-user`
  - CentOS/RHEL → `ec2-user` or `centos`
  - Debian → `admin`
- `hostname-or-ip` → the public IP or DNS of the EC2

**Examples:**
```bash
# SSH into an Ubuntu EC2
ssh -i "mykey.pem" ubuntu@54.123.45.67

# SSH into Amazon Linux 2
ssh -i "mykey.pem" ec2-user@54.123.45.67

# SSH into a private EC2 (from inside the Bastion — no -i needed if using agent)
ssh ubuntu@10.0.2.5

# SSH with custom port (if SSH is running on port 2222 instead of 22)
ssh -i "mykey.pem" -p 2222 ubuntu@54.123.45.67

# SSH and immediately run a command (without interactive shell)
ssh -i "mykey.pem" ubuntu@54.123.45.67 "sudo systemctl status nginx"

# SSH with agent forwarding (for jumping to private instances)
ssh -A -i "mykey.pem" ubuntu@<BASTION-IP>
```

---

### What Happens When You SSH Into an EC2

When you run `ssh -i mykey.pem ubuntu@54.123.45.67`, here's exactly what happens:

1. **Your laptop initiates a TCP connection** to port 22 of the EC2
2. **The EC2's SSH server (sshd) responds** with its public key fingerprint — the first time you connect, it asks: "Are you sure you want to continue connecting?" → type `yes`. This fingerprint is then saved to `~/.ssh/known_hosts` on your laptop.
3. **Key-based authentication:** Your laptop proves identity using the `.pem` private key. The EC2 has the matching public key stored in `~/.ssh/authorized_keys` (placed there by AWS when the instance was launched). If they match → you're authenticated.
4. **An encrypted shell session opens.** You're now at the command prompt of the remote EC2. Every command you type runs on that server.
5. **When you type `exit` or close the terminal**, the SSH session ends. Processes you started interactively (like running a script) will stop. Processes running as services (nginx, pm2 managed apps) continue running.

**Important:** The `.pem` key file on your laptop must have strict permissions:
```bash
chmod 400 mykey.pem
```
If permissions are too open (other users can read it), SSH refuses to use the key and throws a "WARNING: UNPROTECTED PRIVATE KEY FILE" error.

---

## SCP — Secure Copy

### What Is SCP?

**SCP (Secure Copy Protocol)** is a command that lets you **securely transfer files between your local machine and a remote server** (or between two remote servers) over SSH. It uses the same SSH authentication (same key pair).

SCP is how you upload your code, configs, or any files to an EC2 instance, or download log files from it.

---

### SCP Syntax

```bash
# Upload: local → remote
scp -i "key.pem" /local/path/to/file username@remote-ip:/remote/destination/path

# Download: remote → local
scp -i "key.pem" username@remote-ip:/remote/path/to/file /local/destination/path

# Upload a directory recursively (-r flag)
scp -i "key.pem" -r /local/folder/ username@remote-ip:/remote/destination/

# Download a directory recursively
scp -i "key.pem" -r username@remote-ip:/remote/folder/ /local/destination/
```

---

### SCP Examples

```bash
# Upload a single file to EC2
scp -i "mykey.pem" app.js ubuntu@54.123.45.67:/home/ubuntu/

# Upload entire project folder to EC2
scp -i "mykey.pem" -r ./my-node-app/ ubuntu@54.123.45.67:/home/ubuntu/

# Download a log file from EC2 to current directory
scp -i "mykey.pem" ubuntu@54.123.45.67:/var/log/nginx/error.log ./

# Copy key to Bastion (to then jump to private instances)
scp -i "mykey.pem" mykey.pem ubuntu@<BASTION-IP>:~/.ssh/

# Upload to a specific port (if SSH runs on port 2222)
scp -i "mykey.pem" -P 2222 app.js ubuntu@54.123.45.67:/home/ubuntu/
```

**Note:** In SCP, the port flag is `-P` (capital P), while in SSH it's `-p` (lowercase). This is a common mistake.

---

### When to Use SCP vs Other Methods

| Method | When to Use |
|---|---|
| **SCP** | Quick one-off file transfer, copying configs, uploading keys |
| **Git pull on server** | Deploying code — better for production (version controlled) |
| **AWS S3** | Large files, multiple servers need same file |
| **MobaXterm SFTP** | Windows users who prefer a GUI drag-and-drop interface |
| **rsync** | Sync large directories efficiently (only transfers changed files) |

In real DevOps work, you rarely use SCP for code deployment — you use Git or CI/CD pipelines. SCP is used for:
- Transferring config files
- Uploading `.pem` keys to a Bastion
- Downloading log files for local analysis
- Quick one-time transfers

---

---

## The Production Architecture — Image Explained

The image shows a **standard production-grade 3-tier AWS architecture** across two Availability Zones. This is the most commonly deployed pattern in real companies.

Here's every component in the image and what it does:

### AWS Services Shown at the Top
The icons at the top of the diagram (outside the VPC) represent AWS managed services your EC2 instances interact with:
- **S3 bucket** → Object storage for files, images, backups, static assets
- **RDS (Database)** → Managed relational database (MySQL, PostgreSQL)
- **CloudWatch** → Monitoring, logs, alarms, metrics
- **Systems Manager / CodeDeploy** → Deployment management, patching
- **Another service** (possibly ECR or SNS) → Container registry or notifications

These are outside the VPC because they're managed AWS services — you connect to them via VPC Endpoints, the internet, or AWS's private network.

### What The Architecture Does — Flow of a User Request

```
User on internet
      ↓
Application Load Balancer (spans both AZs, in public subnets)
      ↓
Target Group (registered EC2 instances in private subnets)
      ↓
EC2 Server (in Private Subnet, AZ-1 or AZ-2)
      ↓
RDS / S3 / ElastiCache (via VPC Endpoint or private connection)
```

### Component-by-Component Breakdown

**VPC**
The outer boundary. Everything inside is your isolated private network.

**Two Availability Zones**
Everything is deployed across AZ-1 and AZ-2. If AZ-1 has an outage, AZ-2 keeps serving traffic. This is High Availability.

**Public Subnets (one per AZ)**
Each public subnet contains:
- A **NAT Gateway** — lets the private EC2s download updates and reach the internet outbound without being reachable from outside
- The **Application Load Balancer** spans across BOTH public subnets — it lives in the public subnets and receives internet traffic

**Private Subnets (one per AZ)**
Each private subnet contains:
- A **Server (EC2 instance)** — your actual application running here
- No public IPs — nobody from the internet can reach these directly

**Auto Scaling Group (wraps both private subnets)**
The orange box labeled "Auto Scaling group" wraps the servers in both private subnets. ASG monitors load and:
- If traffic spikes → launches more EC2s into both private subnets
- If traffic drops → terminates excess EC2s
- If an EC2 dies → automatically replaces it

**Security Group (around the servers)**
The blue outline labeled "Security group" around the servers controls:
- Allow inbound on app port ONLY from the ALB's Security Group
- Allow inbound SSH ONLY from the Bastion's Security Group (not shown in diagram but implied)
- Allow all outbound (for NAT Gateway access)

**S3 Gateway (VPC Endpoint)**
The icon on the far left labeled "S3 gateway" is a **VPC Gateway Endpoint for S3**. Your private EC2s can access S3 directly through AWS's private network — no internet, no NAT Gateway charges for S3 traffic. Free and faster.

---

## How to Build This Architecture in AWS — Step by Step

This is the sequence of steps to recreate the image in your AWS console.

### Step 1: Create the VPC
- VPC → Create VPC
- IPv4 CIDR: `10.0.0.0/16`
- No IPv6, Tenancy: Default

### Step 2: Create Subnets

Create 4 subnets across 2 AZs:

| Subnet | AZ | CIDR | Type |
|---|---|---|---|
| public-subnet-1 | ap-south-1a | 10.0.1.0/24 | Public |
| public-subnet-2 | ap-south-1b | 10.0.2.0/24 | Public |
| private-subnet-1 | ap-south-1a | 10.0.3.0/24 | Private |
| private-subnet-2 | ap-south-1b | 10.0.4.0/24 | Private |

For both public subnets: enable "Auto-assign public IPv4 address."

### Step 3: Create and Attach Internet Gateway
- Create IGW → attach to your VPC

### Step 4: Create NAT Gateways (one per public subnet)
- Create NAT Gateway in public-subnet-1 → allocate a new Elastic IP
- Create NAT Gateway in public-subnet-2 → allocate a new Elastic IP
- Wait for both to become Available (takes 1–2 minutes)

### Step 5: Create Route Tables

**Public Route Table:**
- Create route table → associate with public-subnet-1 and public-subnet-2
- Add route: `0.0.0.0/0` → Internet Gateway

**Private Route Table 1 (for AZ-1):**
- Create route table → associate with private-subnet-1
- Add route: `0.0.0.0/0` → NAT Gateway in public-subnet-1

**Private Route Table 2 (for AZ-2):**
- Create route table → associate with private-subnet-2
- Add route: `0.0.0.0/0` → NAT Gateway in public-subnet-2

### Step 6: Create Security Groups

You need 3 Security Groups:

**SG-1: ALB Security Group**
| Rule | Port | Source |
|---|---|---|
| Inbound HTTP | 80 | 0.0.0.0/0 |
| Inbound HTTPS | 443 | 0.0.0.0/0 |
| Outbound | All | Anywhere |

**SG-2: EC2 Application Security Group**
| Rule | Port | Source |
|---|---|---|
| Inbound | 80 (or your app port) | SG-1 (ALB Security Group) |
| Inbound SSH | 22 | SG-3 (Bastion Security Group) |
| Outbound | All | Anywhere |

**SG-3: Bastion Host Security Group**
| Rule | Port | Source |
|---|---|---|
| Inbound SSH | 22 | YOUR IP only |
| Outbound | All | Anywhere |

### Step 7: Launch Bastion Host
- EC2 → Launch Instance
- Subnet: public-subnet-1, Auto-assign public IP: Yes
- Security Group: SG-3 (Bastion SG)
- Instance type: t2.micro
- Key pair: create or use existing

### Step 8: Create a Launch Template for the App Servers
- EC2 → Launch Templates → Create
- AMI: Ubuntu 22.04
- Instance type: t2.micro
- Security Group: SG-2 (App SG)
- Key pair: same as Bastion
- User Data: script to install and start your app automatically

### Step 9: Create Target Group
- EC2 → Target Groups → Create
- Target type: Instances
- Protocol: HTTP, Port: 80 (or your app port)
- VPC: your VPC
- Health check path: `/` or `/health`
- Don't register any targets yet — ASG will do it automatically

### Step 10: Create Application Load Balancer
- EC2 → Load Balancers → Create → Application Load Balancer
- Scheme: Internet-facing
- VPC: your VPC
- Subnets: public-subnet-1 and public-subnet-2 (select BOTH)
- Security Group: SG-1 (ALB SG)
- Listener: HTTP port 80 → forward to your Target Group

### Step 11: Create Auto Scaling Group
- EC2 → Auto Scaling Groups → Create
- Launch Template: select the one you created in Step 8
- VPC and Subnets: select private-subnet-1 and private-subnet-2
- Load balancing: attach to existing Load Balancer → select your Target Group
- Health checks: enable ELB health checks
- Group size: Min=1, Desired=2, Max=4
- Scaling policy: Target Tracking → CPU = 50%

ASG will launch 2 instances (one in each AZ), register them in the Target Group, and the ALB will start sending traffic.

### Step 12: Create S3 VPC Endpoint (Gateway)
- VPC → Endpoints → Create
- Service: `com.amazonaws.<region>.s3` (Gateway type)
- Select your VPC and both private route tables
- This adds a route to both private route tables so S3 access goes through AWS's network

### Step 13: Test It
- Go to the ALB's DNS name (from ALB details page)
- Paste it in your browser → your app should load
- The ALB distributes across both EC2 instances

---

*🩷 Read about my Cloud/DevOps Journey here: https://cloudgirllogs.hashnode.dev/*
