---
date: '2024-08-20'
author: 'Florian Bernd'
title: 'Recovering an ESXi host after accidental boot disk passthrough'
subtitle: 'Recovering an  ESXi host after accidental boot disk passthrough'
description: 'Recovering an  ESXi host after accidental boot disk passthrough'
featuredImagePreview: 'thumbnail.jpg'
summary: |-
  In this post, I blog about how I accidentally enabled passthrough for an ESXi boot disk and how to restore the host 
  without completely reinstalling the operating system.

tags:
  - VMware
  - vSphere
  - ESXi

categories:
  - Homelab

draft: false
lastmod: '2024-08-20'
---

Yesterday evening I finally decided to carry out a long-planned upgrade of my 2-node vSphere cluster.

## 1 Introduction

So far I have been using a single `Nvidia GTX 4060 Ti` consumer grade GPU in one of the hosts for transcoding images and videos. The card was passed through to a single VM via direct PCIe passthrough.

I now have several services that would benefit from GPU hardware support and for this reason I have been looking around for vGPU capable cards.

Finally, I was able to acquire two refurbished `Nvidia Tesla T4` cards, one of which will now replace the GTX 4060 Ti. The second card is to be used in the other host, which previously had no GPU installed at all.

## 2 Installing the GPU

To install the GPU in the first server, I put it into maintenance mode, migrated all VMs to the second host and finally shut down the system.

The installation itself took a good while, as the server is mounted in a rack in the basement that is not very easy to access. In the end, however, everything went smoothly - at least that's what I thought at the time.

## 3 The Problem

Full of enthusiasm, I rebooted the host and started testing the GPU.

I quickly realized that although the GPU worked correctly at first glance, for some reason all the settings I made on the system were lost after a restart.

After poking around in vCenter and the shell for a while, I finally found out that the boot NVMe SSD was no longer recognized and ESXi had symlinked the so-called `bootbank` to a temporary and volatile directory.

```shell
[root@esxi01:~] ls -l /
lrwxrwxrwx 1 root root  49 Aug 20 18:24 bootbank -> /tmp/randomid
```

\
What surprised me is the fact that the host still starts properly. I therefore ruled out a hardware-related cause for the time being and instead focused on the configuration.

It suddenly hit me like a blow: I had made a beginner's mistake and had not deactivated the PCIe passthrough of the GTX 4060 Ti before swapping the GPUs.

By installing the new card and changing the physical PCIe slots, the PCIe mapping was changed. The boot NVMe SSD is now assigned the former ID of the GTX 4060 Ti and is then configured by the system for direct PCIe passthrough after ESXi starts. This means that ESXi itself can no longer access the disk.

## 4 The Solution

Now, I was faced with the question of how I could restore the host, if possible without having to completely reinstall ESXi and, if possible, without having to tinker with the hardware again.

So the goal was to *somehow* write the current volatile state back to the correct `bootbank` on the boot SSD.

### 4.1 Mount the Boot SSD

First I had to make sure that the system mounts the boot SSD properly again. To do this I simply deactivated the PCIe passthrough in the vCenter UI.

After that, access to the required VMFS volumes was possible again.

### 4.2 Determine the original Bootbank Location

ESXi always uses two different `BOOTBANK` volumes, one of which contains the actual `bootbank` and the other the `altbootbank`, which is used as a backup.

```shell
[root@esxi01:~] ls -l /vmfs/volumes
drwxr-xr-x 1 root root  8 Jan  1  1970 1f9a6075-1a4651a7-9891-edbc1fcdd109
drwxr-xr-x 1 root root  8 Jan  1  1970 405c1577-34424d96-82ac-2556d560fdce
lrwxr-xr-x 1 root root 35 Aug 21 07:55 BOOTBANK1 -> 405c1577-34424d96-82ac-2556d560fdce
lrwxr-xr-x 1 root root 35 Aug 21 07:55 BOOTBANK2 -> 1f9a6075-1a4651a7-9891-edbc1fcdd109
```

\
To find out which volume actually contains the original `bootbank`, I looked at the access times of `state.tgz`:

```shell
[root@esxi01:~] ls -l /vmfs/volumes/1f9a6075-1a4651a7-9891-edbc1fcdd109 | grep state.tgz
[root@esxi01:~] ls -l /vmfs/volumes/405c1577-34424d96-82ac-2556d560fdce | grep state.tgz
```

\
In my case, the archive on volume `1f9a6075-1a4651a7-9891-edbc1fcdd109` had the much more recent access times.

### 4.3 Restore the original Bootbank Symlink

In order for the state to be persisted again, the `bootbank` symlink must first be redirected to the correct target:

```shell
[root@esxi01:~] ln -sfn /vmfs/volumes/1f9a6075-1a4651a7-9891-edbc1fcdd109 /bootbank

```

### 4.4 Persist the Volatile State

To ensure that the current volatile state (in which PCIe passthrough is already deactivated) is correctly written to the boot SSD, we use a command that should normally be executed before a manual state backup:

```shell
[root@esxi01:~] vim-cmd hostsvc/firmware/sync_config
```

\
Now all that’s left to do is restart the host.

## 5 Conclusion

Direct PCIe passthrough can cause unexpected subtle problems. From now on, before making any hardware changes, I will double-check that PCIe passthrough is disabled for all devices.
