# AWS Storage Lab
## S3, EBS & Lifecycle Policies

---

## Before You Start: The Mental Model

Think of AWS storage like different ways to store things in a building:

| Real World | AWS Equivalent |
|---|---|
| A filing cabinet attached to one desk — only that desk can use it | EBS (block storage attached to one EC2 instance) |
| A shared document server anyone in the building can access over the network | S3 (object storage accessible via HTTPS from anywhere) |
| Moving old files to an off-site archive to save space | S3 Lifecycle policy (transition to cheaper storage classes) |
| Taking a photo of the filing cabinet contents for backup | EBS Snapshot |
| Keeping every version of a document when it changes | S3 Versioning |

The key idea: **S3 is for objects accessed over the internet. EBS is a virtual disk locked to one instance in one AZ. They solve different problems and are often used together.**

---

## Concept 1: Amazon S3 — Object Storage

**S3 (Simple Storage Service)** stores files as objects in buckets. Each object is a file plus its metadata, accessible via a unique URL.

### Key characteristics:
- **Globally unique bucket names** — no two buckets across all AWS accounts can share a name
- **Virtually unlimited storage** — no capacity planning needed
- **Accessed over HTTPS** — from anywhere, by anyone with the right permissions
- **Not a file system** — there are no folders, only key names that look like paths (`invoices/2025/jan.pdf`)
- **11 nines of durability** — AWS replicates objects across multiple AZs automatically

### S3 Storage Classes — Pay for What You Need

| Storage Class | Use Case | Retrieval |
|---|---|---|
| Standard | Frequently accessed data | Immediate |
| Standard-IA | Infrequent access, still needs to be fast | Immediate (small fee) |
| Glacier Flexible Retrieval | Archival, rarely accessed | Minutes to hours |
| Glacier Deep Archive | Long-term archival (years) | Up to 12 hours |
| Intelligent-Tiering | Unknown or changing access patterns | Automatic |

---

## Concept 2: S3 Versioning

When versioning is enabled on a bucket, S3 keeps every version of every object — even after overwriting or deleting.

- Upload `invoice.pdf` → Version 1 stored
- Upload `invoice.pdf` again → Version 2 stored, Version 1 still there
- Delete `invoice.pdf` → a delete marker is added, original versions still recoverable

This protects against accidental overwrites and deletions. In production, versioning should almost always be on.

---

## Concept 3: S3 Lifecycle Rules

Lifecycle rules automate moving objects between storage classes and expiring old versions on a schedule — no manual action required.

A typical lifecycle policy:

```
Day 0    → Object uploaded (Standard)
Day 30   → Automatically moved to Standard-IA
Day 90   → Automatically moved to Glacier
Day 365  → Automatically deleted
          + Noncurrent versions deleted after 60 days
```

This means data that was actively used becomes cheaper over time without any manual work, and old versions don't accumulate indefinitely.

---

## Concept 4: Amazon EBS — Block Storage

**EBS (Elastic Block Store)** is a virtual hard disk attached to an EC2 instance. From the instance's perspective, it looks exactly like a physical disk — it can be formatted, partitioned, and mounted.

### Key characteristics:
- **AZ-locked** — an EBS volume lives in one AZ and can only attach to instances in that same AZ
- **One instance at a time** — a standard EBS volume can only be attached to one EC2 instance at a time
- **Fixed size** — provisioned at a set size (resizable, but requires action)
- **Persistent** — data survives instance stops and reboots (unlike instance store)
- **billed continuously** — charges apply even when the volume is unattached

### EBS Volume Types:

| Type | Use Case |
|---|---|
| gp3 | General purpose SSD — default choice for most workloads |
| gp2 | Older general purpose SSD — gp3 is better and cheaper |
| io2 | High-performance SSD for databases with heavy I/O |
| st1 | Throughput-optimised HDD for large sequential reads |
| sc1 | Cold HDD — lowest cost, infrequently accessed |

---

## Concept 5: EBS Snapshots

A **snapshot** is a point-in-time backup of an EBS volume, stored in S3 behind the scenes (though not visible in the S3 console).

- **Incremental** — the first snapshot copies everything; subsequent snapshots only copy what changed
- **Region-scoped** — snapshots can be copied to other regions for disaster recovery
- **Used to move volumes across AZs** — create a snapshot, then create a new volume from it in a different AZ
- **Billed separately** from the volume — deleting a volume does not delete its snapshots

---

## Concept 6: S3 vs EBS — When to Use Which

| | S3 | EBS |
|---|---|---|
| **Type** | Object storage | Block storage (virtual disk) |
| **Access** | HTTPS from anywhere | Attached to one EC2 in one AZ |
| **Scales** | Virtually unlimited | Fixed size |
| **Good for** | Backups, media, data lakes, static sites | OS disk, databases, low-latency I/O |
| **Billed** | Per GB stored + requests | Per GB provisioned per month |
| **Durability** | 11 nines (multi-AZ by default) | Single AZ (snapshots add resilience) |

**Common pattern in production:** run applications on EBS for low-latency disk access, push backups and exports to S3 with a lifecycle policy that moves them to Glacier over time.

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

## Lab Steps: AWS Console Walkthrough

> Sign in as the IAM admin user (not root) before starting.

---

### Part A — Amazon S3

---

#### Step 1: Create a Bucket

1. Search "S3" in the top bar and open the S3 console.
2. Click **Create bucket**.
3. Fill in:
   - **Bucket name**: `invoicenow-bootcamp-<yourname>-<random>` — must be globally unique across all of AWS
   - **Region**: us-east-1
   - **Block all public access**: leave ON
4. Under **Bucket Versioning**, select **Enable**.
5. Click **Create bucket**.

> **⚠ Public access**
> Block all public access should stay ON unless there is a specific reason (like static website hosting). Most S3 data breaches come from buckets accidentally made public.

---

#### Step 2: Upload Objects and See Versioning in Action

1. Click into the bucket → **Upload** → **Add files**.
2. Pick any file (a sample PDF or text file) → **Upload**.
3. Edit the file locally (change something small), then upload it again with the **exact same filename**.
4. At the top of the Objects list, toggle **Show versions**.
5. Both versions appear — S3 kept the original even after the overwrite.

---

#### Step 3: Create a Lifecycle Rule

1. Inside the bucket, click the **Management** tab.
2. Click **Create lifecycle rule**.
3. Fill in:
   - **Rule name**: `archive-and-expire`
   - **Rule scope**: Apply to all objects in the bucket (acknowledge the warning)
4. Tick and configure the following actions:

   | Action | Setting |
   |---|---|
   | Transition current versions to another storage class | Standard-IA after **30 days**; Glacier Flexible Retrieval after **90 days** |
   | Expire current versions of objects | After **365 days** |
   | Permanently delete noncurrent versions | **60 days** after becoming noncurrent |

5. Review the summary timeline and click **Create rule**.

> **✔ Storage class reminder**
> Standard → hot data. Standard-IA → occasional access. Glacier → archival, rarely touched. Intelligent-Tiering → when access patterns are unpredictable and automation is preferred.

---

### Part B — Amazon EBS

---

#### Step 4: Create and Attach a Volume

1. Go to **EC2** → in the left sidebar under **Elastic Block Store**, click **Volumes**.
2. Click **Create volume**.
3. Fill in:
   - **Volume type**: gp3
   - **Size**: 1 GiB
   - **Availability Zone**: must match the AZ of `web-01` (check the instance details if unsure)
4. Click **Create volume**.
5. Select the new volume → **Actions → Attach volume**.
6. Choose `web-01` as the instance, device name `/dev/sdf`.
7. Click **Attach volume**.

> **⚠ AZ-locked**
> The volume and instance must be in the same AZ. This is a hard constraint — AWS will not allow attaching a volume to an instance in a different AZ.

---

#### Step 5: Format and Mount the Volume

1. Connect to `web-01` via **EC2 Instance Connect** (EC2 → select `web-01` → Connect).
2. In the browser terminal, run:

```bash
lsblk                                    # find the new device — likely xvdf
sudo mkfs -t xfs /dev/xvdf               # format it as XFS
sudo mkdir /data && sudo mount /dev/xvdf /data   # create mount point and mount
df -h /data                              # confirm it's mounted and shows 1GB
```

The volume is now formatted and mounted at `/data` — ready to use like any disk.

---

#### Step 6: Create a Snapshot

1. Go to **EC2 → Volumes**, select the volume just created.
2. Click **Actions → Create snapshot**.
3. Fill in:
   - **Description**: `bootcamp data volume backup`
   - Add a **Name tag**: `bootcamp-snapshot`
4. Click **Create snapshot**.
5. Go to **EC2 → Snapshots** — the snapshot appears and moves to **Completed**.

Snapshots are incremental — the first one copies everything, subsequent ones only copy what changed since the last snapshot.

---

## Verification Checklist

- [ ] S3 bucket created with versioning enabled and public access blocked
- [ ] Uploaded a file twice with the same name and saw both versions
- [ ] Lifecycle rule created with transition and expiration actions
- [ ] EBS volume created in the same AZ as the EC2 instance
- [ ] Volume attached, formatted, and mounted at `/data`
- [ ] Snapshot created and shows Completed status
- [ ] All resources deleted after the lab

---

## Key Concepts to Revisit

1. Why must an EBS volume be in the same AZ as the EC2 instance it attaches to?
2. What happens to an object in S3 when it is deleted and versioning is enabled?
3. A lifecycle rule transitions objects to Glacier after 90 days — what does that mean for retrieval time?
4. Snapshots are stored in S3 — why can they not be seen in the S3 console?
5. An application needs to store user-uploaded profile images accessible globally over HTTPS — S3 or EBS?
6. What is the difference between deleting an EBS volume and deleting its snapshot?
7. Why should Block all public access stay ON by default on S3 buckets?

---

## Cleanup & Teardown

> **⚠ EBS volumes and snapshots bill continuously — even when unattached. Delete everything when done.**

**Order matters — unmount before detaching, detach before deleting.**

| Resource | Where in the Console |
|---|---|
| Unmount volume (inside instance) | EC2 Instance Connect → `sudo umount /data` |
| Detach EBS volume | EC2 → Volumes → select → Actions → **Detach volume** |
| Delete EBS volume | EC2 → Volumes → select → Actions → **Delete volume** |
| Delete snapshot | EC2 → Snapshots → select → Actions → **Delete snapshot** |
| Empty S3 bucket | S3 → select bucket → **Empty** |
| Delete S3 bucket | S3 → select bucket → **Delete** |

> Deleting a volume does not delete its snapshots — they must be deleted separately.

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
