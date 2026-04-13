# OpenCode 统一访问与会话同步方案

## 一、当前架构分析

### 1.1 三种客户端的关系

```
┌─────────────────────────────────────────────────────────────┐
│                      OpenCode 架构总览                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   CLI版                桌面版(Tauri)          Web版          │
│   ┌──────┐            ┌────────────┐       ┌──────────┐    │
│   │opencode│           │  Rust后端   │       │ Solid.js │    │
│   │ serve │           │  (sidecar)  │       │  前端    │    │
│   └──┬───┘            └─────┬──────┘       └────┬─────┘    │
│      │                      │                    │          │
│      │ 直接启动              │ spawn子进程         │ HTTP     │
│      ▼                      ▼                    ▼          │
│   ┌─────────────────────────────────────────────────┐      │
│   │              Hono HTTP Server (Bun)              │      │
│   │         localhost:PORT (默认4096)                 │      │
│   ├─────────────────────────────────────────────────┤      │
│   │  /session  /project  /event(SSE)  /global/health│      │
│   └───────────────────────┬─────────────────────────┘      │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────┐      │
│   │           SQLite 数据库 (WAL模式)                │      │
│   │       ~/.local/share/opencode/opencode.db       │      │
│   └─────────────────────────────────────────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 数据库架构

**数据库路径决策:**
```
优先级:
1. 环境变量 OPENCODE_DB (直接指定路径)
2. Channel后缀: opencode-{channel}.db (如 opencode-claude-practical-gagarin.db)
3. 默认: ~/.local/share/opencode/opencode.db
```

**当前数据库文件:**
| 数据库文件 | Session数量 | 来源 |
|-----------|------------|------|
| opencode.db (210MB) | 456 | 主数据库，桌面版sidecar使用 |
| opencode-claude-practical-gagarin.db | 0 | worktree专用，当前4096使用 |
| opencode-local.db | 未知 | 另一个项目 |

**核心表结构:**
```
ProjectTable
├── id (ProjectID = git首次commit的hash)
├── worktree (项目根路径)
├── vcs ("git" | null)
└── name, icon, commands, sandboxes

SessionTable
├── id (SessionID)
├── project_id → ProjectTable.id
├── workspace_id (可选, 控制平面)
├── directory (创建session时的工作目录)  ← 关键字段
├── parent_id (子session)
├── title, version, slug
├── summary (additions/deletions/files/diffs)
└── time_created, time_updated, time_archived

MessageTable / PartTable (层级消息存储)
├── session_id → SessionTable.id
└── data (JSON)
```

### 1.3 Project ID 生成机制

```typescript
// 通过git获取项目根的最早commit hash作为project_id
git rev-list --max-parents=0 --all | sort | head -1
// 例: "21d80ca68346dfdb8d3556015a723a9217f8566f"
```

**特性:**
- 同一个git仓库的不同clone/worktree → 相同的project_id
- 不同的git仓库 → 不同的project_id
- 非git目录 → 使用fake VCS ID

### 1.4 请求路由与Instance管理

```typescript
// 服务器中间件 (router.ts)
// 每个请求都会:
1. 从 query.directory 或 header["x-opencode-directory"] 提取目录
2. 默认使用 process.cwd()
3. 调用 Instance.provide({ directory }) 创建/复用实例上下文
4. 在该上下文中执行路由处理

// Instance (instance.ts)
// 每个唯一目录一个Instance，缓存在内存中
Instance = {
  directory: string,     // 工作目录
  worktree: string,      // git根目录
  project: ProjectInfo,  // 从数据库加载
}
```

### 1.5 Session 查询逻辑

```typescript
// Session.list() - 项目内查询
conditions = [
  eq(project_id, Instance.project.id),  // 强制按project过滤
  // 可选:
  eq(directory, input.directory),
  eq(workspace_id, input.workspaceID),
  isNull(parent_id),  // roots only
]

// Session.listGlobal() - 跨项目查询（仪表盘用）
conditions = [
  // 不按project_id过滤
  eq(directory, input.directory),  // 可选
  isNull(time_archived),           // 可选
]
```

### 1.6 实时事件系统

```
客户端 ←── SSE (/event) ←── GlobalBus ←── Instance Bus

事件类型:
├── session.created / session.updated / session.deleted
├── message.part.updated / message.part.delta
├── session.status (running/idle/error)
├── project.updated
├── lsp.updated
└── server.connected / server.instance.disposed

特性:
- Per-Instance的Bus (异步上下文隔离)
- GlobalBus桥接所有Instance的事件
- SSE推送到所有连接的客户端
- 心跳: 每10秒发送heartbeat
```

### 1.7 桌面版Sidecar流程

```
桌面应用启动:
1. Rust后端 spawn OpenCode CLI 子进程 (cli::serve)
2. 分配随机端口 (如57985)
3. 轮询 /global/health 等待就绪
4. 返回 ServerReadyData { url, username, password }
5. 前端连接到 localhost:port
6. 前端用localStorage持久化项目列表

特点:
- 每次启动都是新的server实例
- 端口随机分配
- 仅绑定localhost
- 数据库共享但进程独立
```

---

## 二、当前存在的问题

### 2.1 P0: 会话不显示

**根本原因:** 4096后端在worktree目录启动，使用了worktree专用数据库(0个session)

**详细分析:**
```
4096启动目录: G:\claude-project\opencode-local\.claude\worktrees\practical-gagarin
↓
Channel检测: "claude-practical-gagarin"
↓
数据库: opencode-claude-practical-gagarin.db (0 sessions)
↓
API返回: []
```

用户的session在 opencode.db 中 (456 sessions)，project_id = `21d80ca68346dfdb8d3556015a723a9217f8566f`

### 2.2 P1: Windows路径分隔符

**状态:** 修复 commit `812c9814f` 存在但未合并到dev分支

**影响:**
- Session存储时用 `G:\path` (反斜杠)
- 查询时可能用 `G:/path` (正斜杠)
- 导致 `eq(SessionTable.directory, input.directory)` 匹配失败

### 2.3 P2: diffs字段undefined

**状态:** 已修复 (commit `09c588d07`)
- `diffs: row.summary_diffs ?? undefined` → `diffs: row.summary_diffs ?? []`
- Zod schema: `.optional()` → `.default([])`

### 2.4 P3: 非关键错误

| 错误 | 来源 | 影响 |
|------|------|------|
| MCP -32601 Method not found | context7/grep_app不支持prompts | 无影响 |
| rustfmt failed | 桌面版代码格式化 | 格式化功能不可用 |
| oh-my-openagent迁移警告 | opencode.json包名过期 | 仅警告 |

---

## 三、统一访问方案设计

### 3.1 目标

像Claude一样: **一个账号，任意设备，实时同步**

```
┌──────────────────────────────────────────────────────────────┐
│                        目标架构                               │
│                                                              │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐            │
│  │ CLI    │  │ 桌面版  │  │ Web版  │  │ 外网   │            │
│  │ 内网   │  │ 内网   │  │ 内网   │  │ 浏览器  │            │
│  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘            │
│      │           │           │           │                   │
│      └─────────┬─┴───────────┴───────────┘                   │
│                │                                             │
│         ┌──────▼──────┐                                      │
│         │   反向代理   │  (可选: Nginx/Cloudflare Tunnel)     │
│         │  认证/TLS   │                                      │
│         └──────┬──────┘                                      │
│                │                                             │
│         ┌──────▼──────┐                                      │
│         │  OpenCode   │                                      │
│         │  Server     │  0.0.0.0:4096                        │
│         │  (单实例)   │                                      │
│         └──────┬──────┘                                      │
│                │                                             │
│         ┌──────▼──────┐                                      │
│         │   SQLite    │  opencode.db (唯一数据库)             │
│         │   (WAL)     │                                      │
│         └─────────────┘                                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 需要的代码修改

#### 修改1: 数据库路径统一 (必须)

**文件:** `packages/opencode/src/storage/db.ts`

**问题:** Channel机制导致不同启动方式使用不同数据库
**方案:** 
- 提供环境变量 `OPENCODE_DB` 强制指定数据库路径
- 或修改channel逻辑，确保server模式下始终使用主数据库

```typescript
// 改动思路:
// 当以 serve 模式启动时，忽略channel后缀，使用主数据库
if (mode === "serve") {
  dbPath = Global.Path.data + "/opencode.db"
}
```

#### 修改2: 服务器绑定地址 (必须)

**文件:** `packages/opencode/src/server/server.ts`

**当前:** 默认绑定 `127.0.0.1`
**需要:** 支持绑定 `0.0.0.0` 以允许远程访问

```bash
# CLI方式:
opencode serve --hostname 0.0.0.0 --port 4096

# 或环境变量:
OPENCODE_HOST=0.0.0.0 OPENCODE_PORT=4096 opencode serve
```

**已支持:** CLI的 `serve` 命令已有 `--hostname` 和 `--port` 参数

#### 修改3: 认证机制 (必须)

**当前支持:** Basic Auth via `OPENCODE_SERVER_PASSWORD`/`OPENCODE_SERVER_USERNAME`

```bash
# 启动时设置密码:
OPENCODE_SERVER_PASSWORD=your-secret opencode serve --hostname 0.0.0.0
```

**已内置**，但需要确保所有客户端都支持传递认证信息。

#### 修改4: CORS配置扩展 (必须)

**文件:** `packages/opencode/src/server/server.ts`

**当前白名单:**
```
localhost:*, 127.0.0.1:*, tauri://localhost, *.opencode.ai
```

**需要添加:**
- 用户的外网域名
- 或通过环境变量/配置文件自定义CORS

```typescript
// 改动思路:
const customOrigins = process.env.OPENCODE_CORS_ORIGINS?.split(",") ?? []
// 添加到 allowedOrigins 列表
```

#### 修改5: 路径分隔符规范化 (必须, Windows)

**文件:** `packages/opencode/src/session/index.ts` 等

**方案:** Cherry-pick commit `812c9814f` 的改动:
- Session存储时规范化路径 (统一用正斜杠)
- Session查询时规范化输入路径
- Project ID匹配时规范化路径

#### 修改6: 桌面版支持连接远程服务器 (可选)

**当前:** 桌面版总是启动本地sidecar
**需要:** 选项允许桌面版直接连接远程OpenCode server

**文件:** `packages/app/src/context/server.tsx`

**已部分支持:** Web UI已有 `ServerConnection.Http` 类型，支持添加远程服务器

### 3.3 不需要修改的部分

| 组件 | 原因 |
|------|------|
| SSE事件系统 | 已支持多客户端实时推送 |
| Session CRUD | 通过HTTP API操作，客户端无关 |
| SQLite WAL | 已支持并发读写 |
| 项目发现 | 通过directory参数路由，与客户端无关 |
| 消息存储 | 已通过session_id关联，不受影响 |

---

## 四、可执行方案 (分阶段)

### 阶段0: 立即修复当前问题 (预计30分钟)

**目标:** 让4096后端正确显示所有session

```
步骤:
1. 停止当前4096后端
2. 设置 OPENCODE_DB 环境变量指向主数据库
3. 在正确目录下启动后端
4. 验证session显示
```

```bash
# 具体命令:
# 停止旧的
taskkill /PID <4096进程PID> /F

# 在项目目录启动，强制使用主数据库
cd G:\opencode-project\hermes-agent
set OPENCODE_DB=C:\Users\datoo\.local\share\opencode\opencode.db
G:\...\dist\opencode-windows-x64\bin\opencode.exe serve --port 4096
```

**验证标准:**
- `curl http://localhost:4096/session` 返回非空数组
- 浏览器打开4096，左侧显示会话列表
- 无 `diffs.map` 错误

### 阶段1: 路径分隔符修复 (预计1小时)

**目标:** Windows下session查询不再因路径分隔符失败

```
步骤:
1. Cherry-pick或手动应用 812c9814f 的改动
2. 在 Session.list/listGlobal 中添加路径规范化
3. 在 Session.create 中统一存储正斜杠路径
4. 重新编译
5. 测试多目录的session查询
```

### 阶段2: 统一数据库访问 (预计2小时)

**目标:** 任何启动方式都使用同一个数据库

```
步骤:
1. 修改 db.ts: serve模式下忽略channel后缀
2. 添加 OPENCODE_DB 环境变量支持 (如不存在)
3. 修改 server.ts: 添加CORS自定义配置
4. 重新编译测试
```

### 阶段3: 远程访问支持 (预计2小时)

**目标:** 外网可以安全访问OpenCode服务器

```
方案A: 直接暴露 (简单)
1. opencode serve --hostname 0.0.0.0 --port 4096
2. 设置 OPENCODE_SERVER_PASSWORD
3. 路由器端口转发 4096
4. 客户端通过 http://公网IP:4096 连接

方案B: 反向代理 (推荐)
1. opencode serve --hostname 127.0.0.1 --port 4096
2. Nginx反向代理 + TLS证书
3. 或 Cloudflare Tunnel (零配置内网穿透)
4. 客户端通过 https://your-domain.com 连接

方案C: Tailscale/ZeroTier (最安全)
1. 安装Tailscale/ZeroTier
2. opencode serve --hostname 0.0.0.0 --port 4096
3. 客户端通过内网IP访问
4. 自带加密和认证
```

### 阶段4: 多客户端实时同步 (预计1小时验证)

**目标:** 多个浏览器/客户端同时连接，实时看到变化

```
已有基础:
- SSE事件流 (/event) 支持多客户端
- GlobalBus广播所有Instance事件
- 前端sync.tsx监听事件并更新UI

需要验证:
1. 同时打开2个浏览器标签连接4096
2. 在一个标签创建session
3. 另一个标签应自动显示新session
4. 测试事件延迟和可靠性
```

### 阶段5: 桌面版远程连接 (可选, 预计3小时)

**目标:** 桌面版可以选择连接远程服务器而非本地sidecar

```
步骤:
1. 桌面版设置页面添加"连接远程服务器"选项
2. 输入: 服务器URL + 用户名/密码
3. 跳过sidecar启动，直接HTTP连接
4. 保持健康检查和重连逻辑
```

---

## 五、配置参考

### 5.1 服务器启动配置

```bash
# 最小化启动 (本地)
opencode serve --port 4096

# 内网访问
opencode serve --hostname 0.0.0.0 --port 4096

# 带认证的远程访问
OPENCODE_SERVER_PASSWORD=your-secret \
OPENCODE_SERVER_USERNAME=admin \
opencode serve --hostname 0.0.0.0 --port 4096

# 指定数据库
OPENCODE_DB=/path/to/opencode.db \
opencode serve --hostname 0.0.0.0 --port 4096
```

### 5.2 客户端连接

```
Web版:
  打开 http://服务器地址:4096

CLI版:
  OPENCODE_SERVER=http://服务器地址:4096 opencode session list

桌面版:
  设置 → 服务器 → 添加远程服务器 → http://服务器地址:4096
```

### 5.3 Nginx反向代理参考

```nginx
server {
    listen 443 ssl;
    server_name opencode.yourdomain.com;

    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;

    location / {
        proxy_pass http://127.0.0.1:4096;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # SSE支持
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 86400s;
    }
}
```

### 5.4 Cloudflare Tunnel (零配置方案)

```bash
# 安装cloudflared
# 创建tunnel
cloudflared tunnel create opencode
cloudflared tunnel route dns opencode opencode.yourdomain.com

# 配置 ~/.cloudflared/config.yml
tunnel: <tunnel-id>
ingress:
  - hostname: opencode.yourdomain.com
    service: http://localhost:4096
  - service: http_status:404

# 启动
cloudflared tunnel run opencode
```

---

## 六、风险和注意事项

| 风险 | 等级 | 缓解措施 |
|------|------|----------|
| SQLite并发写入冲突 | 低 | WAL模式已处理，但极高并发可能需要考虑PostgreSQL |
| 外网暴露安全 | 高 | 必须启用认证 + TLS，推荐Cloudflare Tunnel |
| 数据库文件损坏 | 中 | 定期备份，启用WAL checkpoint |
| SSE连接数限制 | 低 | Bun原生支持大量并发连接 |
| 路径兼容(Win/Mac/Linux) | 中 | 统一使用正斜杠存储 |
| 大数据库性能 | 中 | 当前210MB，长期需要定期归档旧session |

---

## 七、与Claude的对比

| 特性 | Claude | OpenCode当前 | OpenCode目标 |
|------|--------|-------------|-------------|
| 中心化服务器 | Anthropic云 | 本地/分散 | 统一本地服务器 |
| 多设备同步 | 自动 | 不支持 | 通过连接同一服务器 |
| 实时更新 | WebSocket | SSE (已有) | SSE (已支持) |
| 会话持久化 | 云端 | 本地SQLite | 本地SQLite |
| 外网访问 | 默认 | 不支持 | 通过反向代理/tunnel |
| 认证 | OAuth | Basic Auth | Basic Auth (可扩展) |
| 多项目 | 按Organization | 按git root | 按git root |

**关键差异:** Claude是SaaS，数据在云端。OpenCode是本地优先，通过暴露HTTP服务实现"类云端"体验。好处是数据完全自主可控。
