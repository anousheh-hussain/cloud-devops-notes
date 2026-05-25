[AWS_EC2_Notes.md](https://github.com/user-attachments/files/28220546/AWS_EC2_Notes.md)
# AWS EC2 — Notes + Interview Questions

---

## What Does "EC2" Actually Stand For?

**EC2 = Elastic Compute Cloud**

Let's break each word down:

### "Elastic"
Elastic means **it can grow or shrink based on your needs** — automatically or manually.

Think of a rubber band. You can stretch it when you need more, and it comes back when you don't.

In EC2's case:
- Need more servers during a big sale? Scale up.
- Traffic drops at night? Scale down.
- You only pay for what you use.

This is one of the most important ideas in cloud computing — you're not stuck with a fixed amount of resources.

### "Compute"
Compute simply means **processing power** — CPU, RAM, the ability to run code and applications. EC2 gives you virtual machines in the cloud that can run anything — web apps, databases, scripts, ML models, etc.

### "Cloud"
Cloud means these machines are **hosted on AWS's infrastructure**, not in your office or data center. You access them over the internet.

### What Does the "2" Mean?
The "2" comes from **"Elastic Compute Cloud" having two C's** — EC**C** = E**C**2. AWS just stylized it as EC2. It's not a version number — it's simply a naming convention AWS used to make the abbreviation EC² (like a mathematical square), meaning "EC" squared.

---

## What Is a Hypervisor? (Important Concept)

### First, What's a Virtual Machine (VM)?

A physical server is one big computer with lots of CPU, RAM, and storage.

A **Virtual Machine** is like splitting that one big computer into multiple smaller, separate computers — each running its own operating system, fully isolated from each other.

For example:
- One physical server with 64 CPU cores and 256 GB RAM
- Split into 8 VMs — each gets 8 cores and 32 GB RAM
- Each VM thinks it's a standalone machine. They don't interfere with each other.

### So What is a Hypervisor?

A **Hypervisor** is the software that sits between the physical hardware and the virtual machines. It is responsible for:

- **Creating** VMs on top of physical hardware
- **Allocating** CPU, RAM, and storage to each VM
- **Isolating** VMs from each other so one VM can't see or affect another

Without a hypervisor, you can't have virtual machines.

**Popular hypervisors:**
- VMware ESXi
- Microsoft Hyper-V
- KVM (used inside AWS itself)
- VirtualBox (for personal use on your laptop)

AWS uses a customized hypervisor called **AWS Nitro** (based on KVM) under the hood for EC2.

---

## Why Not Just Buy a Hypervisor and Run Your Own VMs?

This is a great question and comes up in interviews too.

Technically, yes — you can buy a powerful physical server, install VMware on it, and create 10 virtual machines yourself. But here's why real companies don't just do that:

### 1. Massive Upfront Cost
A production-grade physical server with good specs can cost ₹5–20 lakhs or more. You're spending that money before you even have a single user.

With AWS EC2, you pay per hour/second. You can start for less than ₹1/hour for a small instance.

### 2. You Have to Predict the Future
When you buy hardware, you have to decide upfront:
- How many servers do I need?
- How much RAM? How much storage?

If you over-buy → you're wasting money on idle hardware.
If you under-buy → your site goes down when traffic spikes.

With EC2, you just spin up more instances in seconds when you need them.

### 3. Physical Maintenance is Your Problem
With your own servers, you are responsible for:
- Hardware failures (a disk dies? You fix it)
- Power supply issues
- Cooling systems for the server room
- Physical security of the machines
- Network connectivity and redundancy

AWS handles ALL of this. They have massive data centers with redundant power, cooling, security, and hardware. If a physical machine fails, your EC2 instance is automatically moved.

### 4. No Global Reach
If your users are in India, USA, and Europe — you'd need physical servers in all three locations. That costs crores and takes months to set up.

With AWS, you click a button and deploy your application in a data center in the US, Europe, or anywhere in seconds.

### 5. No Elasticity
Your physical server has fixed resources. During a traffic spike, you can't magically add more RAM to it. You'd have to buy new hardware, rack it, configure it, and that takes days or weeks.

EC2 can scale up or down in minutes, automatically.

### Summary Table

| Factor | Your Own Hypervisor (On-Premises) | AWS EC2 |
|---|---|---|
| Upfront cost | Very high | Zero |
| Scaling | Manual, slow, limited | Automatic, instant |
| Maintenance | Your responsibility | AWS's responsibility |
| Global deployment | Expensive and complex | Few clicks |
| Pay model | Fixed (whether used or not) | Pay only for what you use |
| Disaster recovery | You set it up | Built-in options available |

This is why businesses — from startups to enterprises — use cloud providers like AWS instead of running their own hypervisors.

---

## Key Terms You Must Know

### Region
A **Region** is a physical geographic location where AWS has its data centers. For example:
- `ap-south-1` → Mumbai, India
- `us-east-1` → North Virginia, USA
- `eu-west-1` → Ireland, Europe

You choose a region when launching EC2 instances. Always pick the region **closest to your users** for best performance.

### Availability Zone (AZ)
Inside each Region, there are multiple **Availability Zones** — completely separate data centers that are physically isolated from each other but connected with high-speed private networking.

For example, Mumbai Region (`ap-south-1`) has 3 AZs: `ap-south-1a`, `ap-south-1b`, `ap-south-1c`.

**Why does this matter?**
If one AZ has a power failure or fire, the others are completely unaffected. If you deploy your app across 2 AZs, and one goes down — your app is still running in the other. This is called **High Availability**.

**Rule of thumb:** Always deploy across at least 2 AZs for production applications.

### Latency
**Latency** is the time it takes for data to travel from the server to the user and back. It's measured in milliseconds (ms).

- Low latency = fast response. Users experience a snappy app.
- High latency = slow response. Users experience lag.

**Why it matters for EC2:**
If your EC2 instance is in the US but your users are in India, every request has to travel across the world — adding 200–300ms of latency. This makes your app feel slow.

That's why you pick the region closest to your users — it keeps latency low.

### Instance
An **Instance** is just another word for a virtual machine running on EC2. When you "launch an EC2 instance," you're creating and starting a VM in AWS.

---

## Core EC2 Concepts

### 1. AMI — Amazon Machine Image

An AMI is a **template or snapshot** that defines what your EC2 instance will look like when it starts. It includes:
- The operating system (Ubuntu, Amazon Linux, Windows, etc.)
- Any pre-installed software
- Configuration settings

Think of it like a **USB bootable drive** — you use it to set up a machine exactly the way you want.

You can use:
- **AWS-provided AMIs** — Amazon Linux 2, Ubuntu 22.04, Windows Server, etc.
- **Community AMIs** — shared by others (use with caution)
- **Custom AMIs** — you build and save your own (great for deploying identical servers)

### 2. Instance Types

EC2 gives you many different "sizes" of virtual machines. These are called Instance Types.

Each instance type has a name like `t3.micro`, `m5.large`, `c6i.xlarge`.

**How to read an instance type name:**

`t3.micro`
- `t` → Family (General Purpose, Burstable)
- `3` → Generation (3rd gen — newer = better performance/price)
- `micro` → Size (nano < micro < small < medium < large < xlarge < 2xlarge...)

**Common Instance Families:**

| Family | Best For | Examples |
|---|---|---|
| `t` (General/Burstable) | Dev/test, low-traffic apps | t3.micro, t3.small |
| `m` (General Purpose) | Balanced workloads | m5.large, m6i.xlarge |
| `c` (Compute Optimized) | CPU-heavy tasks, gaming servers | c6i.2xlarge |
| `r` (Memory Optimized) | Databases, caching, big data | r6g.large |
| `i` (Storage Optimized) | High IOPS, fast local storage | i3.xlarge |
| `g` / `p` (GPU) | Machine learning, video encoding | g4dn.xlarge |

**Free Tier:** `t2.micro` or `t3.micro` is free for 750 hours/month in the first 12 months.

### 3. Security Groups

A **Security Group** is a virtual firewall for your EC2 instance. It controls:
- **Inbound traffic** → who can connect TO your instance
- **Outbound traffic** → what connections your instance can make OUT

Rules are defined by Protocol, Port, and Source/Destination.

**Example rules for a web server:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Your IP only |
| HTTP | TCP | 80 | Anywhere (0.0.0.0/0) |
| HTTPS | TCP | 443 | Anywhere (0.0.0.0/0) |

**Important:** Security groups are **stateful** — if you allow inbound traffic on port 80, the response is automatically allowed back out. You don't need to add a separate outbound rule for responses.

**Key rule:** By default, all inbound is denied. All outbound is allowed.

### 4. Key Pairs

When you launch an EC2 Linux instance, you don't use a username/password to connect. Instead, you use **SSH Key Pairs** — which are much more secure.

A key pair has two parts:
- **Public Key** → AWS stores this on your EC2 instance
- **Private Key (.pem file)** → You download and keep this on your laptop. Never share it.

When you try to SSH into your instance, your private key and the public key on the server are matched cryptographically. If they match, you're in.

**Important:** If you lose your `.pem` file, you cannot connect to that instance again (unless you use AWS Systems Manager as an alternative).

### 5. EBS — Elastic Block Store

**EBS** is the storage attached to your EC2 instance — essentially the hard drive for your VM.

- When you launch EC2, it automatically comes with a **root EBS volume** (like the C: drive on Windows)
- You can attach additional EBS volumes (like adding more hard drives)
- EBS volumes **persist independently** — even if you stop or terminate the EC2 instance, you can keep the EBS volume and its data (depending on your setting)
- EBS is specific to an AZ — it can only be attached to instances in the same AZ

**Types of EBS volumes:**
- **gp3 / gp2** → General Purpose SSD (most common, good for most use cases)
- **io1 / io2** → Provisioned IOPS SSD (for databases needing very fast read/write)
- **st1** → Throughput Optimized HDD (for big data, logs)
- **sc1** → Cold HDD (cheapest, for infrequently accessed data)

### 6. Elastic IP Address

By default, when you stop and restart an EC2 instance, its **public IP address changes**.

An **Elastic IP** is a **static public IP address** that stays the same — you can assign it to any instance and it won't change even if you restart.

Use case: Your app's domain name (`myapp.com`) is pointing to an IP. If the IP keeps changing, your domain breaks. An Elastic IP fixes this.

**Note:** Elastic IPs are free only when associated with a running instance. If it's sitting unused, AWS charges you for it.

### 7. EC2 Instance States

| State | What It Means |
|---|---|
| **Running** | Instance is on, you're being charged |
| **Stopped** | Instance is off, no compute charge (but EBS storage is still charged) |
| **Terminated** | Instance is permanently deleted, cannot be recovered |
| **Pending** | Instance is starting up |

### 8. EC2 Pricing Models

| Model | How It Works | Best For |
|---|---|---|
| **On-Demand** | Pay per hour/second, no commitment | Dev/test, unpredictable traffic |
| **Reserved Instances** | Commit to 1 or 3 years, get up to 72% discount | Steady, predictable workloads |
| **Spot Instances** | Use AWS's spare capacity at up to 90% discount — but can be interrupted | Batch jobs, data processing, fault-tolerant apps |
| **Savings Plans** | Flexible commitment to a spend amount per hour | Good middle ground between On-Demand and Reserved |

### 9. Auto Scaling

Auto Scaling automatically **adds or removes EC2 instances** based on demand.

- Traffic spikes → Auto Scaling adds more instances
- Traffic drops → Auto Scaling removes instances to save cost

You define:
- **Minimum instances** — always keep at least this many running
- **Maximum instances** — never go above this many
- **Desired capacity** — the target number when traffic is normal

### 10. Load Balancer (ELB)

A **Load Balancer** sits in front of multiple EC2 instances and **distributes incoming traffic evenly** across them.

Without it: All requests hit one server → it gets overloaded.
With it: Requests are spread across 5 servers → none gets overwhelmed.

AWS Load Balancer is called **ELB (Elastic Load Balancer)**.

Types:
- **ALB (Application Load Balancer)** → Layer 7, works with HTTP/HTTPS, routes based on URL path or hostname. Most common for web apps.
- **NLB (Network Load Balancer)** → Layer 4, ultra-fast, handles millions of requests per second
- **CLB (Classic)** → Old, being deprecated

---

## Deploying an Application on EC2 — Full Walkthrough

### What You Need Before Deploying

1. **An AWS Account** (free tier works for learning)
2. **An EC2 instance launched** with the right OS (Ubuntu is most common for DevOps)
3. **A Key Pair (.pem file)** downloaded to your laptop
4. **Security Group** configured to allow SSH (port 22) from your IP, and HTTP (port 80) or your app's port
5. **Your application code** — either on GitHub or ready to transfer

---

### The Confusion Around PuTTY and MobaXterm

This is where beginners get confused, so let's clear it up completely.

#### What is SSH?

**SSH (Secure Shell)** is a protocol that lets you **remotely control a Linux server** from your computer through a terminal (command line). When your EC2 instance is running, it's sitting in an AWS data center somewhere in Mumbai or wherever. You can't physically sit in front of it.

SSH lets you open a terminal on your laptop and type commands that run on that remote server — as if you were physically sitting in front of it.

**On Mac/Linux:** SSH is built-in. You just open Terminal and type `ssh -i yourkey.pem ubuntu@<EC2-Public-IP>` and you're in.

**On Windows:** Windows did NOT have a proper built-in SSH terminal for years. So you needed third-party tools. That's where PuTTY and MobaXterm come in.

*(Note: Windows 10/11 now has built-in OpenSSH, but MobaXterm and PuTTY are still widely used and asked about in interviews.)*

---

### PuTTY — What Is It and What Does It Do?

**PuTTY** is a free, lightweight SSH client for Windows. It lets you connect to your EC2 Linux instance from Windows.

**The catch with PuTTY:**
AWS gives you a `.pem` file as your private key. PuTTY does **not** understand `.pem` files. It uses its own format called `.ppk`.

So you need an extra tool called **PuTTYgen** (comes with PuTTY) to convert your `.pem` → `.ppk`.

**Steps to use PuTTY:**
1. Open PuTTYgen → Load your `.pem` file → Save as `.ppk`
2. Open PuTTY → Enter your EC2 Public IP
3. Go to SSH → Auth → Browse → select your `.ppk` file
4. Click Open → Login as `ubuntu` (for Ubuntu AMI) or `ec2-user` (for Amazon Linux)
5. You're now connected — you'll see the terminal of your remote EC2 server

**What PuTTY can do:**
- SSH into Linux servers
- Execute commands remotely
- Basic terminal functionality

**What PuTTY cannot do:**
- File transfer (you'd need a separate tool like WinSCP)
- Multiple tabs
- GUI-friendly file browsing

---

### MobaXterm — What Is It and What Does It Do?

**MobaXterm** is a more powerful, all-in-one tool for Windows that includes:

- **SSH client** (like PuTTY, but better UI)
- **Built-in SFTP file browser** — drag and drop files between your Windows machine and your EC2 server
- **Multiple tabs** — connect to multiple servers simultaneously
- **Built-in X11 display** — for running graphical Linux apps remotely
- **Terminal with better features** — colors, copy-paste, split-screen, etc.

**The big advantage over PuTTY:** MobaXterm **accepts `.pem` files directly** — no conversion needed.

**Steps to use MobaXterm:**
1. Open MobaXterm → Click Session → SSH
2. Enter your EC2 Public IP in "Remote Host"
3. Check "Specify username" → enter `ubuntu` or `ec2-user`
4. Go to Advanced SSH settings → Use private key → browse to your `.pem` file
5. Click OK → You're connected

On the left panel, you'll automatically see an **SFTP file browser** showing your EC2's file system. You can drag files from Windows directly into your server.

---

### PuTTY vs MobaXterm — Which to Use?

| Feature | PuTTY | MobaXterm |
|---|---|---|
| SSH Connection | ✅ Yes | ✅ Yes |
| Accepts .pem directly | ❌ No (needs .ppk conversion) | ✅ Yes |
| Built-in file transfer | ❌ No | ✅ Yes (SFTP panel) |
| Multiple tabs | ❌ No | ✅ Yes |
| Ease of use for beginners | Okay | Much easier |
| Free | ✅ Yes | ✅ Yes (Home edition) |

**Recommendation:** Use **MobaXterm** for learning. It's much more convenient.

---

### Deploying a Simple Web App on EC2 — Step by Step

Let's say you have a Node.js or Python Flask app on GitHub and you want to run it on EC2.

**Step 1: Launch EC2 Instance**
- Go to AWS Console → EC2 → Launch Instance
- Choose AMI: Ubuntu 22.04 LTS
- Choose Instance Type: t2.micro (free tier)
- Create or select a Key Pair → download the `.pem` file
- Configure Security Group:
  - Allow SSH (port 22) from your IP
  - Allow HTTP (port 80) from anywhere
  - Allow your app's port (e.g., 3000 for Node.js) from anywhere
- Launch the instance

**Step 2: Connect to the Instance**

Using MobaXterm (Windows) or Terminal (Mac/Linux):
```
ssh -i your-key.pem ubuntu@<EC2-Public-IP>
```

You're now inside your EC2 server terminal.

**Step 3: Set Up the Server**
```bash
sudo apt update && sudo apt upgrade -y      # Update packages
sudo apt install git -y                      # Install Git
sudo apt install nodejs npm -y               # For Node.js app
# OR
sudo apt install python3-pip -y              # For Python app
```

**Step 4: Clone Your App**
```bash
git clone https://github.com/yourusername/your-repo.git
cd your-repo
npm install        # Install dependencies
```

**Step 5: Run Your App**
```bash
node app.js        # Or: python3 app.py
```

Your app is now running. Access it at `http://<EC2-Public-IP>:3000`

**Step 6 (Optional but Recommended): Keep App Running with PM2**

If you close MobaXterm/SSH session, the app stops. Use PM2 to keep it running:
```bash
sudo npm install -g pm2
pm2 start app.js
pm2 startup        # Makes it auto-start on reboot
```

**Step 7 (Optional): Point a Domain to Your EC2**
- Assign an **Elastic IP** to your instance (so IP doesn't change)
- Go to your domain registrar → DNS settings
- Add an A record pointing your domain to the Elastic IP
- Access your app at `http://yourdomain.com`

---

## EC2 Architecture Summary

```
Internet
    |
[Load Balancer]
    |
---------------------------------
|               |               |
[EC2 - AZ-1]  [EC2 - AZ-2]  [EC2 - AZ-3]
    |               |               |
[EBS Volume]  [EBS Volume]  [EBS Volume]
---------------------------------
        |
   [Security Group / Firewall]
```

---

---

# Interview Questions & Answers

---

**Q1. What is EC2 and what does the name mean?**

EC2 stands for Elastic Compute Cloud. The "2" represents the two C's in the name — it's shorthand for EC-squared. "Elastic" means resources can scale up or down on demand. "Compute" refers to the processing power — CPU and RAM — provided as virtual machines. "Cloud" means it's hosted in AWS's data centers and accessed over the internet. In simple terms, EC2 lets you rent virtual machines in the cloud and pay only for what you use.

---

**Q2. What is a Hypervisor? How does EC2 use it?**

A hypervisor is software that runs on a physical machine and creates multiple virtual machines on top of it. It handles resource allocation — giving each VM its share of CPU, RAM, and storage — and keeps each VM isolated from others.

EC2 uses AWS's own hypervisor called Nitro, which is a custom version of KVM. When you launch an EC2 instance, the Nitro hypervisor is creating a virtual machine for you on AWS's physical hardware. AWS manages all of this — you only see the virtual machine.

---

**Q3. Why would a company use AWS EC2 instead of just running their own servers with a hypervisor?**

With your own servers, you have a large upfront hardware cost, you have to predict your capacity in advance, and you're responsible for maintenance, power, cooling, and physical security. Scaling up means buying more hardware, which takes days or weeks.

With EC2, there's no upfront cost, you can scale up in minutes, and AWS handles all infrastructure maintenance. You also get global reach — you can deploy in any region around the world instantly. For most businesses, this is more cost-effective and operationally simpler than managing their own data centers.

---

**Q4. What is an AMI?**

AMI stands for Amazon Machine Image. It's a template that contains the operating system, pre-installed software, and configuration needed to launch an EC2 instance. When you launch an instance, you pick an AMI as the base. AWS provides standard AMIs like Ubuntu, Amazon Linux, and Windows. You can also create your own custom AMI from a configured instance — this is useful when you want to launch multiple identical servers quickly.

---

**Q5. What is a Security Group in EC2?**

A Security Group acts as a virtual firewall for an EC2 instance. It controls inbound and outbound traffic using rules based on protocol, port, and source or destination IP. By default, all inbound traffic is denied and all outbound traffic is allowed. Security Groups are stateful — if you allow an incoming request, the response is automatically allowed out. You can attach one or more Security Groups to an instance and modify the rules anytime.

---

**Q6. What is the difference between stopping and terminating an EC2 instance?**

Stopping an instance is like shutting down a computer — the instance is off, you're not charged for compute time, but the EBS storage is still there and you'll still pay for that. You can restart it later, and the data on the disk is preserved.

Terminating an instance is permanent deletion. The instance is destroyed. By default, the root EBS volume is also deleted. There's no way to recover a terminated instance. It's like throwing the computer away.

---

**Q7. What is an Availability Zone and why does it matter?**

An Availability Zone is a physically separate data center within a region. AWS regions have multiple AZs that are connected with high-speed networking but are isolated from each other — they have separate power, cooling, and networking. If one AZ goes down, the others continue to function.

This matters for High Availability. If you deploy your application across two AZs and one has an outage, your app keeps running in the other. A best practice for production workloads is to always deploy across at least two AZs.

---

**Q8. What is latency and how do you minimize it with EC2?**

Latency is the time delay between a user making a request and receiving a response. High latency means your app feels slow.

To minimize latency with EC2, you deploy your instances in the region closest to your users. If your users are in India, you'd use the Mumbai region (ap-south-1). If you have users globally, you'd use a CDN like CloudFront in front of your application, which caches content at edge locations near users around the world.

---

**Q9. What is the difference between On-Demand, Reserved, and Spot Instances?**

On-Demand instances are charged by the hour or second with no commitment. They're flexible but the most expensive. Good for development and unpredictable workloads.

Reserved Instances require a 1 or 3 year commitment but give up to 72% discount compared to On-Demand. Good for steady, predictable workloads like a production database that runs 24/7.

Spot Instances use AWS's spare unused capacity at up to 90% discount, but AWS can terminate them with a 2-minute warning if that capacity is needed. Good for non-critical, fault-tolerant jobs like batch data processing or ML training.

---

**Q10. What is EBS and how is it related to EC2?**

EBS stands for Elastic Block Store. It's the storage — essentially the virtual hard drive — attached to an EC2 instance. Every EC2 instance has at least one EBS volume (the root volume) where the OS is installed. You can attach additional EBS volumes for more storage. EBS volumes persist independently from the instance — you can stop an instance and the data on EBS is still there. However, EBS is tied to a specific Availability Zone.

---

**Q11. What is an Elastic IP and when would you use it?**

An Elastic IP is a static public IP address that you can assign to an EC2 instance. By default, EC2 instances get a public IP that changes every time you stop and restart the instance. If your domain name is pointed to that IP, it breaks when the IP changes. An Elastic IP stays fixed. You'd use it when you need a consistent IP address — for example, when pointing a domain to your server or whitelisting your server's IP in a third-party service.

---

**Q12. What is Auto Scaling?**

Auto Scaling is an AWS feature that automatically adjusts the number of EC2 instances based on real-time demand. You define a minimum, maximum, and desired number of instances. When traffic increases and CPU usage crosses a threshold you set, Auto Scaling launches additional instances. When traffic drops, it terminates unneeded instances. This ensures your application can handle spikes while not wasting money on idle servers during quiet periods.

---

**Q13. What is a Load Balancer and why is it used with EC2?**

A Load Balancer distributes incoming traffic across multiple EC2 instances so no single instance gets overloaded. It also performs health checks — if one instance becomes unresponsive, the Load Balancer stops sending traffic to it and redirects to healthy instances. This improves both performance and availability. In AWS, the Application Load Balancer (ALB) is most commonly used for web applications. Auto Scaling and Load Balancers are almost always used together in production.

---

**Q14. What is a Key Pair in EC2?**

A Key Pair is used for secure SSH access to EC2 Linux instances instead of a password. It consists of a public key (stored by AWS on your EC2 instance) and a private key (a .pem file you download and keep securely). When you try to SSH into your instance, your private key is matched against the public key on the server. If they match, access is granted. If you lose the private key, you lose SSH access to that instance.

---

**Q15. What is PuTTY? What is MobaXterm? How do they differ?**

Both are tools for Windows users to SSH into Linux EC2 instances.

PuTTY is a lightweight SSH client. The downside is it doesn't accept AWS's .pem key files directly — you have to first convert the .pem to .ppk format using PuTTYgen. It also doesn't have a built-in file transfer panel or multiple tabs.

MobaXterm is a more feature-rich tool. It accepts .pem files directly, has a built-in SFTP file browser so you can drag and drop files between your Windows PC and the EC2 server, supports multiple tabs, and has a better interface overall. For beginners and day-to-day DevOps work, MobaXterm is more practical.

---

**Q16. How would you deploy a web application on EC2?**

First, I'd launch an EC2 instance — choosing the right AMI (usually Ubuntu), instance type, and configuring the Security Group to allow SSH on port 22 and HTTP on port 80 or whatever port the app runs on.

Then I'd connect to the instance via SSH using MobaXterm or the terminal with the .pem key file.

Once inside, I'd update the system packages, install the required runtime (Node.js, Python, Java, etc.), clone the application from GitHub, install dependencies, and start the app.

For production, I'd use a process manager like PM2 (for Node.js) to keep the app running after SSH disconnect, set up Nginx as a reverse proxy to forward port 80 traffic to the app's port, and optionally attach an Elastic IP and point a domain to it.

---

*📌 EC2 is one of the most tested AWS topics in interviews — especially combinations like: EC2 + Security Groups, EC2 + Auto Scaling + Load Balancer, and EC2 + IAM Roles (giving EC2 permissions to access S3 etc.). Always connect concepts together when answering.*
