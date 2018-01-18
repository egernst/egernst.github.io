
---
layout: post
title:  Passing a Graphics card to a Clear Container using VFIO
category: posts
---


# Passing a Graphics card to a Clear Container

## Host System setup:

The directions below are specific to an Ubuntu 16.04 host system equipped with a GeForce GTX 550 Ti graphics card.  While the process should be the same, YMMV.

1. Make sure your system supports IOMMU groups

todo -- get directions from SRIOV page
 
2. Make sure your graphics card is in its own IOMMU group

```
$ find /sys/kernel/iommu_groups/ -type l | grep 03:00
/sys/kernel/iommu_groups/17/devices/0000:03:00.1
/sys/kernel/iommu_groups/17/devices/0000:03:00.0
```

In our case, there are two devices within the IOMMU group "17" - 03:00.0 and 03:00.1.  This is a result
of the graphics card being a multi function device for both graphics and audio.  As a result, it is required
that both devices be bound to vfio-pci in order to create a viable IOMMU group for device pass through.

3. Setup the device to bind to vfio-pci rather than its default (ie, nouvueau or other).

To avoid potential graphics instability on the platform, it is advisable to bind to vfio-pci at 
boot time rather than after the host system is up. To achieve this on Ubuntu 16.04, create the
file, describe the device IDs desired and then update the initramfs.  Also, in the case of
neauveau, you'll want to go ahead and blacklist that driver as well.  It is a greedy one.

For vfio, create the file ```/etc/modprobe.d/vfio.conf``` with the folowing contents:
```
options vfio-pci ids=10de:1b80,10de:10f0
options vfio-pci disable_vga=1
```

In our case, ```10de:1244,10de:0bee``` identifies the vendor:device ID for the Nvidia card's
graphics and audio devices.

Graphics device binding, particularly with the nouveau driver, happens early enough that to successfully bind
to vfio-pci, it is required that the graphics and audio driver be explicitly blacklisted.  To do so,
create the file ```/etc/modprobe.d/blacklist-nouveau.conf``` and write the following:

```
blacklist nouveau
options nouveau modeset=0
blacklist snd_hda_intel
```

For these changes to take effect, the initramfs needs to be updated and a reboot is required:

```
$ sudo update-initramfs -u
```

Edit /etc/default/grub in order to make sure that the vfio-pci driver is loaded early, before
other kernel drivers can grab the graphics card devices of interest (note, despite this, blacklist is still required).

```
GRUB_CMDLINE_LINUX="intel_iommu=on rd.driver.pre=vfio-pci video=efifb:off"
```
After editing, it is necesary to update grub.  In our case this was handled by:
```
$ sudo update-grub
```

For these changes to take effect a reboot is required:
```
$ sudo reboot
```

4. Ensure VFIO modules are loaded

```
$ sudo modprobe vfio_iommu_type
$ sudo modprobe vfio-pci
```

To make sure these are loaded on subsequent boots, add these lines to /etc/modules:

```
vfio-pci 
vfio_iommu_type1
```

## Launching a Clear Container with the Graphics Card passed through

Once the host system is configured appropriately for VFIO, starting a Clear Container
with the graphics card attached is achieved with the following Docker command, where ```/dev/vfio/17``` is determined based on the vfio devie path and the IOMMU group to which the graphics card belongs:

```
$ sudo docker run --runtime=cc-runtime --device=/dev/vfio/17 -it egernst/dpdk-enabled-ubuntu
```

Once inside the container, we can verify the device is available:

```
root@6d33a7279d87:/# lspci | grep NVI
00:06.0 VGA compatible controller: NVIDIA Corporation GF116 [GeForce GTX 550 Ti] (rev a1)
00:07.0 Audio device: NVIDIA Corporation GF116 High Definition Audio Controller (rev a1)
```

On the host, the device pass through can be observed on the running qemu's commandline:
```
$ ps -aef| grep qemu | sed 's/ -/\r\n -/g' | grep vfio-pci
 -device vfio-pci,host=03:00.0
 -device vfio-pci,host=03:00.1
 ```
