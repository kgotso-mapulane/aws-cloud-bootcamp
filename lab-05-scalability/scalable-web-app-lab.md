# AWS Scalable Web App Lab
## Launch Template, Application Load Balancer & Auto Scaling Group

---

## Before You Start: The Mental Model

Think of this lab like running a restaurant:

| Real World | AWS Equivalent |
|---|---|
| A recipe card for training new chefs identically | Launch Template |
| The front door and host who directs customers to tables | Application Load Balancer (ALB) |
| The pool of tables available for customers | Target Group |
| A manager who opens more tables when busy, closes them when quiet | Auto Scaling Group (ASG) |
| Health inspector checking each table is properly set | Health Check |
| A customer always returning to the same waiter | Session Stickiness |

The key idea: **a single EC2 instance is a single point of failure. This lab builds a self-healing, elastic web tier that survives instance failure and scales with demand automatically.**

---

## Where Day One Left Off

| Day One | Day Two (this lab) |
|---|---|
| One EC2 instance, one AZ | Fleet of instances across two AZs |
| Manual launch and fix | ASG replaces unhealthy instances automatically |
| Users hit the instance IP directly | Users hit one ALB DNS name; traffic spread across instances |
| Capacity is fixed | Capacity scales up/down with CPU load |

---

## Concept 1: Launch Template

A **Launch Template** is a reusable blueprint that defines everything needed to launch an EC2 instance — AMI, instance type, security group, key pair, and user data. The Auto Scaling Group uses it to launch identical instances every time.

It replaces the older "Launch Configuration" — more flexible, supports versioning, and is the current AWS standard.

**Why not just point the ASG at an existing instance?**  
Hand-built instances are snowflakes — they may have manual changes, different software versions, or configuration drift. A launch template ensures every instance the ASG creates is identical and reproducible.

---

## Concept 2: Application Load Balancer (ALB)

An **ALB** is a Layer 7 (HTTP/HTTPS) load balancer that distributes incoming traffic across multiple targets. It is the single entry point for users — they never need to know individual instance IPs.

Key components:

**Listener** — watches for incoming connections on a port (e.g. HTTP:80). When traffic arrives, it applies rules to decide where to send it.

**Target Group** — a pool of instances (or other targets) that receive traffic. The ALB forwards requests to healthy targets in the group.

**Health Check** — the ALB regularly sends HTTP requests to each instance. If an instance fails a health check, the ALB stops sending traffic to it until it recovers.

**Scheme:**
- **Internet-facing** — has a public DNS name, receives traffic from the internet
- **Internal** — only reachable within the VPC, used for communication between tiers

---

## Concept 3: Defense in Depth with Security Groups

A key security pattern in this lab: lock the instance security group so **only the ALB** can reach the instances on port 80.

```
Internet → ALB (alb-sg: port 80 open) → Instances (web-sg: port 80 only from alb-sg)
```

Without this, anyone who discovers an instance's public IP can bypass the load balancer entirely. With this, all traffic must flow through the ALB — no exceptions.

---

## Concept 4: Auto Scaling Group (ASG)

An **Auto Scaling Group** maintains a fleet of EC2 instances within a defined size range. It automatically:
- Launches instances when capacity is needed
- Terminates instances when demand drops
- Replaces unhealthy instances (self-healing)
- Spreads instances across multiple AZs for fault tolerance

**Three capacity values:**
- **Minimum** — the floor; ASG never goes below this
- **Desired** — the target the ASG tries to maintain at all times
- **Maximum** — the ceiling; ASG never exceeds this

---

## Concept 5: Scaling Policies

**Target Tracking** — the simplest and most common. Set a target metric value (e.g. 50% average CPU) and the ASG automatically adds or removes instances to maintain it. Works like a thermostat.

**Step Scaling** — define specific actions for specific metric thresholds (e.g. add 2 instances if CPU > 70%, add 4 if CPU > 90%).

**Scheduled Scaling** — scale at a set time (e.g. add instances every morning at 8am for predictable traffic patterns).

---

## Concept 6: EC2 Health Checks vs ELB Health Checks

| | EC2 Health Check | ELB Health Check |
|---|---|---|
| What it detects | VM is down | Web server is failing (HTTP not responding) |
| Scenario missed | Apache crashed but instance is running | Never misses this |
| Best for | Basic VM monitoring | Web tier — always enable this |

Always enable ELB health checks for web tiers. A broken-but-running instance still looks healthy to EC2 checks — only the ELB check catches a failed web server.

---

## Concept 7: Session Stickiness

By default the ALB uses round-robin — each request goes to the next healthy instance in rotation. This is ideal for stateless applications.

**Stickiness** makes the ALB pin a client to a specific instance using a cookie (AWSALB). Every request from that client goes to the same instance.

When to use it: only when instances hold session state locally (e.g. in-memory login sessions). The better long-term pattern is **stateless instances** with shared session storage (ElastiCache or DynamoDB) — then any instance can serve any request and stickiness is not needed.

---

## Architecture

```
                        Internet
                            │
                    ┌───────────────┐
                    │   app-alb     │  Internet-facing ALB
                    │   (alb-sg)    │  HTTP:80 listener
                    └───────────────┘
                      │           │
            ┌─────────┘           └─────────┐
            ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│   public-1a (AZ-a)  │         │   public-1b (AZ-b)  │
│   EC2 instance      │         │   EC2 instance      │
│   (web-sg)          │         │   (web-sg)          │
└─────────────────────┘         └─────────────────────┘
            │                               │
            └─────────────┬─────────────────┘
                          ▼
                   ┌─────────────┐
                   │   web-tg    │  Target Group
                   │  (web-asg)  │  min 2 / max 4
                   └─────────────┘
                          │
                   ┌─────────────┐
                   │   web-lt    │  Launch Template
                   │  (blueprint)│
                   └─────────────┘
```

---

## Lab Steps: AWS Console Walkthrough

> Sign in as the IAM admin user (not root) before starting.
> Requires: bootcamp-vpc with web-sg from Labs 2 and 3.

---

### Pre-requisite: Add a Second Public Subnet

An ALB and ASG both require at least two subnets in two different AZs.

1. Go to **VPC → Subnets → Create subnet**.
2. Fill in:
   - **VPC**: `bootcamp-vpc`
   - **Subnet name**: `public-1b`
   - **Availability Zone**: us-east-1b
   - **IPv4 CIDR**: `10.0.3.0/24`
3. Select `public-1b` → **Actions → Edit subnet settings** → enable **Auto-assign public IPv4**.
4. Go to **VPC → Route tables → public-rt → Subnet associations → Edit** → also associate `public-1b`.

Both public subnets now share the same route table and have internet access.

---

### Part A — Launch Template

---

#### Step 1: Create the Launch Template

1. Go to **EC2 → Launch templates → Create launch template**.
2. Fill in:
   - **Name**: `web-lt`
   - **AMI**: Amazon Linux 2023
   - **Instance type**: t3.micro
   - **Key pair**: `bootcamp-key`
3. Under **Network settings**: do NOT select a subnet (the ASG chooses). Select security group `web-sg`.
4. Expand **Advanced details → User data** and paste:

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl enable --now httpd

TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 60')

get() { curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/$1; }

echo "<h1>BPSDynamic web tier</h1><p>Instance: $(get instance-id)</p>\
<p>AZ: $(get placement/availability-zone)</p>" > /var/www/html/index.html
```

5. Click **Create launch template**.

---

### Part B — Application Load Balancer + Target Group

---

#### Step 2: Create a Security Group for the ALB

1. Go to **EC2 → Security groups → Create security group**.
2. Fill in:
   - **Name**: `alb-sg`
   - **VPC**: `bootcamp-vpc`
   - **Inbound rules**: HTTP port 80 from Anywhere-IPv4 (0.0.0.0/0)
3. Click **Create security group**.

4. Now edit `web-sg`:
   - Go to **EC2 → Security groups → web-sg → Inbound rules → Edit**.
   - Change the HTTP (80) source from `0.0.0.0/0` to `alb-sg` (search and select it).
   - Save.

> **ⓘ Defense in depth**
> Users talk to the ALB; the ALB talks to the instances. Locking web-sg's port 80 to alb-sg means nobody can bypass the load balancer to hit an instance directly.

---

#### Step 3: Create the Target Group

1. Go to **EC2 → Target groups → Create target group**.
2. Fill in:
   - **Target type**: Instances
   - **Name**: `web-tg`
   - **Protocol**: HTTP, Port 80
   - **VPC**: `bootcamp-vpc`
3. Health checks: path `/`; Advanced: healthy threshold 2, interval 15s.
4. Click **Next** — on the register targets page, **do not add any instances**. The ASG registers them automatically.
5. Click **Create target group**.

---

#### Step 4: Create the Load Balancer

1. Go to **EC2 → Load balancers → Create load balancer → Application Load Balancer**.
2. Fill in:
   - **Name**: `app-alb`
   - **Scheme**: Internet-facing
3. **Network mapping**: VPC `bootcamp-vpc`, tick both AZs and select `public-1a` and `public-1b`.
4. **Security groups**: select `alb-sg` (remove the default if present).
5. **Listeners**: HTTP:80 → forward to `web-tg`.
6. Click **Create load balancer**.
7. Copy the **DNS name** from the load balancer details — this is the URL used to test.

---

### Part C — Auto Scaling Group

---

#### Step 5: Create the ASG

1. Go to **EC2 → Auto Scaling Groups → Create Auto Scaling group**.
2. Fill in:
   - **Name**: `web-asg`
   - **Launch template**: `web-lt`
3. **Network**: VPC `bootcamp-vpc`, select both `public-1a` and `public-1b`.
4. **Load balancing**: Attach to an existing load balancer → choose from target groups → `web-tg`. Enable **Elastic Load Balancing health checks**.
5. **Group size**: Desired 2, Minimum 2, Maximum 4.
6. Click through and **Create Auto Scaling group**.

---

#### Step 6: Add a Target Tracking Scaling Policy

1. Open `web-asg` → **Automatic scaling** tab → **Create dynamic scaling policy**.
2. Fill in:
   - **Policy type**: Target tracking scaling
   - **Metric**: Average CPU utilization
   - **Target value**: 50
3. Click **Create**.

The ASG now adds instances when average CPU exceeds 50% and removes them when it drops — staying within min 2 / max 4.

---

### Part D — Test It

---

#### Step 7: Verify Load Balancing and Self-Healing

1. Browse to the ALB DNS name. Refresh several times — the Instance ID and AZ change as traffic is spread across instances.
2. Go to **Target groups → web-tg → Targets** — confirm both instances show **healthy**.
3. Terminate one instance: **EC2 → Instances → select one ASG instance → Instance state → Terminate**.
4. Watch the ASG launch a replacement automatically. The ALB stops routing to the terminated instance immediately. That is self-healing.

---

#### Step 8: (Optional) Trigger a Scale-Out

1. Connect to an instance via **EC2 Instance Connect**.
2. Run:

```bash
sudo dnf install -y stress-ng
stress-ng --cpu 2 --timeout 300s
```

3. Watch **web-asg → Activity** and **Instances** — new instances appear as average CPU passes 50%, then terminate after load stops.

> **ⓘ Scaling is not instant**
> Target tracking reacts over a few minutes — CloudWatch needs several data points before acting. This is by design to avoid thrashing (rapidly adding and removing instances).

---

### Part E — Round-Robin and Session Stickiness

---

#### Step 9: Pause the ASG and Launch Three Colour Instances

1. Open `web-asg` → **Edit** → set **Desired = 0** and **Minimum = 0** → **Update**. Wait for its instances to drain.

2. Launch three t3.micro instances (one at a time), mixing AZs. For each use AMI Amazon Linux 2023, key `bootcamp-key`, VPC `bootcamp-vpc`, security group `web-sg`, and paste the script below into User data — changing COLOR and LABEL each time:

```bash
#!/bin/bash
dnf install -y httpd && systemctl enable --now httpd

COLOR=green      # green  |  goldenrod  |  blue
LABEL=GREEN      # GREEN  |  YELLOW     |  BLUE

cat > /var/www/html/index.html <<HTML
<body style="background:$COLOR;text-align:center;font-family:sans-serif">
<h1 style="color:white;font-size:88px;margin-top:120px">$LABEL</h1>
<p style="color:white;font-size:22px">served by $(hostname)</p>
</body>
HTML
```

| Instance name | COLOR | LABEL |
|---|---|---|
| `web-green` | green | GREEN |
| `web-yellow` | goldenrod | YELLOW |
| `web-blue` | blue | BLUE |

---

#### Step 10: Register the Three Instances with the Target Group

1. Go to **EC2 → Target groups → web-tg → Targets → Register targets**.
2. Tick all three colour instances → **Include as pending below → Register pending targets**.
3. Wait until all three show **healthy** (30–60 seconds).

---

#### Step 11: Watch Round-Robin Load Balancing

1. Open the ALB DNS name in the browser and refresh repeatedly.
2. The background cycles GREEN → YELLOW → BLUE as the ALB rotates traffic across instances.

> **✔ Make the rotation obvious**
> Use hard refresh (Ctrl+Shift+R) or open the DNS name in several private/incognito windows side by side — each new session lands on a different colour.

---

#### Step 12: Turn On Stickiness

1. Go to **EC2 → Target groups → web-tg → Attributes → Edit**.
2. Enable **Stickiness**, type **Load balancer generated cookie**, duration 1 hour.
3. Click **Save changes**.

---

#### Step 13: Confirm Traffic is Pinned

1. Refresh the ALB in the browser — the colour stops changing. The AWSALB cookie pins the session to one instance.
2. Open the DNS name in a fresh incognito window — it may land on a different colour, but then that window sticks too.

---

## Verification Checklist

- [ ] Second public subnet added — ALB and ASG span two AZs
- [ ] Launch template `web-lt` created with user data web server
- [ ] `alb-sg` created; `web-sg` locked so only `alb-sg` can reach port 80
- [ ] Target group `web-tg` created with health checks
- [ ] Internet-facing ALB `app-alb` with HTTP:80 forwarding to `web-tg`
- [ ] ASG `web-asg` (min 2 / max 4) across two AZs with ELB health checks
- [ ] Target tracking scaling policy at 50% CPU
- [ ] Refreshing ALB DNS shows different instances; terminating one self-heals
- [ ] Three colour instances registered and healthy in `web-tg`
- [ ] Round-robin confirmed (colours rotate on refresh)
- [ ] Stickiness confirmed (colour stays fixed after enabling)
- [ ] Entire stack torn down

---

## Key Concepts to Revisit

1. Why is a launch template better than pointing the ASG at an existing instance?
2. What is the difference between a listener, a target group, and a health check?
3. Why lock web-sg to only accept traffic from alb-sg instead of 0.0.0.0/0?
4. What is the difference between EC2 health checks and ELB health checks — and why does it matter?
5. A user logs in on instance A. The next request goes to instance B which has no session. What are the two ways to fix this?
6. Scaling isn't instant with target tracking — why is that a feature, not a bug?
7. In what order must resources be deleted and why?

---

## Cleanup & Teardown

> **⚠ The ALB is not free — ~$0.0225/hour. Tear down immediately after the lab.**

**First** — remove Part E extras: turn stickiness off, terminate the three colour instances.

**Then delete in this order — order matters:**

| Order | Resource | Where in the Console |
|---|---|---|
| 1 | Auto Scaling Group | EC2 → Auto Scaling Groups → `web-asg` → Delete |
| 2 | Load balancer | EC2 → Load balancers → `app-alb` → Actions → Delete |
| 3 | Target group | EC2 → Target groups → `web-tg` → Actions → Delete |
| 4 | Launch template | EC2 → Launch templates → `web-lt` → Actions → Delete |
| 5 | ALB security group | EC2 → Security groups → `alb-sg` → Actions → Delete |

> A security group cannot be deleted while something uses it. A target group cannot be deleted while a listener forwards to it. Always delete top-down.

Keep the VPC, subnets, and `bootcamp-key` for future labs.

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
