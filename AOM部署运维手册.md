# HCS AOM 2.6.0 部署运维手册

**适用对象**：交付工程师、运维工程师  
**插件版本**：AOM-Deploy-Plugin-8.6.1.00202604231446  
**AOM 软件版本**：2.6.0（HCS 8.6.1 / HCF 1.0.0）  
**部署方式**：HCCI/FIC 自动化部署（非手工逐条执行脚本）

---

## 目录

1. [概述](#1-概述)
2. [前置条件](#2-前置条件)
3. [配置文件与参数修改位置](#3-配置文件与参数修改位置)
4. [部署操作与重试命令](#4-部署操作与重试命令)
5. [日志路径说明](#5-日志路径说明)
6. [部署后验证](#6-部署后验证)
7. [常见报错与解决方法](#7-常见报错与解决方法)
8. [日常运维命令速查](#8-日常运维命令速查)
9. [交付检查清单](#9-交付检查清单)
10. [附录](#10-附录)

---

## 1. 概述

本手册基于 `AOM-Deploy-Plugin` 插件包编写。AOM 部署由 **HCCI 平台**按 `params/manifest.json` 编排约 18 个任务组、100+ 子步骤自动完成，涵盖：

- 网络/VPC、VM 创建、中间件安装
- CDK 微服务部署、ManageOne 注册
- Console 安装、国密切换（可选）

**重要说明**：交付人员**通常不需要**也**不建议**直接修改插件内 Python 脚本；参数在 **HCCI 工程界面 + LLD** 中配置，部署在 **HCCI 控制台**发起。

### 1.1 插件目录结构（速查）

| 目录/文件 | 作用 |
|-----------|------|
| `plugin.json` | 插件注册（场景：ProjectDeploy / ExpansionCloudService） |
| `params/manifest.json` | 任务编排 DAG、依赖关系 |
| `params/params.cfg` | 参数定义与 LLD IP 映射 |
| `params/param_details.cfg` | 人工填写参数说明 |
| `params/inspect.json` | 软件包校验规则 |
| `params/report_cn.ini` | 错误码中文对照 |
| `params/resource_calc.py` | VM 资源自动计算 |
| `conf/aom/*-input.json` | 微服务部署默认参数（运行时注入） |
| `scripts/` | 各子任务 Python 实现 |
| `tools/` | Shell/YAML 辅助脚本、卸载脚本 |

---

## 2. 前置条件

### 2.1 平台与环境

| 项 | 要求 |
|----|------|
| HCS 方案版本 | 8.6.1 |
| HCF 方案版本 | 1.0.0（如适用） |
| 部署框架 | HCCI/FIC v1 |
| 部署场景 | `ProjectDeploy`（工程部署）或 `ExpansionCloudService`（云服务扩容） |
| 架构 | x86_64 或 ARM（aarch64），工程中选择后插件自动适配 |

### 2.2 必须先完成的云服务（组件依赖）

根据 `cloud_service_conf/AOM/project_conf/ProjectDeploy/comp_check.json`：

| 依赖组件 | 说明 |
|----------|------|
| **AOM-Console** | 控制台（与 AOM 联动勾选） |
| **RegionLB** | 区域负载均衡 |
| **PODLB** | POD 负载均衡 |
| **ALB** | 应用负载均衡 |
| **PaaS-Common** | PaaS 公共组件 |

AOM-Console  additionally 依赖：**CloudConsoleFramework**

### 2.3 manifest 任务级前置依赖

以下云服务任务必须在 AOM 之前成功：

```
AOS → Install_SWR → PaaS-Common → PublicConfig → CACRegionConfig
ManageOne → CDKInstall
install_OBSv3（OBS 对象存储）
CloudLB → CloudLBPost（系统账户注册）
APIGateway（CloudServiceDockingCloudLB）
SDK → Nginx → ConsoleFrameworkConfigurationAfterInstallation（Console 场景）
```

### 2.4 软件包清单

上传至 HCCI 软件仓库，包名需匹配 `params/inspect.json`：

| 场景 | 软件包 | 用途 |
|------|--------|------|
| 全部 | `AOM-Deploy-Plugin-8.6.1.*.zip` | 部署插件 |
| AOM-Console | `ROMA-Factory-AOM-LTS-(*)-All_64_1.zip` | Console 静态资源 |
| Type1&AOM | `ROMA-Factory-AOM-LTS-(*)-All_64_2.zip` ~ `_4.zip` | AOM/LTS 主软件包 |
| Type1&AOM | `resource_10UnifiedAccess2MO_AOM_811R.tar.gz` | MO 适配包（AOM） |
| Type1&AOM | `resource_10UnifiedAccess2MO_LTS_8(*).tar.gz` | MO 适配包（LTS） |
| Max/ExtraMax | `resource_10UnifiedAccess2MO_LTS_POD(*).tar.gz` | LTS POD 区 MO 适配包 |

### 2.5 资源规划

- **LLD IP 规划**：按模板 `IP&VLAN-POD_Service_OM_AOM` 规划全部节点 IP
- **VM 资源**：插件 `params/resource_calc.py` 按规格和是否 HA 自动计算
- **磁盘**：大规格需额外 LTS VPC + POD 区 VM，磁盘配置见 `tools/apm-nodes-disk-mount-config-*.yaml`
- **时间预估**：完整部署约 **8～15 小时**（含解压上传约 2.5h）

### 2.6 规格选型对照

| 工程选项 | 场景 Tag | 特点 |
|----------|----------|------|
| 小规格 | AOMMIN | Base 区部署 ALS 微服务 |
| 中规格 | AOMMedium | 资源略高于 MIN |
| 大规格 | AOMMax | LTS POD 下沉，需独立 VPC |
| 超大 | AOMExtraMax | 更高日志吞吐 |
| 超大 II | AOMExtraMaxII | 最高规格 |
| 专线 | AOMSupportPrivateLine | 需 Nginx VM + ELB |
| 国密 | SMCompatible | 额外 1～2h 国密切换 |
| 容灾 | RegionConHA | VM 数量翻倍 |

---

## 3. 配置文件与参数修改位置

> **交付原则**：参数优先在 **HCCI 工程界面** 和 **LLD Excel** 中填写；插件内文件仅作参考，一般不在现场改插件包。

### 3.1 HCCI 工程界面（主要修改入口）

以下参数定义见 `params/param_details.cfg`，在 HCCI **工程参数**页面填写：

| 参数名 | 适用场景 | 说明 | 示例 |
|--------|----------|------|------|
| **feature_switches** | 全部 AOM | JSON 开关 | `{"CCE":true,"drap":false,"lightweight":false}` |
| **subnet_cidr** | 专线/大规格 | AOM 子网网段 | `71.24.61.64/26` |
| **external_network_name** | 专线 | EIP 外部网络名 | `eip` |
| **day_log** | Max/ExtraMax | 日志量 GB/天 | MIN:1500 / MAX:7500 / MAX-II:15000 |
| **storage_days** | Max/ExtraMax | 日志保留天数 7～365 | 默认 `7` |
| **als_query_data_size** | MIN/Medium | 日志磁盘 GB | MIN:3000 / Medium:5000 |

**feature_switches 填写规则**：

- 已部署 CCE 或与 AOM 同工程部署 CCE → `"CCE": true`
- ServiceStage 含 drap 组件 → `"drap": true`
- manage-aggr 主机数 < 6（轻量化）→ `"lightweight": true`

**storage_days 校验规则**（`params_check.py` 未强制，需人工遵守）：

- 仅支持 7～365 天
- `day_log < 1500` 时，`storage_days` 不能大于 7

### 3.2 LLD 模板（IP 与网络）

节点 IP 在 LLD 中规划，插件通过 `params/params.cfg` 中 `type = lld_ip` 项自动映射，例如：

| LLD 参数名 | 节点 |
|------------|------|
| `zk_01_ip` ~ `zk_03_ip` | ZooKeeper |
| `kafka_01_ip` ~ `kafka_03_ip` | Kafka |
| `etcd_01_ip` ~ `etcd_03_ip` | Etcd |
| `redis_01_ip` ~ `redis_03_ip` | Redis |
| `cassandra_*` / `access_*` / `base_*` / `es_*` | CDK 集群各角色 |
| `als_es_*` | ALS ES 节点 |

网络平面默认：`external_om`  
管理 AZ 默认：`manage-az`（容灾：`dr-manage-az`）

### 3.3 插件内配置文件（参考，一般不现场改）

| 路径 | 用途 | 是否建议修改 |
|------|------|--------------|
| `plugin.json` | 插件注册 | 否 |
| `params/manifest.json` | 任务编排 DAG | 否 |
| `params/params.cfg` | 参数定义与 LLD 映射规则 | 否（由平台计算） |
| `params/param_details.cfg` | 参数填写说明 | 参考 |
| `params/inspect.json` | 软件包校验规则 | 否 |
| `params/resource_calc.py` | VM 资源计算公式 | 否 |
| `conf/aom/*-input.json` | 微服务部署默认参数 | 否（部署时脚本动态注入） |
| `conf/aom_roles.json` | IAM 角色定义 | 否 |
| `conf/api_gateway_*.yaml` | APIG 对接配置 | 否 |
| `conf/static/*.yml` | Console Static 配置 | 否 |
| `tools/apm-nodes-disk-mount-config-*.yaml` | 各规格磁盘挂载 | 参考 |

### 3.4 部署后运行时配置位置

| 位置 | 内容 |
|------|------|
| CDK 集群 `apm` 命名空间 | 微服务 Deployment/ConfigMap/Secret |
| 中间件 VM `/root/script` | 中间件安装脚本与变量 |
| ManageOne CMDB | AOM 三层服务树 |
| IAM | 内置账户 `op_svc_apm`、`op_svc_lts`、`apm_admin` 等 |

---

## 4. 部署操作与重试命令

AOM **没有**单独的“启动命令”，全流程由 HCCI 调度。

### 4.1 部署前准备

1. 确认 HCCI 节点可访问，插件包 `AOM-Deploy-Plugin-*.zip` 已上传
2. 确认 ROMA 软件包 1～4 及 MO 适配包齐全
3. 在工程中勾选 **AOM** + **AOM-Console**（及所需选装组件）
4. 选择规格、是否专线、是否国密、是否 HA
5. 填写工程参数（见 3.1）
6. 导入/核对 LLD IP 规划
7. 确认前置云服务任务全部成功

### 4.2 发起部署（HCCI 控制台）

在 HCCI **工程部署**界面：

1. 选择 Region/POD
2. 执行 **AOM** 相关任务组（平台按 manifest 依赖自动排序）
3. 观察任务进度，失败任务查看日志后重试

**任务组顺序概览**：

```
1.  lts_create_vpc（大规格）
2.  aom_before_install（解压/上传/IAM/OBS）
3.  lts_create_cluster（大规格）
4.  aom_create_vms（9 类 VM + ICAgent）
5.  aom_create_private_line（专线）
6.  als_pod_vms（大规格 POD 区 VM）
7.  aom_install_middleware（Kafka/Etcd/Redis）
8.  lts_pod_install_middleware
9.  aom_register_csl_manageone（NS/Secret/CMDB）
10. lts_pod_register_csl_manageone
11. aom_deploy_microservices（20+ 微服务）
12. lts_pod_deploy_microservices
13. aom_after_install（APIG/DNS/LB/AppCode）
14. aom_mo_cmdb_register
15. aom_register_mo_pkg_and_cert
16. aom_console / lts_console
17. aom_sm_change / lts_sm_change（国密）
18. clean_temp_file_and_reinforce_vms
```

### 4.3 失败任务重试（推荐）

在 **HCCI 部署节点**，用 **root** 执行开发提供的标准重试命令：

```bash
/opt/rootscripts/debug-tools/retry_step.sh execute -i
```

| 参数 | 含义 |
|------|------|
| `execute` | 执行/重试模式（对应底层 `--m execute`） |
| `-i` | 交互模式，按提示输入工程 ID、Region、插件类等 |

**标准重试流程**：

1. 在 HCCI 界面记录：任务 ID（如 `aom_install_kafka`）、错误码（如 `910171`）、工程 ID、Region ID
2. 执行 `retry_step.sh execute -i`，按提示填写
3. 插件类全路径从 `params/manifest.json` 的 `script` 字段获取，例如：

```text
plugins.AOM.scripts.install_middleware.Sub_Job_InstallMiddleWare.InstallKafka
plugins.AOM.scripts.install_services.Sub_Job_Install_AOM.InstallAOMAmsmetric
```

**其他常用命令**：

```bash
# 查看帮助
/opt/rootscripts/debug-tools/retry_step.sh -h

# 回滚某步（慎用，需变更审批）
/opt/rootscripts/debug-tools/retry_step.sh rollback -i
```

若 `/opt/rootscripts/debug-tools/` 不存在，可检查平台默认路径：

```bash
ls -l /opt/cloud/hcci/debug-tools/
```

**底层直调（深度排障，非正常交付流程）**：

```bash
docker exec -it <container_id> bash -c \
  "su hcci -s /bin/bash -c \"export PYTHONPATH=\${PYTHONPATH}:/opt/cloud/hcci/src/HCCI && \
   /opt/python/bin/python3 /opt/cloud/hcci/src/HCCI/common/step_executor.py \
   --pjd <工程ID> --pod <POD_ID> --plp <插件类全路径> --rg <regionId> --srv None --rid None --m execute\""
```

### 4.4 卸载命令（紧急回退）

插件提供 `tools/uninstall_aom.sh`，在 **HCCI 节点 root** 执行：

```bash
cd /path/to/AOM/tools
dos2unix uninstall_aom.sh
bash uninstall_aom.sh <HCCI工程ID> <regionId> >> uninstall_aom.log 2>&1
```

**卸载前务必确认影响范围**（会依次卸载微服务、VM、OBS 桶、CMDB 等）。

### 4.5 重试注意事项

1. **先修根因再重试**（IP 冲突、软件包缺失、OBS/CDK 未就绪等）
2. **只重试失败步骤**，不要从头跑整个工程
3. **`rollback -i` 慎用**，可能删除已创建资源
4. 重试前后在 HCCI 任务日志中搜索 `[AOM]`、`910xxx`、`traceback`

---

## 5. 日志路径说明

排查问题时需区分 **部署日志** 与 **运行日志**，二者位置与用途不同。

### 5.1 日志分类总览

| 类型 | 位置 | 适用场景 |
|------|------|----------|
| **HCCI 部署日志** | HCCI 任务界面 → 失败任务 → 查看日志 | 安装失败、910xxx 错误、脚本 traceback |
| **Pod 标准输出** | `kubectl logs <pod> -n apm` | 容器启动失败、CrashLoop、镜像拉取问题 |
| **Pod 业务日志** | `/var/paas/sys/log/<服务名>/log` | 部署成功后功能异常、接口报错、国密切换 |
| **中间件 VM 日志** | VM 内脚本输出、`/tmp/*.log` | Kafka/Redis/Etcd 安装失败 |
| **磁盘挂载日志** | VM 内 `/tmp/create_vol.log` | 磁盘挂载/分区失败（910032/910035） |

### 5.2 HCCI 部署日志

- **查看方式**：HCCI 部署界面 → 点击失败任务 → **日志/Log**
- **搜索关键词**：`[AOM]`、`failed`、`traceback`、`HCCIException`、`910xxx`、`find pkg`
- **任务映射**：在 `params/manifest.json` 搜索任务 `id`，找到 `script` 字段对应 Python 脚本

### 5.3 Pod 内微服务业务日志（重点）

HCS AOM 微服务部署在 CDK 集群 **`apm`** 命名空间。Pod 内业务日志通用规则：

```text
/var/paas/sys/log/<微服务名>/log
```

**已确认示例**：

| 微服务 | Pod 内日志路径 |
|--------|----------------|
| aom-billing | `/var/paas/sys/log/aom-billing/log` |
| configs | `/var/paas/sys/log/configs` |
| ingester | `/var/paas/sys/log/ingester` |
| compactor | `/var/paas/sys/log/compactor` |
| querier | `/var/paas/sys/log/querier` |
| query-frontend | `/var/paas/sys/log/query-frontend` |
| ruler | `/var/paas/sys/log/ruler` |
| distributor | `/var/paas/sys/log/distributor` |
| store-gateway | `/var/paas/sys/log/store-gateway` |

**查看命令（在 CDK Master 上执行）**：

```bash
# 查找 Pod
kubectl get pod -n apm | grep aom-billing

# 列出日志目录
kubectl exec -it <pod名> -n apm -- ls -la /var/paas/sys/log/aom-billing/log

# 实时跟踪（按实际文件名调整）
kubectl exec -it <pod名> -n apm -- tail -f /var/paas/sys/log/aom-billing/log/*.log

# 标准输出（启动/崩溃类）
kubectl logs <pod名> -n apm --tail=200
kubectl logs <pod名> -n apm --previous    # Pod 重启后看上一次
```

多副本时每个 Pod 各有一份日志，需分别查看：

```bash
kubectl get pod -n apm -l app=aom-billing -o wide
```

### 5.4 ALS/AMS 类服务日志（双路径）

部分 ALS/AMS 微服务同时配置了 Pod 内路径与 `/opt/cloud/logs/` 路径（见 `conf/aom/*-input.json`）：

| 微服务 | 配置路径（Pod/容器内） |
|--------|------------------------|
| als-access | `/opt/cloud/logs/als-access/log` |
| als-config | `/opt/cloud/logs/als-config` |
| als-analysis | `/opt/cloud/logs/als-analysis` |
| als-query | `/opt/cloud/logs/als-query` |
| als-log2metric | `/opt/cloud/logs/als-log2metric` |
| ams-alarm | `/opt/cloud/logs/amsalarm` |

若 `/var/paas/sys/log/<服务名>/log` 不存在，可尝试：

```bash
kubectl exec -it <pod名> -n apm -- ls -la /opt/cloud/logs/
```

Cortex 类服务（ingester/compactor 等）可能同时在宿主机挂载 `/opt/cloud/logs/<服务名>`，Pod 内仍以 `/var/paas/sys/log/<服务名>` 为主。

### 5.5 中间件 VM 日志

中间件安装在独立 VM 上，非 Pod 内：

| 组件 | 排查方式 |
|------|----------|
| Kafka | `ssh root@<kafka_ip>` → `ps -ef \| grep kafka` → 查 9093 端口 |
| Redis | `netstat -tlnp \| grep 6800` |
| Etcd | `netstat -tlnp \| grep 2379` |
| 磁盘挂载 | VM 上 `/tmp/create_vol.log` |
| 安装脚本 | VM 上 `/root/script` 目录 |

辅助脚本位于插件 `tools/`：

```text
create_kafka_topic.sh
create_aom_kafka_topic.sh
open_redis_eval.sh
main.sh / create_vol.sh
```

### 5.6 按问题类型选日志

| 问题现象 | 优先查看 |
|----------|----------|
| HCCI 任务失败（910xxx） | HCCI 部署任务日志 |
| Pod Pending/CrashLoop | `kubectl describe pod` + `kubectl logs` |
| 部署成功但计费/License 异常 | `/var/paas/sys/log/aom-billing/log` |
| 国密 billing 切换失败 | HCCI `aom_sm_change` 任务日志 + aom-billing Pod 日志 |
| Kafka Topic 创建失败 | HCCI 日志 + Kafka VM |
| APIG/DNS 注册失败 | HCCI `aom_after_install` 任务日志 |
| 微服务 910171 | HCCI 日志 → `kubectl logs` → Pod 内 `/var/paas/sys/log/` |

---

## 6. 部署后验证

### 6.1 HCCI 平台

- 全部 AOM 任务组状态为 **成功**
- 无 910xxx 错误码残留

### 6.2 CDK 集群（登录 CDK Master）

```bash
kubectl get pod -n apm
kubectl get deploy -n apm
kubectl get svc -n apm

# 关键微服务应 Running
kubectl get pod -n apm | grep -E "ams-access|ams-metric|dpa-cassandra|als-config|aom-billing"
```

### 6.3 中间件 VM

```bash
# Kafka 示例
ssh root@<kafka_ip>
netstat -tlnp | grep 9093

# Redis 示例
ssh root@<redis_ip>
netstat -tlnp | grep 6800
```

### 6.4 业务验证

| 验证项 | 方法 |
|--------|------|
| Console 访问 | 浏览器打开 AOM/LTS Console 域名 |
| APIG 注册 | 检查 APIG 上 AOM API 是否注册 |
| DNS | `console_domain` 解析是否正确 |
| MO 注册 | ManageOne 云服务列表可见 AOM |
| IAM 账户 | `op_svc_apm` / `op_svc_lts` 可正常鉴权 |

### 6.5 内置账户（部署自动创建）

| 账户 | 类型 | 用途 |
|------|------|------|
| op_svc_apm | 租户 | AOM 服务账户 |
| op_svc_lts | 租户 | LTS 服务账户 |
| apm_admin | 租户 | APM 管理 |
| op_svc_subapm | 人机 | AOM 子账户 |
| op_svc_sublts | 人机 | LTS 子账户 |

密码由平台统一管理；若报 **910273**，说明 op 账户密码与平台记录不一致。

---

## 7. 常见报错与解决方法

完整错误码见 `params/report_cn.ini`。以下为高频问题速查。

### 7.1 安装前阶段

| 错误码 | 含义 | 排查与处理 |
|--------|------|------------|
| **910161** | 上传软件包失败 | 检查 ROMA 包是否齐全、HCCI 磁盘空间、SWR 是否可用 |
| **910021** | 创建 CMDB 服务树失败 | 确认 ManageOne/CMDB 可达；前置 CDKInstall 已成功 |
| **910252** | 创建 IAM 账户失败 | 检查 ManageOne IAM；前置 ManageOne 任务成功 |
| **910132** | 创建 OBS 桶失败 | 确认 `install_OBSv3` 已完成 |
| **910273** | op 账户密码错误 | 在 MO 核对 `op_svc_apm`/`op_svc_lts` 密码 |

**软件包找不到**（日志含 `find pkg`）：核对包名是否匹配 `inspect.json`；x86/ARM 后缀分别为 `-x86_64.zip` / `-aarch64.zip`。

### 7.2 VM 与网络阶段

| 错误码 | 含义 | 排查与处理 |
|--------|------|------------|
| **910031 / 910285** | 创建 VM 失败 | Service OM 查 VM 事件；核对 IP 冲突、规格不足 |
| **910032** | 磁盘挂载失败 | 对照 `tools/apm-nodes-disk-mount-config-<规格>.yaml` |
| **910035** | 磁盘分区失败 | SSH 到 VM；查 `/tmp/create_vol.log` |
| **910272** | VM 初始化失败 | 查 yum 源（910034）、网络连通性 |
| **910279** | 创建 VPC/子网失败 | 核对 `subnet_cidr`、LLD 网段 |
| **910286** | 创建 LTS 集群失败 | 查 LTS Master VM 与 CDK 纳管（910275） |
| **910287** | 创建专线/ELB 失败 | 核对 `external_network_name`、Nginx VM |

### 7.3 中间件阶段

| 错误码 | 含义 | 排查与处理 |
|--------|------|------------|
| **910051** | Kafka 安装失败 | SSH 到 Kafka VM；查 9093 端口 |
| **910061** | Redis 安装失败 | 查 6800 端口；`tools/open_redis_eval.sh` |
| **910091** | Kafka Topic 创建失败 | 确认 Kafka 已启动 |
| **910101** | Kafka 注册 MO 失败 | MO 地址可达；IAM token 有效 |
| **910071** | MoAgent 安装失败 | VM 到 MO 网络连通 |

### 7.4 微服务部署阶段（最高频）

| 错误码 | 含义 | 排查与处理 |
|--------|------|------------|
| **910111 / 910151** | 创建 NS/Secret 失败 | CDK API；IAM 账户是否创建成功 |
| **910171** | 安装微服务失败 | 见下方专项流程 |
| **910181** | 登录 CDK Master 失败 | 核对 CDK Master IP、opsadmin/root 密码 |

**910171 专项排查**：

```bash
# 1. 确认失败微服务名（日志中有 pkg 名）
# 2. CDK Master 上查看 Pod
kubectl get pod -n apm | grep <服务名>
kubectl describe pod <pod名> -n apm
kubectl logs <pod名> -n apm --tail=200

# 3. 部署成功后仍异常 → 查 Pod 内业务日志
kubectl exec -it <pod名> -n apm -- tail -100 /var/paas/sys/log/<服务名>/log/*.log

# 4. 常见原因：镜像拉取失败、中间件未就绪、资源不足、前置微服务未成功
```

### 7.5 安装后注册阶段

| 错误码 | 含义 | 排查与处理 |
|--------|------|------------|
| **910193** | APIG 注册失败 | APIGateway 前置任务；CloudLB 对接 |
| **910201** | DNS 注册失败 | 核对 `console_domain` |
| **910183** | Pod LB 注册失败 | CloudLB、CloudLBPost |
| **910241** | 创建 AppCode 失败 | APIG 对接 CloudLB 前置 |
| **910195 / 910196** | MO 计量话单注册失败 | MO 服务可达性 |
| **910276** | MO 适配包注册失败 | MO 适配 tar.gz 是否上传 |

### 7.6 Console 与国密

| 错误码 | 含义 | 排查与处理 |
|--------|------|------------|
| **910003 / 910011** | Static 安装失败 | `All_64_1.zip`；SDK/ConsoleFramework 前置 |
| **910002 / 910013** | Nginx 配置失败 | Nginx 组件是否部署 |
| **910274** | VM 安全加固失败 | 查加固脚本日志 |
| 国密步骤失败 | SM 切换中断 | 从失败国密子任务重试；勿跳过 soft/hard aksk 顺序 |

---

## 8. 日常运维命令速查

### 8.1 K8s（CDK Master）

```bash
kubectl get pod -n apm -o wide
kubectl get events -n apm --sort-by='.lastTimestamp' | tail -30
kubectl logs -f deploy/ams-metric -n apm --tail=100
kubectl rollout restart deploy/<服务名> -n apm   # 谨慎使用
```

### 8.2 Pod 业务日志

```bash
kubectl exec -it <pod名> -n apm -- tail -f /var/paas/sys/log/aom-billing/log/*.log
kubectl exec -it <pod名> -n apm -- ls -la /opt/cloud/logs/
```

### 8.3 失败任务重试

```bash
/opt/rootscripts/debug-tools/retry_step.sh execute -i
```

### 8.4 中间件

```bash
ssh root@<kafka_ip> "ps -ef | grep kafka; netstat -tlnp | grep 9093"
ssh root@<redis_ip> "netstat -tlnp | grep 6800"
```

### 8.5 错误码速查

```text
AOM/params/report_cn.ini
```

---

## 9. 交付检查清单

**部署前**

- [ ] HCS 8.6.1 环境就绪
- [ ] 前置云服务（AOS/SWR/PaaS-Common/PublicConfig/CDK/MO/OBS/CloudLB/APIG/SDK）已完成
- [ ] 插件包 + ROMA 1～4 包 + MO 适配包已上传
- [ ] 工程已选规格、Console、国密/专线/HA 选项
- [ ] LLD IP 已规划且无冲突
- [ ] `feature_switches`、`day_log`、`storage_days` 等参数已填写

**部署中**

- [ ] 按任务组顺序执行，失败任务先查 HCCI 日志再 `retry_step.sh execute -i`
- [ ] 解压/上传阶段预留足够磁盘和时间（约 2.5h）
- [ ] VM 创建阶段关注 Service OM 资源告警

**部署后**

- [ ] `kubectl get pod -n apm` 全部 Running
- [ ] Console 可访问
- [ ] MO 云服务注册成功
- [ ] APIG/DNS/LB 注册成功
- [ ] 关键服务 Pod 日志目录可访问（如 `/var/paas/sys/log/aom-billing/log`）
- [ ] 临时文件已清理（clean_up_temp_file）

---

## 10. 附录

### 10.1 主要微服务列表

| 微服务 | 说明 |
|--------|------|
| dpa-cassandra | Cassandra 数据库 |
| lts-escluster | ES 集群 |
| ams-access / ams-metric / ams-alarm / ams-view | 指标与告警 |
| als-config / als-access / als-analysis / als-query | 日志服务 |
| aom-billing | 计费 |
| icmgr-controller / icmgr-service | 采集管理 |
| alert-access / alert-mgmt | 告警接入 |
| ingester / compactor | 指标存储 |
| dataforward | 数据转发 |

### 10.2 关键文件索引

| 文件 | 用途 |
|------|------|
| `params/manifest.json` | 任务编排与依赖 |
| `params/param_details.cfg` | 人工参数说明 |
| `params/inspect.json` | 软件包规则 |
| `params/report_cn.ini` | 错误码中文说明 |
| `params/resource_calc.py` | VM 资源计算 |
| `scripts/consts.py` | 规格与版本常量 |
| `tools/uninstall_aom.sh` | 卸载脚本 |

### 10.3 排查信息收集模板

现场升级/Support 时可按此模板收集：

```text
1. 失败任务名/ID：例如 install_service_ams_metric
2. 错误码：例如 910171
3. 平台报错原文：
4. HCCI 任务日志最后 50 行（含 traceback）：
5. 工程规格：MIN/MEDIUM/MAX，是否 HA，是否国密
6. 前置任务是否全部成功：
7. kubectl get pod -n apm 输出：
8. 相关 Pod 业务日志（如 /var/paas/sys/log/aom-billing/log）：
```

---

**文档说明**：本手册依据 `AOM-Deploy-Plugin-8.6.1.00202604231446` 插件代码及现场运维实践整理。现场操作以华为 HCS 官方交付文档为准；若与官方文档冲突，以官方文档优先。

**修订记录**：

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0 | 2026-07-02 | 初版：部署流程、参数、报错、retry_step.sh |
| 1.1 | 2026-07-02 | 合并日志路径说明（HCCI / Pod / VM 三类） |
