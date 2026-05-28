# AI-Infra Platform Technology Selection  
> Each module includes: technology selection, selection rationale, core data models, key API endpoints, and considerations.

---

## Table of Contents

- [AI-Infra Platform Technology Selection](#ai-infra-platform-technology-selection)
  - [Table of Contents](#table-of-contents)
  - [Module 1: Resource Management and Scheduling](#module-1-resource-management-and-scheduling)
    - [Technology Selection](#technology-selection)
    - [Key Design Constraints](#key-design-constraints)
    - [Core Data Models](#core-data-models)
    - [Device State Machine](#device-state-machine)
    - [API Endpoints](#api-endpoints)
  - [Module 2: Runtime and Driver Adaptation Layer](#module-2-runtime-and-driver-adaptation-layer)
    - [Technology Selection](#technology-selection-1)
    - [Key Design Constraints](#key-design-constraints-1)
    - [Core Data Models](#core-data-models-1)
    - [API Endpoints](#api-endpoints-1)
  - [Module 3: Monitoring and Observability](#module-3-monitoring-and-observability)
    - [Technology Selection](#technology-selection-2)
    - [Key Design Constraints](#key-design-constraints-2)
    - [Core Data Models](#core-data-models-2)
    - [API Endpoints](#api-endpoints-2)
  - [Module 4: Task Lifecycle Management](#module-4-task-lifecycle-management)
    - [Technology Selection](#technology-selection-3)
    - [Key Design Constraints](#key-design-constraints-3)
    - [Core Data Models](#core-data-models-3)
    - [Job State Machine](#job-state-machine)
    - [API Endpoints](#api-endpoints-3)
  - [Module 5: Multi-Tenancy and Permission Management](#module-5-multi-tenancy-and-permission-management)
    - [Technology Selection](#technology-selection-4)
    - [Key Design Constraints](#key-design-constraints-4)
    - [Core Data Models](#core-data-models-4)
    - [API Endpoints](#api-endpoints-4)
  - [Module 6: Open API and Operation Management](#module-6-open-api-and-operation-management)
    - [Technology Selection](#technology-selection-5)

---

## Module 1: Resource Management and Scheduling

**Device Discovery and Registration**

+ Automatically scan and register GPU/NPU nodes in the cluster (NVIDIA, Huawei Ascend, Cambricon, Enflame, etc.)
+ Collect device metadata: vendor, model, video memory size, driver version, firmware version, PCIe/NVLink topology
+ Maintain node health status (online/offline/fault/maintenance)
+ Perceive dynamic online/offline of devices and trigger alerts
+ Node Agent actively reports heartbeat (interval 30s, timeout 90s to determine offline), heartbeat Payload contains full device snapshot
+ Device fingerprint deduplication (vendor + model + serial_number triple is unique)
+ Publish node online/offline events to message bus (subscribed by downstream monitoring module)
+ Device tag system (kv tags, support filtering and scheduling by tags)

**Resource Pooling Management**

+ Build multi-dimensional resource pools by vendor, model, computer room, and business line
+ Support GPU virtualization splitting (MIG for NVIDIA, vDVPP for Ascend)
+ Joint resource modeling for CPU/memory/storage/network bandwidth
+ Resource pool capacity planning and prediction (based on historical usage trends)
+ Pool creation rules support AND/OR condition combination (vendor=nvidia AND model=A100 OR H800)
+ Default pool as fallback (devices not matching any custom pool are classified into the default pool)
+ Pool oversubscription coefficient configuration (allow 110% overselling for inference mixed deployment)
+ Pool priority inheritance (sub-pools inherit parent pool priority, which can be overridden)

**Intelligent Scheduling Engine**

+ Multi-policy scheduling: priority queue, Gang Scheduling (full allocation for distributed training), Preemptive Scheduling, Fair Scheduling (Fair Share)
+ Affinity/anti-affinity rules (same machine, cross-machine, cross-rack)
+ Network topology-aware scheduling (prioritize allocation within the same NVSwitch domain / RDMA domain)
+ Cross-vendor heterogeneous job mixed deployment scheduling (automatic model selection based on framework compatibility)
+ Scheduler plugin extension interface (Webhook / Plugin mechanism)
+ Gang Scheduling: full device atomic allocation at one time, block and wait if partially satisfied, support timeout abandonment (timeout_seconds configurable)
+ Preemptive Scheduling: low-priority tasks release devices to high-priority ones, preempted tasks automatically re-enter the queue, and save the current checkpoint path
+ Fragment awareness: when the remaining number of GPUs on the same node < minimum allocation unit, prioritize filling the node (Bin Packing)
+ Topology score: +20 points for allocation within the same NVLink domain, -10 points for cross-RDMA domain, the scheduler selects the node with the highest score
+ Enumeration of scheduling failure reasons: INSUFFICIENT_RESOURCE / NO_COMPATIBLE_DEVICE / QUOTA_EXCEEDED / GANG_TIMEOUT

**Elastic Scaling**

+ Automatically trigger node expansion based on queue water level (integrated with K8s Cluster Autoscaler)
+ Dynamic increase/decrease of elastic workers for training tasks (Elastic Training)
+ Resource recycling and borrowing strategies during idle periods
+ Cross-cluster / cross-cloud resource federated scheduling
  
**UI Pages/Components**
+ Cluster Overview Page: node card matrix, color-coded status, support grouping by rack
+ Device Details Drawer: single-card real-time metric line chart + historical allocation records
+ Resource Pool Management Page: rule editor (tag selector UI) + capacity preview
+ Scheduling Queue Page: real-time refreshing task queue list, support manual adjustment of priority

### Technology Selection

| Sub-function | Selection | Remarks |
|---|---|---|
| Node Agent | **Go** + gRPC | Heartbeat 30s, timeout 90s, full device snapshot |
| Device Metadata Storage | **PostgreSQL** + JSONB | PCIe/NVLink topology stored in JSONB, tag filtering covered by indexes |
| Device Online/Offline Events | **Kafka** + CloudEvents | At-least-once delivery, consumed by monitoring module |
| Service Registration / Offline Perception | **etcd** | Agent lease registration, scheduler Watch |
| Scheduler Core | **Volcano** (K8s extension) | Gang / Fair-Share / Preemption, natively supported |
| Elastic Scaling | **K8s Cluster Autoscaler** | Triggered by queue water level, integrated with public cloud node pools |
| Resource Pool Rule Engine | **Self-developed DSL (Go)** | AND/OR tag conditions, default pool fallback, oversubscription coefficient |
| Topology Scoring Plugin | **Scheduler Plugin (Go)** | NVLink +20 / cross-RDMA −10, Bin Packing |
| Frontend Framework | **React 18 + TypeScript** | Ant Design Pro, ECharts, AntV G6 topology graph |
| Real-Time Data | **WebSocket** + SWR | Real-time refresh of node matrix, device details line chart |

### Key Design Constraints

```
- Device fingerprint deduplication: vendor + model + serial_number triple is unique
- Gang Scheduling: full atomic allocation, timeout abandonment (timeout_seconds configurable)
- Preemptive Scheduling: low-priority release → preempted tasks automatically save checkpoint path → re-enter queue
- Fragment awareness: prioritize Bin Packing to fill the node when remaining GPU on the same node < minimum allocation unit
- Scheduling failure enumeration: INSUFFICIENT_RESOURCE | NO_COMPATIBLE_DEVICE | QUOTA_EXCEEDED | GANG_TIMEOUT
```

### Core Data Models

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

### Device State Machine

```
available ──[Task Allocation]──► allocated
allocated ──[Task Completion]──► releasing
releasing ──[Video Memory Zeroing Completed]──► available
available/allocated ──[Health Check Failed]──► fault
fault ──[Manual Confirmation]──► maintenance
maintenance ──[Repair Completed]──► available
```

### API Endpoints

```
GET    /api/v1/nodes                         # Node list (filtering, pagination)
GET    /api/v1/nodes/{id}                    # Node details + device list
PATCH  /api/v1/nodes/{id}/status             # Manually set node status (maintenance mode)
GET    /api/v1/devices                       # Device list
GET    /api/v1/devices/{id}                  # Device details
GET    /api/v1/pools                         # Resource pool list
POST   /api/v1/pools                         # Create resource pool
PUT    /api/v1/pools/{id}                    # Update resource pool rules
POST   /api/v1/allocations                   # Apply for allocation (scheduling trigger)
DELETE /api/v1/allocations/{id}              # Release allocation
GET    /api/v1/scheduler/queue               # Current scheduling queue
GET    /api/v1/scheduler/decisions/{job_id}  # Scheduling decision details + reasons
```

---

## Module 2: Runtime and Driver Adaptation Layer

**Unified Hardware Abstraction Layer (HAL)**

+ Abstract runtime differences of CUDA / CANN (Ascend) / CNRT (Cambricon) / TOPSRT (Enflame)
+ Unified API for device enumeration, memory allocation, data transmission, and stream synchronization
+ Driver version matrix management and compatibility verification
+ Device enumeration interface: hal.list_devices() → returns unified DeviceInfo structure (shielding vendor differences)
+ Memory management: hal.alloc(device_id, size_mb) / hal.free(handle) / hal.memset(handle, 0) (video memory erasure forced to zero)
+ Data transmission: hal.h2d(src, dst, size) / hal.d2h / hal.d2d (need to judge if cross-device is from the same vendor)
+ Event synchronization: hal.create_stream() / hal.sync_stream() / hal.create_event()
+ Unified error code mapping table: original error codes of each vendor → platform unified error codes (e.g., CUDA error 2 = CANN error 10 = platform ERR_OOM)

**Driver Version Matrix Management**
+ Entry and query of three-dimensional compatibility matrix of Driver × Framework × Device Model
+ Automatic verification before job submission: framework version in image + driver version of target node → return compatibility rating (OK/WARN/BLOCK)
+ Provide recommended alternatives when incompatible (recommended driver version or recommended image)
+ Compatibility matrix change notification (automatically recalculate the list of affected jobs after driver upgrade)

**Multi-Framework Compatibility Adaptation**

+ Support mainstream frameworks such as PyTorch, TensorFlow, MindSpore, PaddlePaddle
+ Maintain Framework × Hardware compatibility matrix (automatically block submission of incompatible combinations)
+ Custom operator registration and cross-hardware bridging

**Operator Library and Algorithm Library Management**

+ Version management of CUDA cuDNN / cuBLAS, Huawei Ascend CANN operator libraries
+ Operator performance Benchmark library (benchmark test data of the same operator on different hardware)
+ Custom operator upload, verification, and release process

**Container Runtime Integration**

+ Device Plugin management (expose GPU/NPU resources to K8s)
+ Driver/runtime injection in container images (avoid strong coupling between images and drivers)
+ Runtime environment isolation (driver environments between different jobs do not interfere with each other)
+ Device Plugin implementation: K8s resource name specification (vendor.com/gpu, such as nvidia.com/gpu, huawei.com/npu)
+ Injection hook before container startup: mount the corresponding vendor runtime (--runtime=nvidia / --runtime=cann)
+ Multi-card topology injection: environment variables CUDA_VISIBLE_DEVICES / ASCEND_VISIBLE_DEVICES set automatically
+ Fault isolation: single container crash does not affect other containers on the same node, device status reset automatically

### Technology Selection

| Sub-function | Selection | Remarks |
|---|---|---|
| HAL Core Implementation | **C++ 17** + Go CGO Bridge | Shield differences of CUDA / CANN / CNRT / TOPSRT |
| NVIDIA Runtime | **CUDA Runtime** + NVML | Device enumeration, video memory alloc/free/memset |
| Huawei Ascend Runtime | **CANN** + ACL | Separate collection of three cores (AI Core / Vector / Cube) |
| Cambricon / Enflame | **CNRT** / **TOPSRT** | Unified DeviceInfo structure to shield differences |
| Error Code Mapping | **Unified ERR_* Enumeration (Go)** | Original codes of each vendor → platform unified codes |
| Compatibility Matrix Storage | **PostgreSQL** | Three-dimensional table of driver × framework × device_model |
| Container Runtime | **containerd** + vendor runtime plugins | nvidia-container-runtime / cann-container-runtime |
| Device Plugin | **K8s Device Plugin (Go)** | nvidia.com/gpu, huawei.com/npu |
| Operator Library Version Management | **Harbor** (OCI) | cuDNN / CANN operator packages, Benchmark results stored in DB |

### Key Design Constraints

```
- HAL Interface Specification:
    hal.list_devices() → []DeviceInfo
    hal.alloc(device_id, size_mb) → handle
    hal.free(handle)
    hal.memset(handle, 0)          // Video memory zeroing, mandatory execution
    hal.h2d / hal.d2h / hal.d2d   // Need to judge if cross-vendor is from the same vendor

- Compatibility Rating: OK | WARN | BLOCK
  Automatic verification before job submission: framework version in image + driver version of target node

- Device Plugin Injection:
    CUDA_VISIBLE_DEVICES    # NVIDIA
    ASCEND_VISIBLE_DEVICES  # Ascend
    Automatically set, no perception for users

- Fault Isolation: single container crash does not affect other containers on the same node, device status reset automatically
```

### Core Data Models

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

### API Endpoints

```
GET  /api/v1/drivers                          # Driver profile list
POST /api/v1/drivers                          # Enter new driver configuration
GET  /api/v1/compatibility/check              # Real-time compatibility verification
                                              # ?framework=&version=&device_model=
GET  /api/v1/compatibility/matrix             # Complete compatibility matrix (support CSV export)
GET  /api/v1/operators                        # Operator library list
POST /api/v1/operators/upload                 # Upload custom operator package
GET  /api/v1/operators/{id}/benchmarks        # Operator benchmark test results
POST /api/v1/operators/{id}/benchmark/run     # Trigger benchmark test (async, return task_id)
```

---

## Module 3: Monitoring and Observability

**Real-Time Metric Collection**

+ GPU-level: utilization rate, video memory usage/bandwidth, power consumption, temperature, ECC error count, NVLink bandwidth
+ Huawei Ascend NPU proprietary metrics: AI Core utilization rate, HBM bandwidth, D2D interconnection utilization rate
+ Node-level: CPU, memory, network I/O, storage I/O
+ Cluster-level: total computing power utilization rate, queue depth, resource fragmentation rate
+ Hierarchical collection frequency: key metrics (utilization rate, temperature, ECC) 10s; general metrics (power consumption, memory bandwidth) 30s; slow metrics (driver version) 5min
+ Ascend proprietary metrics: AI Core utilization rate, Vector Core utilization rate, Cube Core utilization rate (three-core separate collection)
+ Metric missing detection: trigger collector exception alert if no data for 3 consecutive collection cycles
+ Custom metric reporting interface (business parties can Push custom metrics to mount to the job dimension)

**Automated Fault Diagnosis and Alerting**

+ Multi-level alert rules (Warning / Critical / Emergency)
+ GPU Xid error parsing (NVIDIA), CANN fault code parsing (Ascend)
+ Automatic isolation + ticket creation (faulty nodes automatically removed, trigger operation and maintenance tickets)
+ GPU Xid error dictionary (NVIDIA 74 types of Xid comparison tables, including severity and handling suggestions)
+ Ascend fault code dictionary (CANN ErrCode comparison and handling suggestions)
+ Automatic diagnosis process: alert trigger → collect on-site snapshot (dmesg, gpu-smi, memory dump) → compare historical patterns → generate root cause report
+ Multi-card correlation analysis: increase in single-card ECC error rate → health scan of other cards on the same machine → judge if it is a machine-level fault
+ Fault self-healing: configurable self-healing actions (restart container/isolate device/notify operation and maintenance), manual confirmation required for level classification

**Alert Management**
+ Alert convergence and root cause analysis (single-card fault vs machine fault vs rack fault)
+ Alert notification channels: email, WeCom/DingTalk, Webhook, PagerDuty
+ Alert aggregation (merge the same type of alerts for the same device within 5min into one)
+ Alert routing rules (route to different notification channels according to severity, device type, and tenant)
+ On-call escalation chain (Warning notifies level 1, escalates to Critical if unACKed for 30min, then escalates to Emergency after another 30min)
+ Silence window (automatically silence during maintenance window, resume after maintenance ends)

**Topology Visualization**

+ Hierarchical topology graph of computer room → rack → node → GPU card
+ NVLink / NVSwitch / PCIe interconnection topology visualization
+ Heat map display (utilization rate, temperature, fault distribution)

**Full-Link Observability**

+ Trinity of metrics (Prometheus/VictoriaMetrics), logs (Loki/ELK), and tracing (OpenTelemetry/Jaeger)
+ Job-level Profiling data collection (aggregation of NVIDIA Nsight / Ascend Profiler results)
+ Custom Dashboard configuration and sharing (Grafana integration)

### Technology Selection

| Sub-function | Selection | Remarks |
|---|---|---|
| GPU Metric Collection | **DCGM Exporter** + Prometheus Node Exporter | NVIDIA-specific, covering Xid / ECC / NVLink |
| Ascend Metric Collection | **Self-developed Ascend Exporter (Go)** | Separate collection of AI Core / Vector Core / Cube Core |
| Time-Series Database | **VictoriaMetrics** | High cardinality optimization, native PromQL, raw data for 7 days / 1min aggregation for 90 days |
| Logs | **Loki** + Promtail | Unified with Prometheus label system, Grafana associated query |
| Distributed Tracing | **OpenTelemetry SDK** + Jaeger | X-Request-ID full-link transparent transmission |
| Dashboard | **Grafana** | Heat map / topology graph panel, custom Dashboard sharing |
| Alert Engine | **VMAlert** (PromQL expression) | for_duration anti-shake, three-level alerts |
| Notification Routing | **Alertmanager** | WeCom / DingTalk / PagerDuty / Webhook |
| Fault Diagnosis Service | **Self-developed Diagnostic Service (Go)** | Xid / CANN error code dictionary, automatic root cause report |
| Profiling Aggregation | **NVIDIA Nsight** / **Ascend Profiler** + Pyroscope | Job-level Profiling data aggregation |

### Key Design Constraints

```
- Hierarchical collection frequency:
    Key metrics (utilization rate / temperature / ECC): 10s
    General metrics (power consumption / memory bandwidth): 30s
    Slow metrics (driver version): 5min

- Metric missing detection: no data for 3 consecutive collection cycles → trigger collector exception alert

- Alert convergence: merge the same type of alerts for the same device within 5min into one

- On-call escalation chain:
    Warning → notify level 1
    UnACKed for 30min → escalate to Critical
    Another 30min → escalate to Emergency

- Automatic diagnosis process:
    Alert trigger
    → collect on-site snapshot (dmesg / gpu-smi / memory dump)
    → compare historical patterns
    → generate root cause report
    → create operation and maintenance ticket (Jira / internal ITSM Webhook)
```

### Core Data Models

```go
// Alert Rule (stored in PostgreSQL, rule expressions use PromQL)
type AlertRule struct {
    ID             string   `db:"id"`
    Name           string   `db:"name"`
    Expr           string   `db:"expr"`        // PromQL expression
    Severity       string   `db:"severity"`    // warning|critical|emergency
    ForDuration    string   `db:"for_duration"`
    NotifyChannels []string `db:"notify_channels"`
    SilenceWindow  json.RawMessage `db:"silence_window"`
}

// Alert Event (stored in PostgreSQL)
type Alert struct {
    ID                string    `db:"id"`
    RuleID            string    `db:"rule_id"`
    TargetID          string    `db:"target_id"`   // device_id or node_id
    FiredAt           time.Time `db:"fired_at"`
    ResolvedAt        time.Time `db:"resolved_at"`
    Status            string    `db:"status"`       // firing|resolved|silenced
    RootCauseAnalysis string    `db:"root_cause_analysis"`
}

// Diagnostic Report (stored in PostgreSQL)
type DiagnosticReport struct {
    ID          string          `db:"id"`
    TargetType  string          `db:"target_type"`   // device|node|job
    TargetID    string          `db:"target_id"`
    TriggeredBy string          `db:"triggered_by"`  // manual|alert
    Checks      json.RawMessage `db:"checks"`        // [{name, result, detail}]
    Recommendation string       `db:"recommendation"`
}

// Metric Series (stored in VictoriaMetrics, time-series database)
// Tag dimensions: device_id / node_id / job_id
// metric_name examples: gpu_utilization / gpu_memory_used_mb / gpu_temperature_celsius
//                   gpu_ecc_errors_total / gpu_power_watts / nvlink_bandwidth_gbps
```

### API Endpoints

```
GET  /api/v1/metrics/devices/{id}?metric=gpu_util&start=&end=&step=
GET  /api/v1/metrics/nodes/{id}
GET  /api/v1/metrics/jobs/{id}
GET  /api/v1/alerts/rules
POST /api/v1/alerts/rules
GET  /api/v1/alerts/events                   // Support time range and status filtering
POST /api/v1/alerts/events/{id}/ack
POST /api/v1/alerts/events/{id}/silence
POST /api/v1/diagnostics/run                 // body: {target_type, target_id}
GET  /api/v1/diagnostics/reports/{id}
WS   /ws/metrics/live?device_ids=            // Real-time metric push
```

---

## Module 4: Task Lifecycle Management

**Task Submission and Management**

+ Support training tasks (single-machine single-card, single-machine multi-card, multi-machine multi-card)
+ Support inference tasks (online service deployment, offline batch inference)
+ Support Notebook interactive development environment (JupyterHub integration)
+ Task state machine management (Pending → Scheduling → Running → Succeeded/Failed/Killed)
+ Task retry strategy and failure callback

**Job Submission Specification**

+ Submission verification: image compatibility + quota check + storage path reachability, all three pass to enter the queue
+ Resource estimation: give GPU video memory/duration estimation (P50/P90) according to historical similar jobs
+ Job templates: save common configurations as templates for one-click reuse next time
+ Dependent jobs: support DAG dependencies (Job B can only be submitted after Job A succeeds)

**Distributed Training Support**

+ Communication framework: NCCL (NVIDIA)/HCCL (Ascend) automatically injected, no manual configuration required for users
+ Distributed strategy declaration: DP / MP / PP / SP strategies, the platform allocates nodes with corresponding topology according to the strategy
+ Elastic training: support dynamic changes of Worker quantity within [min, max] range (Elastic DDP)
+ Communication performance diagnosis: automatically detect NCCL communication bottlenecks, suggest whether to enable NVLink optimization

**Image and Environment Management**

+ Private image repository (Harbor integration)
+ Pre-built official images (common combinations such as CUDA×PyTorch, CANN×MindSpore)
+ Image build pipeline (base image + user dependency layer)
+ Image security scanning (Trivy/Clair) and compliance admission

**Dataset and Storage Management**

+ Integrate with mainstream storage: Ceph / GPFS / NAS / object storage (S3 / OBS)
+ Dataset registration and version management
+ Data preloading acceleration (cache to local SSD / DRAM)
+ Data access permissions and encryption

**Checkpoint Management**

+ Checkpoint resume (automatic saving + automatic recovery)
+ Checkpoint version management and deduplication storage
+ Cross-cluster migration and recovery support
+ Asynchronous streaming saving of large model Checkpoint (reduce training pause time)
+ Asynchronous Checkpoint: training continues, background streaming write to storage (suitable for large models, reduce pause time)
+ Deduplication storage: only store incremental diff (based on parameter hash comparison)
+ Multi-replica strategy: important checkpoints automatically written to dual storage domains
+ Recovery process UI: select checkpoint version → preview parameters → one-click recovery of jobs

**Task Analysis and Optimization Suggestions**

+ Training efficiency analysis (GPU utilization rate, communication/computation ratio, bottleneck identification)
+ Automatic parameter tuning suggestions (batch size, parallel strategy recommendation)
+ Cost estimation (estimated job cost, resource consumption report)

**Inference Task Exclusive**
+ Online inference: deployed as HTTP service, automatically configure health check, replica count, rolling update strategy
+ Batch inference: file input (CSV/JSON/image package), sharded parallelism, output result aggregation
+ Inference resource specifications: support GPU time slices (MIG/virtualization), multiple inference services share one card

### Technology Selection

| Sub-function | Selection | Remarks |
|---|---|---|
| Distributed Training Orchestration | **Kubeflow Training Operator** | PyTorchJob / TFJob / PaddleJob |
| Elastic Training | **PyTorch Elastic (TorchX)** | Dynamic change of Worker quantity in [min, max] range |
| NCCL / HCCL Injection | **Init Container (Go)** | Automatic injection, zero configuration for users |
| Online Inference Deployment | **KServe** + **Triton Inference Server** | Rolling update, multi-service sharing via MIG time slices |
| Batch Inference | **Argo Workflows** | CSV/JSON sharded parallelism, result aggregation |
| DAG Dependent Jobs | **Argo Workflows** | Job B triggered after Job A succeeds |
| Notebook | **JupyterHub on K8s** | Interactive development environment |
| Image Repository | **Harbor** | Image security scanning (Trivy), Cosign signature |
| Storage Layer | **Ceph RBD / CephFS** + JuiceFS | Dataset version management with DVC, preloading cache to SSD |
| Checkpoint Management | **Self-developed Checkpoint Service** + S3/OBS | Asynchronous streaming write, parameter hash incremental deduplication |
| Job Status Persistence | **PostgreSQL** | Outbox pattern to send Kafka to drive state machine |
| Real-Time Log Streaming | **Loki** + WebSocket | tail mode, stdout/stderr shunting |

### Key Design Constraints

```
- Three items must all pass for submission verification to enter the queue:
    1. Image compatibility (call Module 2 compatibility matrix)
    2. Quota check (call Module 5 quota engine)
    3. Storage path reachability

- Checkpoint asynchronous writing: training continues, background streaming write to storage
- Checkpoint deduplication: only store incremental diff (based on parameter hash comparison)
- Checkpoint multi-replica: important versions automatically written to dual storage domains

- Distributed training strategy declaration: DP / MP / PP / SP
  The platform allocates nodes with corresponding topology according to the strategy (priority within NVLink domain)

- Inference resource specifications: support GPU time slices (MIG / virtualization), multiple services share one card
```

### Core Data Models

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

### Job State Machine

```
draft ──[Submit]──► queued
queued ──[Taken by Scheduler]──► scheduling
scheduling ──[Device Allocation Successful]──► running
scheduling ──[Gang Scheduling Failed]──► queued          // Re-enter queue
running ──[User Pause/Preempted]──► paused
paused ──[Resume]──► queued                       // Re-schedule
running ──[exit code 0]──► succeeded
running ──[Non-zero Exit/Timeout/OOM]──► failed
failed ──[Not Exceed max_retries]──► queued         // Automatic retry
running/queued/paused ──[User Terminate]──► killed
```

### API Endpoints

```
POST   /api/v1/jobs                           # Submit job
GET    /api/v1/jobs                           # Job list (filtering, pagination, sorting)
GET    /api/v1/jobs/{id}                      # Job details
DELETE /api/v1/jobs/{id}                      # Terminate job
POST   /api/v1/jobs/{id}/pause
POST   /api/v1/jobs/{id}/resume
GET    /api/v1/jobs/{id}/logs?stream=&tail=
WS     /ws/jobs/{id}/logs                     # Real-time log streaming
GET    /api/v1/jobs/{id}/metrics
GET    /api/v1/jobs/{id}/checkpoints
POST   /api/v1/jobs/{id}/checkpoints/{ck_id}/restore
POST   /api/v1/validate/job                   # Pre-verification before submission (not enter queue)
GET    /api/v1/images
POST   /api/v1/images/build
GET    /api/v1/images/{id}/scan-report
```

---

## Module 5: Multi-Tenancy and Permission Management

**Tenant and Project System**

+ Four-level hierarchy: Organization → Tenant → Project → User
+ Complete resource isolation between tenants (network, storage, scheduling queue)
+ Project-level quota inheritance and override

**Quota Management**

+ Resource quotas: number of GPU cards, video memory, CPU, memory, storage
+ Time-dimensional quotas: daily/weekly/monthly usage upper limit
+ Burst Quota and priority queue binding
+ Quota over-limit alert and automatic task queuing
+ Quota pre-check: real-time deduction and pre-occupation (reserved) during job submission, settle actual usage after job completion
+ Borrowing mechanism: idle quotas of Tenant A can be temporarily lent to B, automatically recovered when borrowing expires, running jobs are not forcibly interrupted but not renewed
+ Quota alerts: reminder triggered when usage rate reaches 80%, alert triggered at 90%, new job submission prohibited at 100%
+ Historical usage trends: aggregated by day/week/month, support multi-dimensional drill-down by tenant/project/user/GPU model

**RBAC Permission Control**

+ Built-in roles: Platform Administrator, Tenant Administrator, Regular User, Read-only Observer
+ Custom roles and fine-grained permission definition (resource × operation)
+ SSO integration (LDAP / AD / OAuth 2.0 / SAML)
+ API Key management (issuance, revocation, validity period control)

**Metering and Billing**

+ Real-time resource usage collection (precision to minute level)
+ Metering by card-hour, GPU model, network bandwidth dimensions
+ Internal cost attribution report (by tenant/project/user/job)
+ Integration with financial system (bill export, pre-recharge alert)

**Billing Model Refinement**

+ Billing unit price table: independent pricing for different vendors/models (different unit prices for A100 and 910B)
+ Billing granularity: charge by actual occupancy duration (minute level), not by application duration
+ Internal settlement report: monthly bill (PDF export), including details + summary of each project
+ Budget early warning: set monthly budget upper limit for projects, early warning when overspending is expected

**Audit Logs**

+ Full operation audit: login, resource operation, permission change, task submission
+ Tamper-proof audit log storage
+ Audit log retrieval and export (compliance report)
  
**SSO Integration Specification**

+ LDAP attribute mapping: ou → tenant, cn → username, memberOf → role
+ OAuth 2.0 authorization code flow, support PKCE (backendless client scenarios)
+ Session management: JWT validity period 8h, Refresh Token 7 days, support forced offline (revoke all tokens)
+ MFA: TOTP (Google Authenticator compatible), mandatory for administrator operations

### Technology Selection

| Sub-function | Selection | Remarks |
|---|---|---|
| Authentication Center | **Keycloak** | LDAP / AD / OAuth 2.0 (PKCE) / SAML |
| MFA | **TOTP** (built-in Keycloak) | Google Authenticator compatible, mandatory for administrators |
| RBAC Engine | **Casbin (Go)** | Three-dimensional of resource × action × scope, rule hot loading |
| Token Revocation | **Redis Blacklist** | JWT forced offline, Refresh Token 7 days |
| API Key Management | **Self-developed (PostgreSQL + HMAC)** | Issuance, revocation, validity period control |
| Quota Engine | **Self-developed Quota Service (Go)** | Pre-occupy reserved during submission, settle actual usage after completion |
| Metering Collection | **VictoriaMetrics** (minute level) | UsageRecord written to PostgreSQL |
| Billing Report | **Self-developed Billing Service** + WeasyPrint | Monthly bill PDF export, integrated with financial Webhook |
| Audit Logs | **ClickHouse (append-only)** + Hash chain verification | Tamper-proof, support SIEM integration |

### Key Design Constraints

```
- Organization hierarchy: Organization → Tenant → Project → User

- Quota pre-check:
    Submission time: used + reserved + new application ≤ limit (otherwise return QUOTA_EXCEEDED)
    Usage rate alert thresholds: 80% reminder / 90% alert / 100% prohibit new job submission

- Quota borrowing:
    Idle quotas of Tenant A can be lent to B, automatically recovered when burst_until expires
    Running jobs are not forcibly interrupted but not renewed

- JWT specifications:
    access_token validity period: 8h
    refresh_token validity period: 7d
    Forced offline: write to Redis blacklist, verified at gateway layer

- SSO LDAP attribute mapping:
    ou → tenant
    cn → username
    memberOf → role

- Audit log fields: actor / action / resource / timestamp / result
  ClickHouse append-only table, Hash chain anti-tampering
```

### Core Data Models

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

### API Endpoints

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

## Module 6: Open API and Operation Management

**API Gateway Layer**

+ Unified authentication middleware (JWT verification + API Key verification either/or)
+ Request rate limiting: tenant-level (1000 req/min), user-level (100 req/min), IP-level (200 req/min)
+ Request tracing: inject X-Request-ID into each request, transparently transmit to backend services in full link
+ API version management: /api/v1/ long-term support, /api/v2/ grayscale, old version Deprecation notification mechanism
+ OpenAPI Spec automatically generated and hosted (/api/docs), SDK automatically generated from Spec

**External Interfaces**

+ RESTful API (OpenAPI 3.0 specification, including automatic SDK generation)
+ gRPC API (high-frequency call scenarios)
+ Webhook event push (task status change, alert trigger, etc.)
+ K8s CRD extension (native K8s ecosystem integration)

**Console Web UI**

+ Resource overview dashboard (total computing power, utilization rate, queuing status)
+ Task list and details (log streaming view, metric charts)
+ Device management (node list, card-level details, fault records)
+ User and quota management interface
+ Cost report and trend analysis

**CLI / SDK**

+ Command-line tools (submit tasks, view status, download logs)
+ Python SDK / Go SDK
+ Terraform Provider (Infrastructure as Code)

**Platform Operation Tools**

+ Grayscale release / blue-green deployment (platform own component upgrade)
+ Configuration center (runtime parameter hot update)
+ Inspection tasks (regular execution of hardware health detection)
+ Capacity planning dashboard (prediction of resource gap in next N days)
+ Platform component health check endpoints (/healthz, /readyz), integrated with K8s Liveness/Readiness
+ Feature Flag: grayscale release of new features by tenant/user
+ Operation and maintenance operation logs: records of all change operations by platform administrators, stored separately from business audit logs
+ Database migration management: use Migration framework (Flyway/Liquibase), versions can be rolled back

**Web Console Inventory**
```
/dashboard              Overview Dashboard (computing power dashboard, today's usage, pending alerts)
/clusters               Cluster Management (node list, rack view)
/clusters/{id}/devices  Node device list
/pools                  Resource Pool Management
/jobs                   Job List
/jobs/new               Submit Job (wizard-style form, 4 steps)
/jobs/{id}              Job Details (status, logs, metrics, checkpoint)
/images                 Image Management
/monitoring/alerts      Alert Event List
/monitoring/rules       Alert Rule Configuration
/tenants                Tenant Management (Platform Administrator)
/tenants/{id}/quotas    Quota Management
/billing                Billing Report
/settings/profile       Personal Settings
/settings/api-keys      API Key Management
/settings/sso           SSO Configuration (Administrator)
/audit                  Audit Logs
```

### Technology Selection

| Sub-function | Selection | Remarks |
|---|---|---|
| API Gateway | **Kong / APISIX** | Rate limiting plugin, X-Request-ID transparent transmission, JWT + API Key authentication |
| RESTful API Framework | **Go (Gin / Echo)** + OpenAPI 3.0 | /api/docs automatically hosted, SDK automatically generated |
| gRPC | **gRPC-Go** + Protobuf | High-frequency internal communication between scheduler ↔ Agent ↔ HAL |
| Webhook Push | **Kafka** + Self-developed Dispatcher | HMAC-SHA256 signature verification, event external push |
| K8s CRD Extension | **controller-runtime (Go)** | TrainingJob / ResourcePool CRD |
| Configuration Center | **Apollo / Nacos** | Runtime parameter hot update, Feature Flag grayscale by tenant |