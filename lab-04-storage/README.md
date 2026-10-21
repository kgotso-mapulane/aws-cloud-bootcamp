# AWS Cloud Bootcamp — Lab 4: Storage — S3, EBS & Lifecycle

## What This Is

This lab applies AWS storage concepts through hands-on work with S3 and EBS. It covers configuring S3 versioning and lifecycle rules, attaching an EBS volume to an EC2 instance, and creating an EBS snapshot using the AWS Console.

  
**Interface:** AWS Console  
**Goal:** Build and compare object and block storage solutions using Amazon S3 and EBS, and understand when each is appropriate

---

## What I Built

| Component | Setting | Purpose |
|---|---|---|
| S3 bucket | Versioning on, public access blocked | Object storage for files and backups |
| S3 lifecycle rule `archive-and-expire` | IA at 30d, Glacier at 90d, expire at 365d | Automate cost savings as data ages |
| EBS volume | gp3, 1 GiB, same AZ as EC2 | Virtual disk attached to web-01 |
| Volume mount | Formatted XFS, mounted at `/data` | Ready to use like a physical disk |
| EBS snapshot | `bootcamp data volume backup` | Point-in-time backup of the volume |

---

## Architecture

```
AWS Account
│
├── Part A: Amazon S3
│   └── Bucket: invoicenow-bootcamp-<name>-<random>
│       ├── Versioning: enabled
│       ├── Public access: blocked
│       ├── Objects: uploaded files with multiple versions
│       └── Lifecycle rule: archive-and-expire
│           ├── Day 30  → Standard-IA
│           ├── Day 90  → Glacier Flexible Retrieval
│           ├── Day 365 → Expire current versions
│           └── Day 60 after noncurrent → Delete old versions
│
└── Part B: Amazon EBS
    └── Volume: gp3, 1 GiB (same AZ as EC2 instance)
        ├── Attached to: web-01
        ├── Mounted at: /data
        └── Snapshot: bootcamp data volume backup
```

---

## Key Things I Learned

**S3 and EBS are not interchangeable — they solve completely different problems.**  
S3 is object storage accessed over HTTPS from anywhere — files, backups, media.
EBS is a virtual disk locked to one EC2 instance in one AZ — OS, databases,
anything needing low-latency block-level access. Knowing which to reach for
is a fundamental cloud skill.

**EBS is AZ-locked — this is a hard constraint, not a suggestion.**  
A volume can only attach to an instance in the same Availability Zone. To
move data to another AZ, a snapshot must be created first, then a new volume
created from that snapshot in the target AZ.

**S3 versioning protects against accidental overwrites and deletions.**  
When versioning is on, overwriting a file creates a new version — the original
is never lost. Deleting a file adds a delete marker but the original versions
remain recoverable. In production, versioning should almost always be enabled.

**Lifecycle rules automate cost savings without any manual work.**  
Data that starts as hot (Standard) naturally becomes cold over time. A lifecycle
rule moves it through cheaper storage classes automatically — Standard-IA,
then Glacier — and eventually expires it. No human intervention needed once
the rule is set.

**Snapshots are incremental — not full copies each time.**  
The first snapshot copies everything. Every snapshot after that only stores
what changed since the previous one. This keeps storage costs low even when
snapshotting frequently.

**Deleting a volume does not delete its snapshots.**  
Snapshots live independently of the volume they came from. They must be
deleted separately. Easy to forget, and easy to accumulate unexpected costs.

**EBS volumes bill continuously even when unattached.**  
Unlike EC2 instances which can be stopped to pause compute costs, an EBS
volume charges by GB-month regardless of whether it is attached or in use.
Always delete volumes and snapshots when they are no longer needed.

**The common production pattern: EBS for running, S3 for backup.**  
Applications run on EBS for low-latency disk access. Backups and exports go
to S3 with a lifecycle policy that moves them to Glacier over time — keeping
costs low while retaining long-term durability.

---

## What Confused Me Initially

- S3 bucket names being globally unique across all AWS accounts — not just
  within an account or region, but across every AWS customer worldwide
- Snapshots being stored in S3 but not visible in the S3 console — they are
  managed entirely through EC2 and have their own separate console view
- The order of cleanup — unmount inside the OS first, then detach, then delete
  the volume. Skipping the unmount step can corrupt data

---

## Key Concepts

**1. EBS AZ constraint** — EBS is a zone-specific service. A volume is created
in a single Availability Zone and can only attach to an instance in that same
AZ. This is a hard AWS constraint, not a configuration choice. To move data
to another AZ, a snapshot must be created first, then a new volume created
from that snapshot in the target AZ. Snapshots are the bridge between AZs.

**2. S3 versioning and delete markers** — when versioning is enabled and an
object is deleted, S3 does not remove it permanently. Instead it adds a
**delete marker** — a placeholder that hides the object from normal views,
making it appear deleted. The original versions are still there and can be
restored by removing the delete marker manually. This protects against
accidental deletions but restoration requires deliberate action, it does not
happen automatically.

**3. Glacier Flexible Retrieval vs Deep Archive** — these are two different
storage classes with different retrieval times. Glacier Flexible Retrieval
(used in this lab) takes **minutes to a few hours** to retrieve data.
Glacier Deep Archive is for data that is almost never needed and takes
**up to 12 hours**. Choosing the wrong one can mean waiting much longer than
expected to access archived files — worth knowing the difference before
designing a lifecycle policy.

**4. Where snapshots actually live** — snapshots are stored in S3 by AWS
internally, but they are not visible in the S3 console. AWS manages that
storage on behalf of the account. Snapshots are independent of any EC2
instance — they are not attached to EC2, they are simply managed through
the EC2 console for convenience. A snapshot can exist long after the volume
it came from has been deleted.

**5. S3 for globally accessible objects** — an application needing to store
user-uploaded profile images accessible globally over HTTPS belongs in S3.
EBS cannot serve files over the internet — it is a disk attached to one
instance. S3 is the right choice for any object that needs to be reachable
via a URL.

**6. Deleting a volume does not delete its snapshots** — this is a common
and costly mistake. EBS volumes and their snapshots are completely independent.
Deleting a volume leaves all its snapshots intact and still billing by
GB-month. Snapshots must be found and deleted separately in EC2 → Snapshots.
Always check for orphaned snapshots after deleting volumes to avoid
unexpected charges accumulating silently.

**7. Block all public access on S3** — this setting exists purely for
security. S3 buckets and objects are private by default, but permissions
can be misconfigured — accidentally making a bucket public is one of the
most common causes of AWS data breaches. Blocking public access at the
bucket level acts as a safety net that overrides any permissive policies
that might be applied accidentally. It should stay ON unless there is a
deliberate, understood reason to turn it off, such as hosting a static
website where public read access is intentional.

## My Aha Moment — Snapshots

What was surprising is that deleting the original EBS volume leaves the
snapshot completely intact — still there, still billing. A backup outliving
the thing it backed up felt counterintuitive at first, but it makes sense:
the snapshot is an independent copy, not tied to the volume's lifecycle.

That led to another question: if the original volume is deleted and something
needs to be updated, can the snapshot be edited directly? The answer is no —
snapshots are **read-only**. They are a frozen point-in-time copy and cannot
be modified. The way to work with a snapshot is to create a new EBS volume
from it, attach that volume to an instance, and make changes there. The
snapshot stays untouched as the backup.

Also surprising was not seeing snapshots anywhere in the S3 console despite
knowing they are stored in S3 internally. AWS manages that storage completely
behind the scenes — it is not accessible or visible as a normal S3 bucket.
Snapshots live in their own world under EC2 → Snapshots, separate from
everything else.

**S3 charges for DELETE requests too.**  
Most people assume deleting objects saves money immediately. There are actually
request charges for deletions, and with versioning on, "deleting" an object
just adds a delete marker — which is itself a billable object. Costs don't
drop the moment something is deleted.

**An S3 bucket name can never be reused once deleted.**  
Once a bucket is deleted, that name is gone globally for a period of time —
no AWS account anywhere in the world can create a bucket with that exact name
again immediately. In production this makes naming conventions important.
Losing a well-known bucket name because of an accidental deletion can be a
serious problem.

**EBS root volume deletion behaviour depends on launch settings.**  
When launching an EC2 instance, the root EBS volume has a "Delete on
termination" setting. By default it is set to YES for the root volume —
meaning terminating the instance also deletes the root disk permanently.
For additional data volumes attached separately, the default is NO — they
persist after termination and keep billing. Easy to get caught out if not
checked during launch configuration.

**Lifecycle rules have minimum storage duration charges.**  
Objects transitioned to Standard-IA have a minimum 30-day storage charge
regardless of when they are moved or deleted. Moving an object to Standard-IA
after 5 days does not save money — AWS still bills for 30 days in that class.
Lifecycle policies need to account for minimum durations, otherwise the
automation designed to save costs can end up costing more than staying in
Standard.

---

| File | What It Is |
|---|---|
| `README.md` | This file — summary and reflections |
| `s3-ebs-lifecycle-lab.md` | Full concept notes and step-by-step lab walkthrough |

---

## Lab Status

- [x] S3 bucket created with versioning on and public access blocked
- [x] Uploaded a file twice and saw multiple versions
- [x] Lifecycle rule created with transition and expiration actions
- [x] EBS volume created, attached, formatted, and mounted
- [x] Snapshot created and confirmed completed
- [x] All storage resources deleted after lab

## Screenshots

- [x] S3 console — bucket with versioning enabled badge visible → `screenshots/01-s3-versioning-enabled.png`
- [x] S3 console — object versions panel showing two versions of the same file → `screenshots/02-object-versions.png`
- [x] S3 console — lifecycle rule archive-and-expire showing transition actions → `screenshots/03-lifecycle-rule.png`
- [x] EC2 console — Snapshot in Completed state → `screenshots/04-ebs-snapshot.png`
- [x] EC2 Instance Connect — df -h output confirming /data volume mounted → `screenshots/06-volume-mounted.png`

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
