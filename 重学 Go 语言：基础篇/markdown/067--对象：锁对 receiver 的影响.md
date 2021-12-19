### receiver 设定

  * 要修改数据状态，使用 *T。
  * 无需修改状态的小对象或固定值，建议使用T。
  * 大对象使用 *T，避免过高的参数复制成本。
  * 引用类型、字符串、函数等指针包装对象，直接使用 T。
  * 包含锁（mutex）的复合结构，使用 *T，避免锁被复制后失效。
  * 无法确定时使用 *T，以维护唯一实例。

到底用 X 还是用 *X，什么情况用值，什么情况用指针？它们都属于 X 类型的方法，怎么选择有一些简单的规则？

比如修改 X
状态用指针，因为第一个参数是指针才能修改，传进来是拷贝修改不了。第二就是不打算修改而且是个小对象，或者值是固定的建议用拷贝，因为要尽可能地避免去修改。这样在多个线程调用的时候不用进行同步，如果两个线程同时引用，只用指针引用同一个数据要同步处理。如果对象很大的情况下，参数复制成本会很高。

另外引用类型字符串、切片、通道就是包装指针，没有必要再用指针方式。如果复合结构有锁的情况下，需要使用指针，因为复制复合结构变成两个，那同步机制可能失效。方法是用来修改或者维护或者展示原始数据的。

### 锁（mutex）对 receiver 的影响

假设定义数据结构，数据结构内部有个匿名字段用于加锁的，多进程访问需要小心，value 方法 Data 是个复制品，复制品的 Mutex 和原来实例的
Mutex 不是同一个，相当于锁都复制了一把，大家共享同一把锁的时候那把锁才有用，如果一个人手上发把锁就没有用，所以 value
方法锁没有用，因为复制数据的时候锁都复制了一份。

    
    
    type Data struct {
        sync.Mutex //不同锁
        //*sync.Mutex //同一把锁 一级指针
        x int
    }
    
    func (d Data) value() {
        // copy_d.Mutex
        fmt.Printf("v: %p\n", &d.Mutex)
        //fmt.Printf("v: %p\n", d.Mutex)
        d.Lock()
        d.x++
        d.Unlock()
    }
    
    func (d *Data) pointer() {
        fmt.Printf("p: %p\n", &d.Mutex)
        d.Lock()
        d.x++
        d.Unlock()
    }
    
    func main() {
        var d Data
        // var d Data = Data{
        //     Mutex : new(sync.Mutex),
        // }
        fmt.Printf("o: %p\n", &d.Mutex) //输出原始锁的地址
        //fmt.Printf("o: %p\n", d.Mutex)
    
        d.pointer()
        d.value() //地址不一样
    }
    

传值时候需要小心，到底哪些东西复制过来了，是复制指针还是完整复制。如果改成 `*sync.Mutex` 就一样了。

### 方法调用方式汇编调试

方法调用时候会隐性地传递当前实例的指针，不同的语言不同的做法，和普通函数调用有什么区别？因为我们调用方法时候并没有传递
receiver、self、this，在汇编层面究竟怎么传递的？

    
    
    type N int
    
    func (n *N) Inc() {
        *n++
    }
    
    func main() {
        var n N = 0x100
        n.Inc()
        println(n)
    }
    

上面简单的代码，创建了一个类型 N，给这个类型 N 定义 Inc 方法，在 main 函数中创建一个实例 n 调用了 Inc 方法。

编译并调试代码：

    
    
    $ go build -gcflags "-N -l" -o test test.go #编译
    $ gdb test
    $ l
    $ l
    $ b 11
    $ b 6
    $ r
    $ l
    $ p/x &n #$1 = 0xc42003bf68 拿到 n 的信息 f68
    $ set disassembly-flavor intel
    $ disass
    
    
    
    jbe    0x450a4b <main.main+91>
    sub    rsp,0x18 #main 函数分配栈桢空间(三个 8)
    mov    QWORD PTR [rsp+0x10],rbp
    lea    rbp,[rsp+0x10]
    mov    QWORD PTR [rsp+0x8],0x100 #把 n 存入 rsp+0x8，本地变量
    lea    rax,[rsp+0x8] #把 n 的地址放到 rax 中
    mov    QWORD PTR [rsp],rax #rsp 空间就是存放 n 的地址，其实就是编译器隐式地把 this 参数放在第一个参数位置
    call   0x4509b0 <main.(*N).Inc>
    
    
    
    $ c #执行到 inc 方法内部
    $ set disassembly-flavor intel #设置 intel 样式
    $ disass
    
    
    
    sub    rsp,0x10 #inc 方法分配栈桢空间
    mov    QWORD PTR [rsp+0x8],rbp
    lea    rbp,[rsp+0x8]
    mov    rax,QWORD PTR [rsp+0x18] #把 N 的指针放到 rax 中
    test   BYTE PTR [rax],al
    mov    rax,QWORD PTR [rax]
    mov    QWORD PTR [rsp],rax
    mov    rcx,QWORD PTR [rsp+0x18]
    test   BYTE PTR [rcx],al
    inc    rax #调用方法 n++
    mov    QWORD PTR [rcx],rax
    mov    rbp,QWORD PTR [rsp+0x8]
    add    rsp,0x10
    ret
    

可以看到调用 n.Inc() 的时候，虽然没有在参数里面传递 receiver、self、this
引用，编译器实际上替我们完成了这样的操作，编译器会隐式地帮我们传递这样的参数。这就是在书上经常看到的，调用一个方法的时候，编译器会隐式地传递对象实例的引用。

再看一个例子，增加了一个参数：

    
    
    type N int
    
    func (n *N) Inc() {
        *n++
    }
    
    func (n *N) Add(x int) {
        *n += N(x)
    }
    
    func main() {
        var n N = 0x100
        n.Inc()
        n.Add(0x200)
        println(n)
    }
    

增加一个参数，这时候编译器理论上需要传 2 个参数。

编译并调试代码：

    
    
    $ go build -gcflags "-N -l" -o test test1.go #编译
    $ gdb test
    $ l
    $ l
    $ b 16
    $ r
    $ set disassembly-flavor intel
    $ disass
    
    
    
    jbe    0x450aa2 <main.main+114>
    sub    rsp,0x20  #main 函数分配栈桢空间(4 个 8)
    mov    QWORD PTR [rsp+0x18],rbp
    lea    rbp,[rsp+0x18]
    mov    QWORD PTR [rsp+0x10],0x100 #把 n 存入 rsp+0x10
    lea    rax,[rsp+0x10]
    mov    QWORD PTR [rsp],rax
    call   0x4509b0 <main.(*N).Inc>
    lea    rax,[rsp+0x10] #把n的地址放到 rax 中
    mov    QWORD PTR [rsp],rax #rsp 空间就是存放 n 的地址，其实就是编译器隐式地把 this 参数放在这个位置
    mov    QWORD PTR [rsp+0x8],0x200 #把 200 存入 rsp+0x8
    call   0x4509f0 <main.(*N).Add>
    

很显然，当我们调用 n.Add(0x200)
的时候，实际上是有两个参数，第一个参数是对象的引用，可能是个复制品也可能是个指针。接下来会把后面的参数依次往后补上。这就告诉我们隐式传递怎么实现的。

有没有注意到，这种调用跟普通的函数没有什么区别？换句话说， n.Add(0x200) 可以还原成 Add(n,
0x200)。所以在汇编中，没有方法这么一说，方法是出现在语言层面。很显然调用方法时，除了显式参数以外，还需要隐式传递实例的指针。

