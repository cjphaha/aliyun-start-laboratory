# Dubbo-go HelloWorld

## 1. 简介

随着微服务架构的流行，许多高性能 rpc 框架应运而生，dubbo是一个高性能的rpc框架，dubbo-go是阿里开源的dubbo框架的go语言版本，可以实现java和go服务之间的互相调用，本教程将介绍dubbo-go的基础用法，以及相关概念。

## 2 学习目标

+ 了解配置文件的使用
+ 掌握使用dubbo-go编写微服务应用
+ 了解微服务架构的相关概念

## 3. 详细内容

+ 快速上手
+ 运行注册中心
+ 代码解读
+ 平滑关闭

## 4. 准备工作

### 4.1 获取代码

将demo获取下来

```shell
git clone https://github.com/dubbogo/dubbo-samples.git
```

### 4.2 安装注册中心

在微服务架构中，注册中心是最核心的基础服务之一，各个服务将自己的网络地址等信息注册到注册中心，注册中心存储这些数据。服务的消费者从注册中心查询服务提供者的地址，并通过该地址调用服务提供者的接口。

不使用注册中心也可以实现微服务，但是带来的问题是如果某一个服务提供者的ip变动，或者实例有增加或销毁，服务调用者需要再手动修改提供者的网络地址。注册中心提供了服务注册与发现、服务检查等功能。服务调用者只需要通过注册中心便可以调用到服务提供者。常见的注册中心有zookeeper、consul、etcd、nacos等，本案例使用zookeeper，推荐使用docker安装，下面演示一下如何使用docker拉取镜像及运行zookeeper。

```shell
# 拉去zookeeper景象
docker pull zookeeper
# 运行
docker run -d -p 2181:2181 --name zookeeper-container --restart always zookeeper
```

### 4.3 目录结构

demo的文件目录如下：

```shell
.
├── app
├── assembly
└── profiles
```

app是代码目录

assembly是在不同的环境中的执行脚本

profiles是配置文件所在的文件夹

### 4.4 设置配置文件路径

配置文件在dubbo-go中非常重要，注册中心的网络地址、传输协议、日志表等信息都需要写在配置文件中，在项目启动的时候会先加载配置文件。因此在启动配置文件的时候，需要指定一下配置文件的位置，目前支持设置环境变量及添加启动参数的方式来指定路径。对于server端来说有两个变量必须需要设置：CONF_PROVIDER_FILE_PATH、APP_LOG_CONF_FILE，分别应该指向服务端配置文件、日志配置文件。

环境变量的使用方法如下：

```shell
$ export CONF_PROVIDER_FILE_PATH="../profiles/dev/server.yml" 
$ export APP_LOG_CONF_FILE="../profiles/dev/log.yml"
```

client也类似，客户端环境变量的使用方法如下

```shell
$ export CONF_CONSUMER_FILE_PATH="../profiles/dev/client.yml"
$ export APP_LOG_CONF_FILE="../profiles/dev/log.yml"
```

> 1.5.6版本之后可以使用命令行的方式来指定配置文件的路径

server端

```shell
$ go run . -proConf ../profiles/dev/server.yml -logConf ../profiles/dev/log.yml
```

client端

```shell
$ go run . -conConf ../profiles/dev/client.yml -logConf ../profiles/dev/log.yml
```

### 4.5 hessian介绍

在微服务架构中，少不了rpc，rpc即远程过程调用，使用rpc框架可以向调用方屏蔽各种复杂性，向服务提供方也屏蔽各类复杂性，使用rpc带来的好处有如下：

+ 调用方感觉就像调用本地函数一样

+ 服务提供方感觉就像实现一个本地函数一样来实现服务

下图可以更生动的描述rpc框架所起到的作用

![image-20210306150132695](http://cdn.cjpa.top/cdnimages/image-20210306150132695.png)

rpc协议并不是一个具体的协议，可以由开发者自行设计，开发者可以自己对rpc进行封装和注册，对于数据的传输格式以及顺序也可以自己定制，对数据格式的处理需要使用到打解包工具，常见的打解包工具有Protobuf，json，hessian2。dubbo使用的是hessian。

作者在之前的项目中使用过grpc框架，grpc进行序列化需要借助proto文件，在使用的时候需要先用protoc生成开发者所需要的语言的文件，然后再进行调用这样做的好处是，以通过共用一个文件来保证各服务的一致性，但是使用起来并不是那么方便。dubbo/hession就比较简单，但是需要严格控制各个服务之间的数据格式

hessian使用非常简单，真正提供了类似于调用本地方法一样调用接口的功能 ，返回的参数不需要二次解析并且足够轻量。

## 5. 启动server端

在按照4.4中的说明设置好server端之后，可以执行

```shell
go run .
```

便可启动server端

### 5.1 server端配置文件讲解

服务端需要的重要配置有三个字段：services、protocols、registries。

profiles/dev/server.yml:

```yml
registries :
	# 注册中心的名字
  "demoZk":
  	# 注册中心协议
    protocol: "zookeeper"
    # 健康检查超时时间
    timeout    : "3s"
    # zookeeper的地址及端口
    address: "127.0.0.1:2181"
services:
	# 要暴露的rpc-service名
  "UserProvider":
    # 可以指定多个registry，使用逗号隔开;不指定默认向所有注册中心注册
    registry: "demoZk"
    # 暴露的协议名
    protocol : "dubbo"
    # 暴露的服务所处的 interface，相当于dubbo.xml中的interface
    interface : "com.ikurento.user.UserProvider"
    # 负载均衡的策略
    loadbalance: "random"
    warmup: "100"
    # 集群失败策略
    cluster: "failover"
    # 调用的方法
    methods:
    - name: "GetUser"
      retries: 1
      loadbalance: "random"
# 暴露的协议名及端口
protocols:
  "dubbo":
    name: "dubbo"
    port: 20000
```

其中，中间服务的协议名需要和 registries 下的 mapkey 对应，暴露的协议名需要和 protocols 下的 mapkey 对应。

上述例子中，使用了 dubbo 作为暴露协议，使用了 zookeeper 作为中间注册协议，并且给定了端口。如果 zk 需要设置用户名和密码，也可以在配置中写好。

### 5.2 user.go解读

user.go定义了rpc-service结构体以及传输的数据结构

```go
func init() {
	config.SetProviderService(new(UserProvider))
	// ------for hessian2------
	hessian.RegisterPOJO(&User{})
}
type User struct {
	Id   string
	Name string
	Age  int32
	Time time.Time
}
type UserProvider struct {
}
func (u *UserProvider) GetUser(ctx context.Context, req []interface{}) (*User, error) {
	gxlog.CInfo("req:%#v", req)
	rsp := User{"A001", "Alex Stocks", 18, time.Now()}
	gxlog.CInfo("rsp:%#v", rsp)
	return &rsp, nil
}
```

ser 为用户自定义的传输结构体，UserProvider 为用户自定义的 rpc_service；包含一个 rpc 函数，GetUser。

在 init 函数中，调用 config 的 SetProviderService 函数，将当前 rpc_service 注册在框架 config 上，hessian则提供了如5.2所描述的二进制编码的功能，注册传输结构体 User。

### 5.3 server.go解读

server.go

```go
func main() {
   hessian.RegisterPOJO(&User{})
   config.Load()
   initSignal()
}
```

main 函数中只进行了两个操作，首先使用 hessian 注册组件将 User 结构体注册，从而可以在接下来使用 getty 打解包。

之后调用 config.Load 函数，该函数位于框架 config/config_loader.go 内，这个函数是整个框架服务的启动点，  config.Load会从环境变量或启动参数中获取consumer的位置，然后从配置文件中把配置读入框架。

最终开启信号监听 initSignal() 优雅地结束一个服务的启动过程。后文会介绍。

## 6. client端

### 6.1 client端配置文件解读

```yml
registries :
  "demoZk":
    protocol: "zookeeper"
    timeout  : "3s"
    address: "127.0.0.1:2181"
    username: ""
    password: ""
references:
  "UserProvider":
    # 可以指定多个registry，使用逗号隔开;不指定默认向所有注册中心注册
    registry: "demoZk"
    # 协议
    protocol : "dubbo"
    # 暴露的服务所处的 interface，相当于dubbo.xml中的interface
    interface : "com.ikurento.user.UserProvider"
    cluster: "failover"
    # 调用的方法
    methods :
    - name: "GetUser"
      retries: 3
```

client端的配置文件和server端大体一致，其中refrences字段和server端不太一样， refrences 的字段就是对当前服务要主调的服务的配置，其中详细说明了调用协议、注册协议、接口 id、调用方法、集群策略等

### 6.2 user.go解读

client端的use.go的代码和server端基本一致

```go
func init() {
	config.SetConsumerService(userProvider)
	hessian.RegisterPOJO(&User{})
}

type User struct {
	Id   string
	Name string
	Age  int32
	Time time.Time
}

type UserProvider struct {
	GetUser func(ctx context.Context, req []interface{}, rsp *User) error
}
```

其中User结构体一定要保持一致，如果不保持一致会导致传输失败，

### 6.3 client.go解读

```go
func main() {
	hessian.RegisterPOJO(&User{})
	config.Load()
	time.Sleep(3e9)
	gxlog.CInfo("\n\n\nstart to test dubbo")
	user := &User{}
	err := userProvider.GetUser(context.TODO(), []interface{}{"A001"}, user)
	if err != nil {
		panic(err)
	}
	gxlog.CInfo("response result: %v\n", user)
	initSignal()
}
```

client作为调用方，锁执行的步骤要稍微复杂一些，

首先也是将使用 hessian 注册组件将 User 结构体注册，然后加载配置。

userProvider中的GetUser方法可以实现对server端远程服务的调用，可以看到，有了hessian提供打解包，dubbo-go服务之间的调用非常方便。

initSignal()是平滑关闭函数，下文会有介绍。

## 7. 平滑关闭

initSignal

```go
func initSignal() {
	signals := make(chan os.Signal, 1)
	// It is not possible to block SIGKILL or syscall.SIGSTOP
	signal.Notify(signals, os.Interrupt, os.Kill, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)
	for {
		sig := <-signals
		logger.Infof("get signal %s", sig.String())
		switch sig {
		case syscall.SIGHUP:
			// reload()
		default:
			time.AfterFunc(time.Duration(survivalTimeout), func() {
				logger.Warnf("app exit now by force...")
				os.Exit(1)
			})
			// The program exits normally or timeout forcibly exits.
			fmt.Println("provider app exit now...")
			return
		}
	}
}
```

在server和client中都是用到了平滑关闭。

平滑关闭的实现原理很简单，第一步是定义一个chanel，然后监听系统中断的信号，如果收到了中断的信号，就把这个信号传递到chanel中。在接收到chanel之前，会一直循环检测，直到chanel中收到了中断参数，然后系统会执行关闭的相关函数。