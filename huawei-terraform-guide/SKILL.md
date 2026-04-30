---
name: huawei-terraform-guide
description: 华为云 Terraform 部署指南，涵盖 VPC/子网/ECS/EIP/OBS 等资源的 Terraform 配置、数据源查询、常见踩坑及最佳实践
---

# 华为云 Terraform 部署指南

## 概述

本 Skill 提供使用 Terraform 部署华为云资源的实战经验，涵盖 **Provider 配置**、**VPC/子网**、**ECS 实例**、**EIP**、**安全组**、**OBS 桶** 等核心资源，以及大量踩坑经验，帮助减少 Terraform 部署的试错次数。
注意：禁止 terraform apply！除非用户显示指定 apply。

## 环境要求

- 已安装 Terraform（v1.5+）
- 已配置华为云认证环境变量（`HW_ACCESS_KEY` / `HW_SECRET_KEY` / `HW_REGION_NAME`）
- 或使用 Provider 中的 `access_key` / `secret_key` / `region` 参数

## 认证方式

### 方式一：环境变量（推荐）

```powershell
# PowerShell 设置环境变量
$env:HW_ACCESS_KEY="你的AK"
$env:HW_SECRET_KEY="你的SK"
$env:HW_REGION_NAME="cn-north-4"
```

### 方式二：Provider 参数

```hcl
provider "huaweicloud" {
  region     = "cn-north-4"
  access_key = "你的AK"
  secret_key = "你的SK"
}
```

### 实战经验

- **推荐使用环境变量方式**，避免 AK/SK 明文写在代码中
- 环境变量名是 `HW_ACCESS_KEY` / `HW_SECRET_KEY` / `HW_REGION_NAME`（注意前缀是 `HW_` 不是 `HWCLOUD_`）
- 如果同时配置了环境变量和 Provider 参数，Provider 参数优先级更高

---

## Provider 配置

### 基础配置

```hcl
terraform {
  required_providers {
    huaweicloud = {
      source = "huaweicloud/huaweicloud"
    }
  }
}

provider "huaweicloud" {
  region = "cn-north-4"
}
```

### 实战经验

- Provider source 必须是 `huaweicloud/huaweicloud`，不是 `hashicorp/huaweicloud`
- 首次运行需执行 `terraform init` 下载 Provider
- 如果网络环境受限，可配置 Provider 镜像源

---

## 数据源查询

### 可用区查询

```hcl
data "huaweicloud_availability_zones" "this" {}

output "az_names" {
  value = data.huaweicloud_availability_zones.this.names
}
```

### ECS 规格查询

```hcl
data "huaweicloud_compute_flavors" "this" {
  availability_zone = "cn-north-4a"
  performance_type  = "computingv3"
  cpu_core_count    = 2
  memory_size       = 4
}

output "flavor_ids" {
  value = data.huaweicloud_compute_flavors.this.ids
}
```

### 镜像查询

```hcl
data "huaweicloud_images_image" "this" {
  name        = "Ubuntu 22.04 server 64bit"
  most_recent = true
}

output "image_id" {
  value = data.huaweicloud_images_image.this.id
}
```

### 实战经验

- `huaweicloud_availability_zones` 返回的 `names` 是可用区名称列表
- `huaweicloud_compute_flavors` 的 `availability_zone` 是**必填参数**，不填会报错
- `performance_type` 可选值：`computingv3`（通用计算型）、`memory`（内存优化型）等
- `huaweicloud_compute_flavors` 返回的 `ids` 是匹配规格的 ID 列表，取 `ids[0]` 获取第一个
- `huaweicloud_images_image` 的 `name` 支持模糊匹配，`most_recent = true` 取最新版本
- 镜像名称示例：`Ubuntu 22.04 server 64bit`、`CentOS 7.6 64bit`、`Huawei Cloud EulerOS 2.0 64bit`

---

## VPC 和子网

### 完整配置

```hcl
# VPC
resource "huaweicloud_vpc" "this" {
  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# 子网
resource "huaweicloud_vpc_subnet" "this" {
  name       = "my-subnet"
  cidr       = "10.0.1.0/24"
  gateway_ip = "10.0.1.1"
  vpc_id     = huaweicloud_vpc.this.id
}
```

### 实战经验

- VPC 的 `cidr` 支持常见的私有网段：`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`
- 子网的 `gateway_ip` 必须是该子网 CIDR 范围内的第一个可用 IP（通常是 `x.x.x.1`）
- 子网的 `cidr` 必须在 VPC 的 `cidr` 范围内
- 子网创建后 `id` 会自动生成，通过 `huaweicloud_vpc_subnet.this.id` 引用
- 删除资源时，如果子网下有资源（如 ECS），会删除失败，需先删除依赖资源

---

## 安全组

```hcl
resource "huaweicloud_networking_secgroup" "this" {
  name        = "my-sg"
  description = "My security group"
}
```

### 实战经验

- 安全组资源类型是 `huaweicloud_networking_secgroup`（注意是 `networking` 不是 `vpc`）
- 安全组名称在同一个区域下必须唯一
- ECS 引用安全组时用的是安全组的 `name` 而不是 `id`：

```hcl
resource "huaweicloud_compute_instance" "this" {
  security_groups = [huaweicloud_networking_secgroup.this.name]
}
```

---

## EIP（弹性公网 IP）

```hcl
resource "huaweicloud_vpc_eip" "this" {
  publicip {
    type = "5_bgp"
  }
  bandwidth {
    name        = "my-bandwidth"
    size        = 1
    share_type  = "PER"
    charge_mode = "traffic"
  }
}
```

### 实战经验

- `publicip.type` 取值：`5_bgp`（全动态 BGP）、`5_sbgp`（静态 BGP）
- `bandwidth.share_type` 取值：`PER`（独享带宽）、`WHOLE`（共享带宽）
- `bandwidth.charge_mode` 取值：`traffic`（按流量计费）、`bandwidth`（按带宽计费）
- `bandwidth.size` 单位是 Mbit/s
- EIP 创建后会产生费用，测试完成后记得 `terraform destroy` 清理
- ECS 关联 EIP 有两种方式：
  - 方式一：在 ECS 资源中指定 `eip_id = huaweicloud_vpc_eip.this.id`
  - 方式二：先创建 ECS 再单独关联 EIP

---

## ECS 实例

### 完整配置

```hcl
resource "huaweicloud_compute_instance" "this" {
  name              = "my-ecs"
  availability_zone = "cn-north-4a"
  flavor_id         = data.huaweicloud_compute_flavors.this.ids[0]
  image_id          = data.huaweicloud_images_image.this.id

  system_disk_type = "SAS"
  system_disk_size = 40

  network {
    uuid = huaweicloud_vpc_subnet.this.id
  }

  security_groups = [huaweicloud_networking_secgroup.this.name]

  eip_id = huaweicloud_vpc_eip.this.id
}
```

### 实战经验

- `availability_zone` 必须与规格查询时使用的可用区一致
- `flavor_id` 通过数据源 `huaweicloud_compute_flavors` 查询获取
- `image_id` 通过数据源 `huaweicloud_images_image` 查询获取
- `system_disk_type` 可选值：`SAS`（高 IO）、`SSD`（超高 IO）、`GPSSD`（通用型 SSD）、`ESSD`（极速型 SSD）
- `system_disk_size` 单位是 GB，最小 40GB（取决于镜像要求）
- `network.uuid` 是子网的 ID
- `security_groups` 是安全组的**名称列表**（不是 ID 列表）
- 如果不需要 EIP，可以省略 `eip_id` 参数
- ECS 创建后可通过 `access_ip_v4` 获取内网 IP，`public_ip` 获取公网 IP

### 输出示例

```hcl
output "ecs_private_ip" {
  value = huaweicloud_compute_instance.this.access_ip_v4
}

output "ecs_public_ip" {
  value = huaweicloud_compute_instance.this.public_ip
}
```

---

## OBS 桶

### 完整配置

```hcl
# OBS 桶
resource "huaweicloud_obs_bucket" "this" {
  bucket        = "my-test-bucket-20260430"
  storage_class = "STANDARD"
  acl           = "private"
}
```

### 参数说明

| 参数 | 说明 | 必填 | 示例值 |
|------|------|------|--------|
| `bucket` | 桶名称（全局唯一） | 是 | `my-test-bucket-20260430` |
| `storage_class` | 存储类别 | 否（默认 STANDARD） | `STANDARD`、`WARM`、`COLD` |
| `acl` | 访问控制策略 | 否（默认 private） | `private`、`public-read`、`public-read-write` |
| `region` | 区域（默认使用 Provider 的 region） | 否 | `cn-north-4` |

### 实战经验

- **桶名称必须全局唯一**，建议在名称后加日期或随机后缀，如 `my-bucket-20260430`
- 桶名称只能包含小写字母、数字和连字符（`-`），不能以连字符开头或结尾
- 桶名称长度限制：3~63 个字符
- `storage_class` 取值：
  - `STANDARD`：标准存储
  - `WARM`：低频访问存储
  - `COLD`：归档存储
- `acl` 取值：
  - `private`：私有读写（默认）
  - `public-read`：公共读
  - `public-read-write`：公共读写
- 如果桶名称已被占用，`terraform apply` 会报错 `BucketAlreadyExists` 或 `BucketAlreadyOwnedByYou`
- 删除桶时，如果桶内有对象（非空桶），会删除失败，需先清空桶内对象
- OBS 桶的 Terraform Provider 是 `huaweicloud/huaweicloud`，不需要额外安装其他 Provider

---

## 常用 Terraform 命令

```powershell
# 初始化（下载 Provider）
terraform init

# 格式化代码
terraform fmt

# 验证配置
terraform validate

# 查看执行计划
terraform plan

# 应用配置（创建资源）
terraform apply

# 自动确认应用（无需交互）
terraform apply -auto-approve

# 销毁资源
terraform destroy

# 自动确认销毁
terraform destroy -auto-approve

# 查看状态
terraform show

# 查看资源列表
terraform state list
```

---

## 常见踩坑与解决方案

### 坑 1：`terraform init` 失败

**现象**：无法下载 `huaweicloud/huaweicloud` Provider

**原因**：网络问题或 Terraform 镜像源配置问题

**解决**：
```powershell
# 检查网络连接
terraform init

# 如果使用代理，设置环境变量
$env:HTTP_PROXY="http://代理地址:端口"
$env:HTTPS_PROXY="http://代理地址:端口"
```

### 坑 2：认证失败

**现象**：`terraform plan` 或 `apply` 时报认证错误

**原因**：AK/SK 未配置或配置错误

**解决**：
```powershell
# 检查环境变量是否已设置
$env:HW_ACCESS_KEY
$env:HW_SECRET_KEY
$env:HW_REGION_NAME

# 如果未设置，重新设置
$env:HW_ACCESS_KEY="你的AK"
$env:HW_SECRET_KEY="你的SK"
$env:HW_REGION_NAME="cn-north-4"
```

### 坑 3：ECS 规格查询不到结果

**现象**：`huaweicloud_compute_flavors` 数据源返回空列表

**原因**：`availability_zone` 或 `performance_type` 参数不匹配

**解决**：
- 确认可用区名称正确（如 `cn-north-4a`）
- 尝试不指定 `performance_type`，只按 `cpu_core_count` 和 `memory_size` 过滤
- 先用 `hcloud` CLI 查询确认该可用区下存在该规格

### 坑 4：镜像名称不匹配

**现象**：`huaweicloud_images_image` 数据源找不到镜像

**原因**：镜像名称不准确或该区域没有该镜像

**解决**：
- 先用 `hcloud IMS ListImages --__imagetype="gold" --__os_type="Linux" --limit=10` 查询准确的镜像名称
- 镜像名称可能包含版本号差异，如 `Ubuntu 22.04 server 64bit` vs `Ubuntu 22.04.3 LTS 64bit`

### 坑 5：安全组引用方式错误

**现象**：ECS 创建失败，提示安全组不存在

**原因**：`security_groups` 参数传入了安全组 ID 而不是名称

**解决**：
```hcl
# 正确：使用安全组名称
security_groups = [huaweicloud_networking_secgroup.this.name]

# 错误：使用安全组 ID
# security_groups = [huaweicloud_networking_secgroup.this.id]
```

### 坑 6：子网 gateway_ip 错误

**现象**：子网创建失败，提示 gateway_ip 不在 CIDR 范围内

**原因**：`gateway_ip` 不是该子网 CIDR 的第一个可用 IP

**解决**：
- 对于 CIDR `10.0.1.0/24`，gateway_ip 应为 `10.0.1.1`
- 对于 CIDR `192.168.1.0/24`，gateway_ip 应为 `192.168.1.1`

### 坑 7：资源删除失败（依赖关系）

**现象**：`terraform destroy` 时报错，无法删除资源

**原因**：资源之间存在依赖关系，Terraform 未能正确处理

**解决**：
- 检查是否有其他资源依赖了要删除的资源
- 手动删除依赖资源后再执行 `terraform destroy`
- 或使用 `terraform state rm` 移除状态后再手动删除

### 坑 8：桶名称已被占用

**现象**：OBS 桶创建失败，提示 `BucketAlreadyExists`

**原因**：OBS 桶名称全局唯一，已被其他用户占用

**解决**：
- 在桶名称后加随机后缀，如 `my-bucket-20260430-123456`
- 使用更独特的命名前缀

### 坑 9：`terraform plan` 和 `apply` 结果不一致

**现象**：`plan` 显示正常，但 `apply` 时报错

**原因**：plan 是基于当前状态的快照，apply 时状态可能已变化

**解决**：
- 重新执行 `terraform plan` 后再 `apply`
- 检查是否有其他人同时操作了同一资源

### 坑 10：Windows PowerShell 中的路径问题

**现象**：Terraform 命令执行失败，提示路径错误

**原因**：PowerShell 中路径包含空格或特殊字符

**解决**：
```powershell
# 切换到工作目录
cd d:\test\terraform-obs

# 执行命令
terraform init
terraform plan
```

---

## 完整示例：VPC + 子网 + ECS + EIP

```hcl
terraform {
  required_providers {
    huaweicloud = {
      source = "huaweicloud/huaweicloud"
    }
  }
}

provider "huaweicloud" {
  region = "cn-north-4"
}

# 数据源：可用区
data "huaweicloud_availability_zones" "this" {}

# 数据源：规格（2vCPU 4GB）
data "huaweicloud_compute_flavors" "this" {
  availability_zone = "cn-north-4a"
  performance_type  = "computingv3"
  cpu_core_count    = 2
  memory_size       = 4
}

# 数据源：镜像
data "huaweicloud_images_image" "this" {
  name        = "Ubuntu 22.04 server 64bit"
  most_recent = true
}

# VPC
resource "huaweicloud_vpc" "this" {
  name = "terraform-ecs-test-vpc"
  cidr = "10.0.0.0/16"
}

# 子网
resource "huaweicloud_vpc_subnet" "this" {
  name       = "terraform-ecs-test-subnet"
  cidr       = "10.0.1.0/24"
  gateway_ip = "10.0.1.1"
  vpc_id     = huaweicloud_vpc.this.id
}

# 安全组
resource "huaweicloud_networking_secgroup" "this" {
  name        = "terraform-ecs-test-sg"
  description = "Terraform test security group"
}

# 弹性公网IP
resource "huaweicloud_vpc_eip" "this" {
  publicip {
    type = "5_bgp"
  }
  bandwidth {
    name        = "terraform-ecs-test-bandwidth"
    size        = 1
    share_type  = "PER"
    charge_mode = "traffic"
  }
}

# ECS 实例
resource "huaweicloud_compute_instance" "this" {
  name              = "terraform-ecs-test"
  availability_zone = "cn-north-4a"
  flavor_id         = data.huaweicloud_compute_flavors.this.ids[0]
  image_id          = data.huaweicloud_images_image.this.id

  system_disk_type = "SAS"
  system_disk_size = 40

  network {
    uuid = huaweicloud_vpc_subnet.this.id
  }

  security_groups = [huaweicloud_networking_secgroup.this.name]

  eip_id = huaweicloud_vpc_eip.this.id
}

output "ecs_private_ip" {
  value = huaweicloud_compute_instance.this.access_ip_v4
}

output "ecs_public_ip" {
  value = huaweicloud_compute_instance.this.public_ip
}
```

---

## 完整示例：OBS 桶

```hcl
terraform {
  required_providers {
    huaweicloud = {
      source = "huaweicloud/huaweicloud"
    }
  }
}

provider "huaweicloud" {
  region = "cn-north-4"
}

# OBS 桶
resource "huaweicloud_obs_bucket" "this" {
  bucket        = "my-test-bucket-20260430"
  storage_class = "STANDARD"
  acl           = "private"
}

output "bucket_name" {
  value = huaweicloud_obs_bucket.this.bucket
}

output "bucket_endpoint" {
  value = huaweicloud_obs_bucket.this.bucket_domain_name
}
```

---

## 资源清理

```powershell
# 销毁所有资源
terraform destroy -auto-approve

# 如果销毁失败，检查是否有残留资源
terraform state list
```

### 实战经验

- EIP 即使未绑定 ECS 也会产生少量费用
- OBS 桶即使为空也会产生少量存储费用（极小）
- ECS 实例停止状态仍会产生磁盘费用

---

## 参考信息

- 当前环境：Terraform v1.5+，华为云 Provider `huaweicloud/huaweicloud`
- 默认区域：`cn-north-4`
- 可用区：`cn-north-4a`、`cn-north-4b`、`cn-north-4c`、`cn-north-4g`
- 认证方式：环境变量 `HW_ACCESS_KEY` / `HW_SECRET_KEY` / `HW_REGION_NAME`
