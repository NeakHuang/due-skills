# due 项目结构规范

本文档介绍 due 框架的项目结构规范和目录组织。

## 标准项目结构

```
my-game/
├── cmd/                        # 应用入口
│   ├── gate/                   # 网关服务入口
│   │   └── main.go
│   ├── node/                   # 节点服务入口
│   │   └── main.go
│   └── mesh/                   # 微服务入口
│       └── main.go
├── internal/                   # 私有业务逻辑
│   ├── actor/                  # Actor 实现
│   │   ├── player.go
│   │   ├── room.go
│   │   └── npc.go
│   ├── handler/                # 消息处理器
│   │   ├── login.go
│   │   ├── move.go
│   │   └── chat.go
│   ├── logic/                  # 业务逻辑
│   │   ├── user.go
│   │   ├── item.go
│   │   └── battle.go
│   ├── model/                  # 数据模型
│   │   ├── user.go
│   │   ├── item.go
│   │   └── world.go
│   └── middleware/             # 中间件
│       ├── auth.go
│       └── logging.go
├── pkg/                        # 公共库
│   ├── proto/                  # Protobuf 定义
│   ├── utils/                  # 工具函数
│   └── consts/                 # 常量定义
├── config/                     # 配置文件
│   ├── dev.yaml
│   ├── prod.yaml
│   └── local.yaml
├── scripts/                    # 脚本文件
│   ├── deploy.sh
│   └── migration/
├── test/                       # 测试文件
│   ├── actor_test.go
│   └── handler_test.go
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
└── docker-compose.yml
```

## 目录说明

### cmd/
存放应用入口文件，每个子目录对应一个可执行文件。

```
cmd/
├── gate/           # 网关服务
│   └── main.go
├── node/           # 节点服务
│   └── main.go
└── mesh/           # 微服务
    └── main.go
```

### internal/
存放私有业务逻辑，不能被外部模块引用。

#### actor/
Actor 实现目录：

```go
// internal/actor/player.go
package actor

import (
    "context"
    "github.com/dobyte/due/v2/core/actor"
)

type PlayerActor struct {
    *actor.Base
    uid   int64
    data  *PlayerData
}

func NewPlayerActor(opts ...actor.Option) *PlayerActor {
    return &PlayerActor{
        Base: actor.NewBase(opts...),
    }
}

func (a *PlayerActor) OnInit() {
    a.uid = int64(a.ID())
    a.data = loadPlayerData(a.uid)
}

func (a *PlayerActor) OnMessage(ctx context.Context, message *actor.Message) {
    // 处理消息
}
```

#### handler/
消息处理器目录：

```go
// internal/handler/login.go
package handler

import (
    "context"
    "github.com/dobyte/due/network/ws/v2"
)

func LoginHandler(ctx context.Context, conn *ws.Conn, message []byte) {
    // 处理登录
}
```

#### logic/
业务逻辑目录：

```go
// internal/logic/user.go
package logic

import (
    "github.com/dobyte/due/internal/model"
)

type UserLogic struct {
    userModel *model.UserModel
}

func NewUserLogic() *UserLogic {
    return &UserLogic{
        userModel: model.NewUserModel(),
    }
}

func (l *UserLogic) GetUser(uid int64) (*model.User, error) {
    return l.userModel.FindOne(uid)
}
```

#### model/
数据模型目录：

```go
// internal/model/user.go
package model

import "database/sql"

type User struct {
    UID      int64  `json:"uid"`
    Username string `json:"username"`
    Password string `json:"-"`
    Level    int    `json:"level"`
}

type UserModel struct {
    db *sql.DB
}

func NewUserModel(db *sql.DB) *UserModel {
    return &UserModel{db: db}
}

func (m *UserModel) FindOne(uid int64) (*User, error) {
    // 查询数据库
    return &User{}, nil
}
```

### pkg/
存放公共库代码，可以被外部模块引用。

```
pkg/
├── proto/          # Protobuf 定义
├── utils/          # 工具函数
├── consts/         # 常量定义
└── errors/         # 错误定义
```

### config/
配置文件目录：

```yaml
# config/dev.yaml
gate:
  port: 8800
  log_level: debug

node:
  worker_size: 32

database:
  host: localhost
  port: 3306
  user: root
  password: root

redis:
  addr: localhost:6379

consul:
  addr: localhost:8500
```

### scripts/
脚本文件目录：

```
scripts/
├── deploy.sh       # 部署脚本
├── build.sh        # 构建脚本
└── migration/      # 数据库迁移脚本
    ├── 001_init.sql
    └── 002_add_index.sql
```

### test/
测试文件目录：

```go
// test/actor_test.go
package test

import (
    "testing"
    "github.com/dobyte/due/internal/actor"
)

func TestPlayerActor(t *testing.T) {
    // 测试 Actor
}
```

## Makefile 示例

```makefile
.PHONY: build test clean run-gate run-node docker

# 构建
build:
    go build -o bin/gate ./cmd/gate
    go build -o bin/node ./cmd/node

# 测试
test:
    go test -v ./...

# 清理
clean:
    rm -rf bin/
    go clean

# 运行网关
run-gate:
    go run ./cmd/gate

# 运行节点
run-node:
    go run ./cmd/node

# Docker 构建
docker:
    docker build -t my-game:latest .

# Docker Compose
docker-up:
    docker-compose up -d

docker-down:
    docker-compose down
```

## Dockerfile 示例

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 依赖
COPY go.mod go.sum ./
RUN go mod download

# 源码
COPY . .

# 构建
RUN CGO_ENABLED=0 GOOS=linux go build -o gate ./cmd/gate
RUN CGO_ENABLED=0 GOOS=linux go build -o node ./cmd/node

# 运行镜像
FROM alpine:latest

RUN apk --no-cache add ca-certificates tzdata

WORKDIR /app
COPY --from=builder /app/gate .
COPY --from=builder /app/node .
COPY config/ ./config/

EXPOSE 8800

CMD ["./gate"]
```

## 模块依赖关系

```
┌─────────────────────────────────────────┐
│              cmd/                       │
│  (gate, node, mesh 入口)                │
└─────────────────┬───────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
    ▼             ▼             ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│internal/│ │  pkg/   │ │ config/ │
│ actor/  │ │ proto/  │ │  *.yaml │
│ handler/│ │ utils/  │ └─────────┘
│ logic/  │ │ consts/ │
│ model/  │ │ errors/ │
└─────────┘ └─────────┘
```

## 命名规范

### 文件和目录
- 使用小写字母
- 多个单词用连字符分隔：`user-service.go`

### 包名
- 使用小写字母
- 避免使用复数：用 `model` 而非 `models`
- 包名与目录名一致

### 变量和函数
- 导出标识符使用大写字母开头：`User`, `GetUser()`
- 私有标识符使用小写字母开头：`user`, `getUser()`

### 常量
```go
const (
    RouteLogin = 1
    RouteChat  = 2
)
```

## 最佳实践

### ✅ 推荐做法

- 使用 `internal/` 隐藏私有实现
- `cmd/` 只包含入口代码
- `pkg/` 存放可复用代码
- 配置文件与环境分离
- 编写单元测试

### ❌ 避免做法

- 在 `cmd/` 中写业务逻辑
- 循环依赖
- 硬编码配置值
- 忽略错误处理
