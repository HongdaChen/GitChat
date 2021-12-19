由于微服务使用越来越广泛，在 Web 开发中会碰到 RPC 的调用，这里以 gRPC 为例，看看在 Gin 框架中怎样与 gRPC 完成调用。

### gRPC 概述

由谷歌主导开发的 gRPC 框架，使用 HTTP/2 协议并用 Protocol Buffers 作为序列化工具（这和前面官方的 RPC
框架不一样）。提供多语言支持，可以为移动端（iOS/Android）与服务端通讯提供相关的解决方案。

其主要特征：

  * 基于 HTTP/2 提供了连接多路复用、双向流、服务器推送、请求优先级、首部压缩等机制。gRPC 使用了 HTTP2 现有的语义，请求和响应的数据使用 HTTP Body 发送，其他的控制信息则用 Header 表示。
  * gRPC 使用 Protocol Buffers 来定义服务，Protocol Buffers 具有压缩、传输效率高，而且语法简单。
  * gRPC 支持多种语言，并能够基于语言自动生成客户端和服务端代码。gRPC 支持 C、C++、Node.js、Python、Ruby、Objective-C、PHP 和 C# 等语言，还支持 Android 的调用。

可运行如下命令安装 gRPC：

    
    
    go get -u google.golang.org/grpc
    

或者通过访问

> <https://github.com/grpc/grpc-go>

下载 zip 包解压直接安装，但可能需要手工解决依赖包的问题。

在 gRPC 中，客户端应用程序直接调用远程服务器应用程序的方法，就像它是本地对象方法一样，可以轻松地创建分布式应用程序和服务。

gRPC 可定义指定参数和返回类型远程调用的方法，在服务端实现此接口并运行，gRPC
服务端处理客户端调用。在客户端有一个存根它提供与服务端相同的方法。如下图 13-1 所示：

![13%20Gin%20gRPC/015.png](https://images.gitbook.cn/f6c42d60-90df-11ea-867e-e9594b3af56c)

图 13-1 gRPC 方法调用

使用 gRPC 大 致需要三个步骤：

  * 在 .proto 中定义所需要的服务和数据结构
  * 通过 Protocol Buffers 编译器生成特定语言的服务端和客户端代码
  * 使用 Go gRPC API 来开发实现前面所定义的服务

### Gin 框架中调用 gRPC

首先在 gRPC 中，需要使用 Protocol Buffers 来定义服务中的数据类型和方法，如下面定义的 .proto 文件：

    
    
    syntax = "proto3";
    
    option java_multiple_files = true;
    option java_package = "io.grpc.examples.helloworld";
    option java_outer_classname = "HelloWorldProto";
    
    package helloworld;
    
    // 定义服务 greeting
    service Greeter {
      rpc SayHello (HelloRequest) returns (HelloReply) {}
    }
    
    // 请求的消息定义
    message HelloRequest {
      string name = 1;
    }
    
    // 响应的消息定义
    message HelloReply {
      string message = 1;
    }
    

由于在 gRPC 框架中把 Protocol Buffers 作为定义服务的默认协议，所以这里有必要简单解释下 Protocol Buffers 的使用。

`syntax = "proto3"` 表示使用 Protocol Buffers 的 proto3 语法，message HelloRequest
定义了一个名为 HelloRequest 的消息格式，这里的 message 为定义消息格式的关键字，`string name=1` 表示定义一个名为
name 的 string 类型字段，该字段的数字标识符为 1。如果在该消息格式中再增加一个字段，其数字标识符一般递增为 2（或其他值，不能与 1
重复即可）。

在 Protocol Buffers
消息定义中，消息格式中的每个字段都有一个唯一的数字标识符。数字标识符用来在二进制格式中识别各个字段，并且它们一旦使用就不能够再改变。最小的数字标识符从 1
开始，最大到 2^29-1，另外 [19000－19999] 之间的数字标识符不能使用。一般在应用中，每个消息格式的字段数字标识符从 1
开始，根据字段顺序递增。如果 .proto 文件编译输出为 .go 文件，则每个消息格式对应为 Go 语言的结构体。

可在 .proto 文件内使用双斜线（//）进行单行注释。

定义 RPC 服务可以通过关键字 service 来进行，如 service Greeter 则定义了一个 RPC 服务 Greeter。Protocol
Buffer 编译器会根据所选择的语言自动生成相对应的服务接口及存根代码。在该服务中定义了一个 RPC 方法：`SayHello
(HelloRequest) returns (HelloReply)`，输入参数是消息格式 HelloRequest，方法返回消息格式
HelloReply。

使用 Protoc 编译器来编译 .proto 文件并生成对应的 .go 文件（这里使用 Go 语言），运行命令得到编译好的 .go 文件：

    
    
    protoc --go_out=plugins=grpc:. helloworld.proto
    

上述命令中指定 plugins 插件值为 grpc，因为上面的.proto 文件中定义了 RPC 服务，需要设置 plugins 插件值为指定的
gRPC。插件以及指定值为参数 `--go_out=` 的一部分，中间冒号是分隔符，后跟所需要的参数子集。上述命令表示在当前目录，根据定义好的 .proto
文件，编译输出所定义的数据结构和 gRPC 服务的 Go 语言代码文件。

在上面 .proto 文件中，主要定义了 RPC 方法 SayHello()，通过工具生成 .go 文件，下面看看服务端的实现：

[L13/1/greeter_server/main.go](https://github.com/ffhelicopter/learngin)

    
    
    // server 用来实现 helloworld.GreeterServer
    type server struct{}
    
    // SayHello 方法
    func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
        return &pb.HelloReply{Message: "Hello " + in.Name}, nil
    }
    
    func main() {
        lis, err := net.Listen("tcp", ":50051")
        if err != nil {
            log.Fatalf("failed to listen: %v", err)
        }
        s := grpc.NewServer()
        pb.RegisterGreeterServer(s, &server{})
    
        // 在 gRPC 服务上反射注册服务
        reflection.Register(s)
        if err := s.Serve(lis); err != nil {
            log.Fatalf("failed to serve: %v", err)
        }
    }
    

运行服务端程序后，在 50051 端口开启 gRPC 服务，接收客户端的调用，下面是客户端代码：

[L13/1/greeter_client/main.go](https://github.com/ffhelicopter/learngin)

    
    
    func main() {
        // 连接 gRPC 服务端
        conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
        if err != nil {
            log.Fatalf("did not connect: %v", err)
        }
        defer conn.Close()
        client := pb.NewGreeterClient(conn)
    
        // Gin 框架 HTTP 服务设置
        r := gin.Default()
        r.GET("/rest/n/:name", func(c *gin.Context) {
            name := c.Param("name")
    
            // 调用 gRPC 服务的方法并把返回通过 HTTP 响应显示
            req := &pb.HelloRequest{Name: name}
            res, err := client.SayHello(c, req)
            if err != nil {
                c.JSON(http.StatusInternalServerError, gin.H{
                    "error": err.Error(),
                })
                return
            }
    
            c.JSON(http.StatusOK, gin.H{
                "result": fmt.Sprint(res.Message),
            })
        })
    
        // 启动 HTTP 服务
        if err := r.Run(":8052"); err != nil {
            log.Fatalf("could not run server: %v", err)
        }
    }
    

客户端先通过 gRPC 客户端连接 gRPC 服务端，然后启动 Gin 框架的 HTTP 服务。通过 Web 访问的方式，来实现调用 gRPC
方法。也就是说 Gin 框架中是很方便调用 gRPC 方法的。

可以先运行 gRPC 服务端：

    
    
    >go run main.go
    

然后运行客户端：

    
    
    >go run main.go
    

客户端控制台显示：

    
    
    [GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.
    
    [GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
     - using env:   export GIN_MODE=release
     - using code:  gin.SetMode(gin.ReleaseMode)
    
    [GIN-debug] GET    /rest/n/:name             --> main.main.func1 (3 handlers)
    [GIN-debug] Listening and serving HTTP on :8052
    

表明 Gin 框架已经启动，此时通过浏览器访问：<http://localhost:8052/rest/n/999>，页面显示：

    
    
    { "result": "Hello 999"}
    

响应的头信息如下图 13-2：

![13%20Gin%20gRPC/016.png](https://images.gitbook.cn/900b0710-90df-11ea-a24d-ef9510a03f83)

图 13-2 Gin 响应头信息

**本节总结**

  * gRPC 框架特征与原理介绍
  * gRPC 框架下定义服务中的数据类型和方法，服务端与客户端的实现
  * Gin 框架下调用 gRPC 服务端方法

