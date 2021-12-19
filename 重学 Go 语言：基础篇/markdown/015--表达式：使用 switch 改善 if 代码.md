### switch 执行顺序

switch 语句 case 支持变量，有些语言必须是常量，不用显式地写 break，很多语言不写 break，顺序往后执行，Go 语言默认情况下自动终止。

### 使用 switch 改善 if 代码

在很多语言里面都有 if、switch，除了某些语言有限制，看上去差不多，比如 C 语言要求 case 里面必须是常量。

那么这两种分支有什么样的差异，什么时候该用，什么时候用 if，什么时候用 switch，甚至很多人说哪个性能更好，那么我们怎么确认这件事呢？我们以 Go
语言为例子来对比这个事情。

    
    
    func main() {
        switch x := 5; { // 相当于 switch x := 5; true { ... }
        case x > 5:
            println("a")
        case x > 0 && x <= 5: // 不能写成 case x > 0, x <= 5 多条件是OR关系
            println("b")
        default:
            println("z")
        }
    }
    

  * 两者并没有明显的性能差异。
  * if 适合少数大块逻辑分支，switch 适合简明“表”状条件选择。
  * 作为 if 替代的时候，更倾向于表达式的方式。

使用 if 或者 switch
语句主要看各自承载的内容量有多大，以及哪种方式看上去更简单简洁一些。它们有些时候可以替换的。通常情况下不会有很大的性能差异，甚至有可能生成的汇编代码是一样的。

有几种方式来改善，假如分支超过三个考虑用 switch 来表达。switch
看上去更像是表格状的设计，相当于用户表，用户名称用清单形式列出来，像是用数据表选择一个条件，然后执行某个逻辑，它适合用来列出很多条分支的状况，分离数据块和逻辑。

switch 语句更倾向于表达不是很复杂的逻辑分支，意味着不适合特别复杂分支代码。我们倾向于把 switch 语句当作表达式使用。

    
    
    package main
    
    type Week byte
    
    const (
        _ Week = iota
        Monday
        Tuesday
        Wednesday
        Thursday
        Friday
        Saturday
        Sunday
    )
    
    func example(w Week) {
        switch w {
        case Monday, Wednesday, Friday:
            println("1,3,5")
        case Tuesday, Thursday:
            println("2,4")
        default:
            println("6,7")
        }
    
        switch {
        case w < Thursday:
            println("1,2,3")
        case w < Saturday:
            println("4,5")
        default:
            println("6,7")
        }
    }
    

首先应该把 if
语句所有分支代码重构成函数，我们尝试着把它改成switch，改善了以后，快速地通过条件来筛选出一个对应的函数调用，有点像方法表。条件比较复杂不适合使用
switch。

switch 语句更像数据表驱动，if 语句更像逻辑驱动。if 是先有结论，结论匹配条件。switch
更像先给出条件，然后条件去匹配结论。从设计的角度它们的方向是反的。

switch 语句不太适合处理很复杂的逻辑。

### if 和 switch 反汇编对比

    
    
    package main
    
    func tif(x int) {
        if x == 0 {
            println("a")
        } else if x == 1 {
            println("b")
        } else {
            println("c")
        }
    }
    
    func tsw(x int) {
        switch x {
        case 0:
            println("a")
        case 1:
            println("b")
        default:
            println("c")
        }
    }
    
    func main() {
        tif(1)
        tsw(1)
    }
    
    
    
    $ go build -gcflags "-l" -o test test.go #代码优化模式
    $ go tool objdump -s "main\.tif" test
    $ go tool objdump -s "main\.tsw" test
    

汇编指令是一样的，从编译器优化角度，它们是一回事。换句话说，对于所谓的谁性能好谁性能差的说法是有问题的。

这个测试知道两点：

  * 第一，给任何一个结论必须要带上下文，就是当时的环境。
  * 第二，我们应该用什么样的思路确认一件事，运行期测试本身就不公平。

正常思路是看优化过后代码是否一样，因为它们如果逻辑一致的情况下，没有道理各自有不同的优化策略。首先对比的是优化过后所生成的汇编指令是否相同，如果汇编指令一模一样，那么它们就不会有性能问题。

