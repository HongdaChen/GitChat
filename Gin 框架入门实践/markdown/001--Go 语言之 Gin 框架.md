Gin 框架是一个用 Go 语言编写的开源 Web 框架。 因其 API 调用方便，性能优越，已经有越来越多的用户开始使用它。

## 广受欢迎的 Go 语言 Web 框架

根据知名软件开发公司 JetBrains 的调查报告。为更好地了解开发者，JetBrains 于 2019 年初发起了开发人员生态系统调查，调查了约
7000 名开发者。

Go 语言在本次调查中的表现十分值得关注，它被称为“最有前途的编程语言”。因为 Go 在 2017 年的份额只有 8%，现在已达到 18%。此外，多达
(13%) 的开发人员愿意采用或迁移到 Go 语言。

而在“您通常使用哪种（哪些）Go Web 框架？”这项调查中，排名第一的是 Gin 框架，其使用量较去年增长 9%，已达 30%。其次分别是 Echo 和
Beego。

![图 1-1 您通常使用哪种（哪些）Go Web 框架？](https://images.gitbook.cn/15738022561388)

另外，在 GitHub 上该项目的星星数超过 30,000 颗，而 Fork 数量超过 3,500，这在 Go Web
框架中遥遥领先，足以说明用户对其接受程度之高。

当然，在实际的技术选型中，流行并不是让架构师做出选择的最重要条件。最主要的原因应该和 Gin
框架适用于项目的架构需要，另外性能优异程度，社区活跃程度等都会让架构师们不得不做出正确的选择。

既然 Gin 框架如此受欢迎，那么就有必要了解其原理、使用方法以及特点，这样才能知道其受欢迎的原因，同样也让我们在 Web 开发选型时有了判断的依据。

## 安装 Gin 框架

要安装运行 Gin 框架，首先需要确保已经有 Go 语言的运行环境（要求 Go1.8 以上版本）。如果没有则需要安装运行环境，首先需要下载 Go
语言安装包，Go 语言的安装包下载地址为：https://golang.org/dl/ ，
国内可以正常下载地址：https://golang.google.cn/dl/

** 源码编译安装 **

Go 语言是谷歌 2009 发布的第二款开源编程语言。经过十年的版本更迭，目前 Go 语言已经发布 1.13 版本，UNIX/Linux/Mac OS
X，和 FreeBSD 系统下可使用如下源码安装方法：

  1. 下载源码包：
    
        https://golang.google.cn/dl/go1.13.linux-amd64.tar.gz
    

  2. 将下载的源码包解压至 /usr/local 目录：
    
        tar -C /usr/local -xzf go1.13.linux-amd64.tar.gz
    

  3. 将 `/usr/local/go/bin` 目录添加至 PATH 环境变量：
    
        export PATH=$PATH:/usr/local/go/bin
    

  4. 设置 GOPATH，GOROOT 环境变量：

GOPATH 是工作目录，GOROOT 为 Go 的安装目录，这里为 `/usr/local/go/ `

> 注意：MAC 系统下可以使用 `.pkg` 结尾的安装包直接双击来完成安装，安装目录在 `/usr/local/go/` 下。

**Windows 系统下安装 **

在 Windows 系统下可以采用直接安装，下载 `go1.13.windows-amd64.zip` 版本，直接解压到安装目录 D:，然后把 D:PATH
环境变量中。另外，还需要设置 2 个重要环境变量：GOPATH 和 GOROOT。

GOPATH 是项目开发的工作目录，可以有多个，彼此之间用分号隔开，GOROOT 为 Go 的安装目录。

以上三个环境变量设置好后，我们就可以开始正式使用 Go 语言来开发了。

Windows 系统也可以选择 `go1.13.windows-
amd64.msi`，双击运行程序根据提示来操作。安装完成后，可以通过输入命令来检测是否正常安装好。

在 Win 平台下，同时按住 Win+R 两键，快捷打开 CMD（注意：设置环境变量后需要重新打开 CMD），输入 go ，如显示 `go version
go1.13 windows/amd64` 说明 Go 语言运行环境已经安装成功。

Go 语言运行环境成功安装后，就可以开始 Gin 框架的安装了，安装 Gin 框架可以有下面两种方式：

### 使用 go get 命令安装 Gin

  1. 运行 go get 命令下载安装相关包
    
        go get -u github.com/gin-gonic/gin
    

`get -u` 表示下载更新 gin 以及其依赖的相关包。

  2. 在程序中导入相关包
    
        import "github.com/gin-gonic/gin"
    

在 GOPATH 中建立项目目录 web，把 `main.go`
文件保存在其中，源代码如下（[L01/1/main.go](https://github.com/ffhelicopter/learngin)）：

    
    
    // +build go1.8
    
    package main
    import ("net/http""github.com/gin-gonic/gin")
    
    func main() {router := gin.Default()
        router.GET("/", func(c *gin.Context) {c.String(http.StatusOK, "Welcome Gin Server")
        })router.Run()
    }
    

  1. 然后输入命令后回车运行：
    
        go run main.go
    

  2. 接下来打开浏览器，在地址栏输入并访问：locahost:8080，浏览器页面显示：
    
        Welcome Gin Server
    

上面代码很容易就提供了一个 Web 服务，从中或许你可以看出，作为 Web 框架，使用 Gin 来开发 Web 应用将是一件比较轻松的事情。

### 使用 Go modules 安装 Gin

Go 语言在 1.11 版本开始引入 Go modules 进行包依赖管理，从 Go 1.13 开始，模块机制在所有情况下都将会默认启用。所以这里以
Windows 平台下 Go1.13 为例通过 Go modules 来安装 Gin。

为了更好地管理模块，在 Go1.13 中 Go modules 中可以通过设定 GOPROXY 来达到目的，其默认值为
`https://proxy.golang.org`。但这个网址在中国大陆无法访问，云服务提供商七牛云专门为中国 Go 语言开发者提供了一个 Go
模块代理：goproxy.cn。

使用命令：`go env -w GOPROXY=https://goproxy.cn,direct` 设置即可。

由于使用 Go modules，所以一般在 GOPATH 目录外新建立一个项目目录如：web 。在 web 目录中存放上面的 `main.go` 文件。

在 Windows 中打开 CMD 命令行，进入 web 目录后输入命令后回车：

    
    
    go mod init web
    

正常应该出现提示：`go: creating new go.mod: module web`，这说明通过 Go modules 已经在 web
目录成功初始化该模块。

然后输入命令后回车：`` 下图是命令运行时截屏：

![图 1-2 go mod tidy 安装 Gin](https://images.gitbook.cn/15738022561403)

该命令会更新该项目的依赖包并清除未使用的包，在 GOPATH 下的 pkg 目录中可以看到 mod 目录，这个目录中存放 `go mod tidy`
命令执行后下载的依赖包。

从图 1-2 中可以看到，本书安装的是 Gin v1.4.0 版本，本专栏以 v1.4.0 作为标准来讲解 Gin
框架的特性与应用，书中代码也以这个版本作为基础来开发和运行。

另外，对比 v1.4.0 版本与 v1.3.0 版本发现变化还是比较大的，比如在 v1.4.0 版本中已经把 Gin 示例 examples
作为独立项目开源出来，同时 json 目录移动到新的 internal 目录下，且支持更多的函数功能。

然后输入命令后回车：

    
    
    go run main.go
    

接下来打开浏览器，在地址栏输入并访问：`locahost:8080`，浏览器页面显示：

    
    
    Welcome Gin Server
    

至此，Gin 框架已经安装完成了。接下来可以在 Gin
框架下进行应用开发，相信你应该迫不及待地希望开始了。但在正式开始写代码前，我们有必要来了解一些相关的知识，这对我们的 Web 应用开发应该非常有帮助。

在接下来的章节中，所有代码都建议通过 Go modules 模式运行，对此不再做说明，除非有特别情况需要使用 GOPATH 方式，但这会做特别的说明。

** 本节总结 **

  * Gin 框架是当前广受欢迎的 Go 语言 Web 框架；
  * Go 语言运行环境的搭建；
  * Gin 框架的两种安装方式；
  * Go 语言 1.13 版本中 Go modules 设定 GOPROXY。

** 本节问题 **

  1. Go modules 代理模块在 Go1.13 版本中的设置方法与意义？

