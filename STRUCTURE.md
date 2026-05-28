# AI-Infra platfrom contents ✅

### backend - Go Monorepo

```
backend/
├── cmd/                          # 4个微服务入口
│   ├── scheduler/main.go        # 调度器服务
│   ├── agent/main.go            # 节点Agent
│   ├── api-server/main.go       # API网关
│   └── monitor-collector/main.go # 监控采集器
├── internal/                     # 核心业务逻辑
│   ├── common/                  # 公共组件
│   │   ├── config/             # 配置管理
│   │   ├── logger/             # 日志封装
│   │   ├── errors/             # 错误码定义
│   │   └── middleware/         # JWT/限流中间件
│   ├── core/                    # 领域模型
│   │   ├── resource/           # [模块一] 资源调度
│   │   │   ├── model/         # Node/Device实体
│   │   │   ├── repository/    # PostgreSQL持久化
│   │   │   └── service/       # Gang/Fair-Share策略
│   │   ├── runtime/            # [模块二] HAL适配层
│   │   │   ├── hal/           # C++ CGO桥接
│   │   │   ├── matrix/        # 兼容性矩阵
│   │   │   └── container/     # Device Plugin
│   │   ├── task/               # [模块四] 任务管理
│   │   │   ├── engine/        # Kubeflow/Argo集成
│   │   │   ├── checkpoint/    # Checkpoint服务
│   │   │   └── image/         # 镜像扫描
│   │   └── tenant/             # [模块五] 多租户
│   │       ├── quota/         # 配额引擎
│   │       └── billing/       # 计费逻辑
│   ├── observability/           # [模块三] 监控
│   │   ├── metrics/           # Prometheus客户端
│   │   ├── alerting/          # 告警规则
│   │   └── diagnostics/       # 故障诊断
│   └── auth/                    # Keycloak/Casbin
├── pkg/                         # 公开库
│   ├── proto/                  # gRPC定义
│   ├── sdk/                    # Go SDK
│   └── utils/                  # 工具函数
├── migrations/                  # Flyway迁移脚本
├── configs/                     # YAML配置模板
├── scripts/                     # 运维脚本
├── tests/                       # 集成测试
└── go.mod                       # Go依赖管理
```

### frontend - React + TypeScript

```
frontend/
├── .umirc.ts                    # UmiJS配置（路由/代理）
├── package.json                 # npm依赖
└── src/
    ├── index.tsx                # 应用入口（主题注入）
    ├── app.tsx                  # 根组件
    ├── assets/                  # 静态资源
    │   ├── icons/              # SVG图标
    │   └── images/             # 图片资源
    ├── components/              # 通用组件
    │   ├── Layout/             # 侧边栏（互斥展开）
    │   ├── Charts/             # ECharts封装
    │   ├── Tables/             # 高级表格
    │   └── StatusTag/          # 状态标签✅
    ├── layouts/                 # 布局组件
    ├── pages/                   # 页面组件
    │   ├── Dashboard/          # /dashboard
    │   │   ├── ClusterOverview/
    │   │   └── ResourceTrend/
    │   ├── Clusters/           # /clusters
    │   │   ├── NodeList/
    │   │   └── DeviceDetail/
    │   ├── Jobs/               # /jobs
    │   │   ├── JobList/
    │   │   ├── JobSubmit/      # 4步向导
    │   │   └── JobMonitor/     # WebSocket日志
    │   ├── Monitoring/         # /monitoring
    │   │   ├── AlertRules/
    │   │   └── Topology/       # AntV G6
    │   ├── Tenant/             # /tenants
    │   │   └── QuotaManage/
    │   └── Settings/           # /settings
    ├── services/                # API层✅
    │   ├── resource.ts         # 资源管理接口
    │   ├── job.ts              # 任务管理接口
    │   └── monitor.ts          # 监控告警接口
    ├── stores/                  # Redux状态✅
    │   ├── user.ts             # 用户信息
    │   └── global.ts           # 全局状态
    ├── styles/                  # 样式配置✅
    │   └── theme.ts            # 配色方案实现
    ├── utils/                   # 工具函数
    └── public/                  # 公共资源
```

---

## ✅ 已完成的关键文件

### 1. 主题配置 (`frontend/src/styles/theme.ts`)
- ✅ 主色 Indigo `#4B55D9`
- ✅ 辅助色（Success/Warning/Danger/Info）
- ✅ 中性色（标题/正文/次要文字）
- ✅ 圆角规范（4px/8px/16px）
- ✅ 间距规范（4px~40px）
- ✅ 图表配色组（8色palette）
- ✅ 字体层级（12px~24px）
- ✅ 动画规范（0.3s ease-in-out）

### 2. API服务层
- ✅ `services/resource.ts` - 节点/设备/资源池接口
- ✅ `services/job.ts` - 作业生命周期接口
- ✅ `services/monitor.ts` - 监控指标/告警接口

### 3. Redux状态管理
- ✅ `stores/user.ts` - 用户信息/Token/权限
- ✅ `stores/global.ts` - Loading/Error全局状态

### 4. 通用组件
- ✅ `components/StatusTag/index.tsx` - 状态标签（4种状态色）

### 5. 后端入口文件
- ✅ `cmd/scheduler/main.go`
- ✅ `cmd/agent/main.go`
- ✅ `cmd/api-server/main.go`
- ✅ `cmd/monitor-collector/main.go`

### 6. 配置文件
- ✅ `backend/go.mod` - Go依赖（Gin/GORM/Redis/Kafka等）
- ✅ `frontend/package.json` - npm依赖（React/AntD/ECharts等）
- ✅ `frontend/.umirc.ts` - UmiJS路由配置（15+页面路由）

### 7. 文档
- ✅ `src/README.md` - 完整的项目结构说明

---

## 🎨 配色方案落地情况

| 规范项 | 实现位置 | 状态 |
|--------|---------|------|
| 主色 Indigo | `theme.ts:colorPrimary` | ✅ |
| 按钮圆角4px | `theme.ts:components.Button` | ✅ |
| 表格斑马纹 | `theme.ts:components.Table` | ✅ |
| 状态标签颜色 | `StatusTag组件` | ✅ |
| 图表8色palette | `theme.ts:chartColors.palette` | ✅ |
| 字体族PingFang SC | `theme.ts:fontFamily` | ✅ |
| 动画0.3s | `theme.ts:animation.duration` | ✅ |

---



1. **安装依赖**
   ```bash
   cd src/backend && go mod tidy
   cd src/frontend && npm install
   ```

2. **实现核心组件**
   - 侧边栏导航（单选展开逻辑）
   - ECharts图表封装（饼图/柱状图规范）
   - 高级表格组件（固定列/斑马纹）

3. **对接后端API**
   - 实现Axios拦截器（X-Request-ID注入）
   - WebSocket实时日志流
   - Redux异步Action

4. **数据库初始化**
   - 编写Flyway迁移脚本
   - 初始化PostgreSQL表结构
   - 导入兼容性矩阵数据

---

**生成时间**: 2026-05-28  
**符合标准**: Clean Architecture + Ant Design Pro最佳实践
