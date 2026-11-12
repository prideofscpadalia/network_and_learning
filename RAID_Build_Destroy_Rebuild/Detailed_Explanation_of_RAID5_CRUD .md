# Enterprise-Grade Linux Software RAID 5: Deep-Dive Simulation, Fault Tolerance, and Disaster Recovery Lab

[![OS: Fedora Linux](https://img.shields.io/badge/OS-Fedora_Linux-blue.svg)](https://getfedora.org/)
[![Tool: mdadm](https://img.shields.io/badge/Tool-mdadm-orange.svg)](https://en.wikipedia.org/wiki/Mdadm)
[![Storage: RAID 5](https://img.shields.io/badge/Architecture-RAID_5-green.svg)](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_5)

## Overview & Executive Summary

This repository houses a comprehensive, production-grade laboratory setup demonstrating how to construct, test, intentionally degrade, troubleshoot, and structurally recover a **RAID 5** software array on **Fedora Linux** using `mdadm` and virtual loopback block devices. 

By eliminating the need for expensive physical server hardware, this lab emulates enterprise storage virtualization practices, data striping algorithms, real-time XOR parity calculations, and disaster recovery execution in a safe, controlled local environment.

---

## Technical Architecture & Core Concepts

### What is Software RAID via `mdadm`?
Software RAID shifts the responsibility of redundancy and striping from expensive proprietary hardware controller cards to the host operating system kernel. The `mdadm` (Multiple Devices Administrator) utility is the modern standard on Linux distributions (like Fedora) for managing, monitoring, and troubleshooting `md` (multiple device) kernel drivers.

### Why RAID 5?
* **Minimum Requirement:** At least 3 drives.
* **Storage Efficiency:** $\text{Usable Capacity} = (N - 1) \times \text{Drive Size}$, where $N$ is the number of total drives. With three 512 MB virtual disks, our net usable space is ~1 GB.
* **Fault Tolerance:** Distributed parity allows the array to survive the sudden, simultaneous failure of exactly **one** member disk without suffering data loss.

---

## Comprehensive Table of Contents
1. [Prerequisites & Package Installation](#1-prerequisites--package-installation)
2. [Environment Bootstrapping & Virtual Disk Generation (`dd`)](#2-environment-bootstrapping--virtual-disk-generation-dd)
3. [Bridging Files to Block Devices via Loopback (`losetup`)](#3-bridging-files-to-block-devices-via-loopback-losetup)
4. [Array Initialization & Baseline Verification](#4-array-initialization--baseline-verification)
5. [Filesystem Formatting (`ext4`) & Mounting Strategy](#5-filesystem-formatting-ext4--mounting-strategy)
6. [Data Ingestion & Cryptographic Integrity Checks (SHA-256)](#6-data-ingestion--cryptographic-integrity-checks-sha-256)
7. [Simulating Single-Drive Failure & Degraded Mode Processing](#7-simulating-single-drive-failure--degraded-mode-processing)
8. [Real-World Troubleshooting: Hot-Swap Sizing Mismatches & Resolution](#8-real-world-troubleshooting-hot-swap-sizing-mismatches--resolution)
9. [Catastrophic Failure Analysis & Kernel Safety Guardrails](#9-catastrophic-failure-analysis--kernel-safety-guardrails)
10. [Lab Teardown & Resource Purging](#10-lab-teardown--resource-purging)

---

## 1. Prerequisites & Package Installation

Ensure your Fedora system is up to date and install the package management utility for software arrays:

```bash
sudo dnf update -y
sudo dnf install mdadm -y
```

- `-y` flag automatically confirms prompts during installation

Create and move into a dedicated laboratory workspace:
```bash
mkdir -p ~/raid-lab
cd ~/raid-lab
```
- `-p` flag stands for **parents** 
---

## 2. Environment Bootstrapping & Virtual Disk Generation (`dd`)

To simulate raw physical disks without buying hardware, we utilize the data duplicator utility (`dd`) to generate flat binary files filled with null bytes. Each disk is configured to **512 MB**.

```bash
sudo dd if=/dev/zero of=virtual_disk1 bs=1M count=512
sudo dd if=/dev/zero of=virtual_disk2 bs=1M count=512
sudo dd if=/dev/zero of=virtual_disk3 bs=1M count=512
```

**Expected Output:** 
```bash
divyanshu@fedora:~/raid-lab$ sudo dd if=/dev/zero of=virtual_disk1 bs=1M count=512
512+0 records in
512+0 records out
536870912 bytes (537 MB, 512 MiB) copied, 0.350144 s, 1.5 GB/s

divyanshu@fedora:~/raid-lab$ sudo dd if=/dev/zero of=virtual_disk2 bs=1M count=512
512+0 records in
512+0 records out
536870912 bytes (537 MB, 512 MiB) copied, 0.363825 s, 1.5 GB/s

divyanshu@fedora:~/raid-lab$ sudo dd if=/dev/zero of=virtual_disk3 bs=1M count=512
512+0 records in
512+0 records out
536870912 bytes (537 MB, 512 MiB) copied, 0.413023 s, 1.3 GB/s

# This is the confirmation that all three are created
divyanshu@fedora:~/raid-lab$ ls
virtual_disk1  virtual_disk2  virtual_disk3

```
### Deep Dive: Understanding `dd` Architecture
* **`if=/dev/zero`**: The input file(`if`) (`/dev/zero`) is a special kernel character device that yields an endless stream of zero-valued null bytes.
* **`of=virtual_diskX`**: The target output flat file(`of`) path.
* **`bs=1M`**: Sets the input/output block chunk size to 1 Megabyte, maximizing throughput.
* **`count=512`**: Instructs `dd` to copy exactly 512 blocks, resulting in an exact size of 536,870,912 bytes (512 MiB).

**Note** Unlike traditional copy commands (`cp`), `dd` operates at a raw block-stream level, bypassing filesystem layers entirely.

---

## 3. Bridging Files to Block Devices via Loopback (`losetup`)

Linux storage subsystems cannot natively ingest a regular file as a raw block component for a RAID array. We bridge this gap using **loopback devices**.

```bash
sudo losetup --show -f virtual_disk1
sudo losetup --show -f virtual_disk2
sudo losetup --show -f virtual_disk3
```
**Expected Output:** 
```bash
divyanshu@fedora:~/raid-lab$ sudo losetup --show -f virtual_disk1
/dev/loop16
divyanshu@fedora:~/raid-lab$ sudo losetup --show -f virtual_disk2
/dev/loop17
divyanshu@fedora:~/raid-lab$ sudo losetup --show -f virtual_disk3
/dev/loop18
```
* **`-f` (Find)**: Automatically scans loop interfaces (`/dev/loop0`, `/dev/loop1`, etc.) and assigns the first available unused loop device. (On Fedora systems running Flatpaks or Snaps, lower loop indices are often pre-allocated, placing our assignments at higher indices like `/dev/loop16`, `/dev/loop17`, and `/dev/loop18`).
* **`--show`**: Instantly prints the assigned device node path to stdout.
* `losetup` Allow Linus to treat regular files as Block Devices.

---

## 4. Array Initialization & Baseline Verification

Combine the loopback devices into a production-pattern RAID 5 array designated as `/dev/md0`:

```bash
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/loop16 /dev/loop17 /dev/loop18
```
**Expected Output:** 
```bash
divyanshu@fedora:~/raid-lab$ sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/loop16 /dev/loop17 /dev/loop18
[sudo] password for divyanshu: 
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 522240K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
### Deep Dive : `flags` while Initializing
- `--create` Tells `mdadm` to create a brand new RAID array from scratch.
- `--verbose` Prints detailed progress of RAID creation from success, warning, error and detailed layout.
- `/dev/md0` Name of the RAID5 
- `--level=5` Sets RAID level to 5 (RAID5)
- `--raid-devices=3` Specify exact number of drive in RAID5

### Verifying Array Baseline Health
Inspect real-time kernel status: At mdstat as kernal keeps here the real time status of RAID.
```bash
cat /proc/mdstat
```
**Expected Output:**
```text
Personalities : [raid4] [raid5] [raid6] 
md0 : active raid5 loop18[3] loop17[1] loop16[0]
      1044480 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
```
* **`[UUU]`**: All three member devices are **Up**, synchronized, and active.

For detailed hardware/metadata metrics:
```bash
sudo mdadm --detail /dev/md0
```

---

## 5. Filesystem Formatting (`ext4`) & Mounting Strategy

Format the aggregate virtual block device with the `ext4` filesystem, which includes embedded geometry layouts optimized for multi-disk throughput and journal logging:

```bash
sudo mkfs.ext4 /dev/md0

# Expected Output:

divyanshu@fedora:~/raid-lab$ sudo mkfs.ext4 /dev/md0
[sudo] password for divyanshu: 
mke2fs 1.47.2 (1-Jan-2025)
Creating filesystem with 261120 4k blocks and 65280 inodes
Filesystem UUID: b3f8b6c7-5e1f-4ec7-886d-0c5e07f27048
Superblock backups stored on blocks: 
    32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```

+ `mkfs` stands for *make filesystem* 
+ `ext4` is the filesystem

Create a designated mounting directory and attach the array:
```bash
sudo mkdir -p /mnt/raid_test
sudo mount /dev/md0 /mnt/raid_test/
cd /mnt/raid_test
```
- We used `mount` flag to mount the /dev/md0(RAID) to /mnt/raid_test

---

## 6. Data Ingestion & Cryptographic Integrity Checks (SHA-256)

Write test payloads and calculate cryptographic checksums to establish a baseline data integrity standard before inducing failure.

```bash
# 1. Create a structured text note
echo "Raid 5 Parity Test - Data Integrity Check" | sudo tee test_note.txt
## a. The tee command take input from lest side then put it in right side then print it.

# 2. Safely append secondary data using the append flag (-a)
echo "third line test" | sudo tee -a test_note.txt
## b. The a flag means we append in the test_note.txt

# 3. Inject a 20 MB random binary payload using /dev/urandom
sudo dd if=/dev/urandom of=payload.bin bs=1M count=20 status=progress
# Output:

20+0 records in
20+0 records out
20971520 bytes (21 MB, 20 MiB) copied, 0.0854135 s, 246 MB/s

## c. The urandom is virtual character device that generates an endless stream of cryptographically secure pseudorandom bytes on the fly whenever a program reads from it.
## d. The status=progess tells dd command to display live, real time transfer statistics in your terminal.

# 4. Generate SHA-256 cryptographic checksums
sha256sum test_note.txt payload.bin | sudo tee hashes.sha256
## e. SHA256SUM is a digital fingerprint. With tee command we put the input from test_note.txt to 
## f. payload.bin is created as a realistic, large binary test file (usually filled with random data from `/dev/urandom`) so you can properly test how your RAID array handles real-world data across multiple disks. test_note.txt Alone Isn't Enough. It helped increase the size of file so RAID can make it into chunks. Our File is too small.

```

### Why SHA-256 Matters in Storage Engineering
A cryptographic hash acts as a unique 64-character digital fingerprint. Because of the *avalanche effect*, modifying a single bit in the source file entirely shifts the resulting hash string. We use this to prove that data read from a degraded array remains mathematically intact.

---

## 7. Simulating Single-Drive Failure & Degraded Mode Processing

RAID 5 can tolerate the complete loss of one device. Intentionally flag `/dev/loop18` as faulty:

```bash
sudo mdadm --manage /dev/md0 --fail /dev/loop18
```

Verify degraded status:
```bash
cat /proc/mdstat
```
**Expected Output:**
```text
md0 : active raid5 loop18[3](F) loop17[1] loop16[0]
      1044480 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
```
* **`[UU_]`**: Notice the underscore replacing the third drive slot, alongside the `(F)` fault flag on `/dev/loop18`.

### Testing Data Integrity Under Degradation
Run the verification script against the active array:
```bash
sha256sum -c hashes.sha256
```
* **Result:** Both files return `OK`. 

#### The Engineering Mechanism: How Does This Work?
Even with an offline drive, your files remain fully accessible because the Linux kernel dynamically computes missing data blocks in real time using **XOR (Exclusive OR) parity logic** derived from surviving data and parity stripes.

---

## 8. Real-World Troubleshooting: Hot-Swap Sizing Mismatches & Resolution

Remove the failed device to emulate unbolting a dead physical hard drive:
```bash
sudo mdadm --manage /dev/md0 --remove /dev/loop18
```

### The Sizing Mismatch Challenge & Lab Solution
1. **The Replacement Error:** If you create a replacement virtual disk with a minor sizing typo and attempt to hot-add it, `mdadm` will throw an error:
   ```text
   mdadm: /dev/loopX not large enough to join array
   ```
2. **Diagnosis via `blockdev`:** 
   Compare the exact byte block sizes of existing active devices versus your new replacement loop device:
   ```bash
   sudo blockdev --getsize64 /dev/loop18
   536870912
   sudo blockdev --getsize64 /dev/loop19
   512
   ```
3. **Remediation & Resizing:**
   * Detach the misconfigured loop device:
     ```bash
     sudo losetup -d /dev/loop19
     ```
   * Fix permissions and pad the raw file to match exact cluster specifications using `truncate`:
     ```bash
     chmod +w ~/raid-lab/virtual_disk_replacement
     truncate -s 536870912 ~/raid-lab/virtual_disk_replacement
     sudo losetup --show -f ~/raid-lab/virtual_disk_replacement
     ```
   * Re-add the corrected device and monitor the live kernel recovery rebuild progress:
     ```bash
     sudo mdadm --manage /dev/md0 --add /dev/loop19
     cat /proc/mdstat
     ```

---

## 9. Catastrophic Failure Analysis & Kernel Safety Guardrails

RAID 5 is mathematically incapable of surviving two simultaneous drive failures. 

### Why `mdadm` Prevents Failing Two Drives Live
When attempting to mark a second active drive (`/dev/loop17`) as faulty while already degraded, `mdadm` blocks the command:
```text
mdadm: Cannot remove /dev/loop17 from /dev/md0, array will be failed.
```
* **The Reason:** `mdadm` and the Linux kernel maintain safety guardrails to prevent cascading file corruption. Because single parity cannot solve two missing block equations, forcing a second drive failure live on a mounted filesystem would trigger catastrophic kernel `I/O errors` and unrecoverable filesystem corruption. 
* **Proper Procedure:** Catastrophic failure requires stopping the array entirely (`sudo mdadm --stop /dev/md0`), confirming that dual-parity architectures (like RAID 6) or replication (like RAID 10) are mandatory for environments exposed to multi-drive failure risks.

---

## 10. Lab Teardown & Resource Purging

Clean up all temporary loop devices, mounts, and flat files to return your host system to a pristine state:

```bash
# 1. Unmount the filesystem safely
sudo umount /mnt/raid_test

# 2. Stop the active mdadm array
sudo mdadm --stop /dev/md0

# 3. Detach loopback associations
sudo losetup -d /dev/loop16 /dev/loop17 /dev/loop19

# 4. Purge mount directories and virtual disk files
sudo rm -rf /mnt/raid_test
rm -f ~/raid-lab/virtual_disk*
```