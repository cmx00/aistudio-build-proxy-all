# 系统架构与工作原理

## 概述

本项目是一个基于Docker的 Google Gemini API 反向代理系统，通过浏览器自动化和WebSocket隧道技术，实现了对 Google AIStudio Build 的无缝代理访问。该系统解决了直接调用 Gemini API 时可能遇到的网络限制和认证问题。

## 核心组件

系统由三个主要组件构成：

### 1. Camoufox 浏览器实例 (Python)
- **技术栈**: Python + Playwright + Camoufox
- **职责**: 
  - 管理真实浏览器会话
  - 维护 Google 账户登录状态
  - 通过 WebSocket 接收并执行 HTTP 请求
  - 将响应返回给代理服务器

### 2. WebSocket 代理服务器 (Golang)
- **技术栈**: Golang + Gorilla WebSocket
- **职责**:
  - 监听端口 5345，接收外部 HTTP/HTTPS 请求
  - 管理与浏览器实例的 WebSocket 连接池
  - 实现负载均衡（轮询策略）
  - 将 HTTP 请求转换为 WebSocket 消息
  - 支持流式响应（SSE）

### 3. Supervisor 进程管理器
- **职责**:
  - 在 Docker 容器中同时启动和管理 Python 和 Golang 进程
  - 自动重启崩溃的进程
  - 统一日志输出

## 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker Container                          │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Supervisor                              │  │
│  │                                                             │  │
│  │  ┌──────────────────────┐    ┌──────────────────────┐    │  │
│  │  │  Python 进程         │    │  Golang 进程         │    │  │
│  │  │  (run_camoufox.py)   │    │  (go_app_binary)     │    │  │
│  │  │                      │    │                      │    │  │
│  │  │  ┌────────────────┐ │    │  ┌────────────────┐  │    │  │
│  │  │  │  Camoufox      │ │◄───┼──┤  WebSocket     │  │    │  │
│  │  │  │  Browser       │ │ WS │  │  Server        │  │    │  │
│  │  │  │  Instance(s)   │ │───►┼──┤  (Port 5345)   │  │    │  │
│  │  │  └────────────────┘ │    │  └────────────────┘  │    │  │
│  │  │         │            │    │         ▲            │    │  │
│  │  └─────────┼────────────┘    └─────────┼────────────┘    │  │
│  └────────────┼───────────────────────────┼─────────────────┘  │
│               │                            │                    │
└───────────────┼────────────────────────────┼────────────────────┘
                │                            │
                │                            │ HTTP API 请求
                ▼                            │ (带 API Key 认证)
        Google AIStudio                      │
        Build WebSocket                      ▼
        服务器                          外部客户端
                                    (如 Cherry Studio)
```

## 工作流程详解

### 阶段 1: 系统启动

1. **Docker 容器启动**
   - Supervisor 读取 `/etc/supervisor/conf.d/supervisord.conf`
   - 同时启动 Python 和 Golang 进程

2. **Python 进程初始化**
   ```
   run_camoufox.py
   ├── 读取 config.yaml 配置文件
   ├── 为每个实例创建独立进程
   └── 每个进程执行 run_browser_instance()
       ├── 加载 Cookie 文件 (JSON 格式)
       ├── 启动 Camoufox 浏览器
       ├── 注入 Cookie 到浏览器上下文
       ├── 导航到 AIStudio Build URL
       ├── 验证登录状态
       └── 建立与 Golang 服务的 WebSocket 连接
   ```

3. **Golang 代理服务器初始化**
   ```
   main.go
   ├── 启动 HTTP 服务器 (监听 :5345)
   ├── 注册路由:
   │   ├── /v1/ws → WebSocket 连接处理
   │   └── /* → HTTP 请求代理处理
   └── 初始化全局连接池 (globalPool)
   ```

### 阶段 2: WebSocket 连接建立

1. **浏览器实例连接**
   - Python 端通过 WebSocket 连接到 `ws://localhost:5345/v1/ws?auth_token=<token>`
   - Golang 服务器验证 token 并将连接添加到连接池
   - 支持同一用户的多个连接（负载均衡）

2. **连接池管理**
   ```go
   ConnectionPool
   ├── Users: map[string]*UserConnections
   │   └── "user-1"
   │       └── Connections: []*UserConnection
   │           ├── Conn #1 (浏览器实例 1)
   │           ├── Conn #2 (浏览器实例 2)
   │           └── ...
   └── 轮询索引 (NextIndex)
   ```

3. **心跳机制**
   - 浏览器定期发送 `ping` 消息
   - 服务器响应 `pong` 消息
   - 60 秒无活动则断开连接

### 阶段 3: HTTP 请求代理

#### 3.1 请求接收

```
外部客户端 → HTTP Request
                ↓
        Golang 代理服务器
                ↓
        验证 API Key (AUTH_API_KEY)
                ↓
        生成唯一请求 ID (UUID)
```

#### 3.2 请求封装

```go
WSMessage {
    ID: "550e8400-e29b-41d4-a716-446655440000",
    Type: "http_request",
    Payload: {
        method: "POST",
        url: "https://generativelanguage.googleapis.com/v1/models/gemini-pro:generateContent",
        headers: {
            "Content-Type": ["application/json"],
            ...
        },
        body: "{...}"  // JSON 字符串
    }
}
```

#### 3.3 连接选择与消息发送

```
连接池
  ├── 获取用户的连接列表
  ├── 使用轮询算法选择一个连接
  ├── 通过 WebSocket 发送请求消息
  └── 注册等待响应的通道 (pendingRequests)
```

#### 3.4 浏览器端处理

```
Python WebSocket 客户端
  ├── 接收 WSMessage
  ├── 解析请求参数
  ├── 使用 Playwright Page API 执行请求
  │   └── page.evaluate() 或 page.request API
  ├── 获取响应数据
  └── 封装为 WSMessage 返回
```

#### 3.5 响应类型

**类型 A: 标准 HTTP 响应**
```go
WSMessage {
    ID: "550e8400-...",
    Type: "http_response",
    Payload: {
        status: 200,
        headers: {...},
        body: "响应内容"
    }
}
```

**类型 B: 流式响应 (SSE)**
```go
// 1. 流开始
WSMessage {
    ID: "550e8400-...",
    Type: "stream_start",
    Payload: {
        status: 200,
        headers: {...}
    }
}

// 2. 数据块（多次）
WSMessage {
    ID: "550e8400-...",
    Type: "stream_chunk",
    Payload: {
        data: "数据块内容"
    }
}

// 3. 流结束
WSMessage {
    ID: "550e8400-...",
    Type: "stream_end",
    Payload: {}
}
```

**类型 C: 错误响应**
```go
WSMessage {
    ID: "550e8400-...",
    Type: "error",
    Payload: {
        error: "错误信息",
        status: 502
    }
}
```

#### 3.6 响应路由与返回

```
Golang 服务器
  ├── readPump() 接收 WebSocket 消息
  ├── 根据 ID 查找 pendingRequests
  ├── 将消息发送到对应的响应通道
  └── processWebSocketResponse() 处理响应
      ├── 设置 HTTP 响应头
      ├── 写入响应状态码
      ├── 写入响应体（支持流式刷新）
      └── 返回给外部客户端
```

## 关键技术细节

### 1. Cookie 管理

**Cookie 获取方式**（推荐）:
- 使用指纹浏览器（如 AdsPower、Multilogin）
- 登录 Google 账户
- 导出 Cookie 为 JSON 格式
- Cookie 包含 `SID`, `HSID`, `SSID`, `APISID`, `SAPISID` 等关键字段

**Cookie 注入**:
```python
context = browser.new_context()
context.add_cookies(cookies)  # Playwright API
```

**Cookie 验证**:
```python
# 检查是否跳转到登录页面
if "accounts.google.com/v3/signin" in page.url:
    # Cookie 失效
    
# 检查是否显示认证错误
if page.get_by_text("authentication error").is_visible():
    # Cookie 无效或过期
```

### 2. 负载均衡策略

**轮询算法 (Round-Robin)**:
```go
func (p *ConnectionPool) GetConnection(userID string) (*UserConnection, error) {
    userConns.Lock()
    defer userConns.Unlock()
    
    numConns := len(userConns.Connections)
    idx := userConns.NextIndex % numConns
    selectedConn := userConns.Connections[idx]
    userConns.NextIndex = (userConns.NextIndex + 1) % numConns
    
    return selectedConn, nil
}
```

**优势**:
- 均匀分配请求到所有浏览器实例
- 避免单一实例过载
- 提高系统整体吞吐量

### 3. 并发安全

**WebSocket 写入保护**:
```go
type UserConnection struct {
    Conn       *websocket.Conn
    writeMutex sync.Mutex  // 防止并发写入导致数据混乱
}

func (uc *UserConnection) safeWriteJSON(v interface{}) error {
    uc.writeMutex.Lock()
    defer uc.writeMutex.Unlock()
    return uc.Conn.WriteJSON(v)
}
```

**连接池保护**:
```go
type ConnectionPool struct {
    sync.RWMutex  // 读写锁，支持并发读取
    Users map[string]*UserConnections
}
```

**待处理请求**:
```go
var pendingRequests sync.Map  // 线程安全的 Map
```

### 4. 超时机制

| 超时类型 | 时长 | 说明 |
|---------|------|------|
| WebSocket 读取超时 | 60s | 超过此时间无消息则断开连接 |
| HTTP 代理请求超时 | 600s | 单个请求的最大等待时间 |
| 页面导航超时 | 120s | 浏览器导航到目标页面的超时 |
| Spinner 等待超时 | 30s | 等待页面加载指示器消失 |

### 5. 错误处理与重试

**浏览器端**:
```python
try:
    response = page.goto(url, timeout=120000)
    if not response.ok:
        # 记录非 2xx 状态码
        logger.warning(f"HTTP status: {response.status}")
except TimeoutError:
    # 导航超时
    page.screenshot(path="FAIL_timeout.png")
except PlaywrightError as e:
    # 网络错误
    logger.error(f"Network error: {e}")
```

**代理服务器端**:
```go
select {
case msg := <-respChan:
    // 处理响应
case <-ctx.Done():
    // 超时处理
    http.Error(w, "Gateway Timeout", 504)
}
```

## 数据流示例

### 完整的请求-响应流程

```
1. 客户端请求
   POST http://localhost:5345/v1/models/gemini-pro:generateContent
   Header: x-goog-api-key: your_set_api_key_here
   Body: {"contents": [...]}

2. Golang 代理
   ├── 验证 API Key ✓
   ├── 生成 reqID: "abc123"
   ├── 选择连接: user-1 → Conn #2
   └── 发送 WebSocket 消息:
       {
         "id": "abc123",
         "type": "http_request",
         "payload": {
           "method": "POST",
           "url": "https://generativelanguage.googleapis.com/v1/models/gemini-pro:generateContent",
           "headers": {...},
           "body": "{\"contents\": [...]}"
         }
       }

3. Python 浏览器实例 #2
   ├── 接收 WebSocket 消息
   ├── 在浏览器上下文中执行 HTTP 请求
   ├── 等待响应
   └── 返回 WebSocket 消息:
       {
         "id": "abc123",
         "type": "http_response",
         "payload": {
           "status": 200,
           "headers": {"Content-Type": "application/json"},
           "body": "{\"candidates\": [...]}"
         }
       }

4. Golang 代理
   ├── readPump 接收 WebSocket 消息
   ├── 根据 ID "abc123" 找到 respChan
   ├── 发送到通道
   └── processWebSocketResponse:
       ├── w.Header().Set("Content-Type", "application/json")
       ├── w.WriteHeader(200)
       ├── w.Write("{\"candidates\": [...]}")
       └── 返回给客户端

5. 客户端接收
   Status: 200 OK
   Body: {"candidates": [...]}
```

## 安全考虑

### 1. 认证机制

**WebSocket 连接认证**:
- 使用 `auth_token` 参数
- 在生产环境应使用 JWT Token
- 当前简化实现：硬编码 token

**HTTP 代理认证**:
- 通过 `x-goog-api-key` Header 或 URL 参数 `key`
- 与环境变量 `AUTH_API_KEY` 比对
- 认证失败返回 401 Unauthorized

### 2. 数据隔离

- 使用 UserID 隔离不同用户的连接
- 单租户模式：所有请求映射到 "user-1"
- 扩展支持：可通过不同 API Key 映射到不同 UserID

### 3. Cookie 安全

- Cookie 文件存储在本地，不通过网络传输
- 使用 Docker Volume 挂载，与容器内部隔离
- 建议定期更新 Cookie

## 性能优化

### 1. 多实例并发

```yaml
instances:
  - cookie_file: "user1_cookie.json"
    url: "https://aistudio.google.com/..."
  - cookie_file: "user2_cookie.json"
    url: "https://aistudio.google.com/..."
```
- 多个浏览器实例并行处理请求
- 轮询负载均衡提高吞吐量

### 2. 连接复用

- WebSocket 长连接避免频繁建立连接的开销
- HTTP 请求通过同一 WebSocket 通道复用

### 3. 流式响应

- 支持 Server-Sent Events (SSE)
- 数据块即时刷新，降低首字延迟
- 适合 Gemini 的流式生成场景

### 4. 异步处理

- Golang 的 Goroutine 并发模型
- Python 的多进程模型
- 请求处理不阻塞其他请求

## 故障排查

### 常见问题

1. **Cookie 失效**
   - 症状：跳转到登录页面或显示 "authentication error"
   - 解决：重新导出 Cookie

2. **WebSocket 连接断开**
   - 症状：日志显示 "no available client"
   - 解决：检查 Python 进程是否正常运行

3. **代理请求超时**
   - 症状：返回 504 Gateway Timeout
   - 解决：检查网络连接，增加超时时间

4. **浏览器导航失败**
   - 症状：截图显示空白页或错误页
   - 解决：检查代理设置，确认 URL 可访问

### 日志查看

```bash
# Docker 容器日志
docker logs [container_name]

# Python 应用日志
cat camoufox-py/logs/app.log

# 截图诊断
ls camoufox-py/logs/*.png
```

## 扩展与定制

### 添加新的浏览器实例

1. 编辑 `config.yaml`
2. 添加新的 Cookie 文件
3. 重启 Docker 容器

### 修改认证方式

1. 编辑 `golang/main.go` 中的 `authenticateHTTPRequest` 函数
2. 实现自定义的 JWT 或 OAuth 验证
3. 重新构建 Docker 镜像

### 自定义代理逻辑

1. 修改 `golang/main.go` 中的 `handleProxyRequest` 函数
2. 添加请求/响应拦截器
3. 实现自定义的负载均衡策略

## 技术栈总结

| 组件 | 技术 | 版本 | 用途 |
|------|------|------|------|
| 浏览器 | Camoufox | latest | 指纹随机化的 Firefox |
| 自动化 | Playwright | latest | 浏览器控制 |
| Python | Python | 3.11 | 浏览器实例管理 |
| 代理服务器 | Golang | 1.22 | WebSocket 和 HTTP 代理 |
| WebSocket | Gorilla | latest | WebSocket 通信库 |
| 容器 | Docker | latest | 应用打包与部署 |
| 进程管理 | Supervisor | latest | 多进程管理 |

## 总结

本系统通过以下创新设计实现了稳定的 Gemini API 代理：

1. **浏览器隧道技术**: 利用真实浏览器会话绕过网络限制
2. **WebSocket 双向通信**: 高效的请求-响应传输机制
3. **负载均衡**: 多实例并发提高系统容量
4. **容错设计**: 完善的超时、重试和错误处理机制
5. **Docker 容器化**: 简化部署和环境管理

该架构兼顾了**稳定性**、**性能**和**易用性**，适合作为 Google Gemini API 的本地代理解决方案。
