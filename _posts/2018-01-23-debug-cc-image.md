---
layout: post
title: Debug of a Clear Containers image
category: posts
---

# Debug of Agent and base OS in the VM for Clear Containers

Some notes below on how I have been running debug of the Agent and base OS image for Clear Containers.

Yeah, there are better ways to do some of the steps, but this should give you the general idea...

## Update clear containers image to enable debug over socat

First, update the image you are using to create a new debug service.  

### 1. Mount the Clear Containers image

You may need to adjust which loop device you sue, and of course where you mount it.
```
sudo losetup /dev/loop3 /usr/share/clear-containers/clear-containers-debug.img  -o $((2048*512))
sudo mount /dev/loop3 ~/tmp-mnt/
```

### 2. Create service file

Create the file ```~/tmp-mnt/usr/lib/systemd/system/cc-debug-console.service```

```
[Unit]
Description=Container debug console

[Service]
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
StandardInput=tty
StandardOutput=tty
PrivateDevices=yes
Type=simple
ExecStart=/usr/bin/bash
```

Update ```~/tmp-mnt/usr/lib/systemd/system/clear-containers.target``` to include:

```
Wants=cc-debug-console.service
```

### 3. Unmount image

```
sudo umount ~/tmp-mnt
sudo losetup -d /dev/loop3
```

## Connecting to the Clear Containers VM image:

With the base OS image updated to support a shell debug session, let's connect.

### 1. Disable cc-proxy debug

Make sure debug is disabled for cc-proxy within your configuration.toml

A default location for this is at ```/usr/share/defauls/clear-containers/configuration.toml```.
You can check for file location by running cc-runtime cc-env`

### 2. Connect to Clear Container

The console.sock that we will attach to via socat is observable by checking the QEMU command line.  Here's
a sub-optimal script which'll do this for you automagically (assuming you have only a single instance of qemu up):

```
#!/bin/bash
SOCKPORT=`ps -aef| grep qemu | sed 's/ -/\r\n -/g' | grep "console.sock"| awk -F"path=" '{print $2}' | awk -F"," '{print $1}'`
sudo socat stdin,raw,echo=0,escape=0x11 unix-connect:$SOCKPORT
```

After hitting return, you should see your shell:
```
root@clrcont / # 
```

If you don't, your failure is either very early in boot or you likely forgot to disable cc-proxy's debug.

To disconnect from socat, use ctl-q
