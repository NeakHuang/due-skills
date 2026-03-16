# due 网关开发模式 (v2.5.2)

本文档详细介绍 due v2.5.2 框架中 Gate（网关）服务的开发模式。

## Gate 概述

Gate 服务是客户端与游戏服务器之间的入口，负责：
- 客户端连接管理（TCP/KCP/WebSocket）
- 消息协议编解码
- 会话管理
- 消息路由到 Node

## v2.5.2 Gate 完整示例

```go
package main

import (
    "github.com/dobyte/due/locate/redis/v2"
    "github.com/dobyte/due/network/ws/v2"
    "github.com/dobyte/due/registry/consul/v2"
    "github.com/dobyte/due/v2"
    "github.com/dobyte/due/v2/cluster/gate"
)

func main() {
    // 创建容器
    container := due.NewContainer()

    // 创建 WebSocket 服务器
    server := ws.NewServer(
        ws.WithPort(8800),
    )

    // 创建定位器
    locator := redis.NewLocator(
        redis.WithAddr("127.0.0.1:6379"),
    )

    // 创建服务注册
    registry := consul.NewRegistry(
        consul.WithAddr("127.0.0.1:8500"),
    )

    // 创建 Gate 组件
    component := gate.NewGate(
        gate.WithID("gate-001"),
        gate.WithName("gate"),
        gate.WithServer(server),
        gate.WithLocator(locator),
        gate.WithRegistry(registry),
    )

    // 添加组件到容器
    container.Add(component)

    // 启动服务
    container.Serve()
}
```

## 协议支持 (v2.5.2)

due v2.5.2 支持三种主流协议：

### WebSocket 协议

适用于 H5 游戏、即时通讯等场景：

```go
package main

import (
    "github.com/dobyte/due/locate/redis/v2"
    "github.com/dobyte/due/network/ws/v2"
    "github.com/dobyte/due/registry/consul/v2"
    "github.com/dobyte/due/v2"
    "github.com/dobyte/due/v2/cluster/gate"
)

func main() {
    container := due.NewContainer()

    // WebSocket 服务器
    server := ws.NewServer(
        ws.WithPort(8800),
        ws.WithMaxConnNum(10000),
    )

    // 定位器
    locator := redis.NewLocator()

    // 注册中心
    registry := consul.NewRegistry()

    // Gate 组件
    component := gate.NewGate(
        gate.WithServer(server),
        gate.WithLocator(locator),
        gate.WithRegistry(registry),
    )

    container.Add(component)
    container.Serve()
}
```

### TCP 协议

适用于对实时性要求高的游戏，如动作游戏、射击游戏等：

```go
package main

import (
    "github.com/dobyte/due/locate/redis/v2"
    "github.com/dobyte/due/network/tcp/v2"
    "github.com/dobyte/due/registry/consul/v2"
    "github.com/dobyte/due/v2"
    "github.com/dobyte/due/v2/cluster/gate"
)

func main() {
    container := due.NewContainer()

    // TCP 服务器
    server := tcp.NewServer(
        tcp.WithPort(9000),
        tcp.WithMaxConnNum(10000),
    )

    locator := redis.NewLocator()
    registry := consul.NewRegistry()

    component := gate.NewGate(
        gate.WithServer(server),
        gate.WithLocator(locator),
        gate.WithRegistry(registry),
    )

    container.Add(component)
    container.Serve()
}
```

#### TCP 协议配置选项

```go
server := tcp.NewServer(
    tcp.WithPort(9000),             // 监听端口
    tcp.WithMaxConnNum(10000),      // 最大连接数
    tcp.WithMsgSize(4096),          // 最大消息大小（字节）
    tcp.WithSendChanSize(1024),     // 发送缓冲区大小
    tcp.WithHeartbeatInterval(30),  // 心跳间隔（秒）
    tcp.WithHeartbeatHandler(heartbeatHandler), // 心跳处理器
)
```

#### TCP 完整示例（带心跳处理）

```go
package main

import (
    "github.com/dobyte/due/locate/redis/v2"
    "github.com/dobyte/due/network/tcp/v2"
    "github.com/dobyte/due/registry/consul/v2"
    "github.com/dobyte/due/v2"
    "github.com/dobyte/due/v2/cluster/gate"
    "github.com/dobyte/due/v2/log"
)

func main() {
    container := due.NewContainer()

    // TCP 服务器
    server := tcp.NewServer(
        tcp.WithPort(9000),
        tcp.WithMaxConnNum(10000),
        tcp.WithMsgSize(8192),
        tcp.WithSendChanSize(2048),
        tcp.WithHeartbeatInterval(60),
        tcp.WithHeartbeatHandler(handleHeartbeat),
    )

    // 定位器
    locator := redis.NewLocator(
        redis.WithAddr("127.0.0.1:6379"),
    )

    // 注册中心
    registry := consul.NewRegistry(
        consul.WithAddr("127.0.0.1:8500"),
        consul.WithID("gate-tcp-001"),
        consul.WithName("gate"),
    )

    // Gate 组件
    component := gate.NewGate(
        gate.WithID("gate-tcp-001"),
        gate.WithName("gate"),
        gate.WithServer(server),
        gate.WithLocator(locator),
        gate.WithRegistry(registry),
    )

    container.Add(component)
    container.Serve()
}

// handleHeartbeat 心跳处理
func handleHeartbeat(ctx gate.Context) {
    log.Infof("收到心跳 cid=%d, uid=%d", ctx.Session().Cid(), ctx.Session().Uid())
}
```

### KCP 协议

适用于弱网络环境下的实时游戏，如 MOBA、吃鸡等对战游戏：

```go
package main

import (
    "github.com/dobyte/due/locate/redis/v2"
    "github.com/dobyte/due/network/kcp/v2"
    "github.com/dobyte/due/registry/consul/v2"
    "github.com/dobyte/due/v2"
    "github.com/dobyte/due/v2/cluster/gate"
)

func main() {
    container := due.NewContainer()

    // KCP 服务器
    server := kcp.NewServer(
        kcp.WithPort(10000),
        kcp.WithMode(0),  // 0: 快速，1: 正常，2: 流畅
    )

    locator := redis.NewLocator()
    registry := consul.NewRegistry()

    component := gate.NewGate(
        gate.WithServer(server),
        gate.WithLocator(locator),
        gate.WithRegistry(registry),
    )

    container.Add(component)
    container.Serve()
}
```

#### KCP 协议配置选项

```go
server := kcp.NewServer(
    kcp.WithPort(10000),            // 监听端口
    kcp.WithMode(0),                // 模式：0=快速 (20ms RTT), 1=正常 (40ms RTT), 2=流畅 (60ms RTT)
    kcp.WithMaxConnNum(5000),       // 最大连接数
    kcp.WithMsgSize(4096),          // 最大消息大小
    kcp.WithSendChanSize(1024),     // 发送缓冲区大小
)
```

#### KCP 模式说明

| 模式 | 值 | RTT 目标 | 适用场景 |
|------|---|---------|---------|
| 快速 | 0 | 20ms | 竞技游戏、音游 |
| 正常 | 1 | 40ms | MOBA、FPS |
| 流畅 | 2 | 60ms | MMO、休闲游戏 |

#### KCP 完整示例

```go
package main

import (
    "github.com/dobyte/due/locate/redis/v2"
    "github.com/dobyte/due/network/kcp/v2"
    "github.com/dobyte/due/registry/consul/v2"
    "github.com/dobyte/due/v2"
    "github.com/dobyte/due/v2/cluster/gate"
    "github.com/dobyte/due/v2/log"
)

func main() {
    container := due.NewContainer()

    // KCP 服务器 - 使用正常模式
    server := kcp.NewServer(
        kcp.WithPort(10000),
        kcp.WithMode(1),              // 正常模式
        kcp.WithMaxConnNum(5000),
        kcp.WithMsgSize(8192),
        kcp.WithSendChanSize(2048),
    )

    // 定位器
    locator := redis.NewLocator(
        redis.WithAddr("127.0.0.1:6379"),
        redis.WithDB(0),
    )

    // 注册中心
    registry := consul.NewRegistry(
        consul.WithAddr("127.0.0.1:8500"),
        consul.WithID("gate-kcp-001"),
        consul.WithName("gate"),
    )

    // Gate 组件
    component := gate.NewGate(
        gate.WithID("gate-kcp-001"),
        gate.WithName("gate"),
        gate.WithServer(server),
        gate.WithLocator(locator),
        gate.WithRegistry(registry),
    )

    container.Add(component)
    container.Serve()

    log.Info("KCP Gate 服务已启动，端口：10000")
}
```

#### KCP 协议选择建议

- **MOBA/FPS 游戏**：推荐使用模式 1（正常），平衡延迟和带宽
- **音游/格斗游戏**：推荐使用模式 0（快速），最低延迟
- **MMORPG**：推荐使用模式 2（流畅），节省带宽
- **卡牌/回合制**：建议使用 WebSocket，KCP 优势不明显

## 消息协议 (v2.5.2)

### 默认数据包格式

```
┌────────┬────────┬──────┬─────┬─────────┐
│  size  │ header │ route│ seq │ message │
│ 2 bytes│1 byte  │2 bytes│2 bytes│ 可变   │
└────────┴────────┴──────┴─────┴─────────┘
```

- **size**: 数据包总长度（2 字节）
- **header**: 消息头标识（1 字节）
- **route**: 消息路由/类型（2 字节）
- **seq**: 序列号（2 字节，用于请求响应匹配）
- **message**: 消息体（可变长度）

### 心跳包格式

```
┌────────┬────────┬─────────┬──────────────┐
│  size  │ header │ extcode │ heartbeat_time│
│ 2 bytes│1 byte  │1 byte   │ 4 bytes      │
└────────┴────────┴─────────┴──────────────┘
```

## 会话管理 (v2.5.2)

### 基础 Session 操作

在 due v2.5.2 中，Session 由框架自动管理，通过 `ctx.Session()` 获取：

```go
func handler(ctx gate.Context) {
    // 获取 Session
    session := ctx.Session()

    // 获取连接 ID
    cid := session.Cid()

    // 获取绑定的 UID
    uid := session.Uid()

    // 设置 Session 数据
    session.Set("key", "value")

    // 获取 Session 数据
    value := session.Get("key")

    // 删除 Session 数据
    session.Del("key")
}
```

### 玩家绑定 (UID Binding)

将玩家 UID 绑定到 Session 是使用 due 框架的核心操作：

```go
package handler

import (
    "github.com/dobyte/due/v2/cluster/gate"
    "github.com/dobyte/due/v2/codes"
)

// loginReq 登录请求
type loginReq struct {
    UID   int64  `json:"uid"`
    Token string `json:"token"`
}

// loginRes 登录响应
type loginRes struct {
    Code int    `json:"code"`
    Msg  string `json:"msg"`
}

// loginHandler 玩家登录处理器
func loginHandler(ctx gate.Context) {
    req := &loginReq{}
    res := &loginRes{}
    defer func() { ctx.Response(res) }()

    // 解析请求
    if err := ctx.Parse(req); err != nil {
        res.Code = codes.InvalidArgument.Code()
        return
    }

    // TODO: 验证 token 有效性

    // 将 UID 绑定到 Session
    // 绑定后，消息可以通过 UID 路由到该玩家
    if err := ctx.Session().Bind(req.UID); err != nil {
        res.Code = codes.InternalError.Code()
        return
    }

    res.Code = codes.OK.Code()
    res.Msg = "登录成功"
}
```

### 玩家推送消息 (Push by UID)

绑定 UID 后，可以通过 UID 向特定玩家推送消息：

```go
import (
    "context"
    "github.com/dobyte/due/v2/cluster/gate"
)

// SendToUser 向指定玩家推送消息
func SendToUser(ctx context.Context, proxy *gate.Proxy, uid int64, route int32, data interface{}) error {
    return proxy.Push(ctx, uid, route, data)
}

// 使用示例
func notifyPlayer(proxy *gate.Proxy, uid int64) {
    type notifyData struct {
        Title string `json:"title"`
        Content string `json:"content"`
    }

    proxy.Push(context.Background(), uid, 1001, &notifyData{
        Title: "系统通知",
        Content: "欢迎加入游戏!",
    })
}
```

### 玩家批量推送 (Broadcast)

向多个玩家批量推送消息：

```go
// BroadcastToPlayers 向多个玩家推送消息
func BroadcastToPlayers(ctx context.Context, proxy *gate.Proxy, uids []int64, route int32, data interface{}) error {
    for _, uid := range uids {
        if err := proxy.Push(ctx, uid, route, data); err != nil {
            // 处理推送失败（玩家可能已离线）
            log.Warnf("推送失败 uid=%d, error=%v", uid, err)
        }
    }
    return nil
}
```

### 玩家离线处理

当玩家断开连接时，需要清理相关资源：

```go
// onDisconnect 玩家断开连接处理
func onDisconnect(ctx gate.Context) {
    session := ctx.Session()
    uid := session.Uid()

    if uid > 0 {
        log.Infof("玩家离线 uid=%d, cid=%d", uid, session.Cid())

        // TODO: 清理玩家相关数据
        // - 从房间中移除
        // - 通知好友
        // - 保存游戏状态
    }
}
```

### 玩家重连处理

due 支持玩家重连后重新绑定 Session：

```go
// reconnectHandler 重连处理器
func reconnectHandler(ctx gate.Context) {
    req := &struct {
        UID   int64  `json:"uid"`
        Token string `json:"token"`
    }{}

    if err := ctx.Parse(req); err != nil {
        return
    }

    // TODO: 验证 token 有效性

    // 重新绑定 UID，框架会自动处理之前的连接
    if err := ctx.Session().Bind(req.UID); err != nil {
        log.Errorf("重连绑定失败 uid=%d, error=%v", req.UID, err)
        return
    }

    log.Infof("玩家重连成功 uid=%d", req.UID)
}
```

### Session 数据存储

使用 Redis 存储 Session 数据，实现分布式 Session：

```go
import (
    "github.com/dobyte/due/v2/session/redis"
)

// 配置 Redis Session 存储
func setupSession() {
    store := redis.NewStore(
        redis.WithAddr("127.0.0.1:6379"),
        redis.WithPassword(""),
        redis.WithDB(0),
        redis.WithIdleTimeout(300), // 5 分钟空闲超时
    )

    // 设置 Session 存储
    session.SetStore(store)
}
```

### 完整玩家管理示例

```go
package player

import (
    "context"
    "github.com/dobyte/due/v2/cluster/gate"
    "github.com/dobyte/due/v2/codes"
    "github.com/dobyte/due/v2/log"
)

const (
    RouteLogin = 1001
    RouteLogout = 1002
    RouteReconnect = 1003
)

// LoginRequest 登录请求
type LoginRequest struct {
    UID   int64  `json:"uid"`
    Token string `json:"token"`
}

// LoginResponse 登录响应
type LoginResponse struct {
    Code int    `json:"code"`
    Msg  string `json:"msg"`
}

// Login 玩家登录
func Login(ctx gate.Context) {
    req := &LoginRequest{}
    res := &LoginResponse{}
    defer func() { ctx.Response(res) }()

    if err := ctx.Parse(req); err != nil {
        res.Code = codes.InvalidArgument.Code()
        res.Msg = "参数错误"
        return
    }

    // 验证 token
    if !validateToken(req.Token, req.UID) {
        res.Code = codes.Unauthenticated.Code()
        res.Msg = "token 无效"
        return
    }

    // 绑定玩家到 Session
    if err := ctx.Session().Bind(req.UID); err != nil {
        res.Code = codes.InternalError.Code()
        res.Msg = "绑定失败"
        return
    }

    // 设置 Session 数据
    ctx.Session().Set("login_time", time.Now().Unix())

    res.Code = codes.OK.Code()
    res.Msg = "登录成功"
    log.Infof("玩家登录 uid=%d, cid=%d", req.UID, ctx.Session().Cid())
}

// Logout 玩家登出
func Logout(ctx gate.Context) {
    session := ctx.Session()
    uid := session.Uid()

    if uid > 0 {
        log.Infof("玩家登出 uid=%d", uid)
        // 清理玩家数据
        cleanupPlayerData(uid)
    }

    ctx.Response(map[string]int{"code": codes.OK.Code()})
}

// OnDisconnect 玩家断开连接回调
func OnDisconnect(ctx gate.Context) {
    session := ctx.Session()
    uid := session.Uid()

    if uid > 0 {
        log.Infof("玩家断开连接 uid=%d, cid=%d", uid, session.Cid())
        cleanupPlayerData(uid)
    }
}

// validateToken 验证 token（示例）
func validateToken(token string, uid int64) bool {
    // TODO: 实现 token 验证逻辑
    return token != ""
}

// cleanupPlayerData 清理玩家数据
func cleanupPlayerData(uid int64) {
    // TODO: 清理玩家在房间中的数据、通知好友等
}
```

## 消息路由 (v2.5.2)

在 due v2.5.2 中，Gate 自动处理消息路由到 Node，无需手动配置 Match 函数。

消息自动根据 route 转发到对应的 Node Actor：

```
Client → Gate(自动路由) → Node(Proxy.Router 处理)
```

## 配置选项 (v2.5.2)

### Gate 组件配置

```go
component := gate.NewGate(
    gate.WithID("gate-001"),        // 服务 ID
    gate.WithName("gate"),          // 服务名称
    gate.WithServer(server),        // 网络服务器
    gate.WithLocator(locator),      // 定位器
    gate.WithRegistry(registry),    // 注册中心
)
```

### WebSocket 服务器配置

```go
server := ws.NewServer(
    ws.WithPort(8800),              // 端口
    ws.WithMaxConnNum(10000),       // 最大连接数
    ws.WithMsgSize(4096),           // 消息大小限制
    ws.WithHeartbeatInterval(30),   // 心跳间隔（秒）
)
```

### 定位器配置

```go
locator := redis.NewLocator(
    redis.WithAddr("127.0.0.1:6379"),
    redis.WithPassword("password"),
    redis.WithDB(0),
)
```

### 注册中心配置

```go
registry := consul.NewRegistry(
    consul.WithAddr("127.0.0.1:8500"),
    consul.WithID("gate-001"),
    consul.WithName("gate"),
)
```

## 完整示例 (v2.5.2)

```go
package main

import (
    "github.com/dobyte/due/locate/redis/v2"
    "github.com/dobyte/due/network/ws/v2"
    "github.com/dobyte/due/registry/consul/v2"
    "github.com/dobyte/due/v2"
    "github.com/dobyte/due/v2/cluster/gate"
)

func main() {
    // 创建容器
    container := due.NewContainer()

    // WebSocket 服务器
    server := ws.NewServer(
        ws.WithPort(8800),
        ws.WithMaxConnNum(10000),
        ws.WithMsgSize(4096),
        ws.WithHeartbeatInterval(30),
    )

    // 定位器
    locator := redis.NewLocator(
        redis.WithAddr("127.0.0.1:6379"),
    )

    // 注册中心
    registry := consul.NewRegistry(
        consul.WithAddr("127.0.0.1:8500"),
        consul.WithID("gate-001"),
        consul.WithName("gate"),
    )

    // Gate 组件
    component := gate.NewGate(
        gate.WithID("gate-001"),
        gate.WithName("gate"),
        gate.WithServer(server),
        gate.WithLocator(locator),
        gate.WithRegistry(registry),
    )

    // 添加并启动
    container.Add(component)
    container.Serve()
}
```

## v2.5.2 变化说明

**重要**: due v2.5.2 调整了组件使用方式：

1. **使用 Container 统一管理**：所有组件通过 `due.NewContainer()` 管理
2. **Gate 作为组件**：使用 `gate.NewGate()` 创建组件，而非直接创建 ws.NewGate()
3. **自动路由**：消息自动路由到 Node，无需手动 Match 函数
4. **Session 管理**：通过 `ctx.Session()` 获取会话

## 最佳实践

### ✅ 推荐做法

- 使用 Container 统一管理组件
- 配置服务注册实现服务发现
- 配置定位器用于消息路由
- 合理设置消息大小限制
- 配置心跳检测机制

### ❌ 避免做法

- 在 Gate 层执行业务逻辑
- 不使用 Container 直接调用 Serve()
- 忽略配置服务注册和定位器
- 不限制消息大小
