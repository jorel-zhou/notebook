---
title: "IAAS中物理机OS的自动化安装"
permalink: /domains/cloud-os-installation-iaas/
excerpt: "IAAS中物理机OS的自动化安装"
last_modified_at: 2018-11-25T22:21:33-05:00
redirect_from:
  - /theme-setup/
toc: true
---

### 目标
在作者工作的云计算项目中，与阿里云或AWS云中最大的一个业务区别就是支持企业客户购买专用的物理机器而非在某虚拟化平台如VMware vShpere ESXi, KVM, Xen, HyperV等平台上的虚机(VM), 从技术上来说无论是装物理机操作系统还是虚拟化平台的类Unix系统的Hypervisor，从技术原理上基本上都是类似的。所以目标就是能够支持以下操作系统的自动化安装在bare metal的裸机上，操作系统列表可能如下：
1. Windows Server 20xx
2. RedHat Enterprise
3. SUSE Enterprise Linux
4. VMware vShpere ESXi
5. Xen 
6. KVM

### 组件(PXE Boot Server)
1. TFTP Server
2. DHCP Server
3. NFS Root Server
4. Enabled 网卡启动的物理机

### 涉及到的技术
- PXE: PXE(Preboot eXecution Environment) 是一个固件级的软件，一般烧在网卡或者主板上。如果你的机器支持PXE，你会在BIOS可启动设备中看到从网卡启动的选项。(tips: 如果希望自己的电脑启动快点, 一般我们将网卡启动放到硬盘启动之后，当然你的硬盘中应该已经装好了操作系统)
- gPXE: gPXE是PXE的一个开源实现，一般用在那些没有PXE的机器上，你需要将其烧到你的主板或使用本地磁盘，CD或USB盘中。
- DHCP: DHCP (Dynamic Host Configuration Protocol) 服务器分配相应的IP到客户端，它决定了PXE在启动过程中用什么IP去下载启动文件。
- TFTP: TFTP (Trivial File Transfer Protocol) 是PXE用于下载配置文件和kernel文件的一个轻量级文件传输协议，它不需要任何认证，一般情况下只针对指定目录有只读权限。
- PXELINUX: Linux Kernel的PXE启动加载器，在syslinux包中。
- NFS: NFS (Network File System) 用于在客户端挂载特定NFS服务器上的目录，PXE也就是通过它加载到根文件系统进行网络启动。
- Rarpd: 也是一个基于MAC地址提供IP地址分配的服务，目前基本上被DHCP替代但有些特定系统如OpenBSD还在使用它。
- Bootp: 最老的一个分配IP的服务，基本上不用了，它的有些地方与DHCP有冲突。

### 基础原理
1. 在物理机BIOS的启动顺序中设置从网卡启动并开机
2. 开机后，PXE Client中的DHCP Client能够从DHCP服务器中获得一个IP.
> 我们目前的做法是，预先将所有物理服务器连接的Access层(二层)交换机端口与DHCP服务器设置在同一个预定义的VLAN中,如VLAN 100(预定义定义子网`10.100.0.0/255.255.0.0`)，从而保证物理机在启动过程中能够拿到IP, 我们称该VLAN为Provision VLAN(其实也可以直接放在Default VLAN中)，完成OS的安装后将VLAN划到客户VLAN中或使用SDN方案从软件层如vSwitch/vxlan等技术进行网络隔离，以支持多租户模式。
3. 在物理网卡分配IP同时，DHCP服务根据定义在自己配置文件中的机器的网卡的MAC地址通知客户机连接相应的TFTP服务器并下载和运行`pxelinux.0`
4. pxelinux.0会在NFS挂载的目录中寻找`pxelinux.cfg`或者`pxelinux.cfg/default`文件
5. 在上面的配置文件中定义了客户端去哪下载kernel文件(或者还有initrd/miniroot)，然后客户端使用pxelinux配置文件中的参数如`nfsroot=`执行kernel文件
6. `etc/fstab`中定义了客户端应定义了NFS的挂载点做为根文件系统(如WinPE/Linux启动盘)，类似从硬盘启动。

DHCP配置示例
```
option space PXE;
option PXE.mtftp-ip               code 1 = ip-address;
option PXE.mtftp-cport            code 2 = unsigned integer 16;
option PXE.mtftp-sport            code 3 = unsigned integer 16;
option PXE.mtftp-tmout            code 4 = unsigned integer 8;
option PXE.mtftp-delay            code 5 = unsigned integer 8;
option PXE.discovery-control      code 6 = unsigned integer 8;
option PXE.discovery-mcast-addr   code 7 = ip-address;

# Provide PXE clients with appropriate information
  class "pxeclient" {
    match if substring(option vendor-class-identifier, 0, 9) = "PXEClient";
    vendor-option-space PXE;

    # At least one of the vendor-specific PXE options must be set in
    # order for the client boot ROMs to realize that we are a PXE-compliant
    # server.  We set the MCAST IP address to 0.0.0.0 to tell the boot ROM
    # that we can't provide multicast TFTP.

    option PXE.mtftp-ip 0.0.0.0;

    # This is the name of the file the boot ROMs should download.
    filename "pxelinux.0";
  }

  # Provide Etherboot clients with appropriate information
  class "etherboot" {
    match if substring(option vendor-class-identifier, 0, 9) = "Etherboot";
    filename "vmlinuz_arch;
  }

```

### 多系统支持
根据上面的基本原理我们可以看到只要相应修改nfs挂载到的不同的操作系统类型的根文件系统，再利用各不同操作系统的无人值守安装机制即能够是PXE支持多种不同类型的自动化安装。在作者所在的云项目中，利用shell脚本根据客户订单要求的操作系统类型的不同动态修改相应MAC的nfsroot配置，最终将shell包装成RESTful API提供前端调用，以达到多种不同种类的操作系统的网络启动和自动化安装。

以下为各常用操作系统的无人值守安装技术
1. ESXi/CentOS/RHEL - Kickstart
2. Windows - Unattended
3. SUSE - AutoYaST

### 开源工具
[Cobbler](https://cobbler.github.io/) is a Linux installation server that allows for rapid setup of network installation environments