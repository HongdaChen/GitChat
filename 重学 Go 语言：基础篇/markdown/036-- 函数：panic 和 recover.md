### panic 和 recover

相比 error，panic、recover 在使用方法上更接近 `try...catch` 结构化异常。

  * 内置函数，而非语句。
  * 参数为接口类型，可以是任何对象。
  * 无论是否执行 recover，延迟调用都会执行。
  * 仅最后一个 panic 会被捕获。
  * 只能在延迟函数中直接使用 recover。

Go 语言中，panic、recover 和 try…catch 结构化异常非常相似。官方文档推荐不要使用 panic 尽量使用 error。panic
相当于抛出异常，在 defer 里面捕获异常。正常情况下，使用 error 方式来实现，当框架中做流程干涉可以考虑 panic，例如数据库打开错误。

#### 确保 defer 得以执行

    
    
    func main() {
        defer println("b")
        defer func() {
            println("a")
        }()
        panic("error")
    }
    

#### 仅最后一个 panic 会被捕获

    
    
    func main() {
        defer func() {
            fmt.Println(recover())
        }()
        defer func() {
            panic("b")
        }()
        panic("a")
    }
    

#### 必须在延迟函数中直接调用 recover

panic 相当于抛出异常，recover 捕获异常。recover 要通过上一级调用堆栈链来获得它的错误信息，panic 把错误挂到栈上面的。

    
    
    func test() {
        fmt.Println(recover())
    }
    func main() {
        // defer func() { // success
        // fmt.Println(recover())
        // }()
        defer func() { test() }() // failure
        //defer fmt.Println(recover())
        //err := recover()
        //defer fmt.Println(err)
    
        // defer test() // success
        // defer recover() // failure
        panic("error") //raise
    }
    

recover 必须在延迟函数中直接调用。`defer recover()` 捕获不到，`defer func() // success` 可以捕获。

延迟函数必须是顶级的。`defer test()` 可以捕获，`defer func() { test() }()` 在第二级里面，捕获不到。

这样实现是因为在 runtime 层面要维持调用堆栈，如果深度非常深的情况下，并不能直接反应当前 panic 错误。所以 panic
的要求第一必须在延迟函数内部调用，第二必须是第一级延迟函数。

#### 使用匿名函数保护代码片段

    
    
    func test(x, y int) {
        z := 0
        func() {
            defer func() {
                if recover() != nil {
                    z = 0
                }
            }()
            z = x / y
        }()
        println("x / y =", z)
    }
    func main() {
        test(5, 0)
    }
    

使用匿名函数保护不会被零除，这和结构化异常的方式差不多。不推荐这种写法，因为我们推荐检查错误条件是前置条件而不是后置条件。defer
在函数结束的时候才执行，这是一种清理手段，对于算法来说，前置检查条件是算法正常执行的基础，和算法本身不能分割的。

### error 和 panic

  * error 表达一个“正常”返回结果。
  * panic 表达不可恢复，导致系统无法正常工作的错误。
  * error 不会中断执行流程。
  * panic 中断执行流程，立即执行延迟调用。
  * 除非必要，否则优先使用 error 模式。

#### 在 defer/recover 内再次 panic 的意义（log、rethrow）

Go 语言里经常会这样，使用 recover
抓到错误，记录日志然后继续抛出去，这段代码专门用来错误记录。另外一层抓到错误对错误进行处理。很显然它可以中止程序它不关心错误干什么。

为什么要分开呢，因为去掉一段对另一段没有影响，它们中间没有依赖关系，相当于可插拔中间件机制。

    
    
    func test() {
        //对错误进行处理
        defer func() {
            if err := recover(); err != nil {
                os.Exit(1)
            }
        }()
        //错误记录
        defer func() {
            err := recover() //try...except
            log(err)
            panic(err)
        }()
        panic("xxx") //raise
    }
    

