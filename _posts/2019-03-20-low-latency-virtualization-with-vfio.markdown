---
layout: post
title:  "Low latency virtualization with VFIO"
date:   2019-03-20 23:15:31 -0400
categories: linux virtualization
---

*NOTE: This is a work-in progress, don't try following anything until this message is gone.*

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

## Checklist

Basic host configuration:

- [ ] [Make sure you have non-local access to the machine](#Accessing the machine)
- [ ] [Collect all relevant PCI information](#PCI datamining)
- [ ] Configure the VM to run as non-root
- [ ] Pass through GPU sub-identifiers into the VM so that nVidia drivers will install correctly

Guest configuration:

- [ ] Ensure that the nVidia control panel perfomance level is set to Maximum

Critical performance checklist:

- [ ] CPU isolation
- [ ] USB controller passthrough
- [ ] Set CPU scaling policy to max performance
- [ ] Switch all Windows devices to use MSI (masked signal interrupts)

Optional perfomance checlist:

- [ ] Switch Windows guest disks over to virtualized IO device drivers
- [ ] Configure IO threads for the guest
- [ ] Switch the Windows guest network card to a virtualized device driver

## Accessing the machine

You will be yanking your GPU away from the Linux kernel if you follow this guide so make sure you can access the machine over SSH before doing anything or you're gonna be in for a bad time.

## On setting up the VM

There's probably a thousand ways to start and manage a VM in Linux but probably the most popular method is libvirt and virsh. Libvirt provides a really nice abstraction layer on top of different virtualization technologies across different OSs, and virsh is quite a nice management shell that let's you manage the VMs on your system. For something like a VFIO-based virtual machine you will be doing tons and tons of tweaks around the underlying qmeu-kvm command that runs and shell wrappers around that command to optimize and tweak performance so it's almost certainly easiest to start by manually creating your virtual machine command in a simple shell script and then maybe once you get everything working try [converting the script to a libvirt XML domain](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/virtualization_administration_guide/sub-sect-domain_commands-converting_qemu_arguments_to_domain_xml). On my host machine right now this is literally just a [gigantic script](#boot-mediapc.sh) in `/root`, alongside an [on-boot script](#on-boot.sh). Don't jump ahead and try and use either of those though because there's a lot of explaining to do.

## PCI datamining

Run the [check-iommu](#check-iommu) script yourself and save all of the data. The IOMMU groups and the PCI identifiers will come in handy later. Additionally, collect a standard list of your PCI devices with `lspci -nn`. Save everything in a text file.

## GPU selection

There's two common methods here for how the guest gets a GPU - booting with a single GPU and handing it off, or booting with one GPU then handing off a second GPU to the guest. The first is harder and probably stupider but that's what I went with. The second is easier and more common because most Intel CPUs have an integrated GPU onboard that you can use for your Linux host.

There's a bunch complications that arise from passing through a GPU that Linux used to boot. GPU drivers are pretty huge and complex and once attached to a device do *not* like being unbound. So one of the first things in the checklist should be telling your kernel on boot to bind a VFIO driver to the GPU instead of an actual one, and then on boot unbinding any virtual consoles or whatever from the GPU. Side note: You want to do all of the setup in the multiuser runlevel as opposed to the graphical login runlevel. On Debian this is set with:

```bash
systemctl set-default multi-user.target
```

## Boot configuration

Now you want to do a couple things around Kernel parameters and modules. First, update your kernel parameters (in `/etc/default/grub`) to activate IOMMU partitioning for your system:

```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt"
```

Now we want to tell the kernel to blacklist the nVidia drivers. Create the file `/etc/modprobe.d/blacklist-nvidia.conf`:

```bash
blacklist nouveau
blacklist lbm-nouveau
options nouveau modeset=0
alias nouveau off
alias lbm-nouveau off
```

Next we tell the kernel module `vfio-pci` which PCI devices to bind to on boot. Remember the giant list of PCI devices I told you to save? Find your GPU and it's associated audio device and use their vendor/device codes in a new file named `/etc/modprobe.conf/vfio.conf`:

```bash
options vfio-pci ids=10de:1b81,10de:10f0
```

Note these are the exact numbers from the `lspci -nn` output:

```bash
# lspci -nn | grep NVIDIA
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1070] [10de:1b81] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
```

Now is a pretty good time to set up your on-boot script. Literally nothing fancy here, we're not like deploying this to a datacenter or anything. Just create `/root/on-boot.sh`. There's an [example](#on-boot.sh) further down but start a little simple with simple ensuring that the virtual consoles get removed from the GPU and that the PCI device is re-mapped:

```bash
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind
echo '0000:01:00.1' > /sys/bus/pci/devices/0000:01:00.1/driver/unbind
echo 10de 1b81 > /sys/bus/pci/drivers/vfio-pci/new_id
echo 10de 10f0 > /sys/bus/pci/drivers/vfio-pci/new_id
```

Set that file up to run on boot. You can do this in the root crontab easily. Open said file with `crontab -e` and add this single line:

```
@reboot /root/on-boot.sh
```

TODO: I'm not 100% sure the last two lines are necessary, will verify.

That should be it - for now! Reboot, the first or (not) of many. Next boot you'll see the kernel messages fly everywhere (because we removed `quiet` from the kernel options) and then it'll look like it's frozen but at that point it's time to SSH in and do more stuff.

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

### boot-mediapc.sh

```bash
/usr/bin/chrt -r 1 \
/usr/bin/taskset -ac 2,3 \
/usr/bin/qemu-system-x86_64 \
    -enable-kvm -m 8192 \
    -machine q35,accel=kvm,vmport=off,dump-guest-core=off,kernel-irqchip=on \
    -cpu host,kvm=off,hv_synic,hv_stimer,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time,hv_vendor_id=whatever \
    -smp 2,sockets=1,cores=2,threads=1 \
    -realtime mlock=on \
    -pidfile /var/run/vm.mediapc.pid \
    -drive file=/dev/vmpool/mediapc,format=raw \
    -drive file=/dev/sda,format=raw \
    -rtc base=localtime,driftfix=slew \
    -no-hpet \
    -global kvm-pit.lost_tick_policy=discard \
    -global ICH9-LPC.disable_s3=1 \
    -global ICH9-LPC.disable_s4=1 \
    -device ioh3420,bus=pcie.0,addr=1c.0,multifunction=on,port=1,chassis=1,id=root.1 \
    -device vfio-pci,host=01:00.0,bus=root.1,addr=00.0,x-pci-sub-vendor-id=4318,x-pci-sub-device-id=7041,x-vga=on,multifunction=true,romfile=/root/Asus.GTX1070.8192.161102-patched.rom \
    -device vfio-pci,host=01:00.1,bus=root.1,addr=00.1 \
    -device vfio-pci,host=00:1a.0,bus=root.1 \
    -device vfio-pci,host=00:1d.0,bus=root.1 \
    -device vfio-pci,host=00:14.0,bus=root.1 \
    -device e1000,netdev=net0,mac=DE:AD:BE:EF:7A:A0 \
    -netdev tap,id=net0,script=/root/qemu-ifup \
    -no-user-config \
    -nodefaults \
    -display none \
    -vga none \
;
```

### on-boot.sh

```
#!/bin/bash

echo 120 > /proc/sys/vm/stat_interval
echo 0 > /proc/sys/kernel/watchdog
echo 0 > /proc/sys/kernel/nmi_watchdog
echo 3 > /sys/bus/workqueue/devices/writeback/cpumask # Set writeback threads to CPU0,1

echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind
echo 1 > /sys/module/kvm/parameters/ignore_msrs
sleep 1

echo '0000:01:00.1' | sudo tee /sys/bus/pci/devices/0000:01:00.1/driver/unbind
echo 10de 1b81 > /sys/bus/pci/drivers/vfio-pci/new_id
echo 10de 10f0 > /sys/bus/pci/drivers/vfio-pci/new_id

sleep 1
echo 1 > /sys/bus/pci/rescan
sleep 1

# USB bus passthrough
echo "8086 1e2d" > /sys/bus/pci/drivers/vfio-pci/new_id
echo "0000:00:1a.0" > /sys/bus/pci/devices/0000:00:1a.0/driver/unbind
echo "0000:00:1a.0" > /sys/bus/pci/drivers/vfio-pci/bind

echo "8086 1e26" > /sys/bus/pci/drivers/vfio-pci/new_id
echo "0000:00:1d.0" > /sys/bus/pci/devices/0000:00:1d.0/driver/unbind
echo "0000:00:1d.0" > /sys/bus/pci/drivers/vfio-pci/bind

echo "8086 1e31" > /sys/bus/pci/drivers/vfio-pci/new_id
echo "0000:00:14.0" > /sys/bus/pci/devices/0000:00:14.0/driver/unbind
echo "0000:00:14.0" > /sys/bus/pci/drivers/vfio-pci/bind
```

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
