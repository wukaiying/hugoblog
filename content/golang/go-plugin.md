---
title: "golang 实现加载动态库"
date: 2020-02-27T14:13:43+08:00
weight: 70  
markup: mmark  
draft: false  
keywords: ["goplugin"]  
description: "golang 实现加载动态库"  
tags: ["go", "plugin"]  
categories: ["golang"]  
author: "wukaiying"  
---

golang开发中需要用到插件动态加载技术，需要主程序动态加载一些功能模块，不需要修改主程序。相比于java
,java 可以通过加载class文件方式来实现模块加载，而golang通常都是单个二进制文件。
go-plugin则使用另外一种方式实现插件加载，主进程和插件进程是两个独立的进程，二者通过rpc/grpc的方式进行通信，
从而实现主进程调用插件进程。

其次，golang也原生提供一种插件模块加载机制plugins，下面依次进行介绍。

## go-plugin

项目地址https://github.com/hashicorp/go-plugin

### 特征

* 插件是Go接口的实现：这让插件的编写、使用非常自然。对于插件的作者来说，他只需要实现一个Go接口即可；对于插件的用户来说，他只需要调用一个Go接口即可。go-plugin会处理好本地调用转换为gRPC调用的所有细节
* 跨语言支持：插件可以基于任何主流语言编写，同样可以被任何主流语言消费
* 支持复杂的参数、返回值：go-plugin可以处理接口、io.Reader/Writer等复杂类型
* 双向通信：为了支持复杂参数，宿主进程能够将接口实现发送给插件，插件也能够回调到宿主进程
* 内置日志系统：任何使用log标准库的的插件，都会将日志信息传回宿主机进程。宿主进程会在这些日志前面加上插件二进制文件的路径，并且打印日志
* 协议版本化：支持一个简单的协议版本化，增加版本号后可以基于老版本协议的插件无效化。当接口签名变化时应当增加版本
* 标准输出/错误同步：插件以子进程的方式运行，这些子进程可以自由的使用标准输出/错误，并且打印的内容会被自动同步到宿主进程，宿主进程可以为同步的日志指定一个io.Writer
* TTY Preservation：插件子进程可以链接到宿主进程的stdin文件描述符，以便要求TTY的软件能正常工作
* 宿主进程升级：宿主进程升级的时候，插件子进程可以继续允许，并在升级后自动关联到新的宿主进程
* 加密通信：gRPC信道可以加密
* 完整性校验：支持对插件的二进制文件进行Checksum
* 插件崩溃了，不会导致宿主进程崩溃
* 容易安装：只需要将插件放到某个宿主进程能够访问的目录即可

### 用法
这里我们通过一个实例来体验一下go-plugin插件开发。我们需要完成的功能是，主程序暴露了两个方法，插件模块去实现这两个方法，然后主程序加载插件完成功能实现。
最终达成的效果是通过Put方法设置一个key/value对，通过get方法获取key对应的值。
```
Put(key string, value []byte) error
Get(key string) ([]byte, error) ([]byte, error)
```
#### 新建项目
这里我先将完整项目目录结构列出来
```tree
.
├── Makefile
├── cmd
│   ├── cmd
│   ├── kv_wky
│   └── main.go
├── pkg
│   ├── plugins
│   │   └── proto
│   │       ├── plugin.pb.go
│   │       └── plugin.proto
│   └── shared
│       ├── grpc.go
│       └── plugin.go
└── pluginclient
    ├── main.go
    └── pluginclient
```

#### 定义proto
新建pkg/plugins/proto/plugin.proto文件，并生成plugin.pb.go
```proto
syntax = "proto3";
package proto;
message GetRequest {
    string key = 1;
}
message GetResponse {
    bytes value = 1;
}
message PutRequest {
    string key = 1;
    bytes value = 2;
}
message Empty {}
service KV {
    rpc Get(GetRequest) returns (GetResponse);
    rpc Put(PutRequest) returns (Empty);
}
```

#### 定义对外暴露的插件接口
新建pkg/shared/plugin.go
````go
package shared
import (
	"context"
	"goplugin-learn/pkg/plugins/proto"

	"github.com/hashicorp/go-plugin"
	"google.golang.org/grpc"
)
/**
定义plugin和host对接的接口
 */
//plugin和host握手协议
var Handshake = plugin.HandshakeConfig{
	ProtocolVersion: 1,
	MagicCookieKey: "WKY_PLUGIN",
	MagicCookieValue: "wukaiying",
}

// PluginMap is the map of plugins we can dispense.
var PluginMap = map[string]plugin.Plugin{
	"kv_grpc": &KVGRPCPlugin{},
}

// KV is the interface that we're exposing as a plugin.
type KV interface {
	Put(key string, value []byte) error
	Get(key string) ([]byte, error)
}
// This is the implementation of plugin.GRPCPlugin so we can serve/consume this.
type KVGRPCPlugin struct {
	// GRPCPlugin must still implement the Plugin interface
	plugin.Plugin
	// Concrete implementation, written in Go. This is only used for plugins
	// that are written in Go.
	Impl KV
}
func (p *KVGRPCPlugin) GRPCServer(broker *plugin.GRPCBroker, s *grpc.Server) error {
	proto.RegisterKVServer(s, &GRPCServer{Impl: p.Impl})
	return nil
}
func (p *KVGRPCPlugin) GRPCClient(ctx context.Context, broker *plugin.GRPCBroker, c *grpc.ClientConn) (interface{}, error) {
	return &GRPCClient{client: proto.NewKVClient(c)}, nil
}
````

#### proto接口的server端实现和client端实现
新建pkg/shared/grpc.go
```go
package shared

import (
	"goplugin-learn/pkg/plugins/proto"

	"golang.org/x/net/context"
)
/**
这个文件中定义了gprc 对于Impl kv接口的server实现和client端实现
 */
// GRPCClient is an implementation of KV that talks over RPC.
type GRPCClient struct{ client proto.KVClient }
func (m *GRPCClient) Put(key string, value []byte) error {
	_, err := m.client.Put(context.Background(), &proto.PutRequest{
		Key:   key,
		Value: value,
	})
	return err
}
func (m *GRPCClient) Get(key string) ([]byte, error) {
	resp, err := m.client.Get(context.Background(), &proto.GetRequest{
		Key: key,
	})
	if err != nil {
		return nil, err
	}
	return resp.Value, nil
}
// Here is the gRPC server that GRPCClient talks to.
type GRPCServer struct {
	// This is the real implementation
	Impl KV
}
func (m *GRPCServer) Put(
	ctx context.Context,
	req *proto.PutRequest) (*proto.Empty, error) {
	return &proto.Empty{}, m.Impl.Put(req.Key, req.Value)
}
func (m *GRPCServer) Get(
	ctx context.Context,
	req *proto.GetRequest) (*proto.GetResponse, error) {
	v, err := m.Impl.Get(req.Key)
	return &proto.GetResponse{Value: v}, err
}
```
#### 主程序main实现
新建cmd/main.go
```go
func main()  {
	log.SetOutput(ioutil.Discard)
	//we are host, launching the plugin process
	client := plugin.NewClient(&plugin.ClientConfig{
		HandshakeConfig: shared.Handshake,
		Plugins: shared.PluginMap,
		Cmd: exec.Command("sh", "-c", "/Users/wukaiying/go/src/goplugin-learn/pluginclient/pluginclient"),
		AllowedProtocols: []plugin.Protocol{
			plugin.ProtocolNetRPC, plugin.ProtocolGRPC,
		},
	})
	defer client.Kill()

	rpcClient, err := client.Client()
	if err != nil {
		fmt.Println("Error", err.Error())
		os.Exit(1)
	}

	raw, err := rpcClient.Dispense("kv_grpc")
	if err != nil {
		fmt.Println("Error", err.Error())
		os.Exit(1)
	}

	kv := raw.(shared.KV)
	os.Args = os.Args[1:]
	//os.Args = []string{"put","wky", "111"}
	switch os.Args[0] {
	case "get":
		result,err := kv.Get(os.Args[1])
		if err != nil {
			fmt.Println("Error", err.Error())
			os.Exit(1)
		}
		fmt.Println(string(result))
	case "put":
		err := kv.Put(os.Args[1], []byte(os.Args[2]))
		if err != nil {
			fmt.Println("Error", err.Error())
			os.Exit(1)
		}
	default:
		fmt.Printf("Please only use 'get' or 'put', given: %q", os.Args[0])
		os.Exit(1)
	}
	os.Exit(0)
}
```
其中NewClient描述了和plugin端握手协议信息，加载哪个plguin，plugin二进制文件的位置，支持的协议等信息。
```go
client := plugin.NewClient(&plugin.ClientConfig{
		HandshakeConfig: shared.Handshake,
		Plugins: shared.PluginMap,
		Cmd: exec.Command("sh", "-c", "/Users/wukaiying/go/src/goplugin-learn/pluginclient/pluginclient"),
		AllowedProtocols: []plugin.Protocol{
			plugin.ProtocolNetRPC, plugin.ProtocolGRPC,
		},
	})
```
最后完成Get,Put操作调用完成对plugin模块中方法的调用。

#### plugin模块端代码实现
以上都是主程序端代码实现，而plugin模块则需单独开发，plugin模块的开发需要调用主程序端的一些文件。
你可以将plugin模块和主程序放在同一个项目，也可单独新建项目开发，但是你需要在新项目将主程序作为包引入。
本文示例将plugin模块和主程序放在同一个项目中，方便开发。
主程序根目录下新建pluginclient/main.go
```go
package main

import (
	"fmt"
	"goplugin-learn/pkg/shared"
	"io/ioutil"

	"github.com/hashicorp/go-plugin"
)

type KV struct {}

func (KV) Put(key string, value []byte) error {
	value = []byte(fmt.Sprintf("%s\n\nWritten from plugin-go-grpc", string(value)))
	return ioutil.WriteFile("kv_"+key, value, 0644)
}

func (KV) Get(key string) ([]byte, error) {
	return ioutil.ReadFile("kv_"+key)
}

func main()  {
	plugin.Serve(&plugin.ServeConfig{
		HandshakeConfig: shared.Handshake,			//需要引入host上面定义好的一些变量
		Plugins: map[string]plugin.Plugin{
			"kv": &shared.KVGRPCPlugin{Impl: &KV{}},
		},
		GRPCServer: plugin.DefaultGRPCServer,
	})
}
```
代码很简单，你需要具体实现Put和Get两个方法，同时编写与宿主机通信的协议信息。

#### 验证

完成编写后，先对plugin端进行编译生成二进制文件。然后将生成的二进制文件路径添加到主程序的ClientConfig-Cmd参数中(见上文)，再编译主程序，这样一个完整的插件开发就完成了。

完整项目地址：https://github.com/wukaiying/goplugin-learn

## go原生plugins

Go 1.8版本开始提供了一个创建共享库的新工具,称为Plugins.

### 生命周期

plugin编译后的so文件是在主项目中被调用的，他和主项目mian的执行顺序如下：
* main.go的init函数先得到执行
* 执行mian.go的 main函数
* 执行plugin.open(“xxx.so”)打开plugin
* 执行plugin文件中的Init函数
* 最后执行plugin中的被调用的函数或者变量

### 应用场景

* 使用plugin完成可插拔式的模块替换和加载，类似于velero中用户可以替换对象存储plugin模块，更换成自己的对象存储
* 后门程序
* 针对不同的语言环境或者场景加载不同的模块

### 用法

#### 示例一
新建plugin.go
```go
package main

import "fmt"

/**
方法和变量的名字都是要首字母大写，小写会导致调用失败
 */
func Hello()  {
   fmt.Println("hello from plugin.go")
}
var Name = "name from plugin.go"
```
编译plugin.go

```
go build --buildmode=plugin -o plugin.so plugin.go
```

新建use_plugin.go
```go
package main

import (
   "fmt"
   "os"
   "plugin"
)

func main()  {
   p, err := plugin.Open("./plugin.so")
   if err != nil {
      fmt.Println(err)
      os.Exit(1)
   }

   //获取plugin.go中的Hello方法
   symbol,err := p.Lookup("Hello")
   if err != nil {
      fmt.Println(err)
      os.Exit(1)
   }

   //判断symbol类型是不是func()类型
   hello, ok := symbol.(func())
   if !ok {
      fmt.Println(err)
      os.Exit(1)
   }
    //执行plugin中的方法
   hello()

   //获取plugin.go中的变量的值
   //symbol, err = p.Lookup("Name")
   //if err != nil {
   // fmt.Println(err)
   // os.Exit(1)
   //}
   //
   //name, ok := symbol.(string)
   //if !ok{
   // fmt.Println(err)
   // os.Exit(1)
   //}
   //fmt.Println(name)
}
```
编译main.go

```
go build use_plugin.go
```

#### 示例二
新建plugin.go

```go
package main

import (
   "log"
   "os/exec"
   "time"
)

func init() {
   log.Println("plugin init function called")
}

type BadNastyDoctor string

func (g BadNastyDoctor) HealthCheck() error {
   bs, err := exec.Command("bash", "-c", "echo 'test'").CombinedOutput()
   if err != nil {
      return err
   }
   log.Println("now is ",g)
   log.Println("shell has exected ->>>>>", string(bs))
   return nil
}

//相当于new了一个BadNastyDoctor对象
var Doctor = BadNastyDoctor(time.Now().Format(time.RFC3339))

//可以这样访问Doctor的方法
//func test()  {
// Doctor.HealthCheck()
//}

//使用 go build -buildmode=plugin -o=plugin.so plugin.go 来编译该文件
```

新建use_plugin.go

```go
package main

import (
   "fmt"
   "log"
   "os"
   "plugin"
)

type GoodDoctor interface {
   HealthCheck() error
}

func init() {
   log.Println("main int function called")
}

func main() {
   log.Println("main function called")

   //1.open so file load the symbos
   plugin, err := plugin.Open("./plugin.so")
   if err != nil {
      fmt.Println(err)
      os.Exit(1)
   }
   log.Println("plugin opened")

   //2.look up a symbol (an expected function or vairable)
   //这个测试用例中，我们来加载plugin中的Doctor对象
   doc, err := plugin.Lookup("Doctor")
   if err != nil {
      fmt.Println(err)
      os.Exit(1)
   }
   log.Print("read plugin variable Doctor")

   //3.判断从plugin中读到的对象，是不是GoodDoctor类型，plugin 中Doctor对象实现了
   //HealthCheck方法，所以应该是同一个类型
   doctor, ok := doc.(GoodDoctor)
   if !ok {
      fmt.Println("unexpected type from module symbol")
      os.Exit(1)
   }

   //4.使用plugin中的方法
   if err := doctor.HealthCheck(); err != nil {
      log.Println("use plugin doctor failed,", err)
   }
}

/**
由此我们可以看到golang plugin的用法，主程序如果想暴露plugin，就需要暴露一个接口，
本实例中接口为
type GoodDoctor interface {
   HealthCheck() error
}
在plugin go文件中，我们可以定义一个结构体，来实现HealthCheck()这个接口。

这样我们就可以使用plugin组件来读取plugin go文件中的Doctor对象和使用Doctor对象中的
HealthCheck()方法了
 */
```