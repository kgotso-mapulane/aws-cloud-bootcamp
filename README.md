# AWS Cloud Bootcamp

My personal AWS learning journey, documented from scratch. Each lab builds on
the previous one — starting with account security and working up through
networking, compute, storage, and beyond.

---

## Labs

| Lab | Topic | Status |
|---|---|---|
| [Lab 1 — Account Setup & IAM](./lab-01-account-setup/) | Creating an AWS account, securing root, IAM users, groups, roles, billing alerts | Completed |
| [Lab 2 — Networking Fundamentals](./lab-02-networking/) | Custom VPC, subnets, Internet Gateway, route tables, security groups | Completed |
| [Lab 3 — Compute: EC2 & Serverless](./lab-03-compute/) | EC2 web server, IAM role attachment, user data, AWS Lambda | Completed |
| [Lab 4 — Storage: S3, EBS & Lifecycle](./lab-04-storage/) | S3 versioning, lifecycle rules, EBS volumes, snapshots | Completed |
| [Lab 5 — Scalable, Load-Balanced Web App](./lab-05-scalability/) | Launch Template, ALB, Auto Scaling Group, target tracking, session stickiness | Completed |
| [Lab 6 — Account Budgets & Cost Alerts](./lab-06-budgets/) | Monthly cost budget, zero-spend tripwire, SNS fan-out alerting, Cost Explorer | Completed |

---

## Core Concepts

Important concepts that span across multiple labs and are worth understanding deeply.

| Concept | Description |
|---|---|
| [3-Tier Application Architecture](./core-concepts/3-tier-architecture.md) | How presentation, application, and data layers are structured and secured in AWS |
| [Scalable Web Tier — The Why Behind the Build](./core-concepts/scalable-web-tier-explained.md) | Problem statement, service choices, stateless design, and how to explain it as a project |
| [Subnets and CIDR Notation Explained](./core-concepts/subnets-and-cidr-explained.md) | What a subnet actually is, how to read CIDR notation, VPC vs subnet hierarchy |

---

## Approach

Each lab follows the same pattern:
- Understand the concepts before touching the console
- Draw the architecture manually before building it
- Complete the lab in the AWS Console
- Screenshot key console states as evidence of the build
- Document what clicked and what was confusing

The goal is genuine understanding, not just following steps.

---

## Stack

- AWS Console (no CLI or IaC yet — console first to understand what each service does)
- AWS Free Tier

---

*Started July 2026*
