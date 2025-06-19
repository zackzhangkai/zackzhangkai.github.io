---
layout: post
published: true
title: 2025-06-17-NVIDIA MIG 方案调研
categories: [大模型]
tags: [GPU,vGPU]
---
* content
{:toc}


# 背景


# 启用MIG功能

1. `nvidia-smi -i 0 -mig 1`  # 对GPU0启用MIG

2. 查询启用状态

```bash
# nvidia-smi -i 0 -q | grep -i mig -C 3
...

MIG Mode
Current : Enabled
Pending : Enabled
```

3. 查询可以创建的配置的profile

```bash
# nvidia-smi mig -lgip  # 如果这个命令失败多敲几次

+-----------------------------------------------------------------------------+

| GPU instance profiles: |

| GPU Name ID Instances Memory P2P SM DEC ENC |

|=============================================================================|

| 0 MIG 1g.12gb 19 2/7 11.75 No 8 1 0 |

+-----------------------------------------------------------------------------+

| 0 MIG 1g.12gb+me 20 1/1 11.75 No 8 1 0 |

```

# 创建MIG 实例

这个是GPU 0 的配置，ID这列是这个配置1g.12gb的Profile ID，显示可以创建的还有2个实例

4. `nvidia-smi mig -i 0 -cgi 19,1g.12gb,1 -C`  # 在GPU 0上创建1个1g.q12gb实例。注意：如果不加-C,只是创建了GPU实例，还需要加-C创建出计算实例

```bash
# nvidia-smi mig -i 0 -cgi 20,1g.12gb,1 -C

Successfully created GPU instance ID 13 on GPU 0 using profile MIG 1g.12gb (ID 19)

Successfully created GPU instance ID 14 on GPU 0 using profile MIG 1g.12gb (ID 19)

Unable to create a GPU instance on GPU 0 using profile 1: Invalid Argument

Failed to create GPU instances: Invalid Argument

```

5. 验证已经创建好的GI和CI

```bash
# nvidia-smi

...

| MIG devices: |

------------------------------+-----------+-----------------------+

| GPU GI CI MIG | Memory-Usage | Vol| Shared |

|==================+================================+===========+

| 0 11 0 0 | 12MiB / 12032MiB | 8 0 | 1

+------------------+--------------------------------+-----------+-----

| 0 12 0 1 | 12MiB / 12032MiB | 8 0 | 1

```

6. 在两个不同的 GPU 实例上并行运行两个 CUDA 应用程序

```

# nvidia-smi -L

GPU 0: NVIDIA H20 (UUID: GPU-408b6f36-2b41-4a0c-4d68-412db19cb808)

MIG 1g.12gb Device 0: (UUID: MIG-1b24da31-838c-5294-b2b5-22ae3e5f0676)

MIG 1g.12gb Device 1: (UUID: MIG-5a50dd59-de47-534e-b52d-eed94e7eddf9)

```

7. 验证cuda是否可用：`python3 -c "import torch; print(f'CUDA是否可用: {torch.cuda.is_available()}')"`，说明：H20必须要安装NVIDIA驱动和fabric驱动，使用NVLINK才可以

```bash
# nvidia-smi

Wed Jun 11 17:19:55 2025

+---------------------------------------------------------------------------------------+

| NVIDIA-SMI 535.247.01 Driver Version: 535.247.01 CUDA Version: 12.2 |

# systemctl status nvidia-fabricmanager

● nvidia-fabricmanager.service - NVIDIA fabric manager service

Loaded: loaded (/lib/systemd/system/nvidia-fabricmanager.service; enabled; vendor preset: enabled)

Active: active (running) since Wed 2025-06-11 17:15:51 CST; 4min 26s ago

Process: 20280 ExecStart=/usr/bin/nv-fabricmanager -c /usr/share/nvidia/nvswitch/fabricmanager.cfg (code=exited, status=0/SUCCESS)

Main PID: 20282 (nv-fabricmanage)

```

# 指定MIG GPU uuid进行训练 (未实现成功) 

```

import torch

def list_gpu_uuids():

"""列出所有可用GPU及其UUID"""

if not torch.cuda.is_available():

print("没有可用的GPU")

return None

gpu_count = torch.cuda.device_count()

uuids = {}

print(f"发现 {gpu_count} 个GPU:")

for i in range(gpu_count):

uuid = torch.cuda.get_device_properties(i).uuid

name = torch.cuda.get_device_properties(i).name

uuids[uuid] = i

print(f"GPU {i}: {name}, UUID: {uuid}")

return uuids

def select_gpu_by_uuid(target_uuid, uuids=None):

"""根据UUID选择GPU"""

if uuids is None:

uuids = list_gpu_uuids()

if uuids is None:

return False

if target_uuid in uuids:

device_idx = uuids[target_uuid]

torch.cuda.set_device(device_idx)

print(f"已选择GPU {device_idx} (UUID: {target_uuid})")

return True

else:

print(f"未找到UUID为 {target_uuid} 的GPU")

return False

def run_sample_computation():

"""在当前选定的GPU上运行简单的计算任务"""

if not torch.cuda.is_available():

print("无法运行计算任务: 没有可用的GPU")

return

device = torch.device("cuda")

a = torch.randn(1000, 1000, device=device)

b = torch.randn(1000, 1000, device=device)

# 执行矩阵乘法

c = a @ b

# 计算平均值并返回结果

result = c.mean().item()

print(f"计算结果: {result}")

return result

if __name__ == "__main__":

# 步骤1: 列出所有GPU及其UUID

gpu_uuids = list_gpu_uuids()

if gpu_uuids:

# 步骤2: 选择一个GPU (示例中选择第一个GPU的UUID)

# 实际使用时，您可以替换为您想要的特定UUID

first_uuid = next(iter(gpu_uuids.keys()))

select_gpu_by_uuid(first_uuid)

# 步骤3: 在选定的GPU上运行计算任务

run_sample_computation()

```

8. 销毁 ci和gi

```

nvidia-smi mig -dci && nvidia-smi mig -dgi

```

9. 也可以只删除gi下的ci

```

nvidia-smi mig -dci -ci 0,1,2 -gi 1

```

10. `nvidia-smi -i 0 -mig 0` 关装MIG功能