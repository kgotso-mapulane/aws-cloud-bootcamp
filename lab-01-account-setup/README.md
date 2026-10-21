# AWS Cloud Bootcamp — Lab 1: Account Setup & IAM Best Practices

## What This Is

This lab applies AWS identity and access management concepts by securing an AWS account and building a least-privilege identity model using IAM. It covers root account protection, MFA, IAM users and groups, permissions, and service roles using the AWS Console

  
**Interface:** AWS Console  
**Goal:** Build and validate a secure, least-privilege identity and access foundation for AWS workloads.

---

## What I Built

An IAM identity structure from scratch with the following components:

| Component | Setting | Purpose |
|---|---|---|
| Root user | MFA enabled, credentials stored securely | Account owner — used only when absolutely necessary |
| Budget alert | Zero-spend threshold + SMS | Early warning against unplanned charges |
| Password policy | 14 chars, complexity, 90-day expiry | Enforced account-wide for all IAM users |
| Group `Admins` | AdministratorAccess policy | Admin permissions inherited by admin user |
| IAM user `kgotso-admin` | In Admins group, MFA enabled | Daily-use identity — replaces root for all lab work |
| Group `ReadOnly` | ReadOnlyAccess policy | View-only permissions for auditor user |
| IAM user `auditor` | In ReadOnly group | Least-privilege demo — can view, cannot modify |
| Role `EC2-S3-ReadOnly` | Trusted entity: EC2, AmazonS3ReadOnlyAccess | Service role for Lab 3 EC2 instance |

---

## Architecture

```
AWS Account
│
├── Root User (MFA enabled — stored away, rarely used)
│
├── IAM Password Policy (14 chars, complexity, 90-day expiry)
│
├── Budget: Zero-spend alert (email + SMS)
│
├── Admins Group
│   └── Policy: AdministratorAccess
│       └── User: kgotso-admin (MFA enabled) ← used for all lab work
│
├── ReadOnly Group
│   └── Policy: ReadOnlyAccess
│       └── User: auditor ← can view, cannot create or modify
│
└── Role: EC2-S3-ReadOnly
    ├── Trusted entity: EC2
    └── Policy: AmazonS3ReadOnlyAccess
```

---

## Key Things I Learned

**Least privilege is the same principle I apply at work — just in a new context.**  
Give only the permissions needed for the job, nothing more. Familiar concept,
but good to see it enforced structurally in AWS through groups and policies.

**IAM roles make resource communication seamless and secure.**  
A service assumes a role and gets temporary, auto-rotated credentials — no
hardcoded keys, no manual rotation. That's far cleaner than long-lived access
keys, and it maps well to patterns I already see in production environments.

**Admin policy does not automatically grant billing access.**  
AWS gates billing access at the root level, separate from any IAM policy.
Even `AdministratorAccess` won't get an IAM user into the Billing Console
until root explicitly enables IAM billing access in account settings.

**The root user is not just a more powerful admin — it's a different category.**  
Root owns the account. Certain controls (billing access, account closure,
support plan changes) are root-only by design. Treat root credentials like
the keys to the building — secure them and put them away.

**Attach policies to groups, not individual users.**  
If you need to change what a role can do, updating one group policy is far
more maintainable than touching every individual user.

---

## My Aha Moment — The Billing Permission Hunt

After creating the admin user and giving it `AdministratorAccess`, I put the
root credentials away — thinking an admin user would cover everything I needed
for now.

That assumption didn't last long. When I went to create a budget, the Billing
Management Console was blocked for the admin user — access denied.

The fix was to go back to root, find the account-level setting to enable IAM
user access to billing, flip it on, then return to the admin user to finish
creating the budget. Short detour, but an important one.

It reminded me of work where we use **CyberArk EPV (Enterprise Password Vault)**
to securely store and retrieve privileged credentials in production. The same
discipline applies here: know where your most privileged credentials are, and
know exactly when you actually need them.

---

## Least Privilege in Action

One part I genuinely enjoyed was signing in as the `auditor` user and trying
to create an S3 bucket — and watching it get denied. No workaround, no
override, just a flat rejection because the policy says read-only.

It's one thing to understand least privilege as a concept, it's another to
see it enforced live in an AWS environment. That moment made it real.

---

## What Confused Me Initially

- Thinking `AdministratorAccess` means access to *everything* — billing is a
  deliberate exception that requires a root-level toggle
- Putting root credentials away too early and not documenting where they were
  stored, which made retrieving them slower than it needed to be

---

## Files in This Lab

| File | What It Is |
|---|---|
| `README.md` | This file — summary and reflections |
| `aws-account-setup-lab.md` | Full concept notes and step-by-step lab walkthrough |

---

## Lab Status

- [x] Concepts studied and understood before starting
- [x] Root user secured with MFA
- [x] Budget created with SMS alerts
- [x] IAM password policy set
- [x] Admin user created and in use for all lab work
- [x] Read-only user created and confirmed it cannot modify resources
- [x] EC2-S3-ReadOnly role created
- [x] Resources cleaned up (keeping what's needed for Lab 3)

## Screenshots

- [x] IAM console — Users list showing admin user and auditor user → `screenshots/01-iam-users.png`
- [x] IAM console — Admins group with AdministratorAccess policy attached → `screenshots/02-admin-group.png`
- [x] IAM console — Security credentials tab showing MFA enabled on admin user → `screenshots/03-mfa-enabled.png`
- [x] IAM console — EC2-S3-ReadOnly role detail page → `screenshots/04-S3-readonly.png`
- [x] IAM console — EC2-S3-ReadOnly trust relationship showing ec2.amazonaws.com → `screenshots/05-trust-relationship.png`
- [x] Billing console — Budget dashboard showing both budgets → `screenshots/06-budgets.png`
- [x] S3 bucket creation attempt as auditor user — access denied error → `screenshots/07-access-denied.png`

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
