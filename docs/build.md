# CoolCar (租辆酷车) 项目构建文档

## 项目概述

CoolCar 是一个车辆分时租赁系统，采用微服务架构设计，包含前端和后端两大部分：

- **前端**：基于微信小程序，使用 TypeScript 开发
- **后端**：基于 Go 语言的微服务架构，部署在 Kubernetes 集群中

本项目实现了从用户注册、车辆预订、开锁驾驶到还车支付的完整业务流程，采用领域驱动设计理念和敏捷开发方法进行构建。

## 系统架构

### 总体架构

CoolCar 采用前后端分离的微服务架构：

```
客户端层：微信小程序 (TypeScript)
  │
  ▼
API网关层：Gateway 服务
  │
  ▼
业务服务层：Auth服务、Rental服务、Car服务、Blob服务等
  │
  ▼
基础设施层：MongoDB、RabbitMQ、腾讯云COS等
```

### 核心微服务

1. **Auth 服务**：用户认证与授权
2. **Rental 服务**：租车业务逻辑处理
3. **Car 服务**：车辆管理与状态监控
4. **Blob 服务**：文件存储服务
5. **Gateway 服务**：API 网关，统一服务入口
6. **CoolEnv 服务**：开发环境模拟服务

## 开发环境搭建

### 前置条件

- Go 1.15+
- Node.js 12+
- Docker & Docker Compose
- Kubernetes 工具链 (kubectl)
- 微信开发者工具
- Git

### 本地开发环境配置

1. **克隆代码仓库**

```bash
git clone https://your-repo-url/coolcar.git
cd coolcar
```

2. **配置微信小程序开发环境**

```bash
cd wx/miniprogram
npm install
```

然后在微信开发者工具中导入项目，并执行"构建 npm"操作。

3. **配置后端开发环境**

```bash
cd server
go mod tidy
```

## 项目构建流程

### 前端构建

1. **编译 TypeScript 代码**

```bash
cd wx/miniprogram
npm run tsc
```

2. **微信开发者工具中构建小程序**

点击开发者工具中的"编译"按钮，或使用以下脚本自动化构建：

```bash
# 使用微信开发者工具命令行
cli build -p wx/miniprogram
```

### 后端服务构建

后端微服务采用 Docker 容器化部署，每个服务都有对应的 Dockerfile。

#### 使用build.sh构建单个服务

项目提供了统一的构建脚本 `deployment/build.sh`，用于构建各个微服务的 Docker 镜像：

```bash
cd deployment
./build.sh <service-name>
```

例如，构建 auth 服务：

```bash
./build.sh auth
```

该脚本会在 server 目录下执行 Docker 构建命令，生成 `kucar/<service-name>` 镜像。

#### 构建所有服务

可以使用以下命令构建所有后端服务：

```bash
for service in auth blob car gateway rental
do
  ./build.sh $service
done
```

### Docker 镜像详情

每个微服务的 Dockerfile 大致包含以下步骤：

1. 使用 `golang:1.15-alpine` 作为编译环境
2. 配置 Go 模块代理
3. 复制源代码并编译
4. 对于 gRPC 服务，安装 grpc-health-probe 健康检查工具
5. 使用 `alpine:3.13` 作为运行环境，将编译好的二进制文件复制到最终镜像

## 部署流程

### 部署到 Kubernetes

项目采用 Kubernetes 进行容器编排和服务管理。每个微服务都有对应的部署配置文件（YAML）。

#### 部署单个服务

```bash
kubectl apply -f deployment/<service-name>/<service-name>.yaml
```

例如，部署 auth 服务：

```bash
kubectl apply -f deployment/auth/auth.yaml
```

#### 部署所有服务

以下步骤展示了项目的完整部署流程：

1. **部署配置和密钥**

```bash
kubectl apply -f deployment/config/
```

2. **部署基础设施服务（可选，如果使用外部服务则不需要）**

```bash
kubectl apply -f deployment/coolenv/coolenv.yaml
```

3. **按依赖顺序部署业务服务**

```bash
kubectl apply -f deployment/auth/auth.yaml
kubectl apply -f deployment/blob/blob.yaml
kubectl apply -f deployment/car/car.yaml
kubectl apply -f deployment/rental/rental.yaml
kubectl apply -f deployment/gateway/gateway.yaml
```

4. **部署 Istio 服务网格配置（如果使用）**

```bash
kubectl apply -f deployment/config/istio/
```

### 环境变量配置

各服务通过环境变量进行配置，主要包括：

- 微服务地址（如 `CAR_ADDR`, `TRIP_ADDR` 等）
- 数据库连接信息（如 `MONGO_URI`）
- 消息队列连接信息（如 `AMQP_URL`）
- 腾讯云对象存储配置（如 `COS_ADDR`, `COS_SEC_ID` 等）
- 微信小程序配置（如 `WECHAT_APP_ID`, `WECHAT_APP_SECRET`）
- 安全密钥（如 `PRIVATE_KEY_FILE`）

环境变量配置在 Kubernetes 部署文件中以 ConfigMap 和 Secret 的形式管理。

## 服务详情

### Auth 服务

**功能**：提供用户认证和授权，包括微信登录集成和 JWT 令牌管理。

**API**：gRPC 服务，端口 8081

**依赖**：
- MongoDB 数据库
- 微信登录 API

**构建命令**：
```bash
./build.sh auth
```

### Car 服务

**功能**：管理车辆资源，包括车辆状态追踪、位置更新和远程控制。

**API**：gRPC 服务，端口 8081

**依赖**：
- MongoDB 数据库
- RabbitMQ 消息队列
- Trip 服务

**构建命令**：
```bash
./build.sh car
```

### Rental 服务

**功能**：处理租车业务核心逻辑，包括行程管理和计费。

**API**：gRPC 服务，端口 8081

**依赖**：
- MongoDB 数据库
- Car 服务
- Blob 服务
- Auth 公钥

**构建命令**：
```bash
./build.sh rental
```

### Blob 服务

**功能**：处理文件上传和存储，如驾照图片。

**API**：gRPC 服务，端口 8081

**依赖**：
- MongoDB 数据库
- 腾讯云对象存储 COS

**构建命令**：
```bash
./build.sh blob
```

### Gateway 服务

**功能**：API 网关，将 HTTP/RESTful 请求转换为后端 gRPC 调用。

**API**：HTTP 服务，端口 8080

**依赖**：
- 所有后端 gRPC 服务

**构建命令**：
```bash
./build.sh gateway
```

### CoolEnv 服务

**功能**：开发环境模拟服务，提供模拟的基础设施服务如数据库、消息队列等。

**API**：多端口服务
- gRPC: 18001
- HTTP: 18000
- RabbitMQ: 5672 (管理界面: 15672)
- MongoDB: 27017

**构建命令**：
使用预构建镜像 `ccr.ccs.tencentyun.com/coolcar/coolenv:1.2`

## 微信小程序开发

### 构建与运行

1. **安装依赖**

```bash
cd wx/miniprogram
npm install
```

2. **编译 TypeScript**

```bash
npm run tsc
```

3. **在微信开发者工具中构建 NPM**

点击开发者工具中的"工具" -> "构建 npm"。

4. **编译小程序**

点击开发者工具中的"编译"按钮。

### 项目结构

小程序采用 TypeScript 开发，主要组件包括：

- **pages/**: 小程序页面
  - index/: 首页
  - register/: 用户注册
  - driving/: 行程页面
  - mytrips/: 我的行程
- **service/**: 后端服务调用接口
- **utils/**: 通用工具函数
- **components/**: 自定义组件

## Proto 文件生成

后端微服务间通信使用 gRPC，需要生成相应的协议代码：

```bash
cd server
./genProto.sh
```

此脚本会为每个服务生成必要的 Go 代码。

## 测试

### 运行单元测试

```bash
cd server
go test ./...
```

### 运行特定服务的测试

```bash
cd server/auth
go test ./...
```

## 问题排查

### 常见问题解决方案

1. **服务无法启动**
   - 检查环境变量配置是否正确
   - 检查依赖服务是否可用
   - 查看服务日志 `kubectl logs deployment/<service-name>`

2. **服务间通信失败**
   - 检查服务发现配置
   - 确认端口暴露正确
   - 检查网络策略或防火墙规则

3. **构建失败**
   - 确保 Go 和 Node.js 版本正确
   - 检查代理设置

## 运维与监控

### 健康检查

所有 gRPC 服务都集成了健康检查机制，使用 grpc-health-probe 进行探测。

### 日志收集

服务日志通过 Kubernetes 标准输出收集，可以使用以下命令查看：

```bash
kubectl logs deployment/<service-name>
```

## 安全注意事项

1. 敏感信息（如密钥、证书）通过 Kubernetes Secrets 管理
2. JWT 用于服务间身份验证
3. 所有外部 API 通过网关集中管理，实施访问控制

## 版本管理

项目使用 Git 进行版本管理，分支策略如下：
- master: 主分支，同步最新代码
- 特性分支: 根据课程进度组织

## 结语

CoolCar 项目采用现代化的微服务架构和云原生技术栈，为开发者提供了完整的端到端租车系统实现。通过本文档的指导，您应该能够成功构建、部署和维护整个系统。