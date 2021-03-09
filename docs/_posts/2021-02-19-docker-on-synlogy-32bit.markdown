---
layout: post
title: Docker on ARMv7 based Synology DS220
date: 2021-02-19 18:00:20 +0100
description: How to run Docker on ARMv7 based Synology 32b NAS 
img: i-rest.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [Synology, 32b, Docker, ARM, 32b, ARMv7, armhf]
---

# The problem

I love Docker, it makes my work simpler and helps me to keep environment of my computer cleaner. 
I can play with software without a risk of damaging impact of the introduced changes.

ARM platform became more and more [popular in IoT world](https://collabnix.com/building-arm-based-docker-images-on-docker-desktop-made-possible-using-buildx/), good thing is our NAS is based on ARM CPU.

The Synology server is the best fit for HomeAssistant if you have Synology already and thinking to play with smart home solutions
but when you try to configure your NAS as a HomeAssistant you end up with nothing or spend hours to try to make it working.

It only applies to the cheapest Synology products which are running on ARM-based 32bit processors.

You have few options here:
- Try to install HomeAssistant from sources
- Forget about Synology as a runtime environment for Docker
- or do what I did when I've decided to digg deeper and understand a little bit the Synology architecture

## What we need

* SSH access enabled on Synology -> (Go to DSM UC > Control Panel > Terminal & SNMP > Terminal, and tick Enable SSH service)
* [Static binaries of docker](https://github.com/docker-library/official-images#architectures-other-than-amd64) | [Binaries List](https://download.docker.com/linux/static/stable/)
* [Knowledge how to integrate it with our OS](https://docs.docker.com/engine/install/binaries/)

Don't worry I'll explain you step by step how to make it working on your ARM-32bits-based server

## Let's start

### Overview, gathering facts

Please SSH-in to your Synology. If you are using OS X or Linux open up terminal and type

```bash
ssh admin@YOURIP
```

Windows users needs to use the PUTTY client.

#### Synology file structure

Synology is running on build-in volume which is relatively small ~2.3G
You need to know all of your docker images which you download from internet needs to be stored somewhere else.
So where? In your data volume, so for the purpose of this guide I'll name it as *volume1*, and we are assuming it is placed here

```bash
/volume1
```

Call `df -h` to list your volumes

```bash
admin@KFC:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0        2.3G  1.7G  520M  77% /
none            249M     0  249M   0% /dev
/tmp            250M  944K  250M   1% /tmp
/run            250M  2.7M  248M   2% /run
/dev/shm        250M  4.0K  250M   1% /dev/shm
none            4.0K     0  4.0K   0% /sys/fs/cgroup
/dev/vg1000/lv  913G  636G  277G  70% /volume1
```

So mine is the last one on the list, and I'd use it for the Docker images persistence.

#### CPU info

My Synology DS220 `sudo cat /proc/cpuinfo` output:

```bash
admin@KFC:~# cat /proc/cpuinfo 
processor	: 0
model name	: ARMv7 Processor rev 1 (v7l)
BogoMIPS	: 2655.84
Features	: swp half thumb fastmult vfp edsp neon vfpv3 tls 
CPU implementer	: 0x41
CPU architecture: 7
CPU variant	: 0x4
CPU part	: 0xc09
CPU revision	: 1

processor	: 1
model name	: ARMv7 Processor rev 1 (v7l)
BogoMIPS	: 2655.84
Features	: swp half thumb fastmult vfp edsp neon vfpv3 tls 
CPU implementer	: 0x41
CPU architecture: 7
CPU variant	: 0x4
CPU part	: 0xc09
CPU revision	: 1

Hardware	: Marvell Armada 380/381/382/383/384/385/388 (Device Tree)
Revision	: 0000
Serial		: 0000000000000000
```

Ok, so what does it mean to me, if the Docker static list shows the list something like that:

```
aarch64/
armel/
armhf/
ppc64le/
s390x/
x86_64/
```

Useful explanation you can find here:

* The ARM EABI (armel) port targets a range of older 32-bit ARM devices, particularly those used in NAS hardware and a variety of *plug computers.
* The newer ARM hard-float (armhf) port supports newer, more powerful 32-bit devices using version 7 of the ARM architecture specification.
* The 64-bit ARM (arm64) port supports the latest 64-bit ARM-powered devices.

Source: https://www.debian.org/ports/arm/index.en.html

So ours is *armhf*

### Binaries download & testing

Ok, we know a little more about or hardware, so it is a good time to download binaries and proceed with installation.

Go to you home directory and prepare directory for download and extraction of the archive

```bash
cd ~/
mkdir docker_install
cd docker_install
wget URL_TO_DOCKER_BINARY_GOES_HERE
```

You need to replace phrase: `URL_TO_DOCKER_BINARY_GOES_HERE` with valid URL to the most recent docker binary for example [docker-20.10.5](https://download.docker.com/linux/static/stable/armhf/docker-20.10.5.tgz) taken from [here](https://download.docker.com/linux/static/stable/armhf/)

Ok, let's extract the repository

```bash
tar xvf docker-x.x.x.tgz
```
Where `docker-x.x.x.tgz` is downloaded TAR archive

Before we install the extracted files we can test is docker binary compatible with our OS, so let's change directory to extracted one and test the binary like below:

```bash
cd docker
./docker --version
```

the output example:
```bash
Docker version 20.10.5, build 55c4c88
```
The output should be the downloaded docker version info of the docker binary if we receive error message instead it means we downloaded incompatible package.

### Docker installation

You can go to [this](https://docs.docker.com/engine/install/binaries/) document directly or read entire description below

Let's check where you are

```bash
pwd
```

should output

```bash
/admin/docker-install/docker
```

Time to install your docker:

* IMPORTANT you are in extracted directory of docker archive
* What we are going to do right now
  * Change directory to level up just for safety and readability
  * Copy all binaries to `/usr/bin/` directory
  * Cleanup downloaded resources
    
```bash
cd /admin/docker-install
sudo cp docker/* /usr/bin/
```

Start the Docker daemon:

```bash
sudo dockerd &
```

If no errors means it runs :)

Additional configuration

We need to tell Docker we need to store data in our `/volume1` but before we need to create a place for docker there

```bash
mkdir /volume1/docker
```

Docker needs config file for that:

so you need to create/edit following file `/etc/docker/daemon.json`

```bash
{
  "storage-driver": "vfs",
   "iptables": false,
    "bridge": "none",
	"data-root": "/volume1/docker"
}
```

To test docker is able to run call `dockerd` command

``` bash
dockerd
```

To stop press CTRL+C

Reboot your Synology you can type

```bash
sudo reboot
```

Once it is up SSH-in again and type to test your Docker

```bash
sudo docker run hello-world
```

You should see finally tge output from hello-world container

```bash
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm32v5)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

CONGRATULATIONS YOU HAVE INSTALLED DOCKER ON your ARM based NAS!