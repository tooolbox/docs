---
title: API
keywords: api
tags: [api]
sidebar: home_sidebar_cn
lang: cn
summary: Micro的api就是api网关
permalink: /cn/api.html

---

API参考了[API网关模式](http://microservices.io/patterns/apigateway.html)为服务提供了一个单一的公共入口。通过服务发现，Micro API以http方式，将请求动态路由到具体的后台服务接口。我们以下简称**API**。

<p align="center">
  <img src="../images/api.png" />
</p>

## 概览

**API**基于[go-micro](https://github.com/micro/go-micro)开发，所以它天然具备服务发现、负载均衡、编码及RPC通信的能力，故而，**API**也是go-micro体系中的一个微服务，所以它自身也是可插拔的。

有兴趣的同学可以参考[go-plugins](https://github.com/micro/go-plugins)，以了解对micro对gRPC、kubernetes、etcd、nats及rabbitmq等通用工具或组件的支持。

另外，**API**也使用了[go-api](https://github.com/micro/go-api)，这样，它的接口handler处理器也是可以配置的。

## 安装

```shell
go get -u github.com/micro/micro
```

## 运行

```shell
# 默认的端口是8080
micro api
```

## 使用ACME协议

ACME（ Automatic Certificate Management Environment）是由**Let's Encrypt**制定的安全协议。

```
MICRO_ENABLE_ACME=true micro api
```

可以选择是否配置白名单

```
MICRO_ENABLE_ACME=true \
MICRO_ACME_HOSTS=example.com,api.example.com \
micro api
```

## 设置TLS证书

API服务支持TLS证书

```shell
MICRO_ENABLE_TLS=true \
MICRO_TLS_CERT_FILE=/path/to/cert \
MICRO_TLS_KEY_FILE=/path/to/key \
micro api
```

## 设置命名空间

API使用带分隔符的命名空间来在逻辑上区分后台服务及公开的服务。命名空间及http请求路径会用于解析服务名与方法，比如`GET /foo HTTP/1.1`会被路由到`go.micro.api.foo`服务上。 

API默认的命名空间是`go.micro.api`，当然，也可以修改：

```shell
MICRO_NAMESPACE=com.example.api micro api
```

## 示例

我们演示一个3层的服务架构：

- `micro api`: (localhost:8080) - http访问入口
- `api service`: (go.micro.api.greeter) - 对外暴露的API服务
- `backend service`: (go.micro.srv.greeter) - 内网的后台服务

完整示例可以参考：[examples/greeter](https://github.com/micro/examples/tree/master/greeter)

### 运行示例

```shell
# 下载示例
git clone https://github.com/micro/examples

# 运行服务
go run examples/greeter/srv/main.go

# 运行api
go run examples/greeter/api/api.go

# 启动micro api
micro api
```

### 查询

向micro api发起http请求

```shell
curl "http://localhost:8080/greeter/say/hello?name=John"
```

HTTP请求的路径`/greeter/say/hello`会被路由到服务`go.micro.api.greeter`的方法`Say.Hello`上。

绕开api服务并且直接通过rpc调用：

```shell
curl -d 'service=go.micro.srv.greeter' \
     -d 'method=Say.Hello' \
     -d 'request={"name": "John"}' \
     http://localhost:8080/rpc
```
使用JSON的方式执行同一请求：

```shell
curl -H 'Content-Type: application/json' \
     -d '{"service": "go.micro.srv.greeter", "method": "Say.Hello", "request": {"name": "John"}}' \
     http://localhost:8080/rpc
```

## API

micro api提供下面类型的http api接口

```
- /[service]/[method]	# HTTP路径式的会被动态地定位到服务上
- /rpc			# 显式使用后台服务与方法名直接调用
```

请看下面的例子

## Handlers

Handler负责持有并管理HTTP请求路由。

默认的handler使用从注册中心获取的端口元数据来决定指向服务的路由，如果路由不匹配，就会回退到使用"rpc" hander。在注册时，可以通过[go-api](https://github.com/micro/go-api)来配置路由。

API有如下方法可以配置请求handler：

- [`api`](#api-handler) - 负责把内部的RPC服务对外暴露成http接口，它接收并处理http请求，根据URL转成内部RPC请求，并把RPC服务的响应结果返回客户端。
- [`rpc`](#rpc-handler) - 处理json及protobuf格式的POST请求，并转向RPC。
- [`proxy`](#proxy-handler) - 反向代理。
- [`event`](#event-handler) - 处理任意的http请求并向消息总线分发消息。
- [`web`](#web-handler) - http反向代理，支持websocket。

通过[`/rpc`](#rpc-endpoint)入口可以绕开handler处理器。

目前版本（V1）无法支持多个handler并存运行，也即同时只能使用一个handler。

### API Handler

API处理器接收任何的HTTP请求，并且向前转发指定格式的RPC请求。

- Content-Type: 支持任何类型
- Body: 支持任何格式
- Forward Format: 转发格式，[api.Request](https://github.com/micro/go-micro/blob/master/api/proto/api.proto#L11)/[api.Response](https://github.com/micro/go-micro/blob/master/api/proto/api.proto#L21)
- Path: 请求路径，`/[service]/[method]`
- Resolver: 请求解析器，路径会被解析成服务与方法
- Configure: 配置，在启动时指定`--handler=api`或在启动命令前指定环境变量`MICRO_API_HANDLER=api`

### RPC Handler

RPC处理器接收json或protobuf格式的HTTP POST请求，然后向前转成RPC请求。

- Content-Type: `application/json` or `application/protobuf`
- Body: JSON 或者 Protobuf
- Forward Format: **json-rpc**或者**proto-rpc**，与`Content-Type`有关
- Path: `/[service]/[method]`
- Resolver: 请求解析器，路径会被解析成服务与方法
- Configure: 配置，在启动时指定`--handler=rpc`或在启动命令前指定环境变量`MICRO_API_HANDLER=rpc`
- 如果没有设置时，RPC Handler就是**默认**的handler，

### Proxy Handler

代理Handler其实是内置在服务发现中的反向代理服务。

- Content-Type: 支持任何类型
- Body: 支持任何格式
- Forward Format: HTTP反向代理
- Path: `/[service]`
- Resolver: 请求解析器，路径会被解析成服务名
- Configure: 配置，在启动时指定`--handler=proxy`或在启动命令前指定环境变量`MICRO_API_HANDLER=proxy`
- REST风格的服务可以通过API代理，就像常见微服务一样对外提供相应的服务。

### Event Handler

事件处理器使用go-micro的broker代理接收http请求并把请求作为消息传到消息总线上。

- Content-Type: 支持任何类型
- Body: 支持任何格式
- Forward Format: 请求格式得是 [go-api/proto.Event](https://github.com/micro/go-api/blob/master/proto/api.proto#L28L39) 
- Path: 请求路径，`/[topic]/[event]`
- Resolver: 请求解析器，路径会被解析成topic（主题，相当于事件分类）与事件（event）名。
- Configure: 配置，在启动时指定`--handler=event`或在启动命令前指定环境变量`MICRO_API_HANDLER=event`

### Web Handler

Web处理器，它是内置在服务发现中的HTTP反向代理服务，支持web socket。

- Content-Type: 支持任何类型
- Body: 支持任何格式
- Forward Format: HTTP反向代理，包括web socket
- Path: `/[service]`
- Resolver: 请求解析器，路径会被解析成服务名
- Configure:  配置，在启动时指定`--handler=web`或在启动命令前指定环境变量`MICRO_API_HANDLER=web`

### RPC endpoint

**/rpc**端点允许绕过主handler，然后与任何服务直接会话。

- 请求参数
  * `service` - 指定服务名
  * `method` - 指定方法名
  * `request` - 请求body体
  * `address` - 可选，指定特定的目标主机地址

示例：

```
curl -d 'service=go.micro.srv.greeter' \
     -d 'method=Say.Hello' \
     -d 'request={"name": "Bob"}' \
     http://localhost:8080/rpc
```

更多信息查看可运行的示例：[github.com/micro/examples/api](https://github.com/micro/examples/tree/master/api)

## Resolver

解析器，Micro使用命名空间与HTTP请求路径来动态路由到具体的服务。

API命名的空间是`go.micro.api`。可以通过指令`--namespace`或者环境变量`MICRO_NAMESPACE=`设置命名空间。

下面说一下解析器是如何使用的：

### RPC Resolver

RPC解析器示例中的RPC服务有名称与方法，分别是`go.micro.api.greeter`，`Greeter.Hello`。

URL会被解析成以下几部分：

路径	|	服务	|	方法
----	|	----	|	----
/foo/bar	|	go.micro.api.foo	|	Foo.Bar
/foo/bar/baz	|	go.micro.api.foo	|	Bar.Baz
/foo/bar/baz/cat	|	go.micro.api.foo.bar	|	Baz.Cat

带版本号的API URL也可以很容易定位到具体的服务：

Path	|	Service	|	Method
----	|	----	|	----
/foo/bar	|	go.micro.api.foo	|	Foo.Bar
/v1/foo/bar	|	go.micro.api.v1.foo	|	Foo.Bar
/v1/foo/bar/baz	|	go.micro.api.v1.foo	|	Bar.Baz
/v2/foo/bar	|	go.micro.api.v2.foo	|	Foo.Bar
/v2/foo/bar/baz	|	go.micro.api.v2.foo	|	Bar.Baz

### Proxy Resolver

代理解析器只处理服务名，所以处理方案和RPC解析器有点不太一样。

URL会被解析成以下几部分：

路径	|	服务	|	方法
---	|	---	|	---
/foo	|	go.micro.api.foo	|	/foo
/foo/bar	|	go.micro.api.foo	|	/foo/bar
/greeter	|	go.micro.api.greeter	|	/greeter
/greeter/:name	|	go.micro.api.greeter	|	/greeter/:name

{% include links.html %}
