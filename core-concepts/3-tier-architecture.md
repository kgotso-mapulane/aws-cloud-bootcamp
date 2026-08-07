# 3-Tier Application Architecture

> A foundational architecture pattern used across cloud and software engineering.
> Understanding this is essential before designing or working on any multi-layered system.

---

## What It Is

A 3-tier application separates a system into three distinct layers, each with a single responsibility. The layers only communicate with their immediate neighbour — the presentation tier never talks directly to the database.

```
         Users / Browser / Mobile
               │
               ▼
┌──────────────────────────────┐
│       TIER 1: Presentation   │  ← what the user sees
│     (Web / Frontend layer)   │
└──────────────────────────────┘
               │
               ▼
┌──────────────────────────────┐
│       TIER 2: Application    │  ← business logic
│      (Backend / API layer)   │
└──────────────────────────────┘
               │
               ▼
┌──────────────────────────────┐
│        TIER 3: Data          │  ← storage
│       (Database layer)       │
└──────────────────────────────┘
```

---

## Each Tier Explained

**Tier 1 — Presentation**  
What the user interacts with. A website, a mobile app, a web interface. Its only job is to display information and collect input. It sends requests to the application tier and displays whatever comes back. No business logic lives here.

**Tier 2 — Application**  
The brain of the system. Receives requests from the presentation tier, applies business rules, makes decisions, and either returns a response or reads/writes from the database. This is where the actual logic lives — authentication, calculations, workflows, validations.

**Tier 3 — Data**  
Stores and retrieves data. A relational database (RDS, PostgreSQL), a NoSQL store (DynamoDB), or both. It never talks to the presentation tier directly — only the application tier can query it.

---

## In AWS Terms

When drawing this on a whiteboard, map each tier to actual AWS services:

```
                    Internet
                       │
              ┌────────────────┐
              │  Load Balancer │  (ALB)
              └────────────────┘
                       │
         ┌─────────────────────────┐
         │   PUBLIC SUBNET         │
         │   Tier 1: Web Servers   │  EC2 / S3 + CloudFront
         └─────────────────────────┘
                       │
         ┌─────────────────────────┐
         │   PRIVATE SUBNET        │
         │   Tier 2: App Servers   │  EC2 / Lambda / ECS
         └─────────────────────────┘
                       │
         ┌─────────────────────────┐
         │   PRIVATE SUBNET        │
         │   Tier 3: Database      │  RDS / DynamoDB
         └─────────────────────────┘
```

**Key points to cover when drawing:**

- Tier 1 sits in a **public subnet** — it needs to be reachable from the internet
- Tiers 2 and 3 sit in **private subnets** — no direct internet access, only reachable from within the VPC
- A **Load Balancer** (ALB) in front of Tier 1 distributes traffic and adds resilience
- Each tier has its own **Security Group** — Tier 2 only accepts traffic from Tier 1's security group, Tier 3 only accepts traffic from Tier 2's security group
- Spread across **multiple Availability Zones** for fault tolerance

---

## Why the Separation Matters

**Security**  
The database is never exposed to the internet. Even if Tier 1 is compromised, the attacker still can't reach the database directly. Each tier is a security boundary.

**Scalability**  
Each tier scales independently. If the application tier is under heavy load, scale that out without touching the web or database tier.

**Maintainability**  
A change to the frontend doesn't require touching the backend. Teams can work on each tier independently without stepping on each other.

**Well-Architected Framework alignment**  
The security, reliability, and performance efficiency pillars are all addressed by this design.

---

## Connection to Labs

This architecture is built piece by piece throughout the bootcamp:

| Lab | What it contributes to the 3-tier model |
|---|---|
| Lab 1 — IAM | IAM roles and security groups securing each tier |
| Lab 2 — Networking | The VPC, public and private subnets the tiers live in |
| Lab 3 — Compute | EC2 (Tier 1 & 2) and Lambda (Tier 2 alternative) |
| Future labs | RDS for Tier 3, Load Balancer, Auto Scaling |

---

## The One-Line Summary

*"A 3-tier architecture separates presentation, business logic, and data into independent layers — each in its own subnet, each with its own security group, communicating only with the layer directly next to it."*

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
