# Scalable, Load-Balanced Web Tier — The Why Behind the Build

> Understanding the problem, the service choices, and the lessons behind Lab 5.
> Useful context for explaining this as a project on a CV or in an interview.

---

## The Problem Being Solved

Imagine a company launches a website on a single EC2 instance. Everything works
fine until:

**The instance crashes** — the website is down. Every user gets an error. No one
can buy anything, log in, or use the service. The only fix is someone manually
noticing, logging in, and relaunching it. That could take 30 minutes or more.

**Traffic spikes** — a marketing campaign goes live, traffic doubles overnight.
The single instance is overwhelmed, pages load slowly or not at all, customers
leave. To fix it someone has to manually launch more instances, which takes time
and human effort.

**Maintenance is needed** — patching the OS or updating the app requires taking
the server down. That means downtime for every user.

One server is a single point of failure. It cannot scale. It cannot heal itself.
That is the problem. This architecture solves all three.

---

## Why These Specific Services

**Launch Template** — because manually configuring each server by hand is
error-prone and slow. When 10 identical servers are needed in 60 seconds, a
human cannot do that reliably. The template is the source of truth — every
server launched is guaranteed to be identical.

**Application Load Balancer** — because users should never need to know which
server they are talking to, and no single server should carry all the traffic.
The ALB is the single stable entry point. It distributes load, detects broken
servers, and stops sending traffic to them. Without it, users would hit
different IPs manually and one overloaded server would die while others sit idle.

**Auto Scaling Group** — because demand is unpredictable and paying for 10
servers 24/7 when only 2 are needed at night wastes money. The ASG matches
capacity to demand automatically. It also self-heals — if a server dies at 3am,
the ASG replaces it without waking anyone up.

**Two Availability Zones** — because AWS data centres can have outages. If both
servers are in the same building and that building loses power, the app goes
down. Spreading across two AZs means one failure does not take down the whole
system.

**Layered security groups (alb-sg → web-sg)** — because without this, anyone
who finds an instance's IP can bypass the load balancer entirely. Locking the
instances to only accept traffic from the ALB means all traffic flows through
one controlled point.

---

## Lessons to Take From This

**1. Design for failure, not uptime**  
The assumption in this architecture is that individual servers *will* fail. The
design accounts for that from the start rather than hoping it never happens.
This is the core mental shift from traditional IT to cloud architecture.

**2. Automation over manual intervention**  
Every recovery action in this architecture is automated. No one needs to be
woken up when a server dies. This is what separates scalable systems from
fragile ones.

**3. Stateless applications enable everything else**  
When you log into a website, the server needs to remember you are logged in.

A **stateful** server stores that login session in its own memory. Request 1
goes to Server A, which remembers you. Request 2 goes to Server B — which has
no idea who you are and asks you to log in again. That is broken.

A **stateless** server stores nothing locally. The session is saved in a shared
place — like a database (DynamoDB) or a cache (ElastiCache) — that every server
can read. It does not matter whether request 1 goes to Server A and request 2
goes to Server B. Both read the same shared store and find the session.

```
STATEFUL:
User → Server A (remembers login) ✔
User → Server B (knows nothing)   ✗ — broken

STATELESS:
User → Server A → reads session from shared store ✔
User → Server B → reads session from shared store ✔
```

Think of it like a coffee shop. A stateful barista remembers your order in
their head — if they are off sick, you start over. A stateless coffee shop
writes your order on a card that any barista can pick up and fulfil. It does
not matter who serves you.

When any server can handle any request, servers can be added freely during
spikes, removed afterwards, and replaced when they fail — without losing
anyone's session. The moment one server holds unique knowledge, all of that
flexibility disappears. Stateless design is what makes horizontal scaling
possible.

**4. Cost follows demand**  
Min 2 at night, scale to 4 during peak, scale back down. Pay for what is used.
This is fundamentally different from traditional infrastructure where capacity
is bought for peak and wasted the rest of the time.

**5. Security is layered, not bolted on**  
The ALB faces the internet. The instances face only the ALB. The database faces
only the application tier. Each layer only talks to the layer next to it. This
is defense in depth — compromising one layer does not give access to everything.

---

## How to Explain It on a CV and in an Interview

**One-line CV description:**  
*"Designed and deployed a scalable, self-healing web tier on AWS using an
Application Load Balancer, Auto Scaling Group, and Launch Templates across
multiple Availability Zones."*

**What to say when asked to explain it:**  
*"The problem was a single EC2 instance — one point of failure, no ability to
handle traffic spikes. I solved this by putting a load balancer in front of a
fleet of identical instances managed by an Auto Scaling Group. The ASG
automatically replaces failed instances and scales capacity based on CPU load.
Spreading across two Availability Zones means a single data centre failure
does not take the application down. I also layered the security groups so
instances can only be reached through the load balancer, not directly."*

**What makes it stand out:**  
Most people can follow lab steps. What impresses interviewers is explaining
*why* each decision was made — why the ALB gets its own security group, why
ELB health checks are preferred over EC2 checks, why the launch template has
no subnet. Those details show genuine understanding, not just lab completion.

---

## Connection to the Well-Architected Framework

This architecture directly addresses all six pillars:

| Pillar | How this architecture addresses it |
|---|---|
| **Security** | Layered security groups, no direct instance access, traffic only through ALB |
| **Reliability** | Multi-AZ deployment, self-healing via ASG, ELB health checks |
| **Performance Efficiency** | Right-sized instances (t3.micro), scales to demand |
| **Cost Optimisation** | Scales down when idle, no paying for unused capacity |
| **Operational Excellence** | Fully automated recovery, no manual intervention needed |
| **Sustainability** | Scales down when not needed, reduces idle resource consumption |

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
