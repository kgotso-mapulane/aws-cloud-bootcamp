# AWS Cloud Bootcamp — Lab 3: Compute — EC2 & Serverless

## What This Is

This is my personal study work as I learn AWS from scratch. This lab covers
launching an EC2 web server into the VPC from Lab 2, attaching an IAM role,
and contrasting it with a Lambda serverless function — all from the AWS Console.

**Level:** Some IT background  
**Interface:** AWS Console  
**Goal:** Understand the difference between managing a server (EC2) and going serverless (Lambda)

---

## What I Built

| Component | Setting | Purpose |
|---|---|---|
| Key pair `bootcamp-key` | RSA, .pem format | SSH access to the EC2 instance |
| EC2 instance `web-01` | Amazon Linux 2023, t2.micro/t3.micro | Virtual machine running Apache web server |
| User data script | Installs Apache, writes HTML page with AZ | Bootstraps the server at first boot — no manual setup |
| IAM role `EC2-S3-ReadOnly` | Attached to `web-01` | Grants S3 read access via temporary credentials — no keys stored |
| Lambda function `hello-bootcamp` | Python 3.13 | Serverless function returning a greeting — no server to manage |

---

## Architecture

```
AWS Account
│
├── Part A: EC2 Web Server
│   └── VPC: bootcamp-vpc (from Lab 2)
│       └── Public Subnet: public-subnet-a
│           ├── Security Group: web-sg (ports 22, 80, 443)
│           └── EC2 Instance: web-01
│               ├── AMI: Amazon Linux 2023
│               ├── Type: t2.micro / t3.micro
│               ├── Key pair: bootcamp-key
│               ├── User data: installs Apache, serves web page
│               └── IAM Role: EC2-S3-ReadOnly (no keys stored)
│
└── Part B: Lambda Function
    └── Function: hello-bootcamp
        ├── Runtime: Python 3.13
        └── Returns a greeting message
```

---

## Key Things I Learned

**User data runs once at first boot — not every time the instance starts.**  
The bootstrap script installs Apache and writes the HTML page automatically.
No SSH required to set up the server. This is how real servers are
provisioned — setup is baked into the launch configuration.

**The IAM role from Lab 1 finally gets used here.**  
Attaching `EC2-S3-ReadOnly` to the instance means it can call S3 without any
access keys stored on the machine. Temporary, auto-rotated credentials are
fetched from the metadata service at `169.254.169.254`. If the instance
is ever compromised, those credentials expire on their own.

**Serverless means zero servers to manage — not zero infrastructure.**  
Lambda still runs on AWS hardware, but it's never visible, patched, or scaled
manually. A function is handed to AWS and it runs on demand. The tradeoff is
less control over the environment and potential cold starts on idle functions.

**EC2 and Lambda are not competing — they solve different problems.**  
A long-running web server with persistent state belongs on EC2. A short,
event-driven task like processing an upload or responding to an API call
belongs on Lambda. Knowing which to reach for is a core cloud skill.

**Stop vs. terminate — not the same thing.**  
Stopping an instance pauses it and keeps the EBS volume (small cost, quick
restart). Terminating deletes it permanently. For labs, always terminate
when done to avoid consuming Free Tier hours.

**A public IP alone does not mean a web server is reachable.**  
The instance needs to be in a public subnet, have a public IP assigned, sit
behind a security group allowing port 80, and have a running web server.
All four conditions must be true — missing any one of them and the page
won't load.

**EC2 Instance Connect removes the need for a local SSH client.**  
Connecting via the browser terminal in the console is enough to inspect the
instance, check running services, and run commands — no PuTTY or terminal
app needed, especially useful when just getting started.

**The AMI is the starting point, not the finished product.**  
Amazon Linux 2023 boots as a bare OS. User data, manual installs, or
configuration management tools build on top of it. The AMI just ensures
a consistent, clean starting state every time an instance is launched.

**Lambda cold starts are a real tradeoff.**  
If a Lambda function hasn't been invoked recently, the first call takes
slightly longer while AWS initialises the execution environment. For
low-latency production use cases this matters — it's one reason EC2 or
containers are sometimes preferred over Lambda despite the operational overhead.

**Be cost-aware when selecting instance options.**  
Not all instance types cost the same for the same workload. AWS Graviton
instances (`t4g`, `m7g`, `c7g` etc.) use AWS's own ARM-based processors and
are typically 20–40% cheaper than equivalent x86 instances. The AMI just
needs to be ARM-compatible to run on them. In a lab it's easy to click the
default — in production, researching the right instance family can make a
significant cost difference.

**Always keep the AWS Well-Architected Framework (WAF) in mind.**  
Every provisioning decision should be viewed through its six pillars:
Operational Excellence, Security, Reliability, Performance Efficiency,
Cost Optimisation, and Sustainability. It's not just a checklist for
architects — it's a mindset for every resource created, even in a lab.
Asking "is this the right size, is it secure, will it scale, does it
cost more than it needs to?" before clicking launch builds the right habits early.

---

## What Confused Me Initially

- User data feels like running CLI commands but it isn't — it's a script
  baked into the instance launch that runs once on first boot
- The metadata IP `169.254.169.254` looks strange — it's a link-local address
  only reachable from inside the instance itself, not from outside
- Lambda needs an IAM role too — just like EC2, if a Lambda function
  needs to call other AWS services, a role with the right permissions is required

---

## Key Concepts

**1. User data** — a script that runs once at first boot only. It performs the required setup actions when the instance is first launched — installing software, configuring the server, writing files. It does not re-run if the instance is stopped and started again.

**2. Access keys vs IAM role** — if access keys are lost or leaked, the system is at risk until they are manually revoked. With an IAM role, the instance gets temporary credentials that expire automatically after a short period. Even if they land in the wrong hands, they become useless once they expire — no manual action needed.

**3. Instance metadata service** — provides information about the instance itself: its AZ, instance ID, region, attached IAM role, and more. Only the instance can access it because it's available at a link-local IP address (`169.254.169.254`) that is only reachable from within the instance — nothing outside can query it. It's like the instance checking its own identity card.

**4. Lambda reading from DynamoDB** — an IAM role must be created with a policy that allows DynamoDB read access, then attached to the Lambda function. Just like EC2, Lambda needs a role to call other AWS services — it cannot do it on its own.

**5. 20-minute nightly job — EC2 or Lambda?** — EC2. Lambda has a hard maximum runtime of 15 minutes, so a 20-minute job would be killed before it finishes. EC2 has no time limit on how long a process can run.

**6. Stop vs terminate** — stopping an instance pauses it and keeps the EBS volume intact, which incurs a small storage charge but allows a quick restart when needed. Terminating deletes the instance entirely — it cannot be recovered or reused, and no further charges are incurred after termination.

---

## Files in This Lab

| File | What It Is |
|---|---|
| `README.md` | This file — summary and reflections |
| `ec2-serverless-lab.md` | Full concept notes and step-by-step lab walkthrough |

---

## Lab Status

- [x] Key pair created and stored safely
- [x] EC2 instance launched in public subnet and reachable over HTTP
- [x] Connected via EC2 Instance Connect — Apache confirmed running
- [x] IAM role attached — visible on Security tab, no keys stored
- [x] Lambda function created, deployed, and tested successfully
- [x] EC2 instance terminated after lab completion

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
