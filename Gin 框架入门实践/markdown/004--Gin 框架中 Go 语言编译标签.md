在编写依赖特定平台或者 CPU 架构的时候，Go 语言代码需要有不同的实现，可以通过定义编译标签或者是命名约定来实现不同平台的代码管理，这种方式也就是 Go
语言的条件编译。

## 使用 Go 语言编译标签

首先来了解下 Go 语言中条件编译的一种方式：编译标签。在 Go
语言中，可以在源代码里尽量靠源代码文件顶部的地方用注释的方式添加标注，并且标签的结尾添加一个空行，这通常称之为编译标签（build tag）。

Go 语言在构建一个包的时候会读取这个包里的每个源文件并且分析编译标签，这些标签决定了这个源文件是否参与本次编译。

编译标签的四个特征：

  * 编译标签由空格分隔的编译选项以逻辑”或”的关系组成
  * 而编译选项中由逗号分隔的条件项之间是逻辑 "与" 的关系
  * 编译标签的名字由字母 + 数字表示，标签名字前加 ! 表示否定
  * 一个文件可以有多个 +build 构建标记，它们之间的关系是逻辑 "与" 的关系

正确的编译标签的书写方式是在编译标签最后结尾处添加一个空行，这样标签就不会被当做注释，例如：

    
    
    // +build linux windows,amd64
    
    package main
    

上面的编译标签标示此源文件只能在 linux 或者 windows amd64 位平台下编译。而 linux 为编译选项，windows,amd64
这个编译选项中，amd64 为条件项。多个 +build 构建标记一般分成每行一个 +build 构建标记。

## Gin 框架中的编译标签

在 Gin 框架中，也较多地使用了编译标签，例如 internal/json 目录，这个目录目前只有 json 包。在 json
包中，这个包的两个文件分别都使用了标签 jsoniter。

文件 jsoniter.go：

    
    
    // Copyright 2017 Bo-Yi Wu.  All rights reserved.
    // Use of this source code is governed by a MIT style
    // license that can be found in the LICENSE file.
    
    // +build jsoniter
    
    package json
    
    import "github.com/json-iterator/go"
    
    var (
        json = jsoniter.ConfigCompatibleWithStandardLibrary
        // 通过函数值，Marshal 等函数都由 Gin 下的 json 包导出
        Marshal = json.Marshal
        Unmarshal = json.Unmarshal
        MarshalIndent = json.MarshalIndent
        NewDecoder = json.NewDecoder
        NewEncoder = json.NewEncoder
    )
    

文件 json.go：

    
    
    // Copyright 2017 Bo-Yi Wu.  All rights reserved.
    // Use of this source code is governed by a MIT style
    // license that can be found in the LICENSE file.
    
    // +build !jsoniter
    
    package json
    
    import "encoding/json"
    
    var (
        // 通过函数值，Marshal 等函数都由 Gin 下的 json 包导出
        Marshal = json.Marshal
        Unmarshal = json.Unmarshal
        MarshalIndent = json.MarshalIndent
        NewDecoder = json.NewDecoder
        NewEncoder = json.NewEncoder
    )
    

Gin 默认使用标准库 encoding/json 包，但是支持使用标签 jsoniter 来编译。例如：

    
    
    go build -tags=jsoniter .
    

上述命令表示使用 jsoniter.go 文件来编译生成可执行文件。

通过对比上述的两个文件，就可以发现它们分别对第三方包 [github.com/json-
iterator/go](http://github.com/json-iterator/go) 和标准库 encoding/json 包针对 Gin
框架做了编译上的适配包装，在 Gin 框架编译时，可以通过编译标签自由选择性编译。编译时如果指定 -tags=jsoniter，则会选择
jsoniter.go 进行编译，默认情况下一般都没有指定这个标签，所以使用的是标准库的 json 包。

由于第三方包 jsoniter 性能上要优于标准库的 json 包，而且这个包提供 100% 与标准库 json 包兼容选项，加上主要函数签名一致，所以
Gin 这里特意封装了新的 json 包，通过编译标签，程序编译时很容易根据实际标签来选择具体实现的代码包。

如果开发的应用程序需要大量编解码 JSON 数据，程序员为了提高性能，使用 -tags=jsoniter 编译当然是最佳的选择。但一般情况下如应用对
JSON 数据编解码需求很少或甚至没有，那么使用默认编译，采用标准库的 json 包，这对整体性能的影响也不大，也是程序员很常用的选择。

另外通过文件名后缀来进行条件编译由于在 Gin
框架中很少使用，这里只简单说明一下。这种方案比编译标签更简单，在不同的平台和架构可以根据源文件后缀来确定具体源文件，例如 runtime 中的 defs
_darwin_ amd64.go 这个源文件，很典型在苹果系统的 amd64 架构下编译使用。

总之，这里 Gin 框架通过编译标签这个语言特性，也为程序员们提供了一种在 Go 语言中适配不同包的解决方案。这种巧妙解决方案的思路与风格，也许正是 Gin
优秀的某种基因吧。其实不只是 Gin 框架，还有很多优秀的 Go 语言开源项目中也能看到通过编译标签来解决问题的方案，例如 rpcx 框架等。

**本节总结**

  * Go 语言的编译标签；
  * Gin 框架中使用编译标签来解决不同包的适配。

