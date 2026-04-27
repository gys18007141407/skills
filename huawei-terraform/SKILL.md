---
name: huawei-terraform
description: 使用 Terraform 在华为云上创建和管理云资源。面向最终用户屏蔽 Terraform 细节，自动生成当前目录下的 Terraform 配置并执行标准工作流。
---

# Huawei Terraform Skill

## 目标

这个技能用于帮助用户通过 Terraform 管理华为云资源，并尽可能减少用户负担。

核心原则：

1. **所有 Terraform 配置文件必须输出到当前工作目录下**
   - 不要写到临时目录
   - 不要写到隐藏目录
   - 默认输出例如：
      - `main.tf`
      - `variables.tf`
      - `outputs.tf`
      - `terraform.tfvars`
   - 如果场景简单，也可以仅生成 `main.tf`

2. **用户不需要理解 Terraform 语法或实现细节**
   - 不向用户抛出 HCL、provider block、resource block 等概念，除非用户主动要求
   - 与用户沟通时，用“地域、实例规格、系统盘大小、带宽”这类云资源语言，而不是 Terraform 术语

3. **尽量减少用户交互**
   - 只有在缺少关键业务信息时才提问
   - 每次询问资源参数时，必须提供**明确默认值**（默认值需要用户同意使用）
   - 尽量一次性列出所需参数，避免多轮追问

4. **在真正执行 apply 之前，必须获得用户明确同意**
   - 即使底层命令要求使用 `terraform apply --auto-approve`
   - 也必须先向用户展示计划摘要，并请求确认
   - 用户未明确同意前，不得执行 apply

---

## Provider 基线

最小 provider 配置通常如下：

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

生成配置时，默认使用：

- provider: huaweicloud/huaweicloud
- 默认 region: cn-north-4

如果用户未指定地域，应使用默认值 cn-north-4，并在说明中告诉用户可以修改。

---

## 行为规范

1. **先理解用户目标，不先讲 Terraform**
   优先识别用户真正要做的事情，例如：

- 创建 ECS 云服务器
- 创建 VPC / 子网 / 安全组
- 创建 EVS 云硬盘
- 创建 RDS
- 创建负载均衡
- 为已有资源补 Terraform 配置

应以“我要帮你创建什么资源”的方式沟通，而不是“请告诉我 resource block 怎么写”。

2. **参数收集要带默认值**
   理解用户目标后，如果缺失必要参数，必须一次性询问，并给出默认值。

默认参数示例：
- 地域：cn-north-4
- 可用区：cn-north-4a
- ECS 规格：s6.small.1
- 系统盘类型：SAS
- 系统盘大小：40
- VPC 网段：192.168.0.0/16
- 子网网段：192.168.1.0/24
- 公网带宽：5

询问示例：

```
我将使用这些默认值进行操作，你可以进行修改
- 地域：cn-north-4
- 可用区：cn-north-4a
- ECS 规格：s6.small.1
- 系统盘：SAS 40GB
- VPC：192.168.0.0/16
- 子网：192.168.1.0/24
- 公网带宽：5 Mbps
是否采用默认值？
```

注意：
- 所有参数问题都要给默认值
- 尽量把问题合并成一次
- 不要求用户写 Terraform 代码

3. **文件必须写到当前工作目录**
   所有 Terraform 相关文件都必须生成在当前工作目录下。

默认文件策略：

- main.tf: 主资源定义
- variables.tf: 变量定义（可选）
- terraform.tfvars: 默认变量值（可选）
- outputs.tf: 输出定义（可选）

如果是最小场景，可只生成 main.tf。

不要：
- 写入 /tmp
- 写入随机目录
- 写入不可见工作目录
- 让用户手工复制代码

4. **Terraform 执行前检查**
   必须先检查本地是否安装 Terraform。

如果 Terraform 未安装：

- 检查系统架构
- 尝试自动安装 Terraform
- 根据当前系统环境选择合适的安装方式
- 安装后再次验证是否安装成功

自动安装失败时：
- 清楚告诉用户失败原因
- 提供可执行的安装建议
- 不要继续后续 Terraform 步骤

5. **标准执行流程**
   在 Terraform 配置文件生成完成后，按以下顺序执行：

terraform init -no-color
terraform validate -no-color
terraform plan -no-color
terraform apply --auto-approve

如果某一步执行失败，遵守以下额外约束：
- 先修复配置
- 再重新验证
- 不要带着错误进入 plan

执行 terraform plan -no-color 后应向用户总结计划结果，例如：
- 将创建哪些资源
- 预计涉及哪些关键参数
- 是否有删除/替换风险

不要把冗长计划原样全量抛给用户，优先做摘要；如用户要求，再提供完整输出。

重要：apply 前必须征得用户同意 。在真正执行前，必须先向用户明确询问是否继续。
示例：

Terraform 计划已生成。
根据计划，将创建以下资源：
- 1 个 VPC
- 1 个子网
- 1 台 ECS 云服务器
- 1 个安全组

未发现删除类操作。

如果你确认创建这些资源，我将继续执行 apply。

只有在用户明确同意后，才可以执行：

terraform apply --auto-approve

如果用户没有同意、表达犹豫、或提出修改要求：
- 不执行 apply
- 先调整配置，再重新 plan


6. **华为云凭证要求**
   如果 HW_ACCESS_KEY 或者 HW_SECRET_KEY 环境变量未设置，询问用户并保存到全局环境变量供terraform使用
   
   禁止输出 HW_SECRET_KEY。若因凭据问题导致鉴权不通过，询问用户并自动修改 HW_ACCESS_KEY 或者 HW_SECRET_KEY 环境变量（不要让用户手动操作）
   
---

## 用户交互风格

### 应该做的
- 用自然语言解释云资源用途
- 给出合理默认值
- 一次性收集尽量多的必要参数
- 将 Terraform 细节隐藏在背后
- 在关键风险点（尤其 apply）明确确认

### 不应该做的
- 不要要求用户自己写 HCL
- 不要让用户自己运行 Terraform 命令，除非用户明确要求
- 不要在未确认的情况下执行 apply
- 不要省略默认值
- 不要把配置写到当前目录以外的位置

---

## 默认生成策略

如果用户需求不完整，优先采用保守默认值并明确告知。常见默认值可参考：

- region: cn-north-4
- name prefix: demo
- vpc cidr: 192.168.0.0/16
- subnet cidr: 192.168.1.0/24
- ecs flavor: s6.small.1
- system disk type: SAS
- system disk size: 40
- bandwidth size: 5
- security group policy: 仅开放必要端口
- os image: Ubuntu 22.04

如果某资源需要用户提供不可猜测的信息（如密码、公钥、已有资源 ID），则集中一次询问，并给出示例格式。

## 输出要求

默认输出内容应包括：

1. 对用户目标的简要确认
2. 将生成哪些 Terraform 文件
3. 文件写入位置：当前工作目录
4. 使用的默认参数
5. Terraform 执行状态：
- Terraform 是否已安装
- init 结果
- validate 结果
- plan 摘要
6. apply 前确认提示
7. apply 执行结果（仅在用户同意后）

##  示例工作流
示例：用户说“帮我在华为云创建一台 ecs”
应按以下方式处理：

1. 理解目标：创建 ECS
2. 一次性询问必要参数，并带默认值，例如：
- 名称：demo
- 地域：cn-north-4
- 可用区：cn-north-4a
- 规格：s6.small.1
- 系统盘：SAS 40GB
- 操作系统：Ubuntu 22.04
- 公网：是
- 带宽：5 Mbps
3. 在当前工作目录生成 Terraform 文件
4. 检查 Terraform 是否安装；如未安装则自动安装
5. 执行：
- terraform init -no-color
- terraform validate -no-color
- terraform plan -no-color
6. 用自然语言总结计划
7. 请求用户确认
8. 仅在用户明确同意后执行：
- terraform apply --auto-approve

## 错误处理

重要：碰到错误时，应该尽力尝试多种方法来解决问题，而不是直接将问题抛给用户来解决

Terraform 未安装
- 先尝试自动安装
- 安装失败则停止，并告知用户下一步

Provider 初始化失败
- 检查网络、provider source、Terraform 版本兼容性
- 修复后重试 init

凭证缺失
- 明确告诉用户缺少华为云认证信息
- 引导用户提供所需认证方式
- 不泄露敏感信息，不在输出中回显密钥明文

validate / plan 失败
- 优先自动修复显而易见的问题
- 如仍失败，给出简洁原因和下一步建议

apply 失败
- 总结失败原因
- 告知已创建/未创建的资源状态
- 如有必要建议执行回滚或再次 plan
