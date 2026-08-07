# AWS EC2 & Serverless Lab
## Launch a Web Server on EC2 and Contrast It with AWS Lambda

---

## Before You Start: The Mental Model

Think of compute options like different ways to run a restaurant kitchen:

| Real World | AWS Equivalent |
|---|---|
| Renting a full kitchen — managing it, stocking it, cleaning it | EC2 (virtual machine, fully controlled) |
| Handing a recipe to a catering service — cooked on demand | Lambda (serverless function) |
| Health and safety rules for who can enter the kitchen | Security Group |
| A staff badge granting access to the fridge (S3) | IAM Role |
| Instructions pinned to the wall for the first chef who opens up | User Data (bootstrap script) |

The key idea: **EC2 gives full control and full responsibility. Lambda gives zero servers to manage but less control over the environment.**

---

## Concept 1: EC2 — Elastic Compute Cloud

An **EC2 instance** is a virtual machine running in AWS. The operating system, size, network, and software are all configurable.

### Key components when launching an instance:

**AMI (Amazon Machine Image)** — the template for the OS and pre-installed software. The starting snapshot the instance boots from. Amazon Linux 2023 is the current AWS-native Linux distro.

**Instance type** — defines CPU and RAM. `t2.micro` and `t3.micro` are Free Tier eligible — small but sufficient for learning.

**Key pair** — an RSA key pair used for SSH access. AWS holds the public key; the private key (`.pem`) downloads once at creation. Losing it means losing SSH access to that instance.

**Security Group** — the firewall attached to the instance (built in Lab 2). Controls what traffic can reach it.

**User data** — a script that runs once at first boot. Used to install software, configure the server, or set up web pages automatically — no manual SSH required.

**IAM Role** — grants the instance permissions to call AWS services without storing any credentials on the machine.

---

## Concept 2: Instance Metadata Service (IMDS)

When an EC2 instance needs to know things about itself (what AZ it's in, what region, what role it has), it queries the **Instance Metadata Service** at a special IP address: `169.254.169.254`.

This is a link-local address only reachable from within the instance itself — nothing outside can query it.

The bootstrap script in this lab uses IMDS to fetch the Availability Zone so the web page can display which AZ is serving it:

```bash
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 60')

AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)
```

The token step (IMDSv2) is the secure, modern way to query metadata — it prevents certain attacks where malicious code inside the instance tries to steal metadata.

---

## Concept 3: IAM Roles on EC2

The `EC2-S3-ReadOnly` role created in Lab 1 gets used here.

Without a role, if an application on the instance needs to call AWS (e.g. read from S3), the only option is to store access keys directly on the machine — a major security risk. If the instance is compromised, the keys are compromised.

With a role attached, the instance receives **temporary, auto-rotated credentials** via the metadata service. No keys stored anywhere. If the instance is compromised, the credentials expire on their own.

```
Without role:                    With role:
Instance → hardcoded keys        Instance → assumes role → temp credentials
           ↓                                               ↓
        Risk: keys leaked                       Safe: credentials auto-expire
```

---

## Concept 4: Lambda — Serverless Functions

**AWS Lambda** runs code without any servers to manage. A function is provided and Lambda handles everything else — hardware, OS, scaling, patching.

Key characteristics:
- **Event-driven** — runs in response to a trigger (HTTP request, S3 upload, schedule, etc.)
- **Short-lived** — functions time out after a maximum of 15 minutes
- **Stateless** — each invocation is independent; no memory of previous runs
- **Pay per use** — charged per request and per millisecond of execution time

---

## Concept 5: EC2 vs. Lambda — When to Use Which

| | EC2 | Lambda |
|---|---|---|
| **Workload** | Long-running, stateful, full OS control | Short, event-driven, bursty |
| **Scaling** | Manually configured via Auto Scaling | Automatic, per-request |
| **Cost model** | Pay while instance runs | Pay per invocation + duration |
| **Ops burden** | OS patching and maintenance required | AWS manages the runtime |
| **Cold starts** | None | Possible on first/idle invoke |
| **Best for** | Web servers, databases, legacy apps | APIs, automation, data processing |

---

## Architecture

```
AWS Account
│
├── Part A: EC2 Web Server
│   │
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

## Lab Steps: AWS Console Walkthrough

> Sign in as the IAM admin user (not root) before starting.
> This lab builds on Lab 2 — the VPC, public subnet, and web-sg security group must exist.

---

### Part A — Launch an EC2 Web Server

---

#### Step 1: Create a Key Pair

1. Go to **EC2** in the AWS Console (search "EC2" in the top bar).
2. In the left sidebar, under **Network & Security**, click **Key Pairs**.
3. Click **Create key pair**.
4. Fill in:
   - **Name**: `bootcamp-key`
   - **Key pair type**: RSA
   - **Private key file format**: `.pem` (use `.ppk` on Windows with PuTTY)
5. Click **Create key pair** — the private key file downloads automatically.
6. Move it somewhere safe.

> **⚠ The private key cannot be re-downloaded**
> AWS only provides the private key once. Losing it means losing SSH access to any instance using this key pair. EC2 Instance Connect in the browser still works — but keep the file safe anyway.

---

#### Step 2: Launch the EC2 Instance

1. In the left sidebar, click **Instances** → **Launch instances**.
2. Fill in:
   - **Name**: `web-01`
   - **AMI**: Select **Amazon Linux 2023** (Free Tier eligible)
   - **Instance type**: `t2.micro` or `t3.micro`
   - **Key pair**: `bootcamp-key`

3. Under **Network settings**, click **Edit**:
   - **VPC**: `bootcamp-vpc`
   - **Subnet**: `public-subnet-a`
   - **Auto-assign public IP**: Enable
   - **Firewall**: Select existing security group → `web-sg`

4. Expand **Advanced details** → scroll to **User data** at the bottom.
5. Paste the following bootstrap script:

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl enable --now httpd

TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 60')

AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

echo "<h1>BPSDynamic bootcamp — served from $AZ</h1>" > /var/www/html/index.html
```

6. Click **Launch instance**.

> **ⓘ What the user data script does**
> - Updates the OS packages
> - Installs Apache (httpd) web server
> - Starts Apache and enables it on every reboot
> - Fetches the Availability Zone from the instance metadata service
> - Writes a simple HTML page showing which AZ is serving it
>
> Runs once at first boot — no CLI commands needed.

---

#### Step 3: Connect and Verify

1. Wait for the instance to show **Running** with **2/2 status checks passed** (takes 1–2 minutes).
2. Click the instance and copy its **Public IPv4 address**.
3. Open a browser and go to `http://<public-ip>` — the heading served by Apache should appear.

> **✔ No web page showing?**
> Check three things:
> 1. `web-sg` has an inbound rule allowing port 80 from `0.0.0.0/0`
> 2. The instance is in the public subnet with a public IP assigned
> 3. Give user data a minute to finish — Apache may still be installing

4. To connect to the instance from the browser (no SSH client needed):
   - Select `web-01` → click **Connect** → **EC2 Instance Connect** → **Connect**
   - A terminal opens in the browser

5. In that browser terminal, run:
```bash
systemctl status httpd
curl localhost
```
Both confirm Apache is running and serving the web page.

---

#### Step 4: Attach the IAM Role

1. Select `web-01`.
2. Click **Actions → Security → Modify IAM role**.
3. Select `EC2-S3-ReadOnly` from the dropdown.
4. Click **Update IAM role**.
5. Confirm: click on `web-01` → **Security** tab → the role should now show as attached.

The instance can now read from S3 using temporary, auto-rotated credentials — no access keys stored anywhere on the machine.

> **ⓘ Optional check**
> In the EC2 Instance Connect browser terminal, run:
> ```bash
> aws s3 ls
> ```
> S3 buckets will be listed (even if empty) — the instance is authenticating via the role, not stored keys.

---

### Part B — Serverless with AWS Lambda

---

#### Step 5: Create the Lambda Function

1. Search "Lambda" in the top bar and open the Lambda console.
2. Click **Create function**.
3. Select **Author from scratch**.
4. Fill in:
   - **Function name**: `hello-bootcamp`
   - **Runtime**: Python 3.13
   - **Architecture**: x86_64
5. Click **Create function**.

---

#### Step 6: Write and Deploy the Code

1. In the **Code source** editor, replace the existing code with:

```python
import json

def lambda_handler(event, context):
    name = event.get('name', 'bootcamp')
    return {
        'statusCode': 200,
        'body': json.dumps(f'Hello, {name}! Running with zero servers.')
    }
```

2. Click **Deploy**.

---

#### Step 7: Test the Function

1. Click **Test**.
2. Create a new test event:
   - **Event name**: `myTest`
   - **Event JSON**:
   ```json
   { "name": "Kg" }
   ```
3. Click **Save**.
4. Click **Test** again to run it.
5. Inspect the **Execution results** — the response should show:
   - Status code: 200
   - Body: `"Hello, Kg! Running with zero servers."`
   - Log output showing duration and memory used

---

## Verification Checklist

- [ ] EC2 instance `web-01` launched in `public-subnet-a` with a public IP
- [ ] Web page visible in browser at `http://<public-ip>` showing the AZ
- [ ] Connected via EC2 Instance Connect — Apache confirmed running
- [ ] `EC2-S3-ReadOnly` role attached — visible on Security tab
- [ ] Lambda function `hello-bootcamp` created and deployed
- [ ] Lambda test returns status 200 with the greeting message
- [ ] EC2 instance terminated after the lab

---

## Key Concepts to Revisit

Ask yourself these questions — if all can be answered, the lab was understood:

1. What is user data and when does it run?
2. Why is attaching an IAM role to an EC2 instance safer than storing access keys on it?
3. What is the instance metadata service and why can only the instance itself query it?
4. If a Lambda function needs to read from DynamoDB, how would it get permission?
5. A process runs for 20 minutes on a schedule every night — EC2 or Lambda?
6. What is the difference between stopping and terminating an EC2 instance?

---

## Cleanup & Teardown

> **⚠ Terminate the EC2 instance to avoid charges**
> A running instance consumes Free Tier hours continuously. Lambda has a generous always-free tier (1M requests/month) so the function costs nothing, but delete it for tidiness.

| Resource | Where in the Console |
|---|---|
| EC2 instance | EC2 → Instances → select `web-01` → Instance state → **Terminate instance** |
| Key pair (optional) | EC2 → Key Pairs → select `bootcamp-key` → Actions → Delete |
| Lambda function | Lambda → Functions → select `hello-bootcamp` → Actions → Delete |

> **✔ Stop vs. Terminate**
> **Stop** keeps the instance and its EBS volume (small storage cost, quick restart). **Terminate** deletes it permanently. For a lab, always terminate. Keep `bootcamp-key` if continuing to Day Two.

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
