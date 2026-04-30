---
name: huawei-hcloud-guide
description: 华为云 hcloud CLI 工具使用指南
---

# 华为云 hcloud 命令行工具使用指南

## 概述

本 Skill 提供使用华为云命令行工具 `hcloud` 进行云资源查询的实战经验，涵盖 **ECS 规格查询**、**可用区查询**、**VPC/子网/安全组查询**、**云硬盘类型查询**、**EIP 查询**、**IMS 镜像查询**、**价格查询**、**OBS 查询**、**已有资源查询** 等核心场景，以及大量踩坑经验。

## 环境要求

- 已安装 hcloud CLI（版本 7.2.2+）
- 已配置 AK/SK 认证（`hcloud configure set`）
- 已设置默认区域（如 `cn-north-4`）
- 如需调用 BSS 服务，需切换到 `cn-north-1` 区域

## 安装 hcloud

如果没安装 hcloud，使用以下地址下载压缩包后，解压可得到 hcloud 二进制文件，将 hcloud 路径配置到环境变量

### 1. 官方下载地址

所有平台的下载包均托管在华为云 OBS 上：

| 平台 | 架构 | 下载地址 | 包格式 |
|------|------|---------|--------|
| Windows | amd64 / x86_64 | `https://cn-north-4-hdn-koocli.obs.cn-north-4.myhuaweicloud.com/cli/latest/huaweicloud-cli-windows-amd64.zip` | .zip |
| Linux | amd64 / x86_64 | `https://cn-north-4-hdn-koocli.obs.cn-north-4.myhuaweicloud.com/cli/latest/huaweicloud-cli-linux-amd64.tar.gz` | .tar.gz |
| Linux | aarch64 / arm64 | `https://cn-north-4-hdn-koocli.obs.cn-north-4.myhuaweicloud.com/cli/latest/huaweicloud-cli-linux-arm64.tar.gz` | .tar.gz |
| macOS | amd64 / x86_64 | `https://cn-north-4-hdn-koocli.obs.cn-north-4.myhuaweicloud.com/cli/latest/huaweicloud-cli-mac-amd64.tar.gz` | .tar.gz |
| macOS | arm64 (Apple Silicon) | `https://cn-north-4-hdn-koocli.obs.cn-north-4.myhuaweicloud.com/cli/latest/huaweicloud-cli-mac-arm64.tar.gz` | .tar.gz |

### 2. 下载命令示例

**Windows (PowerShell)：**

```powershell
Invoke-WebRequest -Uri "https://cn-north-4-hdn-koocli.obs.cn-north-4.myhuaweicloud.com/cli/latest/huaweicloud-cli-windows-amd64.zip" -OutFile "hcloud.zip"
```

**Linux / macOS：**

```bash
# Linux amd64
curl -O https://cn-north-4-hdn-koocli.obs.cn-north-4.myhuaweicloud.com/cli/latest/huaweicloud-cli-linux-amd64.tar.gz

# macOS arm64
curl -O https://cn-north-4-hdn-koocli.obs.cn-north-4.myhuaweicloud.com/cli/latest/huaweicloud-cli-mac-arm64.tar.gz
```

### 3. 解压

**Windows (PowerShell)：**

```powershell
Expand-Archive -Path "hcloud.zip" -DestinationPath "hcloud_extracted" -Force
```

> ⚠️ **踩坑经验**：Windows 自带的 `Expand-Archive` 解压 zip 包后，`hcloud.exe` 通常在解压目录的根目录下，不在子目录中。如果解压后找不到 `hcloud.exe`，检查解压目录结构。

**Linux / macOS：**

```bash
tar -xzf huaweicloud-cli-linux-amd64.tar.gz
```

### 4. 安装路径

| 平台 | 推荐路径 |
|------|---------|
| Windows | `C:\hcloud\hcloud.exe` |
| Linux | `/usr/local/bin/hcloud` 或 `~/hcloud/hcloud` |
| macOS | `/usr/local/bin/hcloud` 或 `~/hcloud/hcloud` |

**Windows 示例：**

```powershell
# 创建目标目录
New-Item -ItemType Directory -Path "C:\hcloud" -Force

# 复制 hcloud.exe
Copy-Item -Path "hcloud_extracted\hcloud.exe" -Destination "C:\hcloud\hcloud.exe" -Force
```

**Linux / macOS 示例：**

```bash
chmod +x hcloud
sudo mv hcloud /usr/local/bin/hcloud
```

### 5. 配置 PATH 环境变量

**Windows (PowerShell, 永久生效)：**

```powershell
# 将 C:\hcloud 添加到系统 PATH（需要管理员权限）
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "Machine") + ";C:\hcloud", "Machine")
```

> 💡 **提示**：如果不想修改系统 PATH 或者没有权限操作，也可以在每次使用前临时添加：
> ```powershell
> $env:Path = "$env:Path;C:\hcloud"
> ```

**Linux / macOS：**

如果安装到 `/usr/local/bin/`，通常已在 PATH 中，无需额外配置。

如果安装到自定义路径（如 `~/hcloud/`），添加到 `~/.bashrc` 或 `~/.zshrc`：

```bash
echo 'export PATH="$HOME/hcloud:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## 通用技巧

### 1. 跳过证书验证（内网/测试环境）

```bash
hcloud <service> <operation> --cli-skip-secure-verify=true
```

### 2. 只返回指定字段

```bash
hcloud <service> <operation> --cli-query="flavors[].[id,name,vcpus,ram]"
```

### 3. 限制返回条数

```bash
hcloud <service> <operation> --limit=5
```

> ⚠️ **注意**：并非所有 API 都支持 `--limit` 参数。例如 `EVS CinderListVolumeTypes` 就不支持 `--limit`，会报错"不正确的参数:limit"。

### 4. 按条件过滤

```bash
# 精确匹配
hcloud <service> <operation> --<filter_field>="<value>"

# 模糊匹配（如按名称包含）
hcloud <service> <operation> --<filter_field>="*<keyword>*"
```

### 5. 查看当前配置

```bash
hcloud configure list
```

### 6. 查看版本

```bash
hcloud version
```

---

## 场景零：Project ID 查询（几乎所有操作的前置步骤）

> **重要**：很多 API 需要 `project_id`，跨区域查询时也必须指定目标区域的 `project_id`。

### 0.1 查询所有区域的 Project ID

```bash
hcloud IAM KeystoneListProjects
```

返回结果中，每个 project 的 `name` 就是区域名（如 `cn-north-4`），`id` 就是对应的 Project ID。

### 0.2 只提取区域和 Project ID 的对应关系

```bash
hcloud IAM KeystoneListProjects --cli-query="projects[].[name,id]"
```

### 实战经验

- **不是 `hcloud ECS ListProjects`**！ECS 服务没有 `ListProjects` 操作，正确的是 `IAM KeystoneListProjects`
- Project ID 与区域一一对应，每个区域有不同的 Project ID
- 当前默认配置的 Project ID 可通过 `hcloud configure list` 查看
- **跨区域查询时必须同时指定 `--cli-region` 和 `--cli-project-id`**，否则会报错 `Common.0013`

---

## 场景一：可用区查询

### 1.1 查询指定区域的可用区列表

```bash
hcloud ECS NovaListAvailabilityZones --cli-region=cn-north-4
```

### 1.2 跨区域查询可用区

```bash
# 必须同时指定 --cli-region 和 --cli-project-id
hcloud ECS NovaListAvailabilityZones --cli-region=cn-east-3 --cli-project-id=<cn-east-3的ProjectID>
```

### 1.3 查询云硬盘可用区

```bash
hcloud EVS CinderListAvailabilityZones
```

### 实战经验

- **不是 `ListAvailabilityZones`**！正确操作名是 `NovaListAvailabilityZones`
- 可用区命名规则：`<区域名>+字母`，如 `cn-north-4a`、`cn-north-4b`、`cn-north-4c`、`cn-north-4g`
- 不同区域可用区数量不同（如 cn-north-4 有 4 个可用区，cn-east-3 有 5 个）
- 查询 ECS 规格时 `availability_zone` 是必填参数，所以查询规格前必须先知道可用区

---

## 场景二：ECS 规格查询

### 2.1 查询所有可用规格

```bash
hcloud ECS ListFlavors --availability_zone="cn-north-4a"
```

### 2.2 按规格系列过滤（推荐方式）

```bash
# 查询通用入门型 s6 系列
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[?contains(id, 's6')].[id,name,vcpus,ram]"

# 查询通用计算型 c6 系列
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[?contains(id, 'c6')].[id,name,vcpus,ram]"

# 查询通用计算型 c7 系列
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[?contains(id, 'c7')].[id,name,vcpus,ram]"

# 查询内存优化型 m6 系列
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[?contains(id, 'm6')].[id,name,vcpus,ram]"

# 查询 GPU 规格 g6 系列
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[?contains(id, 'g6')].[id,name,vcpus,ram]"

# 查询磁盘增强型 d6 系列
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[?contains(id, 'd6')].[id,name,vcpus,ram]"
```

### 2.3 精确查询某个规格

```bash
# 用 --flavor_id 精确查询（推荐，返回数据量小）
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --flavor_id="s6.large.2"
```

### 2.4 查询所有规格系列列表

```bash
# 提取所有不重复的规格系列前缀
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[].id"
# 然后从结果中提取系列前缀（id 的第一个 . 之前的部分）
```

### 2.5 查询规格总数

```bash
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="length(flavors)"
```

### 实战经验

- `ListFlavors` 的 `availability_zone` 参数是**必填**的，不填会报错
- `vcpus` 和 `ram` 在返回中是字符串类型，查询时需用引号包裹
- `ram` 的单位是 **MB**（如 4096 = 4GB）
- `disk` 字段通常为 `"0"`，表示不附带本地盘
- 规格 ID 命名规则：`<系列>.<规格大小>.<内存倍率>`，如 `s6.small.1`、`s6.large.2`
  - 内存倍率：`1` = 1GB/vCPU，`2` = 2GB/vCPU，`4` = 4GB/vCPU
- **`--flavor_id` 是最高效的查询方式**，避免返回大量数据
- 规格总数可能超过 600 个，**务必使用 `--cli-query` 或 `--flavor_id` 过滤**，不要直接返回全量数据

### ECS 规格系列速查表

| 系列前缀 | 类型 | 说明 |
|---------|------|------|
| s6 | 通用入门型 | 性价比最高，适合轻量应用 |
| t6/t7 | 通用基础型 | 基础性能 |
| c6/c6ne/c6s/c7/c9 | 通用计算型 | 计算密集型应用 |
| m6/m7/m9 | 内存优化型 | 大内存需求 |
| d6/d7/d3/d9i/d7i | 磁盘增强型 | 本地盘大容量 |
| e3/e7 | 大内存型 | 超大内存 |
| g5/g6/g6ne/g6v | GPU 加速型 | GPU 计算/AI |
| p2s/p2v/p2vs | GPU 加速型 | Pi 系列 GPU |
| i3/i7/ir3/ir7 | 高性能计算型 | 高性能存储 |
| pi1/pi2 | 性能型 | 高性能网络 |
| ac7/ac8/ac9 | 鲲鹏计算型 | ARM 架构 |
| kc1/kc2 | 鲲鹏通用计算型 | ARM 架构 |
| x1/x1e/x2e | 超大内存型 | 超大内存 SAP HANA |

---

## 场景三：IMS 镜像查询

### 3.1 查询公共镜像（限制条数）

```bash
hcloud IMS ListImages --__imagetype="gold" --limit=5
```

### 3.2 按操作系统类型和平台过滤

```bash
# 查询所有 Linux 公共镜像
hcloud IMS ListImages --__imagetype="gold" --__os_type="Linux" --limit=10

# 查询 Ubuntu 公共镜像
hcloud IMS ListImages --__imagetype="gold" --__os_type="Linux" --__platform="Ubuntu" --limit=10

# 查询 CentOS 公共镜像
hcloud IMS ListImages --__imagetype="gold" --__os_type="Linux" --__platform="CentOS" --limit=10

# 查询 EulerOS 公共镜像
hcloud IMS ListImages --__imagetype="gold" --__os_type="Linux" --__platform="EulerOS" --limit=10

# 查询 Windows 公共镜像
hcloud IMS ListImages --__imagetype="gold" --__os_type="Windows" --limit=10

# 查询 Huawei Cloud EulerOS 公共镜像
hcloud IMS ListImages --__imagetype="gold" --__os_type="Linux" --__platform="Huawei Cloud EulerOS" --limit=10
```

### 3.3 查询私有镜像

```bash
hcloud IMS ListImages --__imagetype="private"
```

### 3.4 按名称模糊搜索

```bash
# 搜索名称包含 "CentOS" 的镜像
hcloud IMS ListImages --__imagetype="gold" --name="*CentOS*"
```

### 3.5 查询镜像详情

```bash
# 先获取镜像 ID 和关键信息
hcloud IMS ListImages --__imagetype="gold" --__os_type="Linux" --__platform="Ubuntu" --limit=5 --cli-query="images[].[id,name,__os_version,min_disk]"

# 再查询指定镜像详情
hcloud IMS ShowImage --image_id="<镜像ID>"
```

### 实战经验

- **参数名是 `__imagetype`（双下划线），不是 `imagetype`**！用 `--imagetype` 会报错"不正确的参数"
- `__imagetype` 取值：`gold` = 公共镜像，`private` = 私有镜像，`shared` = 共享镜像，`market` = 市场镜像
- `__platform` 是**双下划线**前缀，用于按操作系统平台精确过滤
- `__os_type` 也是双下划线，取值：`Linux`、`Windows`、`Other`
- `name` 参数支持通配符 `*` 进行模糊匹配
- 公共镜像列表非常长，**务必使用 `--limit` + `--__os_type` + `--__platform` 组合过滤**
- 镜像返回字段中 `min_disk` 表示最小系统盘要求（GB），`__os_version` 是完整版本号
- `__platform` 可选值：`Windows`、`Ubuntu`、`RedHat`、`SUSE`、`CentOS`、`Debian`、`OpenSUSE`、`Oracle Linux`、`Fedora`、`Other`、`CoreOS`、`EulerOS`、`Huawei Cloud EulerOS`

---

## 场景四：VPC / 子网 / 安全组查询

### 4.1 查询 VPC 列表

```bash
hcloud VPC ListVpcs/v3
```

### 4.2 查询指定 VPC 下的子网

```bash
hcloud VPC ListSubnets --vpc_id="<VPC_ID>"
```

### 4.3 查询安全组列表

```bash
hcloud VPC ListSecurityGroups/v3
```

### 4.4 查询安全组规则

```bash
hcloud VPC ListSecurityGroupRules/v3 --limit=50
```

### 4.5 查询路由表

```bash
hcloud VPC ListRouteTables --vpc_id="<VPC_ID>"
```

### 4.6 查询 NAT 网关

```bash
hcloud NAT ListNatGateways
```

### 4.7 查询网络 ACL（防火墙）

```bash
hcloud VPC ListFirewall
```

### 实战经验

- VPC 查询推荐用 `ListVpcs/v3`（v3 版本返回信息更完整，包含 cloud_resources 等）
- 安全组查询推荐用 `ListSecurityGroups/v3`
- `ListSubnets` 需要 `--vpc_id` 参数，不能直接列出所有子网
- 安全组规则查询 `ListSecurityGroupRules/v3` **不接受 `--security_group_id` 参数**，只能通过 `--limit` 限制后用 `--cli-query` 过滤
- VPC 返回的 `cloud_resources` 字段显示该 VPC 下关联的资源类型和数量

---

## 场景五：云硬盘（EVS）类型查询

### 5.1 查询所有云硬盘类型

```bash
hcloud EVS CinderListVolumeTypes
```

> ⚠️ **注意**：`CinderListVolumeTypes` **不支持 `--limit` 参数**，会报错"不正确的参数:limit"。直接执行即可返回所有类型。

### 5.2 查询已有云硬盘列表

```bash
hcloud EVS ListVolumes
```

### 实战经验

- 云硬盘类型及说明：

| 类型名 | 说明 | 典型场景 |
|--------|------|---------|
| ESSD | 超高 IO 云硬盘 | 数据库、高性能计算 |
| ESSD2 | 超高 IO 云硬盘（二代） | 更高性能需求 |
| GPSSD | 通用型 SSD 云硬盘 | 通用业务 |
| GPSSD2 | 通用型 SSD 云硬盘（二代） | 更高通用性能 |
| SSD | 超高 IO 云硬盘（旧版） | 兼容旧规格 |
| SAS | 高 IO 云硬盘 | 中等性能需求 |
| SATA | 普通 IO 云硬盘 | 归档、日志 |

- 返回的 `extra_specs.RESKEY:availability_zones` 字段表示该磁盘类型**支持的可用区**
- 返回的 `os-vendor-extended:sold_out_availability_zones` 字段表示该类型**已售罄的可用区**（重要！创建前需检查）
- **ESSD2 在部分可用区可能已售罄**，创建前务必检查 `sold_out_availability_zones`
- SATA 类型在部分可用区也可能已售罄

---

## 场景六：EIP（弹性公网 IP）查询

### 6.1 查询已有 EIP 列表

```bash
hcloud EIP ListPublicips/v3
```

### 6.2 查询带宽列表

```bash
hcloud EIP ListBandwidths
```

### 6.3 查询 EIP 公共池信息

```bash
hcloud EIP ListCommonPools
```

### 实战经验

- EIP 查询推荐用 `ListPublicips/v3`（v3 版本信息更完整）
- `ListCommonPools` 返回可用的 EIP 类型（如 `bgp`、`sbgp`），创建 EIP 时需要指定类型
- EIP 类型：`5_bgp`（全动态 BGP）、`5_sbgp`（静态 BGP）

---

## 场景七：已有 ECS 实例查询（terraform 做不到的功能）

### 7.1 查询所有 ECS 实例详情

```bash
hcloud ECS ListServersDetails
```

### 7.2 查询 ECS 实例的磁盘挂载信息

```bash
hcloud ECS ListServerVolumeAttachments --server_id="<ECS实例ID>"
```

### 7.3 查询 ECS 实例的安全组

```bash
hcloud ECS NovaListServerSecurityGroups --server_id="<ECS实例ID>"
```

### 7.4 查询 ECS 实例的网络接口

```bash
hcloud ECS ListServerInterfaces --server_id="<ECS实例ID>"
```

### 7.5 查询密钥对列表

```bash
hcloud ECS NovaListKeypairs
```

### 7.6 查询服务器组

```bash
hcloud ECS ListServerGroups
```

### 7.7 查询可变配规格列表

```bash
# 查询某个实例可以变配到哪些规格
hcloud ECS ListResizeFlavors --instance_uuid="<ECS实例ID>"
```

### 实战经验

- `ListServersDetails` 返回所有实例的完整信息，包括规格、状态、IP 等
- 查询实例的磁盘/安全组/网络接口等需要先获取 `server_id`
- `ListResizeFlavors` 可以查询某个实例支持变配的目标规格列表，这在 terraform 中无法做到

---

## 场景八：价格查询（BSS 服务）

### 8.1 包月价格查询

```bash
# 切换到 cn-north-1 区域（BSS 服务所在区域）
hcloud BSS ListResourceRatings --cli-region=cn-north-1 --project_id="<PROJECT_ID>" --product_infos.1.cloud_service_type="hws.service.type.ec2" --product_infos.1.resource_type="hws.resource.type.vm" --product_infos.1.resource_spec="s6.small.1.linux" --product_infos.1.region="cn-north-4" --product_infos.1.period_type=2 --product_infos.1.period_num=1 --product_infos.1.subscription_num=1 --product_infos.1.id="1"
```

### 8.2 按需（小时）价格查询

```bash
hcloud BSS ListOnDemandResourceRatings --cli-region=cn-north-1 --project_id="<PROJECT_ID>" --product_infos.1.cloud_service_type="hws.service.type.ec2" --product_infos.1.resource_type="hws.resource.type.vm" --product_infos.1.resource_spec="s6.small.1.linux" --product_infos.1.region="cn-north-4" --product_infos.1.usage_factor="Duration" --product_infos.1.usage_measure_id=4 --product_infos.1.usage_value=1 --product_infos.1.subscription_num=1 --product_infos.1.id="1"
```

### 8.3 查询镜像价格

```bash
# 镜像资源类型为 hws.resource.type.vm.image
hcloud BSS ListResourceRatings --cli-region=cn-north-1 --project_id="<PROJECT_ID>" --product_infos.1.cloud_service_type="hws.service.type.ec2" --product_infos.1.resource_type="hws.resource.type.vm.image" --product_infos.1.resource_spec="<镜像规格>" --product_infos.1.region="cn-north-4" --product_infos.1.period_type=2 --product_infos.1.period_num=1 --product_infos.1.subscription_num=1 --product_infos.1.id="1"
```

### 8.4 查询云硬盘价格

```bash
# 云硬盘资源类型为 hws.resource.type.volume
hcloud BSS ListOnDemandResourceRatings --cli-region=cn-north-1 --project_id="<PROJECT_ID>" --product_infos.1.cloud_service_type="hws.service.type.ec2" --product_infos.1.resource_type="hws.resource.type.volume" --product_infos.1.resource_spec="GPSSD" --product_infos.1.region="cn-north-4" --product_infos.1.usage_factor="Duration" --product_infos.1.usage_measure_id=4 --product_infos.1.usage_value=1 --product_infos.1.subscription_num=1 --product_infos.1.id="1"
```

### 实战经验

- **BSS 服务必须在 `cn-north-1` 区域调用。不要修改用户的区域配置，仅仅在命令行参数指定 --cli-region=cn-north-1 临时查询即可**，即使资源在其他区域
- `project_id` 通过 `hcloud IAM KeystoneListProjects` 获取（不是 `ECS ListProjects`）
- 云服务类型编码：ECS = `hws.service.type.ec2`
- 资源类型编码：
  - 云主机 = `hws.resource.type.vm`
  - 云主机镜像 = `hws.resource.type.vm.image`
  - 云硬盘 = `hws.resource.type.volume`
- 规格命名规则：`<规格ID>.<操作系统>`，如 `s6.small.1.linux`、`s6.small.1.windows`
- `period_type`：2 = 包月，3 = 包年
- `usage_measure_id`：4 = 小时
- 返回的 `amount` 单位为元（CNY）

---

## 场景九：OBS 查询

### 9.1 列出所有 OBS 桶

```bash
hcloud OBS ls
```

### 9.2 列出桶内对象

```bash
hcloud OBS ls obs://<桶名> -limit=10
```

### 实战经验

- **OBS 不复用 hcloud 的 AK/SK 配置！** 需要单独配置
- OBS 配置方式：
  ```bash
  # 方式一：交互式配置（推荐）
  hcloud OBS config -interactive

  # 方式二：命令行配置
  hcloud OBS config -i=<AK> -k=<SK> -e=obs.cn-north-4.myhuaweicloud.com
  ```
- OBS endpoint 格式：`obs.<区域>.myhuaweicloud.com`，如 `obs.cn-north-4.myhuaweicloud.com`
- 配置文件路径：`~/.obsutilconfig`（Windows: `C:\Users\<用户>\.obsutilconfig`）
- 如果未配置 AK/SK，执行 `hcloud OBS ls` 会提示 "Please set ak, sk and endpoint in the configuration file!"
- OBS 的命令风格与其他服务不同，使用 `ls`、`cp`、`rm` 等类 Unix 命令，而非 API 操作名
- 可尝试从环境变量中读取 **Terraform** 使用的环境变量：HW_ACCESS_KEY、HW_SECRET_KEY
---

## 场景十：ELB（弹性负载均衡）查询

### 10.1 查询负载均衡实例列表

```bash
hcloud ELB ListLoadBalancers/v3
```

### 10.2 查询监听器列表

```bash
hcloud ELB ListListeners/v3
```

### 实战经验

- ELB 查询推荐用 v3 版本 API
- `ListLoadBalancers/v3` 返回负载均衡实例列表及其关联的 VPC、子网等信息

---

## 跨区域查询指南

> **这是最容易踩坑的地方！**

### 核心规则

跨区域查询时，**必须同时指定 `--cli-region` 和 `--cli-project-id`**，缺一不可。

### 错误示例

```bash
# 只指定 --cli-region，不指定 --cli-project-id → 报错！
# 错误信息：the current region is [cn-east-3] and does not match with the project name [cn-north-4] in tk
# 错误码：Common.0013
hcloud ECS NovaListAvailabilityZones --cli-region=cn-east-3
```

### 正确做法

```bash
# 第一步：先查询目标区域的 Project ID
hcloud IAM KeystoneListProjects --cli-query="projects[?name=='cn-east-3'].id | [0]"

# 第二步：用 Project ID 查询目标区域
hcloud ECS NovaListAvailabilityZones --cli-region=cn-east-3 --cli-project-id=<cn-east-3的ProjectID>
```

### 实战经验

- hcloud 默认配置中 `projectId` 只对应一个区域，切换区域时 token 中的 project 信息不匹配
- **BSS 服务固定在 `cn-north-1` 区域**，查询价格时也需要指定 `cn-north-1` 的 Project ID
- 建议先通过 `KeystoneListProjects` 一次性获取所有区域的 Project ID 映射，缓存备用

---

## JMESPath 查询踩坑指南

> **hcloud 的 `--cli-query` 使用 JMESPath 语法，但有多个坑！**

### 坑 1：`&&` 组合条件查询会失败

```bash
# 错误！&& 会导致 JMESPath 查询失败，返回原始 JSON
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[?vcpus=='2' && ram=='4096'].[id,name,vcpus,ram]"

# 正确做法：用 API 参数 + 单条件 cli-query 组合
# 先用 --flavor_id 精确查询，或用单条件过滤后再人工筛选
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --flavor_id="s6.large.2"
```

### 坑 2：无法查询含冒号的嵌套字段

```bash
# 错误！os_extra_specs 中的字段名含冒号（如 ecs:performancetype），JMESPath 无法解析
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --cli-query="flavors[0].os_extra_specs.ecs:performancetype"

# 正确做法：先获取完整 JSON，再用其他工具（如 python/jq）提取嵌套字段
hcloud ECS ListFlavors --availability_zone="cn-north-4a" --flavor_id="s6.large.2" > flavor.json
# 然后用 python 提取
python -c "import json; d=json.load(open('flavor.json')); print(d['flavors'][0]['os_extra_specs'])"
```

### 坑 3：`ram` 字段类型不一致

- 在 `ListFlavors` 返回中，`ram` 是**整数**（如 4096）
- 在 `NovaListFlavorsDetails` 返回中，`ram` 也是整数
- 但 `vcpus` 是**字符串**（如 "2"）
- **过滤条件中 `vcpus` 需要用引号，`ram` 不需要引号**（但为安全起见，建议都加引号试试）

### 坑 4：查询结果为空时不会报错

- 如果 `--cli-query` 过滤条件匹配不到任何结果，hcloud 会返回空数组 `[]`，不会报错
- 建议先不加 `--cli-query` 用 `--limit=1` 确认数据存在，再加过滤条件

---

## 常见问题

### Q1: 提示 "Authentication failed"

**原因**：AK/SK 未配置或已过期
**解决**：运行 `hcloud configure set` 重新配置

### Q2: 提示 "Region not supported"

**原因**：当前区域不支持该服务
**解决**：使用 `--cli-region` 切换到正确的区域（如 BSS 需切换到 `cn-north-1`）

### Q3: 提示 "certificate verify failed"

**原因**：内网环境证书问题
**解决**：添加 `--cli-skip-secure-verify=true` 参数

### Q4: 查询结果为空

**原因**：过滤条件过于严格或参数名错误
**解决**：先不加过滤条件查询，确认数据存在后再逐步添加过滤

### Q5: 不知道某个服务有哪些操作

**解决**：运行 `hcloud <服务名> --help` 查看所有可用操作

### Q6: 跨区域查询报错 Common.0013

**原因**：只指定了 `--cli-region` 但未指定 `--cli-project-id`
**解决**：必须同时指定 `--cli-region` 和 `--cli-project-id`，Project ID 通过 `hcloud IAM KeystoneListProjects` 获取

### Q7: 提示 "不支持的operation:ListAvailabilityZones"

**原因**：ECS 服务的可用区查询操作名不是 `ListAvailabilityZones`
**解决**：使用 `NovaListAvailabilityZones`

### Q8: 提示 "不支持的operation:ListProjects"

**原因**：ECS 服务没有 `ListProjects` 操作
**解决**：使用 `hcloud IAM KeystoneListProjects`

### Q9: 提示 "不正确的参数:imagetype"

**原因**：IMS 镜像查询的参数名是 `__imagetype`（双下划线），不是 `imagetype`
**解决**：使用 `--__imagetype="gold"`

### Q10: OBS 提示 "Please set ak, sk and endpoint"

**原因**：OBS 不复用 hcloud 的 AK/SK 配置，需要单独配置
**解决**：运行 `hcloud OBS config -interactive` 配置 OBS 的 AK/SK 和 endpoint

### Q11: JMESPath 查询失败，返回原始 JSON

**原因**：`--cli-query` 中使用了 `&&` 组合条件或含冒号的字段名
**解决**：避免使用 `&&`，改用 API 参数过滤 + 单条件 cli-query；嵌套字段用外部工具提取

### Q12: 返回数据量太大

**原因**：ECS 规格有 600+ 个，公共镜像有数百个
**解决**：
- 规格查询：用 `--flavor_id` 精确查询，或用 `--cli-query` + `contains(id, '系列名')` 过滤
- 镜像查询：用 `--limit` + `--__os_type` + `--__platform` 组合过滤
- 通用：先用 `--limit=1` 确认接口可用，再加过滤条件

---

## 常用服务操作速查

| 服务 | 常用查询操作 | 说明 |
|------|-------------|------|
| IAM | `KeystoneListProjects` | 查询所有区域的 Project ID |
| ECS | `NovaListAvailabilityZones` | 查询可用区 |
| ECS | `ListFlavors` | 查询 ECS 规格（需指定 availability_zone） |
| ECS | `ListServersDetails` | 查询已有 ECS 实例 |
| ECS | `NovaListKeypairs` | 查询密钥对 |
| ECS | `ListResizeFlavors` | 查询可变配规格 |
| IMS | `ListImages` | 查询镜像（注意参数名双下划线） |
| IMS | `ShowImage` | 查询镜像详情 |
| VPC | `ListVpcs/v3` | 查询 VPC |
| VPC | `ListSubnets` | 查询子网（需 vpc_id） |
| VPC | `ListSecurityGroups/v3` | 查询安全组 |
| VPC | `ListSecurityGroupRules/v3` | 查询安全组规则 |
| VPC | `ListRouteTables` | 查询路由表 |
| VPC | `ListFirewall` | 查询网络 ACL |
| EVS | `CinderListVolumeTypes` | 查询云硬盘类型 |
| EVS | `CinderListAvailabilityZones` | 查询云硬盘可用区 |
| EVS | `ListVolumes` | 查询已有云硬盘 |
| EIP | `ListPublicips/v3` | 查询弹性公网 IP |
| EIP | `ListBandwidths` | 查询带宽 |
| EIP | `ListCommonPools` | 查询 EIP 公共池 |
| NAT | `ListNatGateways` | 查询 NAT 网关 |
| ELB | `ListLoadBalancers/v3` | 查询负载均衡 |
| BSS | `ListResourceRatings` | 查询包年包月价格 |
| BSS | `ListOnDemandResourceRatings` | 查询按需价格 |
| OBS | `ls` | 列出桶/对象（需单独配置 AK/SK） |

---

## 参考信息

- 当前环境：hcloud v7.2.2，默认区域 cn-north-4
- 示例价格（cn-north-4 区域）：
  - s6.small.1 Linux：包月 32.2 元，按需 0.07 元/小时
- cn-north-4 可用区：cn-north-4a、cn-north-4b、cn-north-4c、cn-north-4g
- Project ID 可通过 `hcloud IAM KeystoneListProjects` 查询
