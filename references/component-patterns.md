# due 组件使用指南 (v2.5.2)

本文档详细介绍 due v2.5.2 框架提供的核心组件使用方法。

## 日志组件 (log/v2)

due v2.5.2 支持多种日志驱动：Console、File、Aliyun、Tencent。

### Console 日志 (v2.5.2)

```go
import "github.com/dobyte/due/v2/log"

// 设置日志级别
log.SetLevel(log.DebugLevel)

// 输出日志
log.Debug("调试信息")
log.Info("用户登录", "uid", 12345)
log.Warn("警告信息")
log.Error("错误信息", "error", err)
```

### File 日志 (v2.5.2)

```go
import (
    "github.com/dobyte/due/v2/log"
    "github.com/dobyte/due/log/file/v2"
)

// 创建 File 驱动
driver := file.NewDriver(
    file.WithDir("./logs"),
    file.WithFilename("game.log"),
    file.WithLevel(log.InfoLevel),
    file.WithMaxSize(100 * 1024 * 1024),  // 100MB
    file.WithMaxBackups(5),
    file.WithCompress(true),
)

// 设置日志
log.SetDriver(driver)
```

### Aliyun 日志 (v2.5.2)

```go
import (
    "github.com/dobyte/due/v2/log"
    "github.com/dobyte/due/log/aliyun/v2"
)

driver := aliyun.NewDriver(
    aliyun.WithEndpoint("cn-hangzhou.log.aliyuncs.com"),
    aliyun.WithAccessKeyID("your-access-key-id"),
    aliyun.WithAccessKeySecret("your-access-key-secret"),
    aliyun.WithProject("your-project"),
    aliyun.WithLogstore("your-logstore"),
)

log.SetDriver(driver)
```

### Tencent 日志 (v2.5.2)

```go
import (
    "github.com/dobyte/due/v2/log"
    "github.com/dobyte/due/log/tencent/v2"
)

driver := tencent.NewDriver(
    tencent.WithRegion("ap-guangzhou"),
    tencent.WithSecretID("your-secret-id"),
    tencent.WithSecretKey("your-secret-key"),
    tencent.WithTopicID("your-topic-id"),
)

log.SetDriver(driver)
```

### 结构化日志

```go
log.WithFields(log.Fields{
    "uid":   12345,
    "route": 1,
    "cost":  time.Millisecond * 100,
}).Info("用户请求")
```

## 配置组件 (config)

due 支持多种配置中心：Consul、Etcd、Nacos，支持 JSON/YAML/TOML/XML 格式。

### Consul 配置 (v2.5.2)

```go
import (
    "github.com/dobyte/due/config/consul/v2"
)

// 创建配置源
source := consul.NewSource(
    consul.WithAddr("127.0.0.1:8500"),
    consul.WithPath("config/game"),
)

// 创建配置
cfg := config.NewConfig(
    config.WithSource(source),
)

// 加载配置
if err := cfg.Load(); err != nil {
    log.Fatal(err)
}

// 获取配置
port := cfg.Int("gate.port")
host := cfg.Get("gate.host")
```

### Etcd 配置 (v2.5.2)

```go
import (
    "github.com/dobyte/due/config/etcd/v2"
)

source := etcd.NewSource(
    etcd.WithAddr("127.0.0.1:2379"),
    etcd.WithPath("config/game"),
)

cfg := config.NewConfig(config.WithSource(source))
cfg.Load()
```

### Nacos 配置 (v2.5.2)

```go
import (
    "github.com/dobyte/due/config/nacos/v2"
)

source := nacos.NewSource(
    nacos.WithAddr("127.0.0.1:8848"),
    nacos.WithDataID("game-config"),
    nacos.WithGroup("DEFAULT_GROUP"),
)

cfg := config.NewConfig(config.WithSource(source))
cfg.Load()
```

### 配置结构体绑定

```go
type Config struct {
    Gate struct {
        Port int    `json:"port"`
        Host string `json:"host"`
    } `json:"gate"`
    Node struct {
        WorkerSize int `json:"worker_size"`
    } `json:"node"`
}

var c Config
cfg.Scan(&c)
```

### 配置热更新

```go
// 监听配置变化
cfg.Watch(func(key string, value interface{}) {
    log.Infof("配置更新：%s = %v", key, value)
})
```

## 缓存组件 (cache)

due 支持 Redis 和 Memcache 缓存驱动。

### Redis 缓存 (v2.5.2)

```go
import (
    "github.com/redis/go-redis/v9"
    "github.com/dobyte/due/cache/redis/v2"
)

// 创建 Redis 客户端
client := redis.NewClient(&redis.Options{
    Addr:     "127.0.0.1:6379",
    Password: "your-password",
    DB:       0,
    PoolSize: 10,
})

// 基本操作
ctx := context.Background()
client.Set(ctx, "key", "value", time.Hour)
val, _ := client.Get(ctx, "key").Result()
client.Del(ctx, "key")
```

### Redis 高级操作

```go
// Hash 操作
client.HSet(ctx, "user:123", "name", "Alice")
client.HGet(ctx, "user:123", "name").Val()
client.HGetAll(ctx, "user:123").Val()

// List 操作
client.LPush(ctx, "queue", "item1", "item2")
client.RPop(ctx, "queue").Val()
client.LRange(ctx, "queue", 0, -1).Val()

// Set 操作
client.SAdd(ctx, "tags", "tag1", "tag2")
client.SMembers(ctx, "tags").Val()

// ZSet 操作
client.ZAdd(ctx, "leaderboard", redis.Z{Score: 100, Member: "player1"})
client.ZRevRange(ctx, "leaderboard", 0, 9).Val()
```

### Memcache 缓存

```go
import (
    "github.com/dobyte/due/cache/memcache/v2"
)

driver := memcache.NewDriver(
    memcache.WithAddr("127.0.0.1:11211"),
    memcache.WithTimeout(time.Second),
)
```

### 分布式锁 (v2.5.2)

```go
import "github.com/dobyte/due/v2/lock"

// 获取锁
lock := redis.NewLock(client, "resource:123", time.Second*30)

if err := lock.Acquire(context.Background()); err != nil {
    // 获取锁失败
    return
}

defer lock.Release(context.Background())

// 执行业务逻辑
```

## 事件总线 (eventbus)

due 支持多种 EventBus 后端：Redis、NATS、Kafka、RabbitMQ。

### Redis EventBus (v2.5.2)

```go
import (
    "github.com/dobyte/due/eventbus/redis/v2"
)

// 创建 EventBus
bus := redis.NewEventBus(
    redis.WithClient(client),
    redis.WithChannel("game.events"),
)

// 发布事件
bus.Publish(context.Background(), "user.login", &UserLoginEvent{
    UID:  12345,
    Time: time.Now(),
})

// 订阅事件
bus.Subscribe(context.Background(), "user.login", func(event *UserLoginEvent) {
    log.Infof("用户登录：%d", event.UID)
})
```

### NATS EventBus (v2.5.2)

```go
import (
    "github.com/dobyte/due/eventbus/nats/v2"
)

bus := nats.NewEventBus(
    nats.WithAddr("nats://127.0.0.1:4222"),
    nats.WithSubject("game.events"),
)

bus.Publish(context.Background(), "user.login", eventData)
bus.Subscribe(context.Background(), "user.login", handler)
```

### Kafka EventBus (v2.5.2)

```go
import (
    "github.com/dobyte/due/eventbus/kafka/v2"
)

bus := kafka.NewEventBus(
    kafka.WithBrokers([]string{"127.0.0.1:9092"}),
    kafka.WithTopic("game-events"),
    kafka.WithGroupID("game-consumer-group"),
)

bus.Publish(context.Background(), "user.login", eventData)
bus.Subscribe(context.Background(), "user.login", handler)
```

### RabbitMQ EventBus (v2.5.2)

```go
import (
    "github.com/dobyte/due/eventbus/rabbitmq/v2"
)

bus := rabbitmq.NewEventBus(
    rabbitmq.WithAddr("amqp://guest:guest@127.0.0.1:5672/"),
    rabbitmq.WithExchange("game-exchange"),
    rabbitmq.WithRoutingKey("user.login"),
)

bus.Publish(context.Background(), "user.login", eventData)
bus.Subscribe(context.Background(), "user.login", handler)
```

## 服务注册 (registry/v2)

due v2.5.2 支持 Consul、Etcd、Nacos 服务注册。

### Consul Registry (v2.5.2)

```go
import (
    "github.com/dobyte/due/registry/consul/v2"
)

reg := consul.NewRegistry(
    consul.WithAddr("127.0.0.1:8500"),
    consul.WithID("gate-001"),
    consul.WithName("gate"),
    consul.WithHealthCheck(true),
    consul.WithCheckInterval(time.Second*10),
)

// 注册服务会自动在组件启动时执行
component := gate.NewGate(
    gate.WithRegistry(reg),
)
```

### Etcd Registry (v2.5.2)

```go
import (
    "github.com/dobyte/due/registry/etcd/v2"
)

reg := etcd.NewRegistry(
    etcd.WithAddr("127.0.0.1:2379"),
    etcd.WithID("node-001"),
    etcd.WithName("node"),
)
```

### Nacos Registry (v2.5.2)

```go
import (
    "github.com/dobyte/due/registry/nacos/v2"
)

reg := nacos.NewRegistry(
    nacos.WithAddr("127.0.0.1:8848"),
    nacos.WithNamespace("public"),
)
```

### 服务发现

```go
// 获取服务列表
services, err := reg.Discover("gate")

// 获取特定服务
service, err := reg.GetService("gate-001")

// 监听服务变化
reg.Watch("gate", func(services []*registry.Service) {
    for _, s := range services {
        log.Infof("服务：%s - %s:%d", s.ID, s.Address, s.Port)
    }
})
```

## 加密组件 (crypto)

due 支持 RSA 和 ECC 加密。

### RSA 加密 (v2.5.2)

```go
import (
    "github.com/dobyte/due/crypto/rsa/v2"
)

// 生成密钥对
privateKey, publicKey, err := rsa.GenerateKey(2048)

// 公钥加密
ciphertext, err := rsa.Encrypt(publicKey, plaintext)

// 私钥解密
plaintext, err := rsa.Decrypt(privateKey, ciphertext)

// 私钥签名
signature, err := rsa.Sign(privateKey, data)

// 公钥验签
valid, err := rsa.Verify(publicKey, data, signature)
```

### ECC 加密 (v2.5.2)

```go
import (
    "github.com/dobyte/due/crypto/ecc/v2"
)

// 生成密钥对
privateKey, publicKey, err := ecc.GenerateKey()

// 公钥加密
ciphertext, err := ecc.Encrypt(publicKey, plaintext)

// 私钥解密
plaintext, err := ecc.Decrypt(privateKey, ciphertext)
```

## 会话组件 (session)

### Session 管理 (v2.5.2)

在 due v2.5.2 中，Session 通过 Context 获取：

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
}
```

### Session 绑定

```go
// 在 Gate 连接时绑定
func onConnect(ctx gate.Context) {
    // 将连接绑定到 Actor (UID)
    ctx.Session().Bind(uid)
}

// 解除绑定
func onDisconnect(ctx gate.Context) {
    ctx.Session().Unbind()
}
```

## 任务组件 (task)

### 本地任务 (v2.5.2)

```go
import "github.com/dobyte/due/v2/task"

// 创建定时器
timer := task.AfterFunc(time.Second*5, func() {
    log.Info("定时器触发")
})

// 取消定时器
timer.Stop()

// 周期性任务
ticker := task.TickFunc(time.Second*10, func() {
    log.Info("周期性任务")
})

// 停止周期性任务
ticker.Stop()
```

### 分布式任务 (v2.5.2)

```go
import (
    "github.com/dobyte/due/task/redis/v2"
    "github.com/dobyte/due/v2/task"
)

// 创建分布式任务
driver := redis.NewDriver(
    redis.WithClient(client),
    redis.WithKey("task:leader"),
)

task := task.NewTask(
    task.WithDriver(driver),
    task.WithInterval(time.Minute),
    task.WithCallback(func() {
        // 只有领导者会执行
        log.Info("执行分布式任务")
    }),
)

task.Start()
```

## 最佳实践

### ✅ 推荐做法

- 使用结构化日志便于查询分析
- 配置与服务代码分离
- 缓存设置合理的过期时间
- EventBus 用于解耦服务
- 敏感数据使用加密传输

### ❌ 避免做法

- 日志中输出敏感信息
- 硬编码配置值
- 缓存永不过期
- 忽略 EventBus 连接错误
- 使用弱加密算法
