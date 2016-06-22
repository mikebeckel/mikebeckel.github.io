---
layout: single
title: "Migrate From Windows 7 BIOS to Windows 10 UEFI In A Single Task Sequence"
tags: 
 - Windows 10
 - Migration
 - BIOS
 - UEFI
---

While attending the Midwest Management Summit (btw this is a great event that you should consider attending) this year I kept hearing "How do I convert from BIOS to UEFI when migrating to Windows 10?". There are several approaches to address this challenge and each of them have their merits. There's no one right way to solve this; pick the solution that best fits your organization. This post is an introduction to how my team addressed this issue. 

TL:DR: [Target/SCCMOSD-Refresh-Multitool](https://github.com/target/sccmosd-refresh-multitool)

In order to upgrade to our systems to Windows 10 with UEFI/SecureBoot we needed to develop a solution that would migrate our desktops with zero user or technician involvement. In addition it needed to work with devices encrypted with a 3rd party fully disk encryption solution, have no dependency on PXE, and support Dell hardware. Secondarily we had a goal to execute this process in a single SCCM OSD Task Sequence as we felt that this minimized the number of failure points.

The solution we developed is based on a similar solution that I developed when migrating from Windows XP to Windows 7 and is composed of three phases:

* ***Phase 1*** is executed inside the full OS (Windows 7) right after USMT captures the user state to a network share. The goal of this phase is to stage WinPE on the local disk and persist the TSEnv.dat file which is used by the task sequence. 
* ***Phase 2*** is executed from within the WinPE Ramdisk. The goal of this phase is to wipe the disk and reformat with GPT disks, convert the BIOS to UEFI, stage WinPE back on the local disk and with the files necessary to continue the build. 
 * We chose to make this step self-sufficient and not reach out to network shares or web servers for content. While it can be updated to reach out for content we did not feel that the benefit outweighed the risk to an already complex process.  
* ***Phase 3*** is executed from the context of the WinPE Ramdisk that got staged to the local disk in Phase 2.  The goal of this phase is to restart the task sequence where it left off previously.

The repo on GitHub details out each step so if you're interested in more detail continue reading [here](https://github.com/target/sccmosd-refresh-multitool)

We have a few enhancments planned for the near future

* Add support for Windows 7 UEFI to Windows 10 UEFI
* Prevent users from closing out the scripts while executing in phase 2
* Add support for HP hardware
