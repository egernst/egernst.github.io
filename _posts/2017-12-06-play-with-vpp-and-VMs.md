---
layout: post
title: Play session with VMs and VPP
category: posts
---


# Advanced play session with VPP


In all of the examples described below we are connecting a physical interface to a vhost-user interface which is consumed either by a VM image or a Clear Container.  The following startup.conf can be used for all of the examples.

```
unix {
  nodaemon
  log /tmp/vpp.log
  full-coredump
  cli-listen localhost:5002
}

api-trace {
  on
}

api-segment {
  gid vpp
}

cpu {
        main-core 1
        corelist-workers 2-6
}

dpdk {
        dev 0000:05:00.1
        dev 0000:05:00.2
        socket-mem 2048,0
        no-multi-seg
}
```
Note the devices described for DPDK, ```05:00.1``` and ```05:00.2```.  These are specific to the given test setup.

## QEMU instances with two VPP Vhost-user interfaces

For performance testing, a useful topology is to test between two physical intefaces and through a VM.  In our first example, let's look at PHY-> Vhostuser VM -> L2FWD -> Vhostuser VM -> PHY.  To achieve this, we'll need two things: a VM with two vhost user interfaces, and a DPDK enabled VM to run a simple port to port forwarding (testpmd).

![](https://gist.github.com/egernst/01dce46399439fbdbc461ec7c961784f/raw/b8ef9d1a6d21ae97fdbc543f430c5a460c0f765c/gist-networking-1vm-2vhu.png)

### Setup of Network components on host:

1. View interfaces and put into 'up' state

```
vpp# show interfaces  
              Name               Idx       State          Counter          Count     
TenGigabitEthernet5/0/1           1         down      
TenGigabitEthernet5/0/2           2         down 

vpp# set interface state TenGigabitEthernet5/0/1 up

vpp# set interface state TenGigabitEthernet5/0/2 up

vpp# show interfaces  
              Name               Idx       State          Counter          Count     
TenGigabitEthernet5/0/1           1         up       
TenGigabitEthernet5/0/2           2         up   
```

2. Connect each physical interface to an L2 bridge

```
vpp# set interface l2 bridge TenGigabitEthernet5/0/1 1

vpp# set interface l2 bridge TenGigabitEthernet5/0/2 2
```

3. Create, bring up and add vhost-user interfaces to L2 bridges

```
vpp# create vhost-user socket /tmp/sock1.sock server            
VirtualEthernet0/0/0

vpp# create vhost-user socket /tmp/sock2.sock server
VirtualEthernet0/0/1

vpp# set interface state VirtualEthernet0/0/0 up

vpp# set interface state VirtualEthernet0/0/1 up

vpp# set interface l2 bridge VirtualEthernet0/0/0 1

vpp# set interface l2 bridge VirtualEthernet0/0/1 2

vpp# show interface                             
              Name               Idx       State          Counter          Count     
TenGigabitEthernet5/0/1           1         up       
TenGigabitEthernet5/0/2           2         up       
VirtualEthernet0/0/0              3         up       
VirtualEthernet0/0/1              4         up       
local0                            0        down  
```

4. Show resulting bridge setup

```
vpp# show bridge 1 detail                       
  ID   Index   Learning   U-Forwrd   UU-Flood   Flooding   ARP-Term     BVI-Intf   
  1      1        on         on         on         on         off          N/A     

           Interface           Index  SHG  BVI  TxFlood        VLAN-Tag-Rewrite       
    TenGigabitEthernet5/0/1      1     0    -      *                 none             
     VirtualEthernet0/0/0        3     0    -      *                 none             

vpp# show bridge 2 detail                       
  ID   Index   Learning   U-Forwrd   UU-Flood   Flooding   ARP-Term     BVI-Intf   
  2      2        on         on         on         on         off          N/A     

           Interface           Index  SHG  BVI  TxFlood        VLAN-Tag-Rewrite       
    TenGigabitEthernet5/0/2      2     0    -      *                 none             
     VirtualEthernet0/0/1        4     0    -      *                 none  
```



### Qemu commandline:

```
taskset 3C0 qemu-system-x86_64  \
  -enable-kvm -m 8192 -smp cores=4,threads=0,sockets=1 -cpu host \
  -drive file="ubuntu-16.04-server-cloudimg-amd64-disk1.img",if=virtio,aio=threads \
  -drive file="seed.img",if=virtio,aio=threads \
  -nographic -object memory-backend-file,id=mem,size=8192M,mem-path=/dev/hugepages,share=on \
  -numa node,memdev=mem \
  -mem-prealloc \
  -chardev socket,id=char1,path=/tmp/sock1.sock \
  -netdev type=vhost-user,id=net1,chardev=char1,vhostforce \
  -device virtio-net-pci,netdev=net1,mac=00:00:00:00:00:01,csum=off,gso=off,guest_tso4=off,guest_tso6=off,guest_ecn=off,mrg_rxbuf=off \
  -chardev socket,id=char2,path=/tmp/sock2.sock \
  -netdev type=vhost-user,id=net2,chardev=char2,vhostforce \
  -device virtio-net-pci,netdev=net2,mac=00:00:00:00:00:02,csum=off,gso=off,guest_tso4=off,guest_tso6=off,guest_ecn=off,mrg_rxbuf=off
```

## Chaining QEMU instances with VPP
Taking the prior example further, let's look at chaining two QEMU isntances together so we have PHY VM VM PHY with connectiviyt provided by VPP L2 bridges.  Going forward, L2 xconnect makes more sense than bridge (if it is just direct connection).

![](https://gist.github.com/egernst/01dce46399439fbdbc461ec7c961784f/raw/b8ef9d1a6d21ae97fdbc543f430c5a460c0f765c/gist-networking-2vm-4vhu.png)




## Multi-VPP network interfaces with Clear Containers

Prior example was using QEMU by hand.  Let's look at connecting a Clear Container to two vhost-user interfaces in order to facilitate a similar PHY -> VM -> PHY test.  To facilitate this we'll need to setup the networking for attaching two interfaces to a clear container and we'll need to start a container image which has DPDK enabled for a simple test application such as testpmd.

### Installing VPP CNM-plugin and Clear Containers on host system

You will need a VPP CNM plugin on your host system to facilitate testing VPP with Clear Containers in Docker. Details can be found at https://github.com/01org/cc-oci-runtime/blob/networking/vhost-user-poc/documentation/Using-VPP-and-COR.md#install-vpp

You will need Clear Containers installed on your system.  Directions for installation can be found at https://github.com/01org/cc-oci-runtime/blob/master/documentation/Installing-Clear-Containers-on-Ubuntu.md

### Docker image...

Can be found at https://hub.docker.com/r/egernst/dpdk-enabled-ubuntu/


### Launching Clear Container with two vhost-user interfaces

On 2.1, Clear Containers does not support network hotplug. Docker also does not allow for specifiying multiple networks on the commandline.  To work around this we will hotplug a network to a placeholder runc based container, and then start a clear container making use of the placeholder's interfaces.  

1. Clean up

In case we ran through this example before, let's remove the old networks/containers:
```
sudo docker kill placeholder
sudo docker network rm vpp_1
sudo docker network rm vpp_2
sudo docker rm $(sudo docker ps -a -q) 
```

2. Create networks

Add two networks, one for each interface in our resulting Clear Container:
```
sudo docker network create -d=vpp --ipam-driver=vpp --subnet=192.168.1.0/24 --gateway=192.168.1.1 --opt "bridge"="none" vpp_1
sudo docker network create -d=vpp --ipam-driver=vpp --subnet=192.168.2.0/24 --gateway=192.168.2.1 --opt "bridge"="none" vpp_2
```

3. Create network placeholder runc container, which we will shamelessly steal from:

```
sudo docker run --runtime=runc --net=vpp_1 --name placeholder --ip=192.168.1.2 -itd alpine sh
```

4. Add an additional network to placeholder

Add the other network to the existing container:
```
sudo docker network connect vpp_2 placeholder --ip=192.168.2.2
```

5. Start a clear container, stealing the network from placeholder:

```
sudo docker run --runtime=mine --net=container:placeholder -it dpdk-enabled-ubuntu bash
```
