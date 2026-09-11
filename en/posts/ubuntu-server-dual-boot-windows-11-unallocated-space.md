---
layout: post
lang: en
slug: ubuntu-server-dual-boot-windows-11-unallocated-space
title: "Dual booting Ubuntu Server with Windows 11 on a mini PC: why the installer cannot see your free space"
date: 2026-09-09 14:00:00 +0700
description: "Windows Disk Management showed the space I had freed for Ubuntu. The Ubuntu Server installer did not. Two things had to be true at once before it would."
tags: [ubuntu, dual-boot, windows, bitlocker, uefi, amd-igpu]
---

## TL;DR

If you shrank a volume in Windows to make room for Ubuntu and the Ubuntu Server installer
cannot see that space, two things have to be true at once: **the space must be a real
formatted drive in Windows (a D: drive), not left as raw unallocated space**, and **that
drive must not be BitLocker-encrypted** — turn off Device Encryption. Either one alone is
not enough. Once both are true the installer will let you select the drive, and you then
format it as ext4. For EFI, reuse the partition Windows already uses; never create a new
one.

## Hardware and software

| Item | Value |
|---|---|
| Machine | MINIX ER937-AI |
| CPU | AMD Ryzen AI 9 HX 370 (12 cores) |
| iGPU | AMD Radeon 890M — `gfx1150` |
| RAM | 32 GB × 2 DDR5-5600 SO-DIMM = 64 GB (the stock configuration ships 32 GB; this is the factory 64 GB variant) |
| Disk | NVMe PCIe 4.0 1 TB (953.9 GB as reported) |
| OS as shipped | Windows 11 Pro |
| OS added | Ubuntu Server 26.04.1 LTS (ISO from the official download page, never upgraded since) |
| Secure Boot | enabled, left alone |

This model also has an NPU (the marketing figure of 80 TOPS covers CPU + NPU together),
but nothing in this blog touches the NPU. Everything runs on the iGPU.

The disk layout this ended in:

```
nvme0n1     953.9G
├─nvme0n1p1   100M vfat     /boot/efi     ← Windows' existing ESP, now shared
├─nvme0n1p2    16M                        ← Microsoft Reserved (MSR)
├─nvme0n1p3   128G ntfs                   ← Windows 11
└─nvme0n1p4 825.7G ext4     /             ← Ubuntu Server
```

There is no swap partition — the 8 GiB of swap that `free` reports is a file the installer
created on its own, not a separate partition.

## Where this started: Windows was using 11-12 GB before I opened anything

This machine was bought specifically to be an AI station. After the first-boot Windows 11
setup finished and the desktop came up, with not a single application open, 11-12 GB of
RAM was already in use.

For a general-purpose machine that number is fine. But the job this machine exists for is
running language models, where RAM is the resource that decides how large a model you can
load. Handing 11-12 GB to the operating system for nothing is not a good trade.

## Why Server and not Desktop

My first thought was to wipe Windows entirely. My professor suggested dual booting instead
so there would still be a way back if Linux went wrong — good advice, and it mattered
later.

Ubuntu because most AI tooling is documented and tested against it. And **Server, not
Desktop**, for the same reason I was leaving Windows in the first place: Server has no GUI,
no desktop environment, and none of the background services a desktop implies, so it uses
far less RAM.

Measured after 20 hours of uptime with no containers running:

| Operating system | RAM in use at idle |
|---|---|
| Windows 11 (first boot to desktop, nothing opened) | 11-12 GB |
| Ubuntu Server 26.04.1 LTS (20 h uptime, no containers) | 1.4 GB |

```
               total        used        free      shared  buff/cache   available
Mem:            60Gi       1.4Gi        43Gi       8.2Mi        16Gi        59Gi
Swap:          8.0Gi       4.0Ki       8.0Gi
```

About 10 GB of difference. On a machine whose purpose is loading models, 10 GB is not a
rounding error.

## Aside: 64 GB installed, 60 GB visible

This comes back to matter in the next post about running vLLM, so it is worth writing down
here.

`dmidecode` confirms two 32 GB sticks:

```
        Size: 32 GB
        Locator: DIMM 0
        Type: DDR5
        Speed: 5600 MT/s
        Size: 32 GB
        Locator: DIMM 0
        Type: DDR5
        Speed: 5600 MT/s
```

What the kernel reports:

```
MemTotal:       63873580 kB
```

63,873,580 kB is **60.92 GiB** out of the 64 GiB installed. 3.08 GiB is gone.

That missing memory is not all the iGPU's, which is what most people assume (me included,
at first). The biggest single consumer is `crashkernel`, configured in `/proc/cmdline`:

```
crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M
```

Do not guess the reservation from that table. Ask the kernel how many bytes it actually
took:

```bash
cat /sys/kernel/kexec_crash_size
```

```
1073741824
```

**Exactly 1 GiB** — not the 2 GiB that the `64G-128G` bracket appears to promise a 64 GB
machine. By the time the kernel picks a bracket it is looking at the memory left after the
firmware carve-outs, which is already under 64 GiB, so it lands in `32G-64G` and takes
1024M. I had assumed the larger figure in the vLLM post before working through the real
numbers here.

Adding it all up:

| Component | Size | Confirmed by |
|---|---|---|
| RAM installed | 64 GiB | `dmidecode` shows 32 GB × 2 |
| `crashkernel` (kdump) | 1 GiB | `/sys/kernel/kexec_crash_size` |
| VRAM the BIOS gives the iGPU | 512 MiB | `rocm-smi --showmeminfo vram` |
| firmware / ACPI / kernel reserved | ~1.6 GiB | what is left after subtracting |
| **available to the OS (`MemTotal`)** | **60.92 GiB** | `/proc/meminfo` |

The part that matters for the next post: this iGPU **has no memory of its own**. It gets
512 MiB of real VRAM and borrows the rest from system RAM through a mechanism called GTT.
Which means that loading a multi-gigabyte model onto this GPU requires enlarging GTT by
hand — a story for the next post.

## What to do in Windows first

In this order. Do not skip any of it.

1. **Turn off Fast Startup**
   Control Panel → Power Options → Choose what the power buttons do →
   Change settings that are currently unavailable → uncheck "Turn on fast startup"
2. **Turn off Device Encryption**
   Settings → Privacy & security → Device encryption → off
   (wait for decryption to finish; it takes a while)
   This machine runs Windows 11 Pro, which also has the full BitLocker panel at
   Control Panel → BitLocker Drive Encryption, but the Settings toggle above was enough,
   so I never opened it.
3. **Shrink the volume** in Disk Management to free space for Ubuntu
   (this machine keeps 128 GB for Windows and gives the rest to Ubuntu)
4. **Turn that free space into a real drive** — right-click the unallocated space →
   New Simple Volume → make it D:. **Do not leave it as bare unallocated space.**
   The reason is in the section below.

> ⚠️ **Turn off Fast Startup before anything else.** With it on, shutting Windows down
> does not really shut it down — it enters a partial hibernation with its partitions still
> locked. If Linux writes to them in that state, the Windows filesystem can be corrupted.
{: .warn}

## Checking that Device Encryption is really off

Flipping the switch in Settings is not the end of it — decryption still has to finish. To
see whether it has, open Command Prompt as administrator and ask the drive what protectors
it still has:

```
manage-bde -protectors -get C:
```

Once decryption has finished, the command returns no protectors at all: no
`Numerical Password` line, and no 48-digit number to write down, because a drive that is
not encrypted has no key to begin with.

## Making the USB installer

Download the ISO and build the USB following the two official pages:

- <https://ubuntu.com/download/server#how-to-install-tab-lts>
- <https://ubuntu.com/server/docs/tutorial/basic-installation/>

I used Rufus and changed nothing, leaving whatever it selected after I picked the ISO.

## The installer cannot see the free space

This is where the whole job stalled the longest.

The symptom: Windows Disk Management clearly shows the space freed by shrinking. Boot the
Ubuntu Server installer, get to the storage screen, and the installer **sees the disk but
not that free space**, so there is nothing to install onto.

I asked one AI tool about it. It said to turn off Device Encryption / BitLocker in Windows
Settings. Did that — **still no.**

I asked a different one, which supplied the missing piece: after shrinking, do not leave
the space unallocated; create it as a real drive in Windows first (D:).

Once it was a D: drive the installer could see it — but that drive was BitLocker-encrypted
and still unusable. Removing the encryption from it was the last piece; after that it could
be selected and the install ran to completion.

Both conditions are required. Neither one alone does it:

| Condition | Result |
|---|---|
| bare unallocated space | installer does not see it |
| a real drive, but still BitLocker-encrypted | installer sees it, cannot use it |
| a real drive with no BitLocker | selectable, installs fine |

> ⚠️ **The Settings shortcut decrypts the whole machine, not just the target drive.** What
> is actually required is that **the partition you hand to Ubuntu is not BitLocker-encrypted**,
> but the Device encryption switch in Settings is not per-drive — turning it off decrypts
> C: as well. If you want C: to stay encrypted, work drive by drive in
> Control Panel → BitLocker Drive Encryption instead of using this shortcut.
{: .warn}

## In the installer

At the storage screen:

1. Select the drive you prepared (the D: drive made from the shrunk space)
2. Edit it, set the format to **ext4**, and set the mount point to `/`
3. For **EFI, pick the partition Windows already uses — do not create a new one**. On this
   machine that is the 100 MB partition.

> ⚠️ **Never create a second EFI partition.** A UEFI disk has one ESP that every operating
> system shares, each dropping its own boot files into it. If you create a new ESP and
> point Ubuntu at it, the firmware may no longer find Windows' boot files, and Windows
> stops booting.
{: .warn}

One more thing worth ticking on the software selection screen: **OpenSSH Server**. Install
it now and you can do the rest of the work remotely instead of sitting in front of the
machine with a monitor and keyboard attached — which, for a machine meant to be a server,
is how you were going to use it anyway.

## BIOS settings afterwards

The first reboot brought up a GRUB menu with both Ubuntu and Windows Boot Manager, and both
booted.

Then two settings in the BIOS:

1. **Boot priority**: put `ubuntu` first, since this machine is primarily a server. Without
   it, a machine that powers itself back on after an outage sits waiting at Windows.
2. **AC Power Loss = Power On**, so the machine starts by itself when power returns. This
   matters for a machine other people reach over the network: if the power drops while
   nobody is there and it does not come back up on its own, nobody can get in until someone
   walks over and presses the button.

### You do not need to disable Secure Boot

Plenty of Linux installation guides tell you to turn Secure Boot off first. This machine
never did, and the install went through normally. Confirmed afterwards:

```bash
mokutil --sb-state
```

```
SecureBoot enabled
```

Ubuntu ships signed boot files, so it boots under Secure Boot directly.

Leaving it alone buys more than a skipped step: every BIOS setting you change is one more
variable to walk back when something does not boot. If it installs with Secure Boot on,
there is no reason to turn it off.

## What I learned

| What can go wrong | What happens | Fix |
|---|---|---|
| leaving the space unallocated | the Ubuntu Server installer cannot see it | create it as a real drive in Windows first (D:) |
| target drive still BitLocker-encrypted | installer sees the drive but cannot use it | turn off Device Encryption and let decryption finish |
| Fast Startup left on | Windows never really shuts down, partitions stay locked, filesystem corruption risk | turn it off before anything else |
| creating a new EFI partition for Ubuntu | Windows may stop booting | reuse the ESP Windows already uses |
| not installing OpenSSH during setup | you need a monitor and keyboard to continue | tick it on the software selection screen |
| skipping boot priority / AC Power Loss | machine does not return after an outage, or returns into the wrong OS | set both in the BIOS while you are there |

The main lesson: "the installer cannot see the free space" had two stacked causes, and
fixing them one at a time made the first fix look like it had failed when in fact it was
necessary but not sufficient. When one fix does not clear a problem, do not throw it away
before trying the next one on top of it.
