# Cloud Lab 3 Submission — AWS Storage Options

Lab instructions: [Cloud Lab 3 — Understanding AWS Storage Options](https://github.com/RathpiseyAlpha/ITC-I5GIC-CC/blob/main/labs/lab3/lab3-instruction.md)

- **Student name:** Huy Chanchhinghoir
- **Student ID:** e20221469
- **Learner Lab user name:** voclabs/user5619305=HUY_CHANCHHINGHOIR
- **Region:** US East (N. Virginia) `us-east-1`
- **Date completed:** [YYYY-MM-DD]

---

## Lab objectives

This lab demonstrates three AWS storage services:

1. **Amazon S3** — object storage and static website hosting.
2. **Amazon EBS** — block storage attached to an EC2 instance and moved to another instance.
3. **Amazon EFS** — shared file storage mounted by multiple EC2 instances.

All screenshots were taken before cleanup. Console screenshots show the Learner Lab user name at the top-right.

## Submission tree

```text
cc-gic-e20221469/
└── cc-lab-e20221469/
    └── lab3/
        ├── README.md
        └── images/
            ├── 01-s3-bucket-list.png
            ├── 02-s3-objects.png
            ├── 03-s3-website.png
            ├── 04-ebs-volumes.png
            ├── 05-ebs-lsblk.png
            ├── 06-ebs-second-instance.png
            ├── 07-efs-file-system.png
            ├── 08-efs-write.png
            └── 09-efs-read.png
```

## Screenshot checklist

| File | Part | Evidence |
|---|---|---|
| `01-s3-bucket-list.png` | 2.1 | S3 bucket list showing the `static-web-bucket...` bucket |
| `02-s3-objects.png` | 2.1 | `index.html` and `error.html` inside the bucket |
| `03-s3-website.png` | 2.1 | Static website endpoint displaying `index.html` |
| `04-ebs-volumes.png` | 2.2 | `Backup Volume` attached and showing **In-use** |
| `05-ebs-lsblk.png` | 2.2 | `lsblk` output showing the attached 8 GiB volume |
| `06-ebs-second-instance.png` | 2.2 | `instance-2` reading `testfile.txt` after the volume was moved |
| `07-efs-file-system.png` | 2.3 | `clab-efs` and its two mount targets |
| `08-efs-write.png` | 2.3 | `instance-1` creating a file in the EFS mount |
| `09-efs-read.png` | 2.3 | `instance-3` reading the shared file |

---

# Part 2.1 — Amazon S3

## Task 1 — Create a bucket and upload files

Bucket name: `[your static-web-bucket name]`

![S3 bucket list](images/01-s3-bucket-list.png)

The bucket was created in `us-east-1` with Block Public Access disabled so that the later bucket policy could allow website access. I uploaded `index.html` and `error.html`.

![S3 objects](images/02-s3-objects.png)

## Task 2 — Enable static website hosting

The bucket was configured with:

| Setting | Value |
|---|---|
| Hosting type | Host a static website |
| Index document | `index.html` |
| Error document | `error.html` |
| Access policy | Public `s3:GetObject` on objects in this bucket |

![S3 static website](images/03-s3-website.png)

Website endpoint: `[paste the S3 website endpoint here]`

The page loaded successfully after the bucket policy was added. Before the policy was added, the endpoint returned `403 Forbidden`.

---

# Part 2.2 — Amazon EBS

## Task 3 — Launch two EC2 instances

| Instance | Availability Zone | AMI | Instance type | Security group |
|---|---|---|---|---|
| `instance-1` | `us-east-1a` | Ubuntu Server 24.04 LTS | `t3.medium` | `ec2instances-sg` |
| `instance-2` | `us-east-1a` | Ubuntu Server 24.04 LTS | `t3.medium` | `ec2instances-sg` |

## Task 4 — Create an EBS volume

| Setting | Value |
|---|---|
| Name | `Backup Volume` |
| Volume type | `gp3` |
| Size | 8 GiB |
| Availability Zone | `us-east-1a` |

## Task 5 — Attach and mount the volume

![EBS volume](images/04-ebs-volumes.png)

I attached `Backup Volume` to `instance-1` as `/dev/sdf`. Linux showed the device as `[xvdf or nvme device shown by lsblk]`.

Commands used:

```bash
lsblk
sudo mkfs -t ext4 /dev/[device]
sudo mkdir /mnt/my-ebs-volume
sudo mount /dev/[device] /mnt/my-ebs-volume
cd /mnt/my-ebs-volume
echo "This is a test file" > testfile.txt
```

![EBS lsblk output](images/05-ebs-lsblk.png)

## Task 6 — Move the volume to the second instance

I unmounted and detached `Backup Volume` from `instance-1`, attached it to `instance-2`, and mounted it without formatting it again.

![EBS file on second instance](images/06-ebs-second-instance.png)

The `testfile.txt` file and its contents were still available. This demonstrates that an additional EBS volume persists independently of the EC2 instance to which it is attached.

---

# Part 2.3 — Amazon EFS

## Task 7 — Create an EFS file system

| Setting | Value |
|---|---|
| Name | `clab-efs` |
| File system type | Regional |
| Performance mode | Enhanced |
| Throughput mode | Elastic |
| Automatic backups | Disabled |
| Mount target zones | `us-east-1a`, `us-east-1b` |
| Security group | `efs-sg` |
| File system ID | `[paste fs-... here]` |

![EFS file system](images/07-efs-file-system.png)

The `efs-sg` security group allows inbound NFS traffic on port 2049. The file system remained encrypted with the default encryption setting.

## Task 8 — Launch the third EC2 instance

`instance-3` was launched in the default subnet in `us-east-1b`, using Ubuntu Server 24.04 LTS, instance type `t3.medium`, and the existing `ec2instances-sg` security group.

## Task 9 — Mount EFS

I installed `amazon-efs-utils`, created `/home/ubuntu/efsmount/`, and mounted `clab-efs` on both `instance-1` and `instance-3`.

Commands used:

```bash
sudo mkdir /home/ubuntu/efsmount/
cd .. && sudo mount -t efs [fs-id] efsmount/
```

## Task 10 — Share a file between instances

On `instance-1`, I created a file in the EFS mount:

```bash
cd ~/efsmount
sudo su
echo "Creating a sample file from instance 1" >> file_from_1.txt
```

![EFS write from instance 1](images/08-efs-write.png)

On `instance-3`, I listed and read the same file:

```bash
cd ~/efsmount
ls
cat file_from_1.txt
```

![EFS read from instance 3](images/09-efs-read.png)

The file created on `instance-1` was visible on `instance-3`, demonstrating that EFS provides shared file storage for multiple instances.

---

## Comparison of S3, EBS and EFS

| Service | Storage type | Access model | Example use |
|---|---|---|---|
| S3 | Object storage | Objects accessed through APIs or URLs | Static websites, backups and data lakes |
| EBS | Block storage | Attached to one EC2 instance at a time | Instance file systems and databases |
| EFS | File storage | Shared NFS file system | Files shared by multiple instances |

## Questions and answers

### 1. What is the difference between S3, EBS and EFS?

S3 stores independent objects in buckets and is accessed through APIs or URLs. EBS provides raw block storage that is attached to an EC2 instance and formatted with a file system. EFS provides a managed, network file system that multiple EC2 instances can mount at the same time.

### 2. Why did the EBS volume need to be in the same Availability Zone as the instances?

An EBS volume is Availability Zone-specific. It can be attached directly only to instances in the same Availability Zone. A snapshot is needed to create a copy for use in another Availability Zone.

### 3. What happened to the file when the EBS volume moved to `instance-2`?

The file remained because `Backup Volume` was an additional EBS volume, not the root volume. After unmounting and detaching it from `instance-1`, I attached and mounted the same volume on `instance-2` without running `mkfs` again.

### 4. Why could both instances see the EFS file?

EFS is a shared network file system. Both instances mounted the same file system through EFS mount targets, so a file written by `instance-1` was immediately available to `instance-3`.

### 5. What security controls were used?

S3 used Block Public Access settings and a bucket policy. EC2 access used `ec2instances-sg`, and EFS mount targets used `efs-sg` to allow NFS traffic. Public access should be limited to only what the application requires.

---

## Clean up

- [ ] `instance-1`, `instance-2` and `instance-3` terminated.
- [ ] `clab-efs` deleted.
- [ ] `Backup Volume` deleted.
- [ ] `efs-sg` and `ec2instances-sg` deleted.
- [ ] S3 bucket emptied and deleted.
