---
layout: post
published: true
title: 2025-06-17-NVIDIA vGPU 基于虚机方案调研
categories: [大模型]
tags: [GPU,vGPU]
---
* content
{:toc}


# 背景


# vGPU能力概述

NVIDIA的vGPU功能是指将物理GPU分割成多个GPU实例，然后分配给虚拟机使用的能力，以达到GPU超分的目的。属于收费功能，企业注册可以90天试用，注册完之后才可以下载vGPU驱动及License。也并非所有的GPU都支持vGPU能力，可以通过官方网站查询支持的GPU(https://docs.nvidia.com/vgpu/gpus-supported-by-vgpu.html) 。支持vGPU的GPU有：NVIDIA L4、L4O、L40S、 A10、A16、A40、A100、T4、RTX 6000Ada、RTX 5000 Ada、RTX 8000、 RTX 6000、VI0OS、VI00、P10O、P40、P6、P4、M60。

> 如果GPU不支持vGPU，即使GPU通过某种技术手段在宿主机上完成了GPU的切分，也无法分配给虚拟机上使用。原因在于驱动和授权限限制。

## 实现方式：

要实现vGPU能力，需要先在宿主机上将物理GPU的分割成多个GPU实例，然后挂载到虚拟机。挂载到虚拟机上的vGPU还需要安装官方的驱动及授权才可以使用。因此，vGPU实现方式主要分成三步：

1. 物理GPU分割成vGPU实例；
2. vGPU分配给虚拟机；
3. 虚拟机安装vGPU驱动；

在宿主机上分割物理GPU有两种方式：
1. 软件方式：SRIOV技术，通过地址映射，将一个PCI设备映射成多个虚拟设备 
2. 物理方式：硬件分割，通过芯片制作工艺支持，即MIG能力。

### 软件方式：SRIOV方式分割GPU

SRIOV技术是单根输入输出虚拟化功能，通过将单个PCIe物理设备虚拟化成多个独立虚拟设备，如CPU、网卡等。由于GPU也是通过PCI插槽方式插在主板上，因此，也可以使用这种技术，将物理GPU虚拟化成多个虚GPU实例。

软件分割是时分性原理，即在某个时间段将设备分配给某个虚拟设备使用，当有其他设备抢占时，挂起当前任务，去响应优先级更高的任务。在网络和CPU上，这种超分方式还能接受，只是性能受影响。而GPU处理密集型计算任务时，对性能要求非常高。因此，新的GPU开始支持硬件的方式分割GPU，这种方式只在新的GPU芯片上才可使用，即MIG能力。

### 硬件方式：MIG方式分割GPU

MIG是Multi-Instance GPU的缩写，意思是多实例GPU。它是把通过芯片硬件的方式，把一个GPU物理分割成多个GPU实例来用，一般情况最多7个。每个计算实例拥有自己的计算资源，提供更好的性能和隔离。从vGPU 11.1版本开始支持基于MIG技术的vGPU虚拟机实例，最新的GPU一般都支持MIG：A100、A30、H100、H200以及RTX PRO 6000 Blackwell系列。更多查阅官方(https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)查看支持MIG的GPU。

> 说明: H20未在官网上列出，但经验证，可以基于MIG分出多个GPU实例后；由于H20只能基于NVLINK不支持基于PCI通信，分出的实例未必能用，目前还未验证。



## 收费情况
todo

```bash
 NVIDIA® RTX™ Virtual Workstation (vWS), NVIDIA Virtual PC, and NVIDIA Virtual Applications.
```

# 环境
操作系统: Ubuntu 22.04.5 LTS              
内核: Linux 5.15.0-141-generic
GPU: NVIDIA A40
GPU驱动：NVIDIA-Linux-x86_64-550.163.01.run 
vGPU驱动：NVIDIA-GRID-Linux-KVM-550.163.02-550.163.01-553.74


# A40 vGPU操作记录


### 安装nvidia驱动

`./NVIDIA-Linux-x86_64-550.163.01.run  --no-opengl-files --ui=none --no-questions --accept-license  `


安装完成后，nvidia-smi命令可以正常执行成功；如果执行不成功，可以看GPU被什么进程占用。`lspci -vv|grep -i nvidia`可以查看被占用的驱动，我的本地发现是被vifio-pci占用。如果kernel是vfio-pci，则需要手动释放：

解绑gpu脚本：

```bash
$ lspci -nn | grep -i nvidia查询出pci ID
for DEVICE_ID in 0000:e8:00.0 0000:e7:00.0 0000:e6:00.0 0000:e5:00.0 0000:6b:00.0 0000:6a:00.0 0000:68:00.0 0000:67:00.0 ;do echo -n "$DEVICE_ID" > /sys/bus/pci/devices/$DEVICE_ID/driver/unbind
```

### vGPU功能需要修改内核

```bash
$ vi /etc/default/grub
...
intel_iommu=on iommu=pt 
...
```

> 解释：
1. 启用 Intel 的 IOMMU 功能（VT-d）
Intel 的 IOMMU 技术称为 VT-d（Virtualization Technology for Directed I/O），用于管理硬件设备（如显卡、网卡）对系统内存的直接访问（DMA）。
2. 启用 IOMMU 的页表（Page Tables）支持，IOMMU 的核心功能之一是实现 地址转换：设备通过 IOMMU 将自身的 “虚拟地址” 转换为系统的物理地址。pt 代表 Page Tables（页表），启用后，IOMMU 会利用 CPU 的页表机制（如 Intel 的 EPT/VPID 或 AMD 的 NPT）加速地址转换，减少性能损耗。
该参数通常用于优化 二级地址转换（如虚拟机中 Guest OS 的虚拟地址 → 主机物理地址 → IOMMU 转换后的地址），提升设备透传的效率。


生成启动文件

`$ grub-mkconfig > /boot/grub/grub.cfg`

>通过 `dpkg -l | grep grub2` 看是grub2还是grub

### 安装vGPU驱动

vGPU的驱动分成host和guest，在宿主机上需要装host的驱动，在虚拟机中需要装guest驱动。

```bash
# 535 kvm 驱动下载
wget --no-check-certificate "https://griddownloads.nvidia.com/ems/sec/vGPU17.6/NVIDIA-GRID-Linux-KVM-550.163.02-550.163.01-553.74.zip?autho=st=1750064794~exp=1750068394~acl=/ems/sec/vGPU17.6/NVIDIA-GRID-Linux-KVM-550.163.02-550.163.01-553.74.zip~hmac=a5d806ec403b7ddbfe74cd163432eb9a17b0b0eab0eca6ee6c8192ef1a7352b7" -O NVIDIA-GRID-Linux-KVM-550.163.02-550.163.01-553.74.zip
```

下载下来后，解压安装完，会有两个服务：

```bash
# systemctl list-units --all|grep vgpu
  nvidia-vgpu-mgr.service                                                                                                                       loaded    active     running      NVIDIA vGPU Manager Daemon
  nvidia-vgpud.service                                                                                                                          loaded    inactive
```
其中 nvidia-vgpu-mgr服务是一直运行的，另一个服务nvidia-vgpud则是在需要的时候按需运行，并不会一直运行。


**修改 kernel 参数和安装 vGPU Host 驱动以后，一定要 reboot 使其生效。**

安装完后nvidia-smi应该可以看到GPU：

```bash
# nvidia-smi 
Wed Jun 18 14:10:22 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.163.02             Driver Version: 550.163.02     CUDA Version: N/A      |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA A40                     On  |   00000000:67:00.0 Off |                    0 |
|  0%   28C    P8             30W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA A40                     On  |   00000000:68:00.0 Off |                    0 |
|  0%   28C    P8             31W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA A40                     On  |   00000000:6A:00.0 Off |                    0 |
|  0%   28C    P8             31W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA A40                     On  |   00000000:6B:00.0 Off |                    0 |
|  0%   28C    P8             31W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   4  NVIDIA A40                     On  |   00000000:E5:00.0 Off |                    0 |
|  0%   28C    P8             32W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   5  NVIDIA A40                     On  |   00000000:E6:00.0 Off |                    0 |
|  0%   28C    P8             31W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   6  NVIDIA A40                     On  |   00000000:E7:00.0 Off |                    0 |
|  0%   28C    P8             32W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   7  NVIDIA A40                     On  |   00000000:E8:00.0 Off |                    0 |
|  0%   29C    P8             30W /  300W |   22657MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
```

### 启用sriov

调用nvidia的sriov-manage工具，可以指定特定的GPU卡
```bash
/usr/lib/nvidia/sriov-manage -e 0:e8:0.0

# dmesg无报错
# 如果提示file not exist，检查内核 iommu 是否OK，需要reboot
```

> 如果要对所有的卡启用，可以 -h 看help参数，参数中 -e all则是对所有的卡启用sriov。help参数中列出了对卡禁用sriov的方式。
```bash
# /usr/lib/nvidia/sriov-manage -h 
Usage:
    /usr/lib/nvidia/sriov-manage <-e|-d> <ssss:bb:dd.f|ALL>
        Enable/disable Virtual functions on the GPU specified by the PCI_ID=ssss:bb:dd.f.
        If 'ALL' is specified in place of PCI_ID, operate on all the nvidia GPUs in the system.

    /usr/lib/nvidia/sriov-manage -e <ssss:bb:dd.f|ALL> -r 1
		Reset and rescan the GPU followed by enabling Virtual functions on the GPU.

    /usr/lib/nvidia/sriov-manage -h
        Print this help.
```


通过 lspci 命令可查看 GPU 支持的 SR-IOV 属性

```bash
root@control01:~# lspci -s 0:e8:0.0 -vv

    Capabilities: [bcc v1] Single Root I/O Virtualization (SR-IOV)
        IOVCap:    Migration-, Interrupt Message Number: 000
        IOVCtl:    Enable+ Migration- Interrupt- MSE+ ARIHierarchy+
        IOVSta:    Migration-
        Initial VFs: 32, Total VFs: 32, Number of VFs: 32, Function Dependency Link: 00
```

最大分配的vGPU有32个。使用SRIOV后，在 /sys/class/mdev_bus/ 目录中可以列出所有可用于创建 vGPU 的VF 设备的 BDF，A40 的 00:e8:* 的 VF 目录一共是 32 个。

```bash
$ ls /sys/class/mdev_bus/ 
0000:e8:00.4  0000:e8:01.2  0000:e8:02.0  0000:e8:02.6  0000:e8:03.4  0000:e8:04.2
0000:e8:00.5  0000:e8:01.3  0000:e8:02.1  0000:e8:02.7  0000:e8:03.5  0000:e8:04.3
0000:e8:00.6  0000:e8:01.4  0000:e8:02.2  0000:e8:03.0  0000:e8:03.6
0000:e8:00.7  0000:e8:01.5  0000:e8:02.3  0000:e8:03.1  0000:e8:03.7
0000:e8:01.0  0000:e8:01.6  0000:e8:02.4  0000:e8:03.2  0000:e8:04.0
0000:e8:01.1  0000:e8:01.7  0000:e8:02.5  0000:e8:03.3  0000:e8:04.1
```

基于 SRIOV 的 vGPU 特性创建 SR-IOV 类型的 vGPU，遵循以下方法:

  1. 每个 vGPU 实例占用一个 VF 设备，一旦 VF 已经被分配，该 VF 上不可再创建 vGPU。
  2. 每物理 GPU 的 VF 总数 >= 该 GPU 上可创建的 vGPU 最大实例数，例如 A40 单卡的Total VF 数量为 32，而最大 A40 单卡的 vGPU 实例数为 40GB(FB 总量)/4GB(4C 类型vGPU Size) = 10。因此，VF 总数并不是该 GPU 的可创建最大 vGPU 数量。
  3. 最大可创建的 vGPU 实例数可以查询该 vGPU 类型目录中的 description 文件中描述的max_instances 的值。例如:
  
```bash
  root@control01:~# cat /sys/class/mdev_bus/0000\:e8\:00.4/mdev_supported_types/nvidia-555/name 
  NVIDIA A40-1B

  root@control01:~# cat /sys/class/mdev_bus/0000\:e8\:00.4/mdev_supported_types/nvidia-555/description 
  num_heads=4, frl_config=45, framebuffer=1024M, max_resolution=5120x2880, max_instance=32
```

每一个 VF 设备的当前实时可创建 vGPU 数量应查询该 VF 目录下，指定类型目录中available_instances 文件中的值：

1 表示可以在此 VF 设备创建指定的 vGPU 类型实例。
0 则表示不可创建此类型 vGPU，可能是由于当前 VF 已经被占用，或当前 vGPU类型不被支持。
  
```bash
  cat /sys/class/mdev_bus/0000\:e8\:00.4/mdev_supported_types/nvidia-555/available_instances 
  1
```

### Time Sliced 类型的 vGPU 配置概要

下面脚本可以列出所有支持的 mdev 设备名称和 vGPU 设备可用数量
```bash
cd mdev_supported_types
for i in * ; do echo $i, `cat $i/name` `cat $i/ava*` ; done


nvidia-555, NVIDIA A40-1B 1
nvidia-556, NVIDIA A40-2B 1
nvidia-557, NVIDIA A40-1Q 1
nvidia-558, NVIDIA A40-2Q 1
nvidia-559, NVIDIA A40-3Q 1
nvidia-560, NVIDIA A40-4Q 1
nvidia-561, NVIDIA A40-6Q 1
nvidia-562, NVIDIA A40-8Q 1
nvidia-563, NVIDIA A40-12Q 1
nvidia-564, NVIDIA A40-16Q 1
nvidia-565, NVIDIA A40-24Q 1
nvidia-566, NVIDIA A40-48Q 1
nvidia-567, NVIDIA A40-1A 1
nvidia-568, NVIDIA A40-2A 1
nvidia-569, NVIDIA A40-3A 1
nvidia-570, NVIDIA A40-4A 1
nvidia-571, NVIDIA A40-6A 1
nvidia-572, NVIDIA A40-8A 1
nvidia-573, NVIDIA A40-12A 1
nvidia-574, NVIDIA A40-16A 1
nvidia-575, NVIDIA A40-24A 1
nvidia-576, NVIDIA A40-48A 1
```

### 使用uuid创建vgpu设备

```bash
echo 2f82ee72-3ac5-4615-9c0b-327c12e8c83d > /sys/class/mdev_bus/0000:e8:00.4/mdev_supported_types/nvidia-555/create

root@control01:/sys/class/mdev_bus/0000:e8:00.4/mdev_supported_types/nvidia-555# ls devices/
2f82ee72-3ac5-4615-9c0b-327c12e8c83d
```
至此可以看到已经创建成功

再次检查可以创建的数量：

```bash
root@control01:/sys/class/mdev_bus/0000:e8:00.4/mdev_supported_types# cd mdev_supported_types
for i in * ; do echo $i, `cat $i/name` `cat $i/ava*` ; done
-bash: cd: mdev_supported_types: No such file or directory
nvidia-555, NVIDIA A40-1B 0
nvidia-556, NVIDIA A40-2B 0
nvidia-557, NVIDIA A40-1Q 0
nvidia-558, NVIDIA A40-2Q 0
nvidia-559, NVIDIA A40-3Q 0
nvidia-560, NVIDIA A40-4Q 0
nvidia-561, NVIDIA A40-6Q 0
nvidia-562, NVIDIA A40-8Q 0
nvidia-563, NVIDIA A40-12Q 0
nvidia-564, NVIDIA A40-16Q 0
nvidia-565, NVIDIA A40-24Q 0
nvidia-566, NVIDIA A40-48Q 0
nvidia-567, NVIDIA A40-1A 0
nvidia-568, NVIDIA A40-2A 0
nvidia-569, NVIDIA A40-3A 0
nvidia-570, NVIDIA A40-4A 0
nvidia-571, NVIDIA A40-6A 0
nvidia-572, NVIDIA A40-8A 0
nvidia-573, NVIDIA A40-12A 0
nvidia-574, NVIDIA A40-16A 0
nvidia-575, NVIDIA A40-24A 0
nvidia-576, NVIDIA A40-48A 0
```
全部为0。意味着这个VF已经不能再创建vGPU设备。新的vGPU只能创建在不同的VF上了。

此时，可以切换其他的VF，再次创建，直到所有的VF全部绑定；

```bash
root@control01:/sys/class/mdev_bus# ls | grep e8:00
0000:e8:00.4
0000:e8:00.5
0000:e8:00.6
0000:e8:00.7
```

创建第二个：
```bash
root@control01:/sys/class/mdev_bus/0000:e8:04.0/mdev_supported_types/vfio-pci-575# ls devices/
root@control01:/sys/class/mdev_bus/0000:e8:04.0/mdev_supported_types/vfio-pci-575# uuidgen > create
root@control01:/sys/class/mdev_bus/0000:e8:04.0/mdev_supported_types/vfio-pci-575# ls devices/
242e5ea7-90e2-4d6e-afeb-a0b9a656c108
root@control01:/sys/class/mdev_bus/0000:e8:04.0/mdev_supported_types/vfio-pci-575# ls devices/
242e5ea7-90e2-4d6e-afeb-a0b9a656c108

root@control01:/sys/class/mdev_bus/0000:e8:04.0/mdev_supported_types/vfio-pci-575# cat name 
NVIDIA A40-24A
```
所有创建成功的vGPU会在 /sys/bus/mdev/devices下创建同名mdev设备
```bash
root@control01:/sys/class/mdev_bus/0000:e8:04.0/mdev_supported_types/vfio-pci-575# ls /sys/bus/mdev/devices/
242e5ea7-90e2-4d6e-afeb-a0b9a656c108
```
同时，要把uuid保存，写入/etc/rc.local，保证开机自动加载
```bash
$ vi /etc/rc.local

echo 2f82ee72-3ac5-4615-9c0b-327c12e8c83d > /sys/class/mdev_bus/0000:e8:00.4/mdev_supported_types/nvidia-555/create
echo 242e5ea7-90e2-4d6e-afeb-a0b9a656c108 > /sys/class/mdev_bus/0000:e8:04.0/mdev_supported_types/vfio-pci-575/create
```

上面这个是实际创建时候的操作，但是发现上面一条记录  nvidia-555这个命令不会在 /sys/bus/mdev下创建。后面机器重启后这个目录也不存在。下面那个vfio-pci-575会正常。最后经验证，vfio-pci下的vgpu可以正常分配给虚机。

在宿主机上验证已经创建发的vGPU：

```bash
# nvidia-smi vgpu
Wed Jun 18 15:56:14 2025       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 550.163.02             Driver Version: 550.163.02                |
|---------------------------------+------------------------------+------------+
| GPU  Name                       | Bus-Id                       | GPU-Util   |
|      vGPU ID     Name           | VM ID     VM Name            | vGPU-Util  |
|=================================+==============================+============|
|   0  NVIDIA A40                 | 00000000:67:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
|   1  NVIDIA A40                 | 00000000:68:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
|   2  NVIDIA A40                 | 00000000:6A:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
|   3  NVIDIA A40                 | 00000000:6B:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
|   4  NVIDIA A40                 | 00000000:E5:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
|   5  NVIDIA A40                 | 00000000:E6:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
|   6  NVIDIA A40                 | 00000000:E7:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
|   7  NVIDIA A40                 | 00000000:E8:00.0             |   0%       |
|      3251634401  NVIDIA A40-24A | 7cc6...  'sss',debug-thre... |      0%    |
+---------------------------------+------------------------------+------------+
```

使用nvidia-smi命令也可以直接看到：

```bash
root@control01:~# nvidia-smi 
Wed Jun 18 15:57:22 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.163.02             Driver Version: 550.163.02     CUDA Version: N/A      |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA A40                     On  |   00000000:67:00.0 Off |                    0 |
|  0%   27C    P8             30W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA A40                     On  |   00000000:68:00.0 Off |                    0 |
|  0%   28C    P8             31W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA A40                     On  |   00000000:6A:00.0 Off |                    0 |
|  0%   28C    P8             31W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA A40                     On  |   00000000:6B:00.0 Off |                    0 |
|  0%   28C    P8             31W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   4  NVIDIA A40                     On  |   00000000:E5:00.0 Off |                    0 |
|  0%   28C    P8             32W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   5  NVIDIA A40                     On  |   00000000:E6:00.0 Off |                    0 |
|  0%   28C    P8             31W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   6  NVIDIA A40                     On  |   00000000:E7:00.0 Off |                    0 |
|  0%   28C    P8             32W /  300W |       0MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   7  NVIDIA A40                     On  |   00000000:E8:00.0 Off |                    0 |
|  0%   28C    P8             30W /  300W |   22657MiB /  46068MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    7   N/A  N/A    586376    C+G   vgpu                                        22656MiB |
+-----------------------------------------------------------------------------------------+
```
# CloudPods操作


配置 /etc/yunion/host.conf

```bash
# 把所有的vgpu的PF全部写上
nvidia_vgpu_pfs:
- 67:00.0
- 67:00.4
- 67:00.5
- 67:00.6
- 67:00.7
- 67:01.0
- 67:01.1
- 67:01.2
- 67:01.3
- 67:01.4
- 67:01.5
- 67:01.6
- 67:01.7
- 67:02.0
- 67:02.1
- 67:02.2
- 67:02.3
- 67:02.4
- 67:02.5
- 67:02.6
- 67:02.7
- 67:03.0
- 67:03.1
- 67:03.2
- 67:03.3
- 67:03.4
- 67:03.5
- 67:03.6
- 67:03.7
...
```
> pf可以通过lspci | grep -i nvidia查出

重启
` kubectl -n onecloud rollout restart ds default-host`

cloudpods页面创建完虚机后，绑定vGPU设备

进入虚机后，安装guest驱动：

```bash
root@guest-vm:~/Guest_Drivers# ./NVIDIA-Linux-x86_64-550.163.01-grid.run 
Verifying archive integrity... OK
Uncompressing NVIDIA Accelerated Graphics Driver for Linux-x86_64 550.163.01...........................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
root@guest-vm:~/Guest_Drivers# nvidia-smi 
Tue Jun 17 12:05:31 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.163.01             Driver Version: 550.163.01     CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA A40-24A                 On  |   00000000:00:08.0 Off |                  N/A |
| N/A   N/A    P8             N/A /  N/A  |       1MiB /  24576MiB |      0%   Prohibited |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

## 验证虚机中的gpu是否可用


# 参考
1. H卡操作记录 http://vgpu.com.cn/DeploymentGuide/deployment-guide-vgpu-Ampere-GPU.pdf
2. CloudPods启用vGPU设置 https://www.cloudpods.org/docs/guides/onpremise/vminstance/passthrough/vgpu/
3. 支持MIG GPU卡列表： [支持MIG GPU卡列表](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
4. vGPU官方页面  https://docs.nvidia.com/vgpu/index.html#gpu-software-lifecycle  
5. nvidia驱动/license下载地址（授权）： https://ui.licensing.nvidia.com/software 