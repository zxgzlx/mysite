---
title: "Nano游戏框架简介"
date: 2020-05-02T22:41:41+08:00
draft: true
---

### Golang nano游戏框架学习

开源地址：https://github.com/lonng/nano
以下内容转载README.md

#### 基本概念

##### 1.组件(Component)
>nano由松散耦合的Component组成，每个Component完成一些功能。整个应用可以看作是一个>Component容器，完成Component的加载以及生命周期管理。每个Component有Init，AfterInit，>BeforeShutdown，Shutdown等方法，用来完成生命周期管理。
```go
type DemoComponent struct{}

func (c *DemoComponent) Init()           {}
func (c *DemoComponent) AfterInit()      {}
func (c *DemoComponent) BeforeShutdown() {}
func (c *DemoComponent) Shutdown()       {}
```

##### 1.Handler
>Handler用来处理业务逻辑，Handler可以有如下形式的签名：
```go
// 以下的Handler会自动将消息反序列化，在调用时当做参数传进来
func (c *DemoComponent) DemoHandler(s *session.Session, payload *pb.DemoPayload) error {
    // 业务逻辑开始
    // ...
    // 业务逻辑结束

    return nil
}

// 以下的Handler不会自动将消息反序列化，会将客户端发送过来的消息直接当作参数传进来
func (c *DemoComponent) DemoHandler(s *session.Session, raw []byte) error {
    // 业务逻辑开始
    // ...
    // 业务逻辑结束

    return nil
}
```
##### 3.路由(Route)
>route用来标识一个具体服务或者客户端接受服务端推送消息的位置，对服务端来说，其形式一般是..,例如 "Room.Message", 在我们的示例中, Room是一个包含相关Handler的组件, Message是一个定义在 Room中的Handler, Room中所有符合Handler签名的方法都会在nano应用启动时自动注册.
>对客户端来说，其路由一般形式为onXXX(比如我们示例中的onMessage)，当服务端推送消息时，客户端会 有相应的回调。
##### 4.会话(Session)
>Session对应于一个客户端会话, 当客户端连接服务器后, 会建立一个会话, 会话在玩家保持连接期间可以 用于保存一些上下文信息, 这些信息会在连接断开后释放.
##### 5.组(Group)
>Group可以看作是一个Session的容器，主要用于需要广播推送消息的场景。可以把某个玩家的Session加 入到一个Group中，当对这个Group推送消息的时候，所有加入到这个Group的玩家都会收到推送过来的消 息。一个玩家的Session可能会被加入到多个Group中，这样玩家就会收到其加入的Group推送过来的消息。

##### 6.请求(Request), 响应(Response), 通知(Notify), 推送(Push)
>nano中有四种消息类型的消息, 分别是:
>请求(Request)
>响应(Response)
>通知(Notify)
>推送(Push)

* 客户端发起Request到服务器端，服务器端处理后会给其返回响应Response;
* Notify是客户端发给服务端的通知，也就是不需要服务端给予回复的请求;
* Push是服务端主动给客户端推送消息的类型。
#### 示例
Server
```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/lonnng/nano"
	"github.com/lonnng/nano/component"
	"github.com/lonnng/nano/serialize/json"
	"github.com/lonnng/nano/session"
)

type (
	// define component
	Room struct {
		component.Base
		group *nano.Group
	}

	// protocol messages
	UserMessage struct {
		Name    string `json:"name"`
		Content string `json:"content"`
	}

	NewUser struct {
		Content string `json:"content"`
	}

	AllMembers struct {
		Members []int64 `json:"members"`
	}

	JoinResponse struct {
		Code   int    `json:"code"`
		Result string `json:"result"`
	}
)

func NewRoom() *Room {
	return &Room{
		group: nano.NewGroup("room"),
	}
}

func (r *Room) AfterInit() {
	nano.OnSessionClosed(func(s *session.Session) {
		r.group.Leave(s)
	})
}

// Join room
func (r *Room) Join(s *session.Session, msg []byte) error {
	s.Bind(s.ID()) // binding session uid
	s.Push("onMembers", &AllMembers{Members: r.group.Members()})
	// notify others
	r.group.Broadcast("onNewUser", &NewUser{Content: fmt.Sprintf("New user: %d", s.ID())})
	// new user join group
	r.group.Add(s) // add session to group
	return s.Response(&JoinResponse{Result: "sucess"})
}

// Send message
func (r *Room) Message(s *session.Session, msg *UserMessage) error {
	return r.group.Broadcast("onMessage", msg)
}

func main() {
	nano.Register(NewRoom())
	nano.SetSerializer(json.NewSerializer())
	nano.EnableDebug()
	log.SetFlags(log.LstdFlags | log.Llongfile)

	http.Handle("/web/", http.StripPrefix("/web/", http.FileServer(http.Dir("web"))))

	nano.SetCheckOriginFunc(func(_ *http.Request) bool { return true })
	nano.Listen(":3250", nano.WithIsWebsocket(true))
}
```
1. 首先, 导入这个代码片段需要应用的包
2. 定义Room组件
3. 定义所有全后端交互可能用到的协议结构体(实际项目中可能使用Protobuf)
4. 定义所有的Handler, 这里包含Join和Message
5. 启动我们的应用
	- 注册组件
	- 设置序列化反序列器
	- 开启调试信息
	- 设置log输出信息
	- Set WebSocket check origin function
	- 开始监听WebSocket地址":3250"

#### 路由压缩
文章介绍：https://github.com/lonng/nano/blob/master/docs/route_compression_zh_CN.md
使用protobuf减少开销
```go
nano.SetSerializer(protobuf.NewSerializer())
```
#### 通信协议格式
文章介绍：https://github.com/lonng/nano/blob/master/docs/communication_protocol_zh_CN.md
#### 服务器API
https://godoc.org/github.com/lonnng/nano