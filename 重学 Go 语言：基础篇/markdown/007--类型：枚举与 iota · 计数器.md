### 枚举是什么？

枚举其实是固定且有限的类别，比如春夏秋冬，亦或是 KB/MB/GB/TB 等。

  * 没有明确意义上的 enum 定义；
  * 借助 iota 实现常量组自增值；
  * 可使用多个 iota，各自单独计数；
  * 中断须显式恢复。

枚举是非常常见的类型，通常情况下指的是一种一连串或者连续性的定义，它的总数是固定的，比如星期、月份、容量、颜色。它是有一定的规律并且可以用一连串顺序数字代替。

枚举在其他语言里用的比较多，但是在 Go 语言里没有明确意义上的枚举定义。iota 实际上是常量组里面实现自增的操作，严格来说和枚举没多大关系。

我们用枚举其实明确定义一个类型，iota 只是编译器的一些类似于宏函数之类的东西。

### iota 的具体含义

    
    
    const (
        _ = 1 << (10 * iota)
        KB
        MB
        GB
    )
    
    const (
        _, _ = iota, iota * 10 // 0, 0 * 10
        a, b                   // 1, 1 * 10
        c, d                   // 2, 2 * 10
    )
    
    func main() {
        println(KB, MB, GB)
    }
    

iota 是编译器，为我们产生连续性数字。其实质是一个计数器，它从零开始计数，每行添加一。它是给编译器看的占位符，告诉编译器在一组里递增数字，每一组
iota 会重新进行计算。iota 可以作为表达式里面其中的操作数。

上面例子中定义两组常量组，在常量组按行来计数，在常量组里在下面没有定义右值的情况下，就是把上面表达式复制下来，例如常量 KB 是把表达式 `1 << (10
* iota)` 复制下来。

如果是有多组，也可以定义多个。上面例子第二组中，iota 在第一行是 0，第二行是 1，这种语法糖非常别扭。

很显然，这样一个枚举组是很好维护的，它代表着有规律的东西，枚举值是让编译器替我们自己生成结果的一种常量。

### 为什么对枚举定义类型

在 Go 语言里面枚举实质上是常量，其它语言枚举是 enum 特定类型，这就会造成一定的麻烦。所以我们习惯于 **对枚举定义类型**
，虽然不能阻止，但便于阅读。

    
    
    type color byte //自定义类型
    
    const (
        red color = iota //指定常量类型
        yellow
        blue
    )
    
    // test 函数参数是byte类型
    func test(c byte) {
        println(c)
    }
    
    // printColor 函数参数是color类型
    func printColor(c color) {
        println(c)
    }
    
    func main() {
        printColor(yellow)
        printColor(45) // 45并未超出color类型取值范围
        var c byte = 100
        test(c)
        //错误：cannot use c (type byte) as type color in argument to printColor
        printColor(c)
        const cc byte = 100
        //错误：cannot use cc (type byte) as type color in argument to printColor
        printColor(cc)
    }
    

上面例子定义颜色值 red、yellow、blue。函数 test 接收颜色，它的类型默认是 byte，iota
就相当于整型计数器，所有常量的类型全部是整型。如果想接收参数，就是 int 类型，但是调用方传任何整型都没有问题。定义和颜色类型一点关系都没有。

我们通常的做法是对枚举类型定义一个自定义类型指定基础类型 `type color byte`，定义类型 `red color =
iota`。不定义类型默认的情况下是整数，定义类型的好处是在参数里面只接收指定的类型。我们期望的是枚举类型只有三个值，使用的时候就需要 switch
判断是不是这三个值，因为枚举类型是编程当中非常常见的一种范例。

定义函数 `func printColor(c color)`，这样一来我们知道输入参数是 color 类型。

  * printColor(yellow)：yellow 是 color 类型肯定合法的。
  * printColor(45)：45 是常量，常量是魔法数字字面量，它会隐式转换。45 并未超出 color 类型取值范围是合法的。

这里需要注意的是 byte 类型的取值范围是从 0 到 255，三个枚举值只是 0 到 255 当中的三个并不表示 byte 类型只有这三个值。

我们是先定义 color，然后为 color 定义常量值，就相当于先有 byte 类型，然后为 byte 类型定义几个常量，那么 byte
类型的值显然不是常量定义的几个，byte 类型的取值范围和后面定义没有关系。同样地不能表示说 color 取值就是定义的三个。

  * printColor(c)：c 是 byte 类型变量，因为 printColor 参数是 color 类型，变量会严格判断认为是两种类型，所以不合法。
  * printColor(cc)：同样是常量 cc，我们定义常量给常量值确切类型的时候，隐式地告诉编译器调用时候需要做类型检查。`const cc = 100` 定义魔法数字 100 并指定符号名，别的地方引用符号名直接展开，是常量符号定义，但是 `const cc int = 100` 除了符号定义以外还指定类型。

