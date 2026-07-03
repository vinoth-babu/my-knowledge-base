# Creating a ZFS RAID0 Pool on Rocky Linux

## Overview

From the `lsblk` output:

* `nvme2n1` and `nvme0n1` are already being used for the operating system in Software RAID1 (`md124`, `md125`, `md126`, `md127`).
* The only unused 7 TB NVMe drives are:

  * `/dev/nvme1n1`
  * `/dev/nvme3n1`

This guide creates a **ZFS RAID0 (striped) pool** using these two drives.

> **Warning**
>
> RAID0 provides **no redundancy**. If either drive fails, **all data in the pool will be lost**. Use RAID0 only when maximum performance is required and data is backed up elsewhere.

---

# Step 1 – Verify the Drives Are Unused

Check the filesystem information:

```bash
lsblk -f
```

Verify that no filesystem signatures exist:

```bash
blkid /dev/nvme1n1
blkid /dev/nvme3n1
```

If old filesystem or RAID signatures are present, remove them.

Remove filesystem signatures:

```bash
wipefs -a /dev/nvme1n1
wipefs -a /dev/nvme3n1
```

Erase the partition tables:

```bash
sgdisk --zap-all /dev/nvme1n1
sgdisk --zap-all /dev/nvme3n1
```

---

# Step 2 – Create the ZFS RAID0 Pool

Create a striped ZFS pool named **tank**:

```bash
zpool create -f \
    -o ashift=12 \
    tank \
    /dev/nvme1n1 \
    /dev/nvme3n1
```

---

# Step 3 – Verify the Pool

Check the pool status:

```bash
zpool status
```

Check pool capacity:

```bash
zpool list
```

Check mounted filesystems:

```bash
zfs list
```

---

# Expected Configuration

| Item            | Value                          |
| --------------- | ------------------------------ |
| Pool Name       | `tank`                         |
| RAID Level      | RAID0 (Stripe)                 |
| Devices         | `/dev/nvme1n1`, `/dev/nvme3n1` |
| Usable Capacity | ~14 TB                         |
| Performance     | Maximum                        |
| Redundancy      | None                           |

---

# Mount Point

By default, the pool is mounted at:

```text
/tank
```

Verify:

```bash
df -h
```

---

# Basic Performance Test

Write test:

```bash
fio --name=write_test \
    --directory=/tank \
    --size=50G \
    --rw=write \
    --bs=1M \
    --iodepth=32 \
    --ioengine=libaio \
    --direct=1 \
    --group_reporting
```

Read test:

```bash
fio --name=read_test \
    --directory=/tank \
    --size=50G \
    --rw=read \
    --bs=1M \
    --iodepth=32 \
    --ioengine=libaio \
    --direct=1 \
    --group_reporting
```

---

# Useful ZFS Commands

Display pool status:

```bash
zpool status
```

List pools:

```bash
zpool list
```

List datasets:

```bash
zfs list
```

Show pool properties:

```bash
zpool get all tank
```

Destroy the pool (deletes all data):

```bash
zpool destroy tank
```

---

# Notes

* `ashift=12` optimizes the pool for modern 4K-sector NVMe SSDs.
* RAID0 offers the highest throughput by striping data across both drives.
* Ensure important data is backed up before using RAID0 in production.
