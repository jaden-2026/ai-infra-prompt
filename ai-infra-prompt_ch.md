# AI-Infra 平台技术选型  
> 每个模块包含：技术选型、选型理由、核心数据模型、关键 API 端点、注意事项。

## 布局结构
```
─────────────────────────────────────────┐
│ Sidebar    │ Header (sticky, 240px)     │
│ (fixed)    ├───────────────────────────┤
│ 240px      │ Content                   │
│            │ (margin-left: 240px)      │
│            │                           │
└────────────┴───────────────────────────┘
```

## 前端
开发语言：Typescript
组件库：Ant Design + Vite + React Router
---

## 目录

- [模块一：资源管理与调度](#模块一资源管理与调度)
- [模块二：运行时与驱动适配层](#模块二运行时与驱动适配层)
- [模块三：监控与可观测性](#模块三监控与可观测性)
- [模块四：任务生命周期管理](#模块四任务生命周期管理)
- [模块五：多租户与权限管理](#模块五多租户与权限管理)
- [模块六：开放 API 与运营管理](#模块六开放-api-与运营管理)
- [横切：安全与合规基座](#横切安全与合规基座)
- [全局技术栈速查表](#全局技术栈速查表)

---

## 模块一：资源管理与调度

**设备发现与注册**

+ 自动扫描并注册集群内 GPU/NPU 节点（NVIDIA、华为昇腾、寒武纪、燧原等）
+ 采集设备元数据：厂商、型号、显存大小、驱动版本、固件版本、PCIe/NVLink 拓扑
+ 节点健康状态维护（在线/离线/故障/维护）
+ 设备动态上下线感知与告警
+ 节点 Agent 主动上报心跳（间隔 30s，超时 90s 判定离线），心跳 Payload 包含全量设备快照
+ 设备指纹去重（vendor + model + serial_number 三元组唯一）
+ 节点上下线事件发布至消息总线（下游监控模块订阅）
+ 设备标签系统（kv 标签，支持按标签筛选调度）

**资源池化管理**

+ 按厂商、型号、机房、业务线构建多维资源池
+ 支持 GPU 虚拟化切分（MIG for NVIDIA、vDVPP for 昇腾）
+ CPU/内存/存储/网络带宽联合资源建模
+ 资源池容量规划与预测（基于历史用量趋势）
+ 池创建规则支持 AND/OR 条件组合（vendor=nvidia AND model=A100 OR H800）
+ 默认池兜底（未匹配任何自定义池的设备归入 default 池）
+ 池容量超配系数配置（允许 110% 超卖，用于推理混部）
+ 池优先级继承（子池继承父池优先级，可覆盖）

**智能调度引擎**

+ 多策略调度：优先级队列、Gang 调度（分布式训练全量分配）、抢占调度、公平调度（Fair Share）
+ 亲和性/反亲和性规则（同机、跨机、跨机架）
+ 网络拓扑感知调度（优先同 NVSwitch 域 / RDMA 域内分配）
+ 跨厂商异构作业混部调度（根据框架兼容性自动选型）
+ 调度插件扩展接口（Webhook / Plugin 机制）
+ Gang 调度：全量设备一次性原子分配，部分满足则阻塞等待，支持超时放弃（timeout_seconds 可配）
+ 抢占调度：低优先级任务释放设备给高优先级，被抢占任务自动重入队列，保存当前 checkpoint 路径
+ 碎片感知：同一节点剩余 GPU 数 < 最小分配单位时优先填满该节点（Bin Packing）
+ 拓扑得分：同 NVLink 域内分配得分 +20，跨 RDMA 域扣分 -10，调度器取最高分节点
+ 调度失败原因枚举：INSUFFICIENT_RESOURCE / NO_COMPATIBLE_DEVICE / QUOTA_EXCEEDED / GANG_TIMEOUT

**弹性扩缩容**

+ 基于队列水位自动触发节点扩容（对接 K8s Cluster Autoscaler）
+ 训练任务弹性工作者动态增减（Elastic Training）
+ 空闲时段资源回收与借用策略
+ 跨集群 / 跨云资源联邦调度
  
**UI页面/组件**
+ 集群总览页：节点卡片矩阵，颜色编码状态，支持按机架分组
+ 设备详情抽屉：单卡实时指标折线图 + 历史分配记录
+ 资源池管理页：规则编辑器（tag 选择器 UI）+ 容量预览
+ 调度队列页：实时刷新的任务排队列表，支持手动调整优先级

### 技术选型

| 子功能 | 选型 | 备注 |
|---|---|---|
| 节点 Agent | **Go** + gRPC | 心跳 30s，超时 90s，全量设备快照 |
| 设备元数据存储 | **PostgreSQL** + JSONB | PCIe/NVLink 拓扑用 JSONB，标签筛选覆盖索引 |
| 设备上下线事件 | **Kafka** + CloudEvents | 至少一次投递，监控模块消费 |
| 服务注册 / 离线感知 | **etcd** | Agent 租约注册，调度器 Watch |
| 调度器核心 | **Volcano**（K8s 扩展） | Gang / Fair-Share / 抢占，原生支持 |
| 弹性扩缩容 | **K8s Cluster Autoscaler** | 队列水位触发，对接公有云节点池 |
| 资源池规则引擎 | **自研 DSL（Go）** | AND/OR tag 条件，default 池兜底，超配系数 |
| 拓扑打分插件 | **Scheduler Plugin（Go）** | NVLink +20 / 跨 RDMA −10，Bin Packing |
| 前端框架 | **React 18 + TypeScript** | Ant Design Pro，ECharts，AntV G6 拓扑图 |
| 实时数据 | **WebSocket** + SWR | 节点矩阵实时刷新，设备详情折线图 |

### 关键设计约束

```
- 设备指纹去重：vendor + model + serial_number 三元组唯一
- Gang 调度：全量原子分配，超时放弃（timeout_seconds 可配）
- 抢占调度：低优先级释放 → 被抢占任务自动保存 checkpoint 路径 → 重入队列
- 碎片感知：同节点剩余 GPU < 最小分配单位时优先 Bin Packing 填满
- 调度失败枚举：INSUFFICIENT_RESOURCE | NO_COMPATIBLE_DEVICE | QUOTA_EXCEEDED | GANG_TIMEOUT
```

### 核心数据模型

```go
// Node
type Node struct {
    ID              string    `db:"id"`
    Hostname        string    `db:"hostname"`
    IP              string    `db:"ip"`
    RackID          string    `db:"rack_id"`
    ClusterID       string    `db:"cluster_id"`
    Status          string    `db:"status"` // online|offline|fault|maintenance
    created_at      time.Time `db:"created_at"`
    LastHeartbeatAt time.Time `db:"last_heartbeat_at"`
}

// Device
type Device struct {
    ID             string `db:"id"`
    NodeID         string `db:"node_id"`
    Vendor         string `db:"vendor"`   // nvidia|huawei|cambricon|enflame
    ModelName      string `db:"model_name"`
    DeviceIndex    int    `db:"device_index"`
    TotalMemoryMB  int    `db:"total_memory_mb"`
    DriverVersion  string `db:"driver_version"`
    NVLinkGroupID  string `db:"nvlink_group_id"`
    Status         string `db:"status"`   // available|allocated|fault|disabled
}

// ResourcePool
type ResourcePool struct {
    ID               string          `db:"id"`
    Name             string          `db:"name"`
    FilterRules      json.RawMessage `db:"filter_rules"` // {vendor, model, tags}
    Priority         int             `db:"priority"`
    SchedulingPolicy string          `db:"scheduling_policy"` // fifo|fair|priority
}

// Allocation
type Allocation struct {
    ID          string    `db:"id"`
    JobID       string    `db:"job_id"`
    DeviceIDs   []string  `db:"device_ids"`
    NodeID      string    `db:"node_id"`
    RequestedAt time.Time `db:"requested_at"`
    AllocatedAt time.Time `db:"allocated_at"`
    ReleasedAt  time.Time `db:"released_at"`
    Status      string    `db:"status"` // pending|allocated|releasing|released
}
```

### 设备状态机

```
available ──[任务分配]──► allocated
allocated ──[任务结束]──► releasing
releasing ──[显存归零完成]──► available
available/allocated ──[健康检测失败]──► fault
fault ──[人工确认]──► maintenance
maintenance ──[修复完成]──► available
```

### API 端点

```
GET    /api/v1/nodes                         # 节点列表（过滤、分页）
GET    /api/v1/nodes/{id}                    # 节点详情 + 设备列表
PATCH  /api/v1/nodes/{id}/status             # 手动设置节点状态（维护模式）
GET    /api/v1/devices                       # 设备列表
GET    /api/v1/devices/{id}                  # 设备详情
GET    /api/v1/pools                         # 资源池列表
POST   /api/v1/pools                         # 创建资源池
PUT    /api/v1/pools/{id}                    # 更新资源池规则
POST   /api/v1/allocations                   # 申请分配（调度触发）
DELETE /api/v1/allocations/{id}              # 释放分配
GET    /api/v1/scheduler/queue               # 当前调度队列
GET    /api/v1/scheduler/decisions/{job_id}  # 调度决策详情 + 原因
```

---

## 模块二：运行时与驱动适配层

**统一硬件抽象层（HAL）**

+ 抽象 CUDA / CANN（昇腾）/ CNRT（寒武纪）/ TOPSRT（燧原）等运行时差异
+ 统一设备枚举、内存分配、数据传输、流同步 API
+ 驱动版本矩阵管理与兼容性校验
+ 设备枚举接口：hal.list_devices() → 返回统一 DeviceInfo 结构（屏蔽厂商差异）
+ 内存管理：hal.alloc(device_id, size_mb) / hal.free(handle) / hal.memset(handle, 0)（显存擦除强制归零）
+ 数据传输：hal.h2d(src, dst, size) / hal.d2h / hal.d2d（跨设备需判断是否同厂商）
+ 事件同步：hal.create_stream() / hal.sync_stream() / hal.create_event()
+ 错误码统一映射表：各厂商原始错误码 → 平台统一错误码（如 CUDA error 2 = CANN error 10 = 平台 ERR_OOM）

**驱动版本矩阵管理**
+ 驱动 × 框架 × 设备型号 三维兼容性矩阵录入与查询
+ 作业提交前自动校验：镜像内框架版本 + 目标节点驱动版本 → 返回兼容性评级（OK/WARN/BLOCK）
+ 不兼容时给出推荐替代方案（推荐驱动版本或推荐镜像）
+ 兼容性矩阵变更通知（驱动升级后自动重新计算受影响作业列表）

**多框架兼容适配**

+ 支持 PyTorch、TensorFlow、MindSpore、PaddlePaddle 等主流框架
+ 框架 × 硬件 兼容矩阵维护（自动阻断不兼容组合提交）
+ 自定义算子注册与跨硬件桥接

**算子库与算法库管理**

+ CUDA cuDNN / cuBLAS、昇腾 CANN 算子库版本管理
+ 算子性能 Benchmark 库（相同算子在不同硬件上的基准测试数据）
+ 自定义算子上传、验证、发布流程

**容器运行时集成**

+ Device Plugin 管理（GPU/NPU resource 暴露给 K8s）
+ 容器镜像内驱动/运行时注入（避免镜像与驱动强耦合）
+ 运行时环境隔离（不同作业间驱动环境互不干扰）
+ Device Plugin 实现：K8s 资源名称规范（vendor.com/gpu，如 nvidia.com/gpu、huawei.com/npu）
+ 容器启动前注入钩子：挂载对应厂商 runtime（--runtime=nvidia / --runtime=cann）
+ 多卡拓扑注入：环境变量 CUDA_VISIBLE_DEVICES / ASCEND_VISIBLE_DEVICES 自动设置
+ 故障隔离：单容器崩溃不影响同节点其他容器，设备状态自动重置

### 技术选型

| 子功能 | 选型 | 备注 |
|---|---|---|
| HAL 核心实现 | **C++ 17** + Go CGO 桥接 | 屏蔽 CUDA / CANN / CNRT / TOPSRT 差异 |
| NVIDIA 运行时 | **CUDA Runtime** + NVML | 设备枚举、显存 alloc/free/memset |
| 华为昇腾运行时 | **CANN** + ACL | 三核（AI Core / Vector / Cube）分离采集 |
| 寒武纪 / 燧原 | **CNRT** / **TOPSRT** | 统一 DeviceInfo 结构屏蔽差异 |
| 错误码映射 | **统一 ERR_* 枚举（Go）** | 各厂商原始码 → 平台统一码 |
| 兼容性矩阵存储 | **PostgreSQL** | driver × framework × device_model 三维表 |
| 容器运行时 | **containerd** + 厂商 runtime 插件 | nvidia-container-runtime / cann-container-runtime |
| Device Plugin | **K8s Device Plugin（Go）** | nvidia.com/gpu、huawei.com/npu |
| 算子库版本管理 | **Harbor**（OCI） | cuDNN / CANN 算子包，Benchmark 结果入 DB |

### 关键设计约束

```
- HAL 接口规范：
    hal.list_devices() → []DeviceInfo
    hal.alloc(device_id, size_mb) → handle
    hal.free(handle)
    hal.memset(handle, 0)          // 显存归零，强制执行
    hal.h2d / hal.d2h / hal.d2d   // 跨厂商需判断是否同厂商

- 兼容性评级：OK | WARN | BLOCK
  作业提交前自动校验：镜像内框架版本 + 目标节点驱动版本

- Device Plugin 注入：
    CUDA_VISIBLE_DEVICES    # NVIDIA
    ASCEND_VISIBLE_DEVICES  # 昇腾
    自动设置，用户无感知

- 故障隔离：单容器崩溃不影响同节点其他容器，设备状态自动重置
```

### 核心数据模型

```go
type DriverProfile struct {
    ID                 string   `db:"id"`
    Vendor             string   `db:"vendor"`
    RuntimeName        string   `db:"runtime_name"` // cuda|cann|cnrt|topsrt
    DriverVersion      string   `db:"driver_version"`
    RuntimeVersion     string   `db:"runtime_version"`
    SupportedFrameworks []FrameworkSupport
    DeviceModels       []string
}

type CompatibilityMatrix struct {
    DriverProfileID  string `db:"driver_profile_id"`
    Framework        string `db:"framework"`
    FrameworkVersion string `db:"framework_version"`
    Status           string `db:"status"` // supported|degraded|unsupported
    Notes            string `db:"notes"`
}

type OperatorLibrary struct {
    ID               string            `db:"id"`
    Vendor           string            `db:"vendor"`
    Name             string            `db:"name"`
    Version          string            `db:"version"`
    Checksum         string            `db:"checksum"`
    BenchmarkResults []BenchmarkResult
}
```

### API 端点

```
GET  /api/v1/drivers                          # 驱动配置文件列表
POST /api/v1/drivers                          # 录入新驱动配置
GET  /api/v1/compatibility/check              # 实时兼容性校验
                                              # ?framework=&version=&device_model=
GET  /api/v1/compatibility/matrix             # 完整兼容性矩阵（支持 CSV 导出）
GET  /api/v1/operators                        # 算子库列表
POST /api/v1/operators/upload                 # 上传自定义算子包
GET  /api/v1/operators/{id}/benchmarks        # 算子基准测试结果
POST /api/v1/operators/{id}/benchmark/run     # 触发基准测试（异步，返回 task_id）
```

---

## 模块三：监控与可观测性

**实时指标采集**

+ GPU 级：利用率、显存占用/带宽、功耗、温度、ECC 错误数、NVLink 带宽
+ 昇腾 NPU 专有指标：AI Core 利用率、HBM 带宽、D2D 互联利用率
+ 节点级：CPU、内存、网络 I/O、存储 I/O
+ 集群级：总算力利用率、队列深度、资源碎片率
+ 采集频率分级：关键指标（利用率、温度、ECC）10s；一般指标（功耗、内存带宽）30s；慢指标（驱动版本）5min
+ 昇腾专有指标：AI Core 利用率、Vector Core 利用率、Cube Core 利用率（三核分离采集）
+ 指标缺失检测：连续 3 个采集周期无数据则触发采集器异常告警
+ 自定义指标上报接口（业务方可 Push 自定义指标挂载到作业维度）

**故障诊断自动化与告警**

+ 多级告警规则（Warning / Critical / Emergency）
+ GPU Xid 错误解析（NVIDIA）、CANN 故障码解析（昇腾）
+ 自动隔离 + 工单创建（故障节点自动摘除、触发运维工单）
+ GPU Xid 错误字典（NVIDIA 74种 Xid 对照表，含严重程度和处理建议）
+ 昇腾故障码字典（CANN ErrCode 对照处理建议）
+ 自动诊断流程：告警触发 → 采集现场快照（dmesg、gpu-smi、内存 dump）→ 比对历史模式 → 生成根因报告
+ 多卡关联分析：单卡 ECC 错误率上升 → 同机其他卡健康度扫描 → 判断是否机器级别故障
+ 故障自愈：可配置自愈动作（重启容器/隔离设备/通知运维），需人工确认级别分三档

**告警管理**
+ 告警收敛与根因分析（单卡故障 vs 机器故障 vs 机架故障）
+ 告警通知渠道：邮件、企业微信/钉钉、Webhook、PagerDuty
+ 告警聚合（同一设备 5min 内相同类型告警合并为一条）
+ 告警路由规则（按严重程度、设备类型、所属租户路由到不同通知渠道）
+ On-call 升级链（Warning 通知一级，30min 未 ACK 升级 Critical，再 30min 升级 Emergency）
+ 静默窗口（维护窗口期间自动静默，维护结束后恢复）

**拓扑可视化**

+ 机房 → 机架 → 节点 → GPU 卡 层级拓扑图
+ NVLink / NVSwitch / PCIe 互联拓扑可视化
+ 热力图展示（利用率、温度、故障分布）

**全链路可观测**

+ 指标（Prometheus/VictoriaMetrics）、日志（Loki/ELK）、追踪（OpenTelemetry/Jaeger）三位一体
+ 作业级 Profiling 数据采集（NVIDIA Nsight / 昇腾 Profiler 结果汇聚）
+ 自定义 Dashboard 配置与分享（Grafana 集成）

### 技术选型

| 子功能 | 选型 | 备注 |
|---|---|---|
| GPU 指标采集 | **DCGM Exporter** + Prometheus Node Exporter | NVIDIA 专用，覆盖 Xid / ECC / NVLink |
| 昇腾指标采集 | **自研 Ascend Exporter（Go）** | AI Core / Vector Core / Cube Core 三核分离 |
| 时序数据库 | **VictoriaMetrics** | 高基数优化，原生 PromQL，原始 7 天 / 1min 聚合 90 天 |
| 日志 | **Loki** + Promtail | 与 Prometheus 标签体系统一，Grafana 关联查询 |
| 链路追踪 | **OpenTelemetry SDK** + Jaeger | X-Request-ID 全链路透传 |
| Dashboard | **Grafana** | 热力图 / 拓扑图面板，自定义 Dashboard 分享 |
| 告警引擎 | **VMAlert**（PromQL 表达式） | for_duration 防抖，三级告警 |
| 通知路由 | **Alertmanager** | 企业微信 / 钉钉 / PagerDuty / Webhook |
| 故障诊断服务 | **自研 Diagnostic Service（Go）** | Xid / CANN 错误码字典，自动根因报告 |
| Profiling 汇聚 | **NVIDIA Nsight** / **昇腾 Profiler** + Pyroscope | 作业级 Profiling 数据聚合 |

### 关键设计约束

```
- 采集频率分级：
    关键指标（利用率 / 温度 / ECC）：10s
    一般指标（功耗 / 内存带宽）：30s
    慢指标（驱动版本）：5min

- 指标缺失检测：连续 3 个采集周期无数据 → 触发采集器异常告警

- 告警收敛：同一设备 5min 内相同类型告警合并为一条

- On-call 升级链：
    Warning → 通知一级
    30min 未 ACK → 升级 Critical
    再 30min → 升级 Emergency

- 自动诊断流程：
    告警触发
    → 采集现场快照（dmesg / gpu-smi / 内存 dump）
    → 比对历史模式
    → 生成根因报告
    → 创建运维工单（Jira / 内部 ITSM Webhook）
```

### 核心数据模型

```go
// 告警规则（存 PostgreSQL，规则表达式用 PromQL）
type AlertRule struct {
    ID             string   `db:"id"`
    Name           string   `db:"name"`
    Expr           string   `db:"expr"`        // PromQL 表达式
    Severity       string   `db:"severity"`    // warning|critical|emergency
    ForDuration    string   `db:"for_duration"`
    NotifyChannels []string `db:"notify_channels"`
    SilenceWindow  json.RawMessage `db:"silence_window"`
}

// 告警事件（存 PostgreSQL）
type Alert struct {
    ID                string    `db:"id"`
    RuleID            string    `db:"rule_id"`
    TargetID          string    `db:"target_id"`   // device_id 或 node_id
    FiredAt           time.Time `db:"fired_at"`
    ResolvedAt        time.Time `db:"resolved_at"`
    Status            string    `db:"status"`       // firing|resolved|silenced
    RootCauseAnalysis string    `db:"root_cause_analysis"`
}

// 诊断报告（存 PostgreSQL）
type DiagnosticReport struct {
    ID          string          `db:"id"`
    TargetType  string          `db:"target_type"`   // device|node|job
    TargetID    string          `db:"target_id"`
    TriggeredBy string          `db:"triggered_by"`  // manual|alert
    Checks      json.RawMessage `db:"checks"`        // [{name, result, detail}]
    Recommendation string       `db:"recommendation"`
}

// 指标序列（存 VictoriaMetrics，时序库）
// 标签维度：device_id / node_id / job_id
// metric_name 示例：gpu_utilization / gpu_memory_used_mb / gpu_temperature_celsius
//                   gpu_ecc_errors_total / gpu_power_watts / nvlink_bandwidth_gbps
```

### API 端点

```
GET  /api/v1/metrics/devices/{id}?metric=gpu_util&start=&end=&step=
GET  /api/v1/metrics/nodes/{id}
GET  /api/v1/metrics/jobs/{id}
GET  /api/v1/alerts/rules
POST /api/v1/alerts/rules
GET  /api/v1/alerts/events                   # 支持时间范围、状态过滤
POST /api/v1/alerts/events/{id}/ack
POST /api/v1/alerts/events/{id}/silence
POST /api/v1/diagnostics/run                 # body: {target_type, target_id}
GET  /api/v1/diagnostics/reports/{id}
WS   /ws/metrics/live?device_ids=            # 实时指标推送
```

---

## 模块四：任务生命周期管理

**任务提交与管理**

+ 支持训练任务（单机单卡、单机多卡、多机多卡）
+ 支持推理任务（在线服务部署、离线批量推理）
+ 支持 Notebook 交互式开发环境（JupyterHub 集成）
+ 任务状态机管理（Pending → Scheduling → Running → Succeeded/Failed/Killed）
+ 任务重试策略与失败回调

**作业提交规范**

+ 提交校验：镜像兼容性 + 配额检查 + 存储路径可达性，三项全通才入队
+ 资源预估：根据历史同类作业，给出 GPU 显存/时长预估（P50/P90）
+ 作业模板：可保存常用配置为模板，下次一键复用
+ 依赖作业：支持 DAG 依赖（作业 B 需等作业 A 成功后才提交）

**分布式训练支持**

+ 通信框架：NCCL（NVIDIA）/ HCCL（昇腾）自动注入，无需用户手动配置
+ 分布式策略声明：DP / MP / PP / SP 策略，平台根据策略分配对应拓扑的节点
+ 弹性训练：支持 Worker 数量在 [min, max] 范围内动态变化（Elastic DDP）
+ 通信性能诊断：自动检测 NCCL 通信瓶颈，建议是否开启 NVLink 优化

**镜像与环境管理**

+ 私有镜像仓库（Harbor 集成）
+ 预置官方镜像（CUDA×PyTorch、CANN×MindSpore 等常用组合）
+ 镜像构建流水线（基础镜像 + 用户依赖层）
+ 镜像安全扫描（Trivy/Clair）与合规准入

**数据集与存储管理**

+ 对接主流存储：Ceph / GPFS / NAS / 对象存储（S3 / OBS）
+ 数据集注册与版本管理
+ 数据预加载加速（缓存至本地 SSD / DRAM）
+ 数据访问权限与加密

**Checkpoint 管理**

+ 断点续训（自动保存 + 自动恢复）
+ Checkpoint 版本管理与去重存储
+ 跨集群迁移恢复支持
+ 大模型 Checkpoint 异步流式保存（减少训练暂停时间）
+ 异步 Checkpoint：训练继续，后台流式写入存储（适合大模型，减少暂停）
+ 去重存储：只存增量 diff（基于参数 hash 比较）
+ 多副本策略：重要 checkpoint 自动跨存储域双写
+ 恢复流程 UI：选择 checkpoint 版本 → 预览参数 → 一键恢复作业

**任务分析与优化建议**

+ 训练效率分析（GPU 利用率、通信/计算比、瓶颈识别）
+ 自动调参建议（batch size、并行策略推荐）
+ 成本估算（作业预计费用、资源消耗报告）

**推理任务专属**
+ 在线推理：部署为 HTTP 服务，自动配置健康检查、副本数、滚动更新策略
+ 批量推理：文件输入（CSV/JSON/图片包），分片并行，输出结果汇聚
+ 推理资源规格：支持 GPU 时间片（MIG/虚拟化），多个推理服务共享一张卡

### 技术选型

| 子功能 | 选型 | 备注 |
|---|---|---|
| 分布式训练编排 | **Kubeflow Training Operator** | PyTorchJob / TFJob / PaddleJob |
| 弹性训练 | **PyTorch Elastic（TorchX）** | Worker 数量 [min, max] 动态变化 |
| NCCL / HCCL 注入 | **Init Container（Go）** | 自动注入，用户零配置 |
| 在线推理部署 | **KServe** + **Triton Inference Server** | 滚动更新，MIG 时间片多服务共享 |
| 批量推理 | **Argo Workflows** | CSV/JSON 分片并行，结果汇聚 |
| DAG 依赖作业 | **Argo Workflows** | 作业 B 等作业 A 成功后触发 |
| Notebook | **JupyterHub on K8s** | 交互式开发环境 |
| 镜像仓库 | **Harbor** | 镜像安全扫描（Trivy），Cosign 签名 |
| 存储层 | **Ceph RBD / CephFS** + JuiceFS | 数据集版本管理用 DVC，预加载缓存至 SSD |
| Checkpoint 管理 | **自研 Checkpoint Service** + S3/OBS | 异步流式写入，参数 hash 增量去重 |
| 作业状态持久化 | **PostgreSQL** | Outbox 模式发 Kafka 驱动状态机 |
| 实时日志流 | **Loki** + WebSocket | tail 模式，stdout/stderr 分流 |

### 关键设计约束

```
- 提交校验三项全通才入队：
    1. 镜像兼容性（调用模块二兼容性矩阵）
    2. 配额检查（调用模块五配额引擎）
    3. 存储路径可达性

- Checkpoint 异步写入：训练继续，后台流式写入存储
- Checkpoint 去重：只存增量 diff（基于参数 hash 比较）
- Checkpoint 多副本：重要版本自动跨存储域双写

- 分布式训练策略声明：DP / MP / PP / SP
  平台根据策略分配对应拓扑节点（NVLink 域内优先）

- 推理资源规格：支持 GPU 时间片（MIG / 虚拟化），多服务共享一卡
```

### 核心数据模型

```go
type Job struct {
    ID           string          `db:"id"`
    Name         string          `db:"name"`
    TenantID     string          `db:"tenant_id"`
    ProjectID    string          `db:"project_id"`
    UserID       string          `db:"user_id"`
    Type         string          `db:"type"`   // training|inference_batch|inference_online|notebook
    Status       string          `db:"status"` // draft|queued|scheduling|running|paused|succeeded|failed|killed
    Priority     int             `db:"priority"` // 1-100
    ResourceSpec json.RawMessage `db:"resource_spec"`
    // {gpu_count, gpu_vendor, gpu_model, memory_mb, cpu_cores}
    Framework    string          `db:"framework"` // pytorch|tensorflow|mindspore|paddlepaddle
    ImageURI     string          `db:"image_uri"`
    Command      string          `db:"command"`
    RetryPolicy  json.RawMessage `db:"retry_policy"` // {max_retries, backoff_seconds}
    TimeoutSecs  int             `db:"timeout_seconds"`
    EstimatedCost float64        `db:"estimated_cost"`
}

type Checkpoint struct {
    ID          string          `db:"id"`
    JobID       string          `db:"job_id"`
    Step        int             `db:"step"`
    Epoch       int             `db:"epoch"`
    StoragePath string          `db:"storage_path"`
    SizeBytes   int64           `db:"size_bytes"`
    IsBest      bool            `db:"is_best"`
    Metadata    json.RawMessage `db:"metadata"` // {val_loss, val_acc, ...}
}

type Image struct {
    ID           string          `db:"id"`
    Name         string          `db:"name"`
    Tag          string          `db:"tag"`
    RegistryURL  string          `db:"registry_url"`
    Digest       string          `db:"digest"`
    SizeBytes    int64           `db:"size_bytes"`
    ScanStatus   string          `db:"scan_status"` // pending|passed|failed|skipped
    Vulnerabilities json.RawMessage `db:"vulnerabilities"`
}
```

### 作业状态机

```
draft ──[提交]──► queued
queued ──[调度器取走]──► scheduling
scheduling ──[设备分配成功]──► running
scheduling ──[gang 调度失败]──► queued          // 重回队列
running ──[用户暂停/被抢占]──► paused
paused ──[恢复]──► queued                       // 重新调度
running ──[exit code 0]──► succeeded
running ──[非零退出/超时/OOM]──► failed
failed ──[未超过 max_retries]──► queued         // 自动重试
running/queued/paused ──[用户终止]──► killed
```

### API 端点

```
POST   /api/v1/jobs                           # 提交作业
GET    /api/v1/jobs                           # 作业列表（过滤、分页、排序）
GET    /api/v1/jobs/{id}                      # 作业详情
DELETE /api/v1/jobs/{id}                      # 终止作业
POST   /api/v1/jobs/{id}/pause
POST   /api/v1/jobs/{id}/resume
GET    /api/v1/jobs/{id}/logs?stream=&tail=
WS     /ws/jobs/{id}/logs                     # 实时日志流
GET    /api/v1/jobs/{id}/metrics
GET    /api/v1/jobs/{id}/checkpoints
POST   /api/v1/jobs/{id}/checkpoints/{ck_id}/restore
POST   /api/v1/validate/job                   # 提交前预校验（不入队）
GET    /api/v1/images
POST   /api/v1/images/build
GET    /api/v1/images/{id}/scan-report
```

---

## 模块五：多租户与权限管理

**租户与项目体系**

+ 组织 → 租户 → 项目 → 用户 四级层次
+ 租户间资源完全隔离（网络、存储、调度队列）
+ 项目级配额继承与覆盖

**配额管理**

+ 资源配额：GPU 卡数、显存、CPU、内存、存储
+ 时间维度配额：日/周/月使用上限
+ 突发借用配额（Burst Quota）与优先级队列绑定
+ 配额超限告警与自动任务排队
+ 配额预检：作业提交时实时扣减预占（reserved），作业结束后结算实际用量
+ 借用机制：A 租户空闲配额可临时借给 B，借用到期自动回收，运行中作业不强制中断但不续期
+ 配额告警：使用率达到 80% 触发提醒，90% 触发告警，100% 禁止新作业提交
+ 历史用量趋势：按天/周/月聚合，支持按租户/项目/用户/GPU 型号多维下钻

**RBAC 权限控制**

+ 内置角色：平台管理员、租户管理员、普通用户、只读观察者
+ 自定义角色与细粒度权限定义（资源 × 操作）
+ SSO 集成（LDAP / AD / OAuth 2.0 / SAML）
+ API Key 管理（颁发、吊销、有效期控制）

**计量与计费**

+ 实时资源用量采集（精度到分钟级）
+ 按卡时、GPU 型号、网络带宽分维度计量
+ 内部成本归因报告（按租户/项目/用户/作业）
+ 对接财务系统（账单导出、预充值告警）

**计费模型细化**

+ 计费单价表：不同厂商/型号独立定价（A100 与 910B 单价不同）
+ 计费粒度：按实际占用时长计费（分钟级），不按申请时长
+ 内部结算报告：月度账单（PDF 导出），包含每个项目的明细 + 汇总
+ 预算预警：项目设置月度预算上限，预计超支时提前告警

**审计日志**

+ 全操作审计：登录、资源操作、权限变更、任务提交
+ 不可篡改审计日志存储
+ 审计日志检索与导出（合规报告）
  
**SSO 集成规范**

+ LDAP 属性映射：ou → tenant，cn → username，memberOf → role
+ OAuth 2.0 授权码流程，支持 PKCE（无后端客户端场景）
+ 会话管理：JWT 有效期 8h，Refresh Token 7 天，支持强制下线（吊销所有 token）
+ MFA：TOTP（Google Authenticator 兼容），管理员操作强制要求

### 技术选型

| 子功能 | 选型 | 备注 |
|---|---|---|
| 认证中心 | **Keycloak** | LDAP / AD / OAuth 2.0（PKCE）/ SAML |
| MFA | **TOTP**（Keycloak 内置） | Google Authenticator 兼容，管理员强制 |
| RBAC 引擎 | **Casbin（Go）** | resource × action × scope 三维，规则热加载 |
| Token 吊销 | **Redis 黑名单** | JWT 强制下线，Refresh Token 7 天 |
| API Key 管理 | **自研（PostgreSQL + HMAC）** | 颁发、吊销、有效期控制 |
| 配额引擎 | **自研 Quota Service（Go）** | 提交时预占 reserved，结束后结算实际用量 |
| 计量采集 | **VictoriaMetrics**（分钟级） | UsageRecord 写入 PostgreSQL |
| 账单报告 | **自研计费服务** + WeasyPrint | 月度账单 PDF 导出，对接财务 Webhook |
| 审计日志 | **ClickHouse（仅追加）** + Hash 链校验 | 不可篡改，支持 SIEM 对接 |

### 关键设计约束

```
- 组织层级：Organization → Tenant → Project → User

- 配额预检：
    提交时：used + reserved + 新申请 ≤ limit（否则返回 QUOTA_EXCEEDED）
    使用率告警阈值：80% 提醒 / 90% 告警 / 100% 禁止新作业提交

- 配额借用：
    A 租户空闲配额可借给 B，burst_until 到期自动回收
    运行中作业不强制中断，但不续期

- JWT 规格：
    access_token 有效期：8h
    refresh_token 有效期：7d
    强制下线：写入 Redis 黑名单，网关层验证

- SSO LDAP 属性映射：
    ou → tenant
    cn → username
    memberOf → role

- 审计日志字段：actor / action / resource / timestamp / result
  ClickHouse 仅追加表，Hash 链防篡改
```

### 核心数据模型

```go
type Quota struct {
    ID           string    `db:"id"`
    ScopeType    string    `db:"scope_type"`  // platform|tenant|project
    ScopeID      string    `db:"scope_id"`
    ResourceType string    `db:"resource_type"` // gpu_count|gpu_memory_mb|cpu_cores|storage_gb
    Limit        int64     `db:"limit"`
    Used         int64     `db:"used"`
    Reserved     int64     `db:"reserved"`
    BurstLimit   int64     `db:"burst_limit"`
    BurstUntil   time.Time `db:"burst_until"`
}

type UsageRecord struct {
    JobID       string    `db:"job_id"`
    TenantID    string    `db:"tenant_id"`
    ProjectID   string    `db:"project_id"`
    UserID      string    `db:"user_id"`
    DeviceType  string    `db:"device_type"`
    DeviceCount int       `db:"device_count"`
    StartAt     time.Time `db:"start_at"`
    EndAt       time.Time `db:"end_at"`
    GPUSeconds  int64     `db:"gpu_seconds"`
    CostAmount  float64   `db:"cost_amount"`
    Currency    string    `db:"currency"`
}

type RoleBinding struct {
    UserID    string `db:"user_id"`
    RoleID    string `db:"role_id"`
    ScopeType string `db:"scope_type"` // platform|tenant|project
    ScopeID   string `db:"scope_id"`
}
```

### API 端点

```
POST /api/v1/auth/login
POST /api/v1/auth/oauth/callback
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
GET  /api/v1/tenants
POST /api/v1/tenants
GET  /api/v1/tenants/{id}/quotas
PUT  /api/v1/tenants/{id}/quotas
GET  /api/v1/tenants/{id}/usage?period=
GET  /api/v1/projects
POST /api/v1/projects
GET  /api/v1/users
POST /api/v1/users/invite
GET  /api/v1/roles
POST /api/v1/role-bindings
GET  /api/v1/audit-logs?actor=&action=&resource=&start=&end=
GET  /api/v1/billing/bills?tenant=&month=
```

---

## 模块六：开放 API 与运营管理

**API 网关层**

+ 统一鉴权中间件（JWT 验证 + API Key 验证二选一）
+ 请求限流：租户级（1000 req/min），用户级（100 req/min），IP 级（200 req/min）
+ 请求追踪：每个请求注入 X-Request-ID，全链路透传到后端服务
+ API 版本管理：/api/v1/ 长期支持，/api/v2/ 灰度，旧版本 Deprecation 通知机制
+ OpenAPI Spec 自动生成并托管（/api/docs），SDK 从 Spec 自动生成

**对外接口**

+ RESTful API（OpenAPI 3.0 规范，含 SDK 自动生成）
+ gRPC API（高频调用场景）
+ Webhook 事件推送（任务状态变更、告警触发等）
+ K8s CRD 扩展（原生 K8s 生态集成）

**控制台 Web UI**

+ 资源总览仪表盘（算力总量、利用率、排队情况）
+ 任务列表与详情（日志流式查看、指标图表）
+ 设备管理（节点列表、卡级详情、故障记录）
+ 用户与配额管理界面
+ 成本报告与趋势分析

**CLI / SDK**

+ 命令行工具（提交任务、查看状态、下载日志）
+ Python SDK / Go SDK
+ Terraform Provider（基础设施即代码）

**平台运营工具**

+ 灰度发布 / 蓝绿部署（平台自身组件升级）
+ 配置中心（运行参数热更新）
+ 巡检任务（定期执行硬件健康检测）
+ 容量规划看板（未来 N 天资源缺口预测）
+ 平台组件健康检查端点（/healthz，/readyz），接入 K8s Liveness/Readiness
+ 功能开关（Feature Flag）：可按租户/用户灰度放量新功能
+ 运维操作日志：平台管理员的所有变更操作记录，与业务审计日志分开存储
+ 数据库迁移管理：使用 Migration 框架（Flyway/Liquibase），版本可回滚

**Web控制台清单**
```
/dashboard              总览仪表盘（算力大盘、今日用量、待处理告警）
/clusters               集群管理（节点列表、机架视图）
/clusters/{id}/devices  节点设备列表
/pools                  资源池管理
/jobs                   作业列表
/jobs/new               提交作业（向导式表单，4步）
/jobs/{id}              作业详情（状态、日志、指标、checkpoint）
/images                 镜像管理
/monitoring/alerts      告警事件列表
/monitoring/rules       告警规则配置
/tenants                租户管理（平台管理员）
/tenants/{id}/quotas    配额管理
/billing                计费报告
/settings/profile       个人设置
/settings/api-keys      API Key 管理
/settings/sso           SSO 配置（管理员）
/audit                  审计日志
```

### 技术选型

| 子功能 | 选型 | 备注 |
|---|---|---|
| API 网关 | **Kong / APISIX** | 限流插件，X-Request-ID 透传，JWT + API Key 鉴权 |
| RESTful API 框架 | **Go（Gin / Echo）** + OpenAPI 3.0 | /api/docs 自动托管，SDK 自动生成 |
| gRPC | **gRPC-Go** + Protobuf | 调度器 ↔ Agent ↔ HAL 高频内部通信 |
| Webhook 推送 | **Kafka** + 自研 Dispatcher | HMAC-SHA256 签名验证，事件外推 |
| K8s CRD 扩展 | **controller-runtime（Go）** | TrainingJob / ResourcePool CRD |
| 配置中心 | **Apollo / Nacos** | 运行参数热更新，Feature Flag 按租户灰度 |
| 数据库迁移 | **Flyway** | 版本可回滚，CI 自动执行 |
| 服务网格 | **Istio** | 全链路 mTLS，容器网络隔离 |
| 灰度 / 蓝绿发布 | **Argo Rollouts** | 平台自身组件升级 |
| IaC | **自研 Terraform Provider**（Go SDK 封装） | 资源池 / 配额声明式管理 |

### 关键设计约束

```
- 限流规格：
    租户级：1000 req/min
    用户级：100 req/min
    IP 级：200 req/min

- API 版本策略：
    /api/v1/ 长期支持（LTS）
    /api/v2/ 灰度
    旧版本 Deprecation 头通知（Sunset 响应头）

- Webhook 事件类型：
    job.status_changed / alert.fired / alert.resolved
    node.offline / node.recovered / quota.exceeded

- 健康检查端点：
    /healthz   → K8s Liveness
    /readyz    → K8s Readiness

- Web 控制台路由清单：
    /dashboard                  总览仪表盘
    /clusters                   集群管理
    /clusters/{id}/devices      节点设备列表
    /pools                      资源池管理
    /jobs                       作业列表
    /jobs/new                   提交作业（4 步向导）
    /jobs/{id}                  作业详情（状态/日志/指标/checkpoint）
    /images                     镜像管理
    /monitoring/alerts          告警事件
    /monitoring/rules           告警规则
    /tenants                    租户管理（平台管理员）
    /tenants/{id}/quotas        配额管理
    /billing                    计费报告
    /settings/api-keys          API Key 管理
    /settings/sso               SSO 配置
    /audit                      审计日志
```

---

## 横切：安全与合规基座

| 领域 | 选型 | 关键配置 |
|---|---|---|
| 传输安全 | **Istio mTLS** | 支持国密 SM2/SM4 扩展 |
| 密钥管理 | **HashiCorp Vault** | KMS 集成，密钥自动轮换 |
| 存储加密 | **Ceph 落盘加密** + Vault | 密钥存 Vault，定期轮换 |
| 显存清除 | **HAL memset 归零**（强制） | 任务结束释放时执行，不可跳过 |
| 镜像安全 | **Trivy**（CVE 扫描）+ **Cosign**（签名）| 高危漏洞自动阻断准入，SBOM 生成 |
| 不可篡改审计 | **ClickHouse 仅追加** + Hash 链 | 支持 Splunk / Elasticsearch SIEM 对接 |
| 合规认证目标 | 等保 2.0 三级 / ISO 27001 / SOC 2 | 条款映射文档另维护 |

---

## 全局技术栈速查表

### 后端

| 类别 | 技术 |
|---|---|
| 主要语言 | Go（服务层）、C++ 17（HAL 层）、Python（脚本/工具） |
| Web 框架 | Gin / Echo |
| gRPC | gRPC-Go + Protobuf |
| 关系型 DB | PostgreSQL（主数据）|
| 时序 DB | VictoriaMetrics |
| 列式 DB | ClickHouse（审计日志）|
| 缓存 | Redis（Token 黑名单 / 热数据）|
| 消息队列 | Kafka + CloudEvents |
| 服务注册 | etcd |
| 任务编排 | Volcano、Kubeflow Training Operator、Argo Workflows |
| 容器运行时 | containerd |
| 服务网格 | Istio |
| 配置中心 | Apollo / Nacos |
| 数据库迁移 | Flyway |
| 密钥管理 | HashiCorp Vault |

### 前端

| 类别 | 技术 |
|---|---|
| 框架 | React 18 + TypeScript |
| UI 组件库 | Ant Design Pro |
| 图表 | ECharts |
| 拓扑图 | AntV G6 |
| 实时数据 | WebSocket + SWR / React Query |

### 基础设施 & 可观测

| 类别 | 技术 |
|---|---|
| 容器编排 | Kubernetes |
| 调度扩展 | Volcano |
| 弹性伸缩 | K8s Cluster Autoscaler |
| 镜像仓库 | Harbor |
| 存储 | Ceph RBD / CephFS + JuiceFS（S3 代理）|
| 指标采集 | DCGM Exporter + Prometheus Node Exporter |
| 时序存储 | VictoriaMetrics |
| 日志 | Loki + Promtail |
| 追踪 | OpenTelemetry + Jaeger |
| Dashboard | Grafana |
| 告警 | VMAlert + Alertmanager |
| 灰度发布 | Argo Rollouts |
| IaC | Terraform（自研 Provider）|

---

*本文档版本：v1.0 · 对应 Infra-detail.md 初版*
