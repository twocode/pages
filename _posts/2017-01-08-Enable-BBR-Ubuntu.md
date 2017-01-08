---
title: Enable BBR on Ubuntu
date: 2017-01-08 19:00
layout: post
category: projects
comments: true
tags: linux network ubuntu bbr
description: Bottleneck Bandwidth and RTT (BBR) is the new TCP congestion and traffic control algorithm introduced to kernel 4.9. Enabling it can hugely benefits to the performance of shadowsocks.

---

The server in use is a *v16.04 Ubuntu hosted on Azure*. 

**Download Kernel and install**

```
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.9/linux-headers-4.9.0-040900_4.9.0-040900.201612111631_all.deb

wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.9/linux-headers-4.9.0-040900-generic_4.9.0-040900.201612111631_amd64.deb

wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.9/linux-image-4.9.0-040900-generic_4.9.0-040900.201612111631_amd64.deb

sudo dpkg -i *.deb

dpkg -l | grep linux-image
```

(*For some linux distros, you need to update grub.cnf, for Ubuntu, there is no need*)

**Reboot**

Now check if the kernel has been installed.

```
uname -r
4.9.0-040900-generic
```

**Enable BBR**

```
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf 
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
sysctl net.ipv4.tcp_available_congestion_control
```

BBR has taken effect obviously since there is no more dropping FPS and jittering when watching Youtube HD.


<br />

