# AWS Networking Fundamentals Lab
## Build a Custom VPC with Public & Private Subnets

---

## Before You Start: The Mental Model

Think of AWS networking like setting up a private office building:

| Real World | AWS Equivalent |
|---|---|
| The entire building plot | VPC (Virtual Private Cloud) |
| A floor or wing of the building | Subnet |
| The building's address on a street | Internet Gateway |
| Floor directory / signs pointing where to go | Route Table |
| Security guard checking who enters a room | Security Group |
| Building-wide access badge policy | Network ACL |

The key idea: **nothing in AWS can talk to anything else by default.** You explicitly build every connection.

---

## Concept 1: What Is a VPC?

A **VPC (Virtual Private Cloud)** is your own logically isolated section of the AWS cloud. It's a private network that you fully control — your own IP address range, your own subnets, your own routing rules.

Every AWS account has a **default VPC** already set up. You're building a **custom one** because:
- You control the IP range
- You decide what's public vs. private
- It's how real production environments work

### CIDR Notation Refresher
CIDR (Classless Inter-Domain Routing) defines an IP range using a base address + a prefix length.

```
10.0.0.0/16
│         │
│         └── prefix length: 16 bits are fixed, 16 bits are free
│              → 2^16 = 65,536 possible addresses
└── base address
```

| CIDR | Fixed Bits | Free Bits | # of Addresses |
|---|---|---|---|
| 10.0.0.0/16 | 16 | 16 | 65,536 |
| 10.0.1.0/24 | 24 | 8 | 256 |
| 10.0.2.0/24 | 24 | 8 | 256 |

> AWS reserves 5 addresses in every subnet (network, router, DNS, future use, broadcast), so a /24 gives you 251 usable IPs.

Your VPC (`10.0.0.0/16`) contains both subnets (`10.0.1.0/24` and `10.0.2.0/24`) — they fit inside the larger /16 range. That's the hierarchy:

```
VPC: 10.0.0.0/16  (65,536 addresses)
├── Public Subnet:  10.0.1.0/24  (256 addresses, in AZ-a)
└── Private Subnet: 10.0.2.0/24  (256 addresses, in AZ-b)
```

---

## Concept 2: Subnets — Public vs. Private

A **subnet** is a slice of your VPC's IP range, pinned to one Availability Zone (AZ). AZs are physically separate data centers within a region — using multiple AZs gives you fault tolerance.

**What makes a subnet "public"?**
Not just its name — it needs two things:
1. A route to an Internet Gateway in its route table
2. Resources inside it that have public IP addresses

**What makes a subnet "private"?**
It has no direct route to the internet. Resources here can only be reached from within the VPC (or through a NAT Gateway if they need outbound internet — not covered in this lab).

---

## Concept 3: Internet Gateway (IGW)

The **Internet Gateway** is the door between your VPC and the public internet. It's:
- Horizontally scaled and highly available (AWS manages it)
- **One per VPC**
- Bidirectional: handles both inbound and outbound traffic

Just attaching an IGW to your VPC doesn't make anything public. You also need to tell traffic *how to get there* — that's the route table's job.

---

## Concept 4: Route Tables

A **route table** is a set of rules (routes) that determine where network traffic is directed.

Every subnet must be associated with a route table. Each route says: "for traffic going to destination X, send it to target Y."

Your public route table will have:

| Destination | Target | Meaning |
|---|---|---|
| 10.0.0.0/16 | local | Traffic within the VPC stays local |
| 0.0.0.0/0 | igw-xxxxxxx | Everything else goes to the Internet Gateway |

The `0.0.0.0/0` route is the **default route** — it's a catch-all for any traffic not matched by a more specific rule. Pointing it at the IGW is what makes a subnet truly public.

The **private subnet** will only have the `local` route — no path to the internet.

---

## Concept 5: Security Groups vs. Network ACLs

This is one of the most important distinctions in AWS networking.

### Security Groups (SGs) — Stateful Firewall
- Operate at the **resource level** (attached to an EC2 instance, RDS, etc.)
- **Stateful**: if you allow inbound traffic, the return traffic is automatically allowed (AWS tracks the connection state)
- Only have **allow** rules — you cannot write a "deny" rule
- Changes take effect immediately

### Network ACLs (NACLs) — Stateless Firewall
- Operate at the **subnet level** (applied to all resources in a subnet)
- **Stateless**: you must explicitly allow BOTH inbound AND outbound traffic (no connection tracking)
- Have both **allow and deny** rules
- Rules are evaluated in order by rule number (lowest number first)

### Side-by-Side Comparison

| Feature | Security Group | Network ACL |
|---|---|---|
| Level | Resource (instance) | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Return traffic | Automatic | Must be explicit |
| Default behavior | Deny all inbound | Allow all (default NACL) |
| Use case | Per-instance firewall | Subnet-wide guardrails |

**For this lab**, you're only creating a Security Group. NACLs exist (AWS creates a default one), but you won't modify them.

Your `web-sg` security group will allow:
- Port 22 (SSH) — to connect to the server via terminal
- Port 80 (HTTP) — regular web traffic
- Port 443 (HTTPS) — encrypted web traffic

---

## Architecture You're Building

```
AWS Region
└── VPC: bootcamp-vpc (10.0.0.0/16)
    │
    ├── Internet Gateway (igw) ──── Internet
    │
    ├── Public Subnet (10.0.1.0/24) [AZ-a]
    │   ├── Route Table: 0.0.0.0/0 → igw
    │   └── Security Group: web-sg (allow 22, 80, 443)
    │
    └── Private Subnet (10.0.2.0/24) [AZ-b]
        └── Route Table: local only (no internet route)
```

---

## Lab Steps: AWS Console Walkthrough

> Sign in as your IAM admin user (not root) before starting.

---

### Step 1: Create the VPC

1. Go to **VPC** in the AWS Console (search "VPC" in the top bar)
2. In the left sidebar, click **Your VPCs**
3. Click **Create VPC**
4. Fill in:
   - **Resources to create**: VPC only *(not "VPC and more" — you want to understand each piece)*
   - **Name tag**: `bootcamp-vpc`
   - **IPv4 CIDR block**: `10.0.0.0/16`
   - **IPv6 CIDR block**: No IPv6 CIDR block
   - **Tenancy**: Default
5. Click **Create VPC**

**Why "VPC only"?** The "VPC and more" wizard auto-creates subnets, IGW, and route tables for you. That's fine later, but right now you want to build each piece manually so you understand what each one does.

---

### Step 2: Create the Public Subnet

1. In the left sidebar, click **Subnets**
2. Click **Create subnet**
3. Fill in:
   - **VPC ID**: Select `bootcamp-vpc`
   - **Subnet name**: `public-subnet-a`
   - **Availability Zone**: Choose the first AZ in your region (e.g., `us-east-1a`)
   - **IPv4 CIDR block**: `10.0.1.0/24`
4. Click **Create subnet**

---

### Step 3: Create the Private Subnet

1. Click **Create subnet** again
2. Fill in:
   - **VPC ID**: Select `bootcamp-vpc`
   - **Subnet name**: `private-subnet-b`
   - **Availability Zone**: Choose a **different** AZ (e.g., `us-east-1b`)
   - **IPv4 CIDR block**: `10.0.2.0/24`
3. Click **Create subnet**

> **Why different AZs?** Real architectures spread resources across AZs for fault tolerance. If one data center has an issue, the other AZ keeps running.

---

### Step 4: Create and Attach an Internet Gateway

1. In the left sidebar, click **Internet Gateways**
2. Click **Create internet gateway**
3. Fill in:
   - **Name tag**: `bootcamp-igw`
4. Click **Create internet gateway**
5. You'll see it in state **Detached** — it's created but not connected to anything yet
6. Click **Actions → Attach to VPC**
7. Select `bootcamp-vpc`
8. Click **Attach internet gateway**

The state should now show **Attached**. The IGW exists, but traffic can't flow through it yet — no route table points to it.

---

### Step 5: Create a Public Route Table

The default route table that came with your VPC only has a `local` route. You need a new one for the public subnet.

1. In the left sidebar, click **Route Tables**
2. Click **Create route table**
3. Fill in:
   - **Name**: `public-rt`
   - **VPC**: Select `bootcamp-vpc`
4. Click **Create route table**

**Add the internet route:**
1. Select `public-rt` from the list
2. Click the **Routes** tab at the bottom
3. Click **Edit routes**
4. Click **Add route**:
   - **Destination**: `0.0.0.0/0`
   - **Target**: Select **Internet Gateway**, then select `bootcamp-igw`
5. Click **Save changes**

**Associate the route table with the public subnet:**
1. Click the **Subnet associations** tab
2. Click **Edit subnet associations**
3. Check `public-subnet-a`
4. Click **Save associations**

Now `public-subnet-a` has a path to the internet. The private subnet still only has the local route (through the main/default route table — that's fine for now).

---

### Step 6: Enable Auto-Assign Public IPs on the Public Subnet

By default, instances launched in a subnet don't get a public IP. You need to enable this for the public subnet.

1. Go back to **Subnets**
2. Select `public-subnet-a`
3. Click **Actions → Edit subnet settings**
4. Check **Enable auto-assign public IPv4 address**
5. Click **Save**

---

### Step 7: Create the Security Group

1. In the left sidebar, click **Security Groups**
2. Click **Create security group**
3. Fill in:
   - **Security group name**: `web-sg`
   - **Description**: `Security group for web tier - allow SSH, HTTP, HTTPS`
   - **VPC**: Select `bootcamp-vpc` *(important — don't leave it on the default VPC)*
4. Under **Inbound rules**, click **Add rule** three times:

   | Type | Protocol | Port | Source | Notes |
   |---|---|---|---|---|
   | SSH | TCP | 22 | My IP | Only your IP can SSH in |
   | HTTP | TCP | 80 | 0.0.0.0/0 | Anyone can browse |
   | HTTPS | TCP | 443 | 0.0.0.0/0 | Anyone can browse securely |

5. Leave **Outbound rules** as default (all traffic allowed out)
6. Click **Create security group**

> **Why "My IP" for SSH?** Allowing SSH from `0.0.0.0/0` (anywhere) is a major security risk — bots scan for open port 22 constantly. Restricting to your IP is a basic but important security practice.

---

## Verification Checklist

Before calling the lab done, verify each component:

- [x] **VPC** `bootcamp-vpc` exists with CIDR `10.0.0.0/16`
- [x] **Public subnet** `public-subnet-a` exists with CIDR `10.0.1.0/24` in AZ-a
- [x] **Private subnet** `private-subnet-b` exists with CIDR `10.0.2.0/24` in AZ-b (different from AZ-a)
- [x] **Internet Gateway** `bootcamp-igw` is **attached** to `bootcamp-vpc`
- [x] **Route table** `public-rt` has routes: `10.0.0.0/16 → local` and `0.0.0.0/0 → bootcamp-igw`
- [x] **Route table** `public-rt` is associated with `public-subnet-a`
- [x] **Public subnet** has auto-assign public IP enabled
- [ ] **Security group** `web-sg` is in `bootcamp-vpc` with inbound rules for 22, 80, 443
- [ ] **Private subnet** has NO route to the IGW

---

## Key Concepts to Revisit

**Ask yourself these questions — if you can answer them, you understood the lab:**

1. What's the difference between a subnet being "named" public vs. actually being public?
2. Why does attaching an IGW alone not give internet access?
3. If you have two resources in the same VPC but different subnets, can they talk to each other without an IGW?
4. Why is a Security Group called "stateful"? What does that mean in practice?
5. What would happen if you associated the public route table with the private subnet?
6. Why use a /16 for the VPC but /24 for subnets?

---

## What This Enables

This VPC is now the foundation for everything else you'll build in AWS:
- **EC2 instances** go into subnets and use security groups
- **RDS databases** go into private subnets (no internet exposure)
- **Load balancers** sit in public subnets
- **NAT Gateways** (next step) let private subnet resources reach the internet for updates without being reachable from outside

You've built the network backbone. Every other AWS service runs on top of exactly this kind of structure.

---

## Cleanup (When You're Done)

To avoid any unexpected charges, delete resources in this order (reverse creation order):

1. Security Group: `web-sg`
2. Route Table: `public-rt` (first disassociate it from the subnet)
3. Subnets: `public-subnet-a` and `private-subnet-b`
4. Internet Gateway: detach from VPC first, then delete
5. VPC: `bootcamp-vpc`

> The VPC itself has no cost, but if you later launch EC2 instances or NAT Gateways inside it, those have costs. The networking components created in this lab (VPC, subnets, IGW, route tables, security groups) are all free.
