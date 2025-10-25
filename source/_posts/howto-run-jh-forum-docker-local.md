---
title: 如何在docker和本地环境运行jh论坛
date: 2025-10-25 01:24:47
tags: jhlt
category: tech
---

本文介绍如何在 Docker 和纯本地环境下运行 JH 论坛项目，包括环境配置、常见问题及解决方案。

## 方式一：Docker + 本地环境（推荐）

### 快速启动步骤

1. **打开项目**：使用 `IntelliJ IDEA` 或 `Visual Studio Code` 打开项目

2. **修改配置文件**：编辑 `application.yml`，更新以下配置：
   - 数据库账号密码
   - Redis 密码（如果设置了）
   - Nacos 配置信息

3. **启动 Docker 容器**：
   ```bash
   docker compose up
   ```
   > **注意**：启动前请确保 Docker Desktop 已运行

4. **等待容器启动**：等待所有容器成功挂起

5. **运行论坛**：在本地 IDE 中启动论坛应用

### 常见问题排查

#### 问题 1：网络请求失败

**错误信息**：
```
failed to do request: Get…………error
```

**原因**：网络环境不稳定

**解决方法**：多次重试 `docker compose up` 命令

#### 问题 2：镜像拉取失败（403 Forbidden）

**错误信息**：
```
✘ sentinel-dashboard Error     unknown: failed to resolve reference "docker.io/bladex/sentinel-dashboard:1.8.8": unexpected status from HEAD re...                          10.4s
Error response from daemon: unknown: failed to resolve reference "docker.io/bladex/sentinel-dashboard:1.8.8": unexpected status from HEAD request to https://docker.m.daocloud.io/v2/bladex/sentinel-dashboard/manifests/1.8.8?ns=docker.io: 403 Forbidden
```

**原因**：Docker 镜像源配置问题或镜像源不可用

**解决方法**：
1. 检查 Docker 镜像源配置（优先排查）
2. 更换可用的镜像源
3. 确认网络连接正常

#### 问题 3：镜像不在白名单

**错误信息**：
```
error from registry: 🚫-> https://github.com/DaoCloud/public-image-mirror/issues/2328 🔗 这镜像不在白名单. this image is not in the allowlist.
```

**原因**：使用的镜像源对该镜像有访问限制

**解决方法**：
1. 检查并更换镜像源
2. 使用官方 Docker Hub 或其他可用镜像源
3. 联系运维人员配置镜像白名单

---

## 方式二：纯本地环境

如果不使用 Docker，可以选择完全在本地环境运行论坛。这种方式需要手动配置所有依赖服务。

### 第一步：前期准备

#### 1. 配置 Nacos

Nacos 是服务配置中心，需要正确配置才能使用。

**操作步骤**：

1. **设置单机模式**：
   - 打开 `conf/application.properties` 文件
   - 将 `mode` 修改为 `standalone`

2. **配置账号密码**：
   - 在 Nacos 中设置账号密码
   - **重要**：同步修改项目 `application.yml` 中的 Nacos 账号密码，否则会导致连接失败

3. **配置运行地址**：
   - 在 `conf/application.properties` 中添加：
     ```properties
     nacos.inetutils.ip-address=127.0.0.1
     ```
   - **注意**：本地运行建议使用 `127.0.0.1`，也可以根据需要修改为其他地址
   - **重要**：同步修改项目 `application.yml` 中的 Nacos 地址配置

#### 2. 配置 Redis

Redis 用于缓存和会话管理。

**配置选项**：

- **方案一（不设置密码）**：
  - Redis 安装后不设置密码
  - 在项目 `application.yml` 中注释掉 `spring.data.redis.password` 配置项

- **方案二（设置密码）**：
  - 为 Redis 设置密码
  - 在项目 `application.yml` 的 `spring.data.redis.password` 中配置相应密码


### 第二步：修改项目配置文件

除了上述 Nacos 和 Redis 的配置，还需要对项目的 `application.yml` 进行以下修改：

#### 1. 重命名配置文件

首次打开项目时，配置文件名为 `application.example.yml`，需要：
- 将文件名改为 `application.yml`
- 修改配置文件中的导入路径：

```yml
spring:
  config:
    import: "nacos:nacos-config-application.properties?refresh=true"
```

> **说明**：原配置为 `nacos:nacos-config-application-example.properties`，需要删除文件名中的 `-example`

#### 2. 配置 Cube 和 User-Center

配置文件最下方的 `cube` 和 `user-center` 配置项默认为空，这些是论坛核心服务配置。

**获取方式**：联系论坛开发人员获取相应的配置信息

#### 3. 修改 Dubbo 协议

由于已知兼容性问题，需要将 Dubbo 协议从 `tri` 改为 `dubbo`：

```yml
dubbo:
  protocol:
    name: dubbo  # 原值为 tri，需要修改为 dubbo
```

> **重要**：如果不修改此配置，Dubbo 服务将无法正常连接

### 第三步：启动前检查

在启动论坛之前，需要完成以下检查和准备工作：

#### 1. 检查端口占用

论坛的 Dubbo 服务使用 **50052** 端口，启动前需确保该端口未被占用。

**检查方法**（Linux/Mac）：
```bash
lsof -i:50052
```

**检查方法**（Windows）：
```cmd
netstat -ano | findstr 50052
```

**处理方式**：
- 如果端口被占用，需要先关闭占用该端口的进程
- 否则 Dubbo 将无法启动，导致服务间通信失败

#### 2. 启动依赖服务

**必须先启动以下服务**，否则论坛无法正常运行：

##### 启动 Nacos

**启动方式**：
```bash
# Linux/Mac
sh startup.sh -m standalone

# Windows
startup.cmd -m standalone
```

**启动成功标志**：
命令行最后一行显示：
```
Nacos started successfully in stand alone mode. use embedded storage
```

##### 启动 Redis

**启动方式**：
```bash
# Linux/Mac
redis-server

# Windows
redis-server.exe
```

**启动成功标志**：
- 使用 `redis-cli` 连接后不会闪退
- 显示配置的 IP 地址提示符，如 `127.0.0.1:6379>`

#### 3. 数据库配置优化

在数据源配置中，建议添加 `allowPublicKeyRetrieval=true` 参数以避免连接问题：

```yml
spring:
  datasource:
    url: jdbc:mysql://your_ip/your_database?useSSL=false&autoReconnect=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
```

> **说明**：原配置文件中可能没有 `allowPublicKeyRetrieval=true` 参数，建议添加

### 第四步：启动论坛

确认以上所有配置和服务都已就绪后：

1. 在 IDE 中点击运行按钮启动论坛应用
2. 等待应用加载，观察控制台输出

**启动成功标志**：
- 控制台输出 "帖子热度计算中" 等业务相关日志
- 短期内（1-2分钟）没有报错信息
- 应用进入稳定运行状态

