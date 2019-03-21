---
layout: post
title:  "Low latency virtualization with VFIO"
date:   2019-03-20 23:15:31 -0400
categories: linux virtualization
---

For a long the media PC in our house has just been a bog-standard Windows installation that performs pretty light duty but is always on. Windows is pretty good for a TV interface (high-DPI scaling and everything works quite well in Windows 10), it plays back every form of media you can imagine via Kodi and MPC-HC in a pinch, and it runs games.

That being said when doing the whole media server part it wasn't really ideal as the large volume storage options leave a lot to be desired compared to ZFS and even really LVM, and there's a lot of things I'd like to use a Linux machine for that a Raspberry Pi (which I also use) won't cut it for. Basically don't ask me to justify this horror-show further than that.

Anyways I started this stupid setup by first just trying to convert the media PC to a Linux install but Linux suuuuuuucks when trying to do something as novel as use it as a computer on a TV that's normal human distance from yourself. Media playback is probably fine but games - despite a lot of effort - is still a non-starter.

I had read and experimented with the concept of running Windows as a VM for these reasons previously and since I now had a second OS installed (Debian Stretch) on a different, isolated drive I figured why not see on a lark if my thousand-year old system supported a VFIO configuration. If you're new to the term VFIO (Virtual Function I/O) is a pretty advanced hardware and software virtualization feature that allows the hypervisor to directly hand off PCI devices to a guest. Generally those can be any devices but for most people going down this path it's the video card so that they can play games under Linux.

## Testing the waters

The key pre-requisite to make this "easy" is that your motherboard and CPU have to cooperate in allowing the segmentation of your PCI devices into IOMMU groups that act as partitions for operating systems running under a hypervisor. Shockingly when checking the groupings with [this script](#check-iommu) my computer was pretty much a perfect candidate for this. The video card itself is isolated and not intermingled with anything else the host might need:

```
IOMMU Group 0 00:00.0 Host bridge [0600]: Intel Corporation 2nd Generation Core Processor Family DRAM Controller [8086:0100] (rev 09)
IOMMU Group 10 00:1f.0 ISA bridge [0601]: Intel Corporation Z77 Express Chipset LPC Controller [8086:1e44] (rev 04)
IOMMU Group 10 00:1f.2 SATA controller [0106]: Intel Corporation 7 Series/C210 Series Chipset Family 6-port SATA Controller [AHCI mode] [8086:1e02] (rev 04)
IOMMU Group 10 00:1f.3 SMBus [0c05]: Intel Corporation 7 Series/C216 Chipset Family SMBus Controller [8086:1e22] (rev 04)
IOMMU Group 11 03:00.0 Ethernet controller [0200]: Qualcomm Atheros AR8151 v2.0 Gigabit Ethernet [1969:1083] (rev c0)
IOMMU Group 1 00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200/2nd Generation Core Processor Family PCI Express Root Port [8086:0101] (rev 09)
IOMMU Group 1 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1070] [10de:1b81] (rev a1)
IOMMU Group 1 01:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
IOMMU Group 2 00:14.0 USB controller [0c03]: Intel Corporation 7 Series/C210 Series Chipset Family USB xHCI Host Controller [8086:1e31] (rev 04)
IOMMU Group 3 00:16.0 Communication controller [0780]: Intel Corporation 7 Series/C216 Chipset Family MEI Controller #1 [8086:1e3a] (rev 04)
IOMMU Group 4 00:1a.0 USB controller [0c03]: Intel Corporation 7 Series/C216 Chipset Family USB Enhanced Host Controller #2 [8086:1e2d] (rev 04)
IOMMU Group 5 00:1b.0 Audio device [0403]: Intel Corporation 7 Series/C216 Chipset Family High Definition Audio Controller [8086:1e20] (rev 04)
IOMMU Group 6 00:1c.0 PCI bridge [0604]: Intel Corporation 7 Series/C216 Chipset Family PCI Express Root Port 1 [8086:1e10] (rev c4)
IOMMU Group 7 00:1c.4 PCI bridge [0604]: Intel Corporation 7 Series/C210 Series Chipset Family PCI Express Root Port 5 [8086:1e18] (rev c4)
IOMMU Group 8 00:1c.6 PCI bridge [0604]: Intel Corporation 82801 PCI Bridge [8086:244e] (rev c4)
IOMMU Group 8 04:00.0 PCI bridge [0604]: Intel Corporation 82801 PCI Bridge [8086:244e] (rev 41)
IOMMU Group 9 00:1d.0 USB controller [0c03]: Intel Corporation 7 Series/C216 Chipset Family USB Enhanced Host Controller #1 [8086:1e26] (rev 04)
```

## On setting up the VM

## Machine configuration

* CPU: Intel i5-2500
* Memory: 16GB of no-name probably mismatched memory
* Drives: One 500GB drive for the Linux host operating system, one 120GB drive for the Windows 10 guest (this barely matters)
* Motherboard: Gigabyte GA-Z77M-D3H-MVP (older than dirt)
* GPU: ASUS nVidia 1070
* Guest OS: Windows 10, 64-bit
* Host OS: Debian Stretch

## Configuration files and options

## Scripts

### check-iommu

```bash
#!/bin/bash
shopt -s nullglob
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done;
```
