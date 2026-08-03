# AWS Account Setup & IAM Best Practices Lab
## Secure a Brand-New AWS Account and Build a Least-Privilege Identity Model

---

## Before You Start: The Mental Model

Think of AWS identity and access like a company's security system:

| Real World | AWS Equivalent |
|---|---|
| The building owner with master keys | Root user |
| A senior employee with broad access | IAM user with AdministratorAccess |
| A department (e.g. Finance, Engineering) | IAM group |
| A visitor badge for a specific task | IAM role |
| A door access rule ("Finance can enter floor 3") | IAM policy |
| Security audit log | AWS CloudTrail |

The key idea: **in AWS, nothing has access to anything by default.** Every permission must be explicitly granted.

---

## Concept 1: The Root User

When you create an AWS account, the email address you signed up with becomes the **root user**. It has unrestricted access to everything in the account — billing, account closure, deleting all resources, everything.

**You should almost never use the root user for day-to-day work.** The only tasks that genuinely require root are a small list of account-level actions (like enabling IAM billing access, changing the support plan, or closing the account).

The moment your account is set up, your job is to:
1. Secure the root user (MFA + strong password)
2. Create an IAM admin user for daily work
3. Put the root credentials somewhere safe and stop using them

---

## Concept 2: IAM — Identity and Access Management

IAM is the service that controls **who can do what** in your AWS account. The four building blocks:

### Users
A **user** is an identity representing a person (or application) that needs access to AWS. Each user has their own credentials — a password for Console access, and optionally access keys for programmatic access.

### Groups
A **group** is a collection of users. You attach permissions to the group, and every user in the group inherits those permissions. This is how you manage permissions at scale — change the group policy once instead of updating every individual user.

### Policies
A **policy** is a JSON document that defines permissions. It says what **actions** are allowed (or denied) on what **resources** under what **conditions**.

AWS provides hundreds of pre-built **managed policies** (like `AdministratorAccess` and `ReadOnlyAccess`) you can attach without writing any JSON.

### Roles
A **role** is an identity with permissions that can be **assumed temporarily** — by a service, a user, or an external identity. No long-lived credentials. This is how EC2 instances, Lambda functions, and other AWS services get permissions securely.

```
IAM Summary:
├── Users      → individual identities (people or apps)
├── Groups     → collections of users sharing the same permissions
├── Policies   → the actual permission rules (what's allowed/denied)
└── Roles      → temporary identities assumed by services or users
```

---

## Concept 3: Least Privilege

**Least privilege** means granting only the permissions needed to do a specific job — nothing more.

| Example | Too Much | Just Right |
|---|---|---|
| Developer needs to deploy code | AdministratorAccess | Access only to CodeDeploy + S3 |
| App reads from a database | Full RDS access | Read-only access to one table |
| Auditor reviews infrastructure | Full account access | ReadOnlyAccess policy |

In a lab or personal account, `AdministratorAccess` for your own admin user is acceptable. In production, every identity should have the minimum permissions required for its specific job.

---

## Concept 4: IAM Roles vs. Access Keys

This is one of the most important security concepts in AWS.

| | IAM User (Access Keys) | IAM Role |
|---|---|---|
| **Credentials** | Long-lived, static keys | Temporary, auto-rotated |
| **Used by** | A person or application | A service or a user assuming it |
| **Risk if leaked** | High — keys work until manually revoked | Low — credentials expire automatically |
| **Rotation** | Manual | Automatic |
| **Best for** | Human Console sign-in | Service-to-service communication |

**The rule:** wherever a service (EC2, Lambda, etc.) needs to call AWS, use a role. Never embed access keys in application code or config files. You'll put a role on an EC2 instance in Lab 3.

---

## Concept 5: The 2025 Free Tier Model

> Applies to accounts created on or after **15 July 2025**.

New accounts get:
- **$200 in credits** ($100 at sign-up, $100 more as you use core services)
- A **free account plan** — no charges until you choose to upgrade
- Expires after **6 months** or when credits run out

Over **30 always-free services** (including IAM) remain free regardless of plan. Accounts created before that date keep the legacy 12-month Free Tier.

IAM — users, groups, roles, and policies — **always free**, no matter what.

---

## Architecture You're Building

```
AWS Account
│
├── Root User (secured with MFA — used only when absolutely necessary)
│
├── IAM Password Policy (enforced account-wide)
│
├── Admins Group
│   └── Policy: AdministratorAccess
│       └── User: YourName-admin (MFA enabled)
│
├── ReadOnly Group
│   └── Policy: ReadOnlyAccess
│       └── User: auditor
│
└── IAM Role: EC2-S3-ReadOnly
    └── Trusted entity: EC2
    └── Policy: AmazonS3ReadOnlyAccess
```

---

## Lab Steps: AWS Console Walkthrough

> **Duration:** 60–75 min | **Level:** Some IT background | **Cost:** AWS Free Tier

---

### Part A — Account Setup & Root Protection

---

#### Step 1: Create the AWS Account

1. Go to **aws.amazon.com** and choose **Create an AWS Account**.
2. Enter a root email address and an account name.
   > Use a dedicated email address for this account, not a personal inbox you share with other things.
3. Choose the **free account plan** when prompted.
4. Complete identity verification and enter a credit/debit card.
   > Required even for the free plan — you will **not** be charged while on the free account plan.
5. Select the **Basic support plan** (free).
6. Sign in as root to confirm the account is active.

> **⚠ The root user is all-powerful**
> The root user can close the account, change billing, and delete everything. After setup, treat the credentials like the keys to the building — store them safely and use them as rarely as possible.

---

#### Step 2: Enable MFA on the Root User

1. Sign in as root.
2. Open the **IAM** console (search "IAM" in the top bar).
3. A red security recommendations banner will appear — click it, or navigate to **My Security Credentials** from the account menu (top right).
4. Under **Multi-factor authentication (MFA)**, choose **Assign MFA device**.
5. Select **Authenticator app**.
6. Open your authenticator app (Google Authenticator, Authy, 1Password, etc.) and scan the QR code.
7. Enter two consecutive 6-digit codes to confirm the setup.
8. Click **Add MFA**.

> **✔ Best practice**
> Also set a strong, unique root password and store it in a password manager. If the root user has any access keys listed under **Access keys**, delete them — the root user should have no programmatic keys at all.

---

#### Step 3: Set a Billing Safety Net

1. Open **Billing and Cost Management** (search "Billing" in the top bar).
2. In the left sidebar, go to **Budgets** → **Create budget**.
3. Choose **Zero spend budget** — this alerts you the moment any real charge appears.
   - Alternatively, set a **$5 monthly cost budget** if you want a small buffer.
4. Add your email address as the alert recipient.
5. Optionally add an **SMS phone number** for alerts — if you always have your phone on you, this is a faster way to catch unplanned charges.
6. Click **Create budget**.

> **ⓘ Why this matters**
> Even on the free plan, a misconfigured resource can generate charges once credits run out. A budget is your early-warning system. You'll go deeper on cost management in the Day Two budgets lab.

> **⚠ Note on billing access**
> If you're signed in as an IAM user (not root) and can't access the Billing Console, it's because IAM billing access hasn't been enabled at the root level yet. Go to root → Account settings → enable **IAM user and role access to Billing information**, then retry as the IAM user. You'll hit this if you try to set the budget after creating your admin user — it's a deliberate AWS security gate.

---

### Part B — IAM Identities in the Console

---

#### Step 4: Set an Account Password Policy

1. In the **IAM** console, go to **Account settings** in the left sidebar.
2. Under **Password policy**, click **Edit**.
3. Configure:
   - Minimum password length: **14**
   - Require at least one uppercase letter: ✔
   - Require at least one lowercase letter: ✔
   - Require at least one number: ✔
   - Require at least one non-alphanumeric character: ✔
   - Password expiration: **90 days**
4. Click **Save changes**.

This policy applies to all IAM users in the account — they'll be required to meet these rules when setting or changing their password.

---

#### Step 5: Create the Admins Group

1. In **IAM**, go to **User groups** → **Create group**.
2. Fill in:
   - **Group name**: `Admins`
3. Under **Attach permissions policies**, search for and select `AdministratorAccess`.
4. Click **Create user group**.

> **✔ Attach policies to groups, not users**
> This is a best practice. If you need to change what an admin can do, you update the group policy once — not every individual user.

---

#### Step 6: Create Your IAM Admin User

1. In **IAM**, go to **Users** → **Create user**.
2. Fill in:
   - **User name**: e.g. `YourName-admin`
   - Check **Provide user access to the AWS Management Console**
   - Select **I want to create an IAM user**
   - Set a strong password (or auto-generate and note it down)
3. Click **Next**.
4. Under **Set permissions**, select **Add user to group** and choose `Admins`.
5. Click **Next**, then **Create user**.
6. Note the **Console sign-in URL** — it looks like `https://<account-id>.signin.aws.amazon.com/console`.
7. **Sign out of root** and sign back in as your new IAM admin user.

---

#### Step 7: Enable MFA on the Admin User

1. Signed in as the admin user, go to **IAM** → **Users** → click your username.
2. Go to the **Security credentials** tab.
3. Under **Multi-factor authentication (MFA)**, click **Assign MFA device**.
4. Repeat the same process as root — authenticator app, scan QR code, enter two consecutive codes.

From this point on, use the IAM admin user for all lab work. Root stays locked away.

---

#### Step 8: Create a Read-Only User (Least-Privilege Demo)

1. In **IAM**, go to **User groups** → **Create group**.
   - **Group name**: `ReadOnly`
   - Attach policy: `ReadOnlyAccess`
   - Click **Create user group**.

2. In **IAM** → **Users** → **Create user**.
   - **User name**: `auditor`
   - Enable Console access, set a password.
   - Add to the `ReadOnly` group.
   - Create the user.

3. Open a **private/incognito browser window** and sign in as `auditor`.
4. Try to create something — for example, go to **S3** and attempt to create a bucket.
5. The action will be denied.

This is least privilege in action: `auditor` can see everything, but cannot change anything. The policy says so, and AWS enforces it.

---

### Part C — IAM Roles

---

#### Step 9: Create a Service Role for EC2

A role lets an AWS service call other AWS services securely — no credentials stored anywhere.

1. In **IAM**, go to **Roles** → **Create role**.
2. Under **Trusted entity type**, select **AWS service**.
3. Under **Use case**, select **EC2**. Click **Next**.
4. Search for and attach the policy `AmazonS3ReadOnlyAccess`. Click **Next**.
5. Fill in:
   - **Role name**: `EC2-S3-ReadOnly`
   - **Description**: `Allows EC2 instances to read from S3`
6. Click **Create role**.

This role will be attached to an EC2 instance in Lab 3. When that instance needs to read from S3, it assumes this role and gets temporary credentials automatically — no access keys required.

> **✔ Rule of thumb**
> Prefer roles over access keys wherever a service needs to call AWS. For human sign-in, prefer IAM Identity Center (SSO) over long-lived access keys.

---

## Verification Checklist

Before calling the lab done, confirm each item:

- [x] Root user has MFA enabled; root access keys removed (or never existed)
- [x] Budget created with email (and optionally SMS) alerts
- [x] Account password policy set (14 chars, complexity, 90-day expiry)
- [x] `Admins` group created with `AdministratorAccess` policy
- [x] IAM admin user created, added to `Admins` group, with MFA enabled
- [x] Signed in as IAM admin user — not root — for the rest of the lab
- [x] `ReadOnly` group created with `ReadOnlyAccess` policy
- [x] `auditor` user created, added to `ReadOnly` group
- [x] Confirmed `auditor` cannot create resources (denied when attempted)
- [x] `EC2-S3-ReadOnly` role created with `AmazonS3ReadOnlyAccess`, trusted entity EC2

---

## Key Concepts to Revisit

Ask yourself these questions — if you can answer them, you understood the lab:

1. Why should you almost never use the root user for daily work?
2. What's the difference between attaching a policy to a user vs. attaching it to a group?
3. Why does an IAM admin user need root to enable billing access — doesn't `AdministratorAccess` cover everything?
4. What makes a role more secure than long-lived access keys for service-to-service communication?
5. If you wanted to give a new team member read-only access to all AWS services, what's the fastest and most maintainable way to do it?
6. What happens to a role's credentials when the session expires?

---

## Cleanup & Teardown

IAM resources are free, so cleanup here is about security hygiene and keeping things tidy.

| What | Where in the Console |
|---|---|
| Remove user from group | IAM → Users → the user → Groups tab → Remove |
| Delete demo user | IAM → Users → select `auditor` → Delete |
| Delete demo group | IAM → User groups → select `ReadOnly` → Delete |
| Delete demo role (optional) | IAM → Roles → select `EC2-S3-ReadOnly` → Delete |

> **⚠ Keep these for the rest of the bootcamp**
> Do NOT delete: your admin user, the `Admins` group, MFA devices, the password policy, the `EC2-S3-ReadOnly` role (used in Lab 3), or the budget.

---

## My Notes & Reflections

### Concepts That Resonated

**Least Privilege** — Not new to me; I already apply this at work. Good to see it reinforced as a core AWS security pillar. Same idea: give only the permissions needed, nothing more.

**IAM Roles** — What clicked for me is how roles enable seamless, secure communication between AWS resources without hardcoding credentials. A service assumes a role and gets just the permissions it needs, dynamically. Elegant, and it maps well to patterns I already see in production environments.

### My Aha Moment — The Billing Permission Hunt

After creating the admin user and attaching the admin policy, I put the root user credentials away — thinking I wouldn't need them anytime soon. An admin user with `AdministratorAccess` felt like enough for where I was in the learning journey.

That assumption didn't last long. When I went to create a budget, I needed to access the Billing Management Console — and the admin user couldn't get in. Access was blocked.

The fix required going back to the root user: AWS has a specific account-level setting that must be enabled by root to allow IAM users (even admins) to access the Billing Console at all. I had to dig out the root credentials I had just put away, log in as root, enable IAM access to billing, then switch back to the admin user to finally create the budget.

Good early lesson: **admin policy doesn't automatically grant billing access** — that gate is controlled by root, deliberately. Knowing when you actually need the root user, and having those credentials accessible when the moment comes, matters.

This reminded me of work where we use **CyberArk EPV (Enterprise Password Vault)** to securely store and retrieve privileged credentials in production. The discipline of knowing where your most privileged credentials live — and when you actually need them — is just as relevant in the cloud.

### Budget Threshold & SMS Alerts

One of my favourite parts of this lab. I set a budget threshold and added SMS as the alert channel — since I always have my phone on me, I'll catch any unplanned spend early. Simple but effective, and something I'll carry into every future lab.

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
