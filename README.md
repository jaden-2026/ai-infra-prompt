# ai-infra-prompt
+ This is a prompt for building an AI-Infra platform via vibe-coding. Feel free to star it. It works seamlessly with Qoder, Cursor, Codex, Trae and so on.
+ 这个是一个通过vibe-coding创建ai-infra的prompt，欢迎star，支持通过Qoder、Cursor、Codex、Trea
# AI-Infra

> 面向异构算力集群的私有化 AI 基础设施平台 —— 统一管理 GPU/NPU 资源、任务调度、监控告警与多租户计费。

[English](./README.en.md) · 简体中文

---

## 这是什么

AI-Infra 是一套**私有化部署的 AI 算力 PaaS**，帮助企业在自有数据中心统一管理来自不同厂商的异构 GPU/NPU 集群。

它解决的核心问题：

- 多个业务团队抢占同一批 GPU，没有公平的调度与隔离机制
- NVIDIA、华为昇腾、寒武纪、燧原等不同硬件需要各自运维，驱动和框架兼容性难以追踪
- 训练任务失败后缺乏现场诊断能力，排障效率低
- 资源用量无法按业务线归因，成本核算困难

AI-Infra 不介入模型业务逻辑，专注于**算力资源申请、释放、监控**三件事，通过统一 API 向上层的大模型服务平台、MLOps 平台、智能体开发平台提供算力底座。

---

## 功能模块

```
┌─────────────────────────────────────────────────────────────┐
│                        Web 控制台 / CLI / SDK                │
├─────────────────────────────────────────────────────────────┤
│              开放 API 网关（REST · gRPC · Webhook）          │
├──────────────┬──────────────┬───────────────┬───────────────┤
│  资源管理     │  任务生命周期  │  监控可观测性  │  多租户权限   │
│  & 调度引擎  │  管理         │               │  & 计费       │
├──────────────┴──────────────┴───────────────┴───────────────┤
│              运行时与驱动适配层（HAL）                        │
├─────────────────────────────────────────────────────────────┤
│   NVIDIA CUDA  │  华为昇腾 CANN  │  寒武纪 CNRT  │  燧原 TOPSRT │
└─────────────────────────────────────────────────────────────┘
```

### 模块一：资源管理与调度

- **设备自动发现**：扫描并注册集群内 GPU/NPU 节点，采集厂商、型号、显存、拓扑等元数据；节点 Agent 30s 心跳，90s 超时自动摘除
- **多维资源池**：按厂商 / 型号 / 机房 / 业务线构建资源池，支持 AND/OR 规则组合、超配系数、优先级继承
- **智能调度引擎**：
  - Gang 调度（分布式训练全量原子分配）
  - 抢占调度（低优先级释放 → 自动保存 checkpoint → 重入队列）
  - Fair-Share 公平调度
  - 拓扑感知打分（NVLink 域 +20 / 跨 RDMA 域 −10）
  - Bin Packing 碎片优先填满
- **弹性扩缩容**：队列水位触发 K8s Cluster Autoscaler，支持弹性训练 Worker 动态增减

### 模块二：运行时与驱动适配层

- **统一 HAL**：屏蔽 CUDA / CANN / CNRT / TOPSRT 运行时差异，统一设备枚举、内存管理、流同步 API
- **兼容性矩阵**：驱动 × 框架 × 设备型号三维兼容性校验，作业提交前自动 BLOCK 不兼容组合
- **容器运行时集成**：Device Plugin 暴露 GPU/NPU 资源给 K8s，Init Container 自动注入 NCCL/HCCL，环境变量自动设置

### 模块三：监控与可观测性

- **多级指标采集**：GPU 利用率 / 显存 / 温度 / ECC 错误 / NVLink 带宽，采集频率按重要性分级（10s / 30s / 5min）
- **故障自动诊断**：Xid 错误字典（NVIDIA 74 种）+ CANN 故障码字典，告警触发后自动采集现场快照、生成根因报告、创建运维工单
- **多级告警**：Warning / Critical / Emergency 三级，On-call 升级链，5min 内同类告警聚合，支持静默窗口
- **全链路可观测**：Prometheus 指标 + Loki 日志 + OpenTelemetry 追踪三位一体，Grafana Dashboard

### 模块四：任务生命周期管理

- **多类型任务**：训练（单机单卡 / 单机多卡 / 多机多卡）、在线推理、批量推理、Notebook
- **分布式训练**：DP / MP / PP / SP 策略声明，平台按策略分配对应拓扑节点，NCCL/HCCL 自动注入
- **Checkpoint 管理**：异步流式写入、参数 hash 增量去重、跨存储域双写、UI 版本选择一键恢复
- **镜像安全**：Harbor 私有仓库，Trivy CVE 扫描，Cosign 签名验证，高危漏洞自动阻断准入

### 模块五：多租户与权限管理

- **四级租户体系**：Organization → Tenant → Project → User，租户间资源完全隔离
- **配额引擎**：提交时预占 reserved，结束后结算实际用量；80% 提醒 / 90% 告警 / 100% 禁止新作业
- **配额借用**：租户间临时借用空闲配额，到期自动回收，运行中作业不中断
- **RBAC**：Casbin 三维权限（resource × action × scope），SSO 集成（LDAP / OAuth 2.0 / SAML），TOTP MFA
- **计量计费**：分钟级精度，按卡时 + GPU 型号独立定价，月度账单 PDF 导出

### 模块六：开放 API 与运营管理

- RESTful API（OpenAPI 3.0）、gRPC、Webhook 事件推送、K8s CRD 扩展
- Python SDK / Go SDK / Terraform Provider
- Feature Flag 按租户灰度，配置中心热更新，Argo Rollouts 蓝绿发布

---

## 技术栈

| 层次 | 技术选型 |
|---|---|
| 后端语言 | Go（服务层）、C++ 17（HAL 层） |
| Web 框架 | Gin / Echo |
| 关系型数据库 | PostgreSQL |
| 时序数据库 | VictoriaMetrics |
| 列式数据库 | ClickHouse（审计日志） |
| 缓存 | Redis |
| 消息队列 | Kafka |
| 服务注册 | etcd |
| 容器编排 | Kubernetes + Volcano |
| 任务编排 | Kubeflow Training Operator、Argo Workflows |
| 服务网格 | Istio |
| 监控 | VictoriaMetrics + Loki + Jaeger + Grafana |
| 前端 | React 18 + TypeScript + Ant Design Pro |
| 镜像仓库 | Harbor |
| 密钥管理 | HashiCorp Vault |

完整技术选型及选型理由见 [docs/tech-stack.md](./docs/tech-stack.md)。

---

## 快速开始

### 前置条件

- Kubernetes 1.24+
- Helm 3.10+
- kubectl 已配置集群访问
- 至少一个带 GPU/NPU 的工作节点

### 部署

```bash
# 1. 添加 Helm 仓库
helm repo add ai-infra https://charts.ai-infra.io
helm repo update

# 2. 创建命名空间
kubectl create namespace ai-infra

# 3. 安装（最小化配置）
helm install ai-infra ai-infra/ai-infra \
  --namespace ai-infra \
  --set global.storageClass=standard \
  --set postgresql.enabled=true \
  --set redis.enabled=true

# 4. 等待 Pod 就绪
kubectl rollout status deployment/ai-infra-api -n ai-infra

# 5. 获取访问地址
kubectl get svc ai-infra-console -n ai-infra
```

生产环境部署请参考 [docs/deployment/production.md](./docs/deployment/production.md)。

---

## 项目结构

```
ai-infra/
├── agent/                  # 节点 Agent（Go）——心跳上报、设备采集
├── api/                    # API 服务（Go + Gin）
│   ├── handlers/           # HTTP Handler
│   ├── middleware/         # 鉴权、限流、日志中间件
│   └── openapi/            # OpenAPI 3.0 Spec
├── scheduler/              # 调度器扩展（Volcano Plugin）
├── hal/                    # 硬件抽象层（C++ 17 + CGO 桥接）
│   ├── cuda/
│   ├── cann/
│   ├── cnrt/
│   └── topsrt/
├── quota/                  # 配额引擎（Go）
├── billing/                # 计费服务（Go）
├── diagnostic/             # 故障诊断服务（Go）
├── checkpoint/             # Checkpoint 管理服务（Go）
├── console/                # Web 控制台（React 18 + TypeScript）
├── charts/                 # Helm Chart
├── docs/                   # 文档
│   ├── tech-stack.md       # 技术选型（含选型理由）
│   ├── architecture.md     # 架构设计
│   ├── api-reference.md    # API 参考
│   └── deployment/         # 部署指南
├── deploy/                 # Kubernetes Manifests / Kustomize
└── scripts/                # 运维脚本
```

---

## 文档

| 文档 | 说明 |
|---|---|
| [架构设计](./docs/architecture.md) | 整体架构、模块边界、数据流 |
| [技术选型](./docs/tech-stack.md) | 分模块技术选型及选型理由 |
| [API 参考](./docs/api-reference.md) | 完整 REST API 文档 |
| [部署指南](./docs/deployment/production.md) | 生产环境部署 |
| [开发指南](./docs/development.md) | 本地开发环境搭建 |
| [贡献指南](./CONTRIBUTING.md) | 如何参与贡献 |

---

## 边界说明

AI-Infra **只负责**：算力资源申请、释放、监控。

AI-Infra **不负责**：

| 上层系统 | 该系统负责的事 |
|---|---|
| 大模型服务平台 | 模型版本管理、API 网关、流量分发、限流熔断 |
| 智能体开发平台 | 智能体编排、工具调用、记忆管理 |
| MLOps / 数据平台 | 数据清洗、特征工程、实验追踪（MLflow 等） |

---

## 路线图

- [ ] 核心调度引擎（模块一）
- [ ] HAL 层 NVIDIA CUDA 实现（模块二）
- [ ] 基础监控采集与告警（模块三）
- [ ] 训练任务生命周期管理（模块四）
- [ ] 多租户与配额引擎（模块五）
- [ ] HAL 层昇腾 CANN 适配
- [ ] 跨集群联邦调度
- [ ] Terraform Provider
- [ ] 寒武纪 / 燧原 HAL 适配

---

## 贡献

欢迎 Issue 和 Pull Request。

---

## License

[Apache 2.0](./LICENSE)
