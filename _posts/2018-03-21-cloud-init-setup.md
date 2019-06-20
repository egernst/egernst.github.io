---
layout: post
title: Setting up a cloud image to use with KVM/QEMU
category: posts
---

I do this often enough to forget the details in between. Here's to not starting from scratch next time...

## Setup the image


 1. Grab a xenial cloud image:
```
 wget https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img  
```
 2. Create a qcow from this original image:
```
 qemu-img create -f qcow2 -o backing_file=xenial-server-cloudimg-amd64-disk1.img mine.image.qcow2
```
 3. Create a seed 
```
$ cat seed
#cloud-config
password: ubuntu
chpasswd: { expire: False }
ssh_pwauth: True
```

 4. Create the cloud seed image to be used on initial boot:
```
cloud-localds --disk-format qcow2 seed.img seed
```

 5. Initial boot
```
 kvm -hda mine.image.qcow2 -nographic -cdrom seed.img
``` 
Update image credentials if desired

## Boot the image

VMN should be adjusted if you are booting multiple virtual machines, and IMAGE points to the qcow created in step 2.

```
VMN=${VMN:=1}

qemu-system-x86_64 \
    -enable-kvm \
    -smp sockets=1,cpus=4,cores=2 -cpu host \
    -m 1024 \
    -vga none -nographic \
    -drive file="$IMAGE",if=virtio,aio=threads,format=qcow2 \
    -cdrom seed.img \
    -netdev user,id=mynet0,hostfwd=tcp::${VMN}0022-:22,hostfwd=tcp::${VMN}2375-:2375 \
    -device virtio-net-pci,netdev=mynet0 \
    -debugcon file:debug.log -global isa-debugcon.iobase=0x402 $@
```

During initial testing, there were connectivity issues (corp proxy) when running apt-get inside of the VM.  To
resolve this, update the /etc/environment settings for no_proxy.  Make sure to include the IP, localhost and
hostname (ubuntu in this case).  This could be due to hostname not being set appropriately in the image.


