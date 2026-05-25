[AWS_IAM_Notes.md](https://github.com/user-attachments/files/28220303/AWS_IAM_Notes.md)
# AWS IAM — Notes + Interview Questions

---

## What is IAM?

**IAM = Identity and Access Management**

IAM is AWS's way of controlling **who can access your AWS account** and **what they're allowed to do** inside it.

Think of it like a security guard + rulebook for your AWS account.

- The **security guard** checks your identity → this is **Authentication**
- The **rulebook** decides what you're allowed to do → this is **Authorization**

---

## Authentication vs Authorization — Explained Simply

### 🔐 Authentication — "Who are you?"

Authentication is the process of **proving your identity**.

When you log into AWS with your username and password, AWS is asking:
> *"Are you really who you say you are?"*

That's authentication. It happens **before** you get access to anything.

**Examples in AWS IAM:**
- Logging into AWS Console with email + password
- Using Access Key ID + Secret Access Key via CLI
- MFA (Multi-Factor Authentication) — a second proof of identity

---

### ✅ Authorization — "What are you allowed to do?"

Once AWS knows **who you are**, it then checks:
> *"Okay, but are you allowed to do this specific action?"*

That's authorization. It happens **after** authentication.

**Examples in AWS IAM:**
- You're logged in (authenticated), but can you delete an S3 bucket? → That depends on your **policy** (authorization)
- A Lambda function is running (it's authenticated using a Role), but can it read from DynamoDB? → Depends on what permissions the Role has

**Simple analogy:**
> Authentication = showing your ID card at the office gate
> Authorization = your access card only opens certain floors, not all of them

---

## Core Components of IAM

### 1. Users
- Represents a **real person** or an application that needs AWS access
- Each user gets their own credentials (password or access keys)
- By default, a new IAM user has **zero permissions**

### 2. Groups
- A **collection of users**
- You attach policies to a group, and all users in that group inherit those permissions
- Makes managing permissions easy — instead of giving permissions to 10 users one by one, add them to a group

### 3. Roles
- Like a user, but meant for **services or applications**, not humans
- A Role is **assumed temporarily** — it gives temporary credentials
- Common use: giving an EC2 instance permission to access S3, without hardcoding any credentials
- You can also assume a role as a user (cross-account access, etc.)

### 4. Policies
- A **JSON document** that defines what is allowed or denied
- Attached to Users, Groups, or Roles
- Every policy has:
  - **Effect** → Allow or Deny
  - **Action** → What AWS action (e.g., `s3:GetObject`, `ec2:StartInstances`)
  - **Resource** → Which specific resource (e.g., a specific S3 bucket ARN, or `*` for all)

**Example Policy (Allow reading from S3):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### 5. MFA (Multi-Factor Authentication)
- Adds a second layer of authentication
- After entering password, user must also enter a one-time code from an app (like Google Authenticator)
- Strongly recommended for the root account and admin users

---

## Types of Policies

| Policy Type | Description |
|---|---|
| **Managed Policy** | Created and managed by AWS (ready-made) or by you. Can be reused and attached to multiple identities. |
| **Inline Policy** | Directly embedded inside a single user, group, or role. Not reusable. |
| **Identity-based Policy** | Attached to IAM identities (users, groups, roles). Controls what that identity can do. |
| **Resource-based Policy** | Attached directly to a resource (like an S3 bucket). Controls who can access that resource. |

---

## IAM Best Practices

- **Never use root account** for daily tasks — create an IAM admin user
- **Principle of Least Privilege** — give only the permissions that are actually needed, nothing extra
- **Enable MFA** — especially for root and admin accounts
- **Use Roles for EC2/Lambda** — never hardcode access keys inside code or servers
- **Rotate Access Keys** — if you use access keys, change them regularly
- **Use Groups** — assign permissions to groups, not individual users
- **Use IAM Access Analyzer** — it tells you if your resources are exposed to the internet or external accounts

---

## IAM is Global

IAM is **not region-specific**. IAM users, roles, and policies work across all AWS regions in your account.

---

## How IAM Evaluates Permissions (Policy Evaluation Logic)

When you make a request in AWS, IAM checks in this order:

1. Is there an **explicit Deny** anywhere? → Request is **DENIED immediately**
2. Is there an **explicit Allow**? → Request is **ALLOWED**
3. If neither → **DENIED by default** (implicit deny)

> Deny always wins over Allow.

---

## Root Account vs IAM User

| | Root Account | IAM User |
|---|---|---|
| Created when | AWS account is first created | Manually created inside IAM |
| Access level | Full access to everything, always | Only what policies allow |
| Recommended for daily use | ❌ No | ✅ Yes |
| Can be restricted | ❌ No | ✅ Yes |

---

---

# Interview Questions & Answers

---

**Q1. What is IAM in AWS?**

IAM stands for Identity and Access Management. It's the AWS service that controls who can access your AWS account and what actions they can perform. Using IAM, you can create users, groups, and roles, and attach policies to them to define their permissions. IAM is global — it applies across all AWS regions.

---

**Q2. What is the difference between Authentication and Authorization in AWS IAM?**

Authentication is about verifying identity — it answers "who are you?" For example, when you log into the AWS console with a username and password, that's authentication.

Authorization happens after authentication — it answers "what are you allowed to do?" For example, even after logging in, whether you can start an EC2 instance or delete an S3 bucket depends on the policies attached to your IAM identity. That's authorization.

In simple terms — authentication gets you inside the building, authorization controls which rooms you can enter.

---

**Q3. What are IAM Users, Groups, and Roles? How are they different?**

A **User** represents a person or application that needs AWS access. Each user has their own credentials and permissions.

A **Group** is a collection of users. You assign permissions to the group, and all users in that group inherit those permissions. It simplifies permission management.

A **Role** is similar to a user, but it's meant for AWS services or applications, not people. Roles provide temporary credentials. For example, if you want an EC2 instance to access S3, you attach a role to that EC2 instead of hardcoding access keys.

---

**Q4. What is an IAM Policy?**

A policy is a JSON document that defines permissions. It specifies what actions are allowed or denied on which resources. Policies are attached to users, groups, or roles. Every statement in a policy has three key elements — Effect (Allow/Deny), Action (what operation), and Resource (which AWS resource).

---

**Q5. What is the Principle of Least Privilege?**

It means giving users or services only the minimum permissions they need to do their job — nothing more. For example, if an application only needs to read from S3, you give it only `s3:GetObject` permission, not full S3 access. This reduces the risk of accidental or malicious damage.

---

**Q6. What is the difference between an IAM Role and an IAM User?**

A User is for humans or applications that need long-term, permanent credentials. A Role is for AWS services, applications, or cross-account access, and it provides temporary credentials. You never hardcode a role's credentials because they're generated on the fly and automatically expire. Roles are the recommended way to give permissions to EC2 instances, Lambda functions, etc.

---

**Q7. What happens if there's a conflict between Allow and Deny in IAM policies?**

Deny always wins. IAM evaluates all applicable policies, and if there is even one explicit Deny, the request is rejected regardless of any Allow statements. Also, if there's no explicit Allow, the request is denied by default — this is called an implicit deny.

---

**Q8. What is MFA in IAM and why is it important?**

MFA stands for Multi-Factor Authentication. It adds a second layer of security on top of the regular password. Even if someone gets hold of your password, they still can't log in without the second factor — usually a one-time code from an authenticator app. It's especially important for the root account and admin users since those have the highest level of access.

---

**Q9. What is the root account and should you use it daily?**

The root account is the account created when you first sign up for AWS. It has unrestricted access to everything in the account and cannot be limited by any IAM policy. You should NOT use it for daily tasks. The best practice is to create an IAM admin user immediately after signing up and use that for everything. The root account should be locked down with MFA and used only for specific tasks like closing the account or changing billing settings.

---

**Q10. What is the difference between an Inline Policy and a Managed Policy?**

A Managed Policy is a standalone policy that can be attached to multiple users, groups, or roles. It's reusable and easier to manage at scale. AWS also provides pre-built managed policies like `AdministratorAccess` or `AmazonS3ReadOnlyAccess`.

An Inline Policy is embedded directly into a specific user, group, or role. It can't be shared or reused. It gets deleted when the identity it's attached to is deleted. Inline policies are useful when you want a strict one-to-one relationship between the policy and the identity.

---

**Q11. What is an IAM Role used for in EC2?**

When an EC2 instance needs to interact with other AWS services (like reading from S3 or writing to DynamoDB), you attach an IAM Role to it. The EC2 instance then automatically gets temporary credentials to perform those actions. This is much safer than hardcoding access keys inside your application or on the server.

---

**Q12. What is IAM Access Analyzer?**

IAM Access Analyzer is a feature that automatically scans your AWS environment and tells you if any resources (like S3 buckets, IAM roles, KMS keys) are shared with external accounts or made public. It helps you identify unintended access that could be a security risk.

---

**Q13. What is a Resource-based Policy? How is it different from an Identity-based Policy?**

An identity-based policy is attached to an IAM identity (user, group, role) and controls what that identity can do.

A resource-based policy is attached directly to an AWS resource — for example, an S3 bucket policy. It controls who (which accounts, users, or services) can access that specific resource. Resource-based policies are useful for cross-account access scenarios.

---

**Q14. Is IAM a regional or global service?**

IAM is a global service. IAM users, groups, roles, and policies are not tied to any specific AWS region. When you create an IAM user, they can operate across all regions in your account.

---

**Q15. What is the AWS Security Token Service (STS)? How does it relate to IAM?**

STS is the service that provides temporary, short-lived credentials when an IAM Role is assumed. When an EC2 instance or Lambda assumes a role, STS generates an Access Key, Secret Key, and a Session Token — all of which expire after a set time. This is more secure than using permanent credentials, because even if the temporary credentials leak, they'll expire on their own.

---

*📌 Tip for interviews: Always tie your answers back to real use cases — mentioning EC2 + Roles, S3 bucket policies, or the Least Privilege principle shows practical understanding, not just bookish knowledge.*
