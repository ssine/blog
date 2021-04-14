---
title: "Set Up and Application of HPE MicroServer"
date: 2021-04-14T01:05:20+08:00
---

__Note: This article is mainly translated by Google translator, please turn to the Chinese version for more accurate expression if possible.__

I bought HPE MicroServer Gen 10 Plus a few months ago, and it has been running stably since it was configured. Let’s talk about the configuration and application of the home server. Buying a host and turning it on at home throughout the year will be specially used as a server. The price will be much cheaper than a cloud server with the same configuration. This is at the cost of the usual maintenance cost and lower stability of the home environment, but if there are living conditions and technology , This is still an affordable and interesting option. A typical scenario is self-built cloud disks, which require high hard disks. However, cloud hosts with large hard disk capacity, large bandwidth, and sufficient traffic are usually very expensive. At this time, it is very convenient to use a home server. You can add hard disks at any time. The network will not have bandwidth and traffic bottlenecks.

# Installed

HPE official only sells the MicroServer body and iLO expansion card, and there are some things (mainly hard drives) that need to be purchased and installed by yourself. The following are links to Taobao for some items that will be used later.

-   [TOOLFREE 2.5 to 3.5 inch hard drive adapter box](https://m.tb.cn/h.4KYHEL0?sm=ad1309)
-   [Western Digital Blue Disk 1T 2.5 inch SATA3 SSD](https://m.tb.cn/h.4ooR65C?sm=fd8d34)
-   [Greenlink PCIE to NVME expansion card](https://m.tb.cn/h.4pgbuV8?sm=3b1128)
-   [Western Digital Blue Disk SN550 1TB](https://m.tb.cn/h.4LkV8IU?sm=644fe9)
-   [Display box transparent plastic](https://m.tb.cn/h.4oo7xSg?sm=f735fd)

## Hardware Configuration

The configuration of MicroServer itself:

-   Processor Intel Xeon E-2224 Quad-Core 3.4GHz 8MB CPU, Up To 4.6GHz Turbo
-   Memory 16GB (2 x 8GB) DDR4 PC4-21300 2666MHz Unbuffered Memory

I won’t go into details. If the memory is ECC, the memory will be very stable. There are four network ports.

I am personally sensitive to sound. The sound of a computer's mechanical hard drive at home is enough to make me feel annoyed. Therefore, for a server that needs to operate 24 hours a day, SSD is still the priority. The machine itself has four 3.5-inch SATA disk bays, but there is no NVME SSD slot on the motherboard, so if you want a high-speed SSD, you can only transfer from PCIE. I plan to install a 1T SSD as the main hard disk of the virtual machine, and the other four disk slots are reserved for the NAS system, so I started with a PCIE to NVME expansion card and an SSD hard disk. The SATA disk bay also temporarily started with a 2.5-inch SATA SSD and a 2.5 to 3.5 transfer box, so that the SSD can be placed in the SATA disk bay smoothly.

For details of component installation, please refer to the MicroServer user manual:[Chinese](https://psnow.ext.hpe.com/doc/a00073430zh_cn)  [English](https://psnow.ext.hpe.com/doc/a00073430en_us) 。

## System selection installation

A bare server like this usually has two options, install a virtualization management system and install virtual machines in it, or install the system directly on the host. The advantage of the former is that it is convenient to manage many virtual machines and can perform operations such as backup and migration. The disadvantage is that virtualization will have some performance losses. After checking the data, it is found that computing performance is almost unaffected, mainly due to the IO performance brought by virtual hard disk The decline is basically within 10%. The advantage of the latter is that the operation is more familiar, but the disadvantage is that the stability is much worse than that of a dedicated server management system.

For personal use, server bare metal management systems mainly include VMWare's ESXi and open source Proxmox VE. The former is reliable and stable, and the latter is more like Linux + special virtualization support. Since HPE MicroServer officially has a customized ESXi image to ensure driver compatibility, the ESXi system is used for convenience. ESXi can be understood as an operating system that only contains VMWare Workstation virtual machine management software, and of course there are some operating system functions, such as file management.

As for the system installation, you still need HDMI or DP to connect the monitor, burn the image to the U disk, set the BIOS to boot from the USB, and then install the machine.

-   Mirror download link: https://my.vmware.com/cn/group/vmware/downloads/details?downloadGroup=OEM-ESXI70-HPE&productId=974
-   Official free long-term valid Lisense: https://my.vmware.com/en/group/vmware/evalcenter?p=free-esxi7

## joy

iLO is an expansion card for remote control. There are two PCIE slots in ESXi. One is for SSD and the other is for iLO. The iLO card comes with a network port (plus the original four to make five) , Is to prevent the host's network port from being filled up and making it impossible to control the machine. If the bandwidth usage is not high, you can also plug the network cable into the host's network port. iLO can share a network port with other hosts with different IP.

iLO provides a complete solution for remote control of the host. The host can be controlled on and off through the Web service, and the display output of the host can also be seen through the console. The initial password of iLO is located on a label affixed to the server. For details, please refer to the iLO user manual:[Chinese](https://psnow.ext.hpe.com/doc/a00048134zh_cn)  [English](https://support.hpe.com/hpesc/public/docDisplay?docId=a00048134en_us) 。

# application

After the host is set up, you can install the virtual machine, upload the desired operating system image to ESXi, and then create the virtual machine. Storage space is allocated on demand, and it will be troublesome to back up a virtual machine that is too large.

## Virtual machine configuration

ESXi supports many types of virtual machines such as Linux and Windows. The internal configuration of the virtual machine is irrelevant to this article. But one thing to mention here is how to dynamically expand the disk space of a virtual machine. The disks of the virtual machine are in vmdk format. After ESXi expands the vmdk size, enter the virtual machine (here is Centos8, use LVM to manage the disk), and run`fdisk` You can see that the display size is inconsistent, enter`w q` Can be repaired.

Then create a new primary partition (GPT supports 128 instead of 4, just create it randomly), and choose Linux LVM as the file system type.

Then there is the LVM thing (refer to[Here](https://www.cnblogs.com/gaojun/archive/2012/08/22/2650229.html) ), Run`pvcreate /dev/sdax` Add a new physical volume, and then`vgdisplay` View all volume groups, put the newly added physical volume into the volume group that needs to be added`vgextend cl /dev/sdax` , And then you can expand the logical volume in the volume group:`lvdisplay` ， `lvextend -l +xxx cl-home` . Finally, expand the file system to the volume size:`xfs_growfs /dev/mapper/cl-home`(The virtual disk device is in`fdisk -l` Find here).

For NAS services, it is necessary to do hard disk pass-through, which allows the virtual machine to directly control the entire hard disk, including temperature and other information, and also allows the HDD to hibernate when not in use. Create a pass-through virtual hard disk in ESXi and add it to the corresponding virtual machine.

## Some services

As for what you want to use a home server for, it varies from person to person. Here is a brief list of the services I am currently running:

-   Jupyter Lab, see[Build a Jupyter Lab container with multiple cores](https://ssine.ink/posts/versatile-jupyter-lab/)
-   Matrix chat server Synapse, see[Self-built chat server, robot and multi-platform bridge](https://ssine.ink/posts/matrix-bot-and-bridges/)
-   RSS related:
  -   [RSSHub](https://docs.rsshub.app/) , Used to provide RSS feeds
  -   [Tiny Tiny RSS](https://tt-rss.org/) , RSS Web client for reading
-   Mongodb, store some formatted personal information
-   Grafana, Followed Information Board
-   NextCloud, cloud disk, directly connected to 1T external SSD hard disk
-   Bitwarden, password management
-   Calibre, e-book management, not very useful, it is recommended to develop the habit of reading books first
