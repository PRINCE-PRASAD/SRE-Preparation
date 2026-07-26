#  Linux Boot Process Explained (Beginner Friendly)

Understanding the Linux boot process is one of the most common DevOps, Linux Administrator, and SRE interview topics.

The complete boot sequence is:

```text
BIOS/UEFI → GRUB → Kernel + initramfs → Mount Root Filesystem → systemd (PID 1) → Targets → Services → Login
```

---

#  Boot Process Overview

```text
Power ON
   │
   ▼
BIOS / UEFI
   │
   ▼
GRUB Bootloader
   │
   ▼
Linux Kernel + initramfs
   │
   ▼
Mount Real Root Filesystem
   │
   ▼
systemd (PID 1)
   │
   ▼
Start Targets & Services
   │
   ▼
Login Prompt / Desktop
```

---

#  Step 1: BIOS / UEFI

When you press the **Power Button**, the motherboard firmware starts first.

Depending on the hardware, it is either:

- BIOS (Legacy)
- UEFI (Modern)

## Responsibilities

- Check CPU
- Check RAM
- Check Keyboard
- Detect SSD/HDD
- Find a bootable device

This hardware check is called:

> **POST (Power-On Self-Test)**

Example:

```text
Power On

Checking CPU...
✔ CPU OK

Checking RAM...
✔ RAM OK

Checking SSD...
✔ SSD Found

Boot Device Found
```

Once everything is verified, BIOS/UEFI hands control to the bootloader.

---

#  Step 2: GRUB (Bootloader)

**GRUB** stands for:

> **GRand Unified Bootloader**

Its primary job is to load the operating system.

Typical GRUB menu:

```text
Ubuntu
Ubuntu (Recovery Mode)
Windows
```

GRUB loads two important components into memory:

- Linux Kernel
- initramfs

Then GRUB's job is finished.

---

#  Step 3: Linux Kernel

The **Kernel** is the heart of Linux.

Once loaded, it starts initializing the operating system.

### Responsibilities

- Initialize CPU
- Initialize Memory
- Load Device Drivers
- Detect USB Devices
- Detect Network Cards
- Initialize Storage Controllers

Example:

```text
Loading CPU...

Loading Memory...

Loading USB Drivers...

Loading Network Drivers...

Loading Storage Drivers...
```

However, the kernel still cannot access the actual Linux filesystem.

For that, it needs **initramfs**.

---

#  Step 4: initramfs

**initramfs** stands for:

> **Initial RAM Filesystem**

Think of it as a **temporary mini Linux** that runs entirely in RAM.

It contains:

- Storage Drivers
- Filesystem Drivers
- RAID Tools
- LVM Tools
- Required Kernel Modules

Its purpose is to locate and mount the real Linux installation.

Example:

```text
Searching for Root Filesystem...

Found:

/dev/nvme0n1p2

Mounting Root Filesystem...
```

---

#  Step 5: Mount the Real Root Filesystem

Initially, Linux is running from the temporary filesystem in RAM.

Now it switches to the real filesystem stored on disk.

Temporary filesystem:

```text
RAM
```

↓

Real filesystem:

```text
/

├── bin
├── boot
├── dev
├── etc
├── home
├── lib
├── opt
├── root
├── usr
├── var
```

This process is known as:

> **Switch Root**

After this point, Linux is running from the actual disk.

---

#  Step 6: systemd (PID 1)

Once the root filesystem is available, the kernel starts the very first userspace process:

```text
systemd
```

This process always has:

```text
PID = 1
```

Verify it:

```bash
ps -p 1
```

Output:

```text
PID TTY      TIME CMD
1   ?        00:00 systemd
```

---

## What is PID 1?

PID 1 is the first process started by the Linux kernel.

On modern Linux distributions, it is **systemd**.

It is responsible for starting every other process.

---

#  What Does systemd Do?

Think of **systemd** as the manager of the operating system.

It starts essential services like:

- SSH
- Docker
- NGINX
- Cron
- Networking
- Logging
- Databases
- Kubernetes kubelet

Example:

```text
systemd
│
├── sshd
├── docker
├── nginx
├── cron
├── redis
├── kubelet
└── network
```

---

#  Step 7: Targets

systemd organizes startup using **Targets**.

A target represents the desired state of the system.

Common targets:

| Target | Purpose |
|---------|----------|
| `multi-user.target` | Server Mode (CLI) |
| `graphical.target` | Desktop GUI |
| `rescue.target` | Recovery Mode |
| `emergency.target` | Minimal Emergency Shell |

Examples:

Server:

```text
multi-user.target
```

Desktop:

```text
graphical.target
```

Check the default target:

```bash
systemctl get-default
```

---

#  Step 8: Services Start

Finally, systemd starts all services required for the selected target.

Examples:

```bash
systemctl list-units --type=service
```

Typical output:

```text
sshd.service
docker.service
nginx.service
cron.service
NetworkManager.service
```

At this point, Linux is fully operational.

---

#  Complete Boot Flow

```text
Power Button
     │
     ▼
BIOS / UEFI
(Hardware Check / POST)
     │
     ▼
GRUB Bootloader
(Loads Kernel + initramfs)
     │
     ▼
Linux Kernel
(Initializes Hardware & Drivers)
     │
     ▼
initramfs
(Finds Real Root Filesystem)
     │
     ▼
Mount Root Filesystem
(Switch Root)
     │
     ▼
systemd (PID 1)
     │
     ▼
Load Target
(Server / Desktop)
     │
     ▼
Start Services
(SSH, Docker, NGINX, Database...)
     │
     ▼
Login Prompt / Desktop Ready
```

---

# 🏢 Easy Real-Life Analogy

Imagine opening a company office every morning.

| Linux Component | Real-Life Analogy |
|-----------------|-------------------|
| BIOS/UEFI | Security guard opens the building |
| GRUB | Receptionist asks which office should open |
| Kernel | CEO enters and takes control |
| initramfs | Temporary toolbox to prepare the office |
| Root Filesystem | Actual office becomes available |
| systemd | HR Manager hires all employees |
| Services | Employees (SSH, Docker, NGINX, Database) start working |

---

#  Useful Commands

### Check Boot Time

```bash
systemd-analyze
```

---

### View Boot Logs

```bash
journalctl -b
```

---

### Show PID 1

```bash
ps -p 1
```

---

### Check Running Services

```bash
systemctl list-units --type=service
```

---

### Check Default Target

```bash
systemctl get-default
```

---

### Check Failed Services

```bash
systemctl --failed
```

---

#  Interview Questions

### 1. What is the first software that runs after powering on the machine?

**Answer:** BIOS or UEFI firmware.

---

### 2. What is the purpose of GRUB?

**Answer:** GRUB loads the Linux Kernel and initramfs and allows selecting the operating system.

---

### 3. What is initramfs?

**Answer:** A temporary filesystem loaded into RAM that helps the kernel locate and mount the real root filesystem.

---

### 4. What is PID 1?

**Answer:** `systemd` (on most modern Linux distributions).

---

### 5. What does systemd do?

**Answer:** Starts the operating system by launching targets, services, daemons, networking, logging, and other essential processes.

---

### 6. Difference between BIOS and UEFI?

| BIOS | UEFI |
|------|------|
| Legacy firmware | Modern firmware |
| Supports MBR | Supports GPT |
| Limited to 2 TB disks | Supports very large disks |
| Text-based | GUI support available |
| Slower boot | Faster boot |

---

#  Key Takeaways

Remember this sequence:

```text
Power On
    ↓
BIOS / UEFI
    ↓
GRUB
    ↓
Kernel
    ↓
initramfs
    ↓
Mount Root Filesystem
    ↓
systemd (PID 1)
    ↓
Targets
    ↓
Services
    ↓
Login Prompt
```

This is the standard Linux boot process and a fundamental concept for Linux, DevOps, Cloud, and Site Reliability Engineering (SRE) interviews.
