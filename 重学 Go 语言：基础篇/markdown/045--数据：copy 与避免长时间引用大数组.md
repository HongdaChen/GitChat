### copy

    
    
    func main() {
        a := [...]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
        s1 := a[6:]
        s2 := a[4:7]
        fmt.Println(copy(s2, s1), a) // 3 [0 1 2 3 6 7 8 7 8 9]
    }
    

  * 在两个 slice 间复制数据，复制长度以段 slice 为准。
  * 允许指向同一底层数组，允许目标区间重叠。

两个切片之间拷贝数据的时候。第一，这两个切片可以指向同一个底层数组，第二，切片复制区间可以重叠。复制的时候，s1 容量是 4，s2 容量是
3，复制长度以短的为准。s1 数据拷贝到 s2 相当于把 678 覆盖 456，底层数组变成
0123678789。复制需要知道就是它允许指向同一个数组，它们区间可以重叠。copy 返回值是复制多少数据。

### copy 区域重叠（overlap）

copy 操作在 C 语言里很常见的，它的特点是区域是可以重叠的。可以把数组 A 片段拷贝到 B 片段，其中有一段区域是重叠的。

copy 函数实际上调用 memmove，说白了就是通过交换的方式去实现。在同一个切片内部搬移数据，而且搬移空间可以重叠。

    
    
    func main() {
        a := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
        copy(a[2:], a[6:8])
        fmt.Println(a)
    }
    

为什么会有 copy 函数，还会涉及性能问题，我们在两个切片中间复制数据，有不同方式，第一种方式是循环的方式拷贝过来，第二种方式用 append
追加，第三种方式是拷贝函数。

    
    
    func testFor(a, b []byte) int {
        for i := 0; i < len(a); i++ {
            b[i] = a[i]
        }
        return len(b)
    }
    
    func testAppend(a, b []byte) int {
        b = append(b, a...)
        return len(b)
    }
    
    func testCopy(a, b []byte) int {
        copy(b, a)
        return len(b)
    }
    
    func main() {
        source := []byte{1, 2, 3, 4}
    
        //复制显然切片长度是相同的，因为是完全复制
        b1 := make([]byte, len(source))
        fmt.Println(testFor(source, b1), b1)
    
        //为了避免发生内存扩张，指定容量
        b2 := make([]byte, 0, len(source))
        fmt.Println(testAppend(source, b2), b2[:len(source)])
    
        //拷贝指定相应长度的内存
        b3 := make([]byte, len(source))
        fmt.Println(testCopy(source, b3), b3)
    }
    

测试用例。为了公平起见复制源有 1M，从 dev/urandom 设备读随机数据，使用 init 函数确保在性能测试之前数据就准备好了。

    
    
    var source []byte
    
    func init() {
        source = make([]byte, 1<<20)
        rand.Read(source) // /dev/urandom
    }
    
    func BenchmarkFor(b *testing.B) {
        x := make([]byte, len(source))
        b.ResetTimer()
    
        for i := 0; i < b.N; i++ {
            _ = testFor(source, x)
        }
    }
    
    func BenchmarkAppend(b *testing.B) {
        // 一次性把内存分配好
        //x := make([]byte, len(source))
        //自己分配内存
        var x []byte
        b.ResetTimer()
    
        for i := 0; i < b.N; i++ {
            _ = testAppend(source, x)
        }
    }
    
    func BenchmarkCopy(b *testing.B) {
        x := make([]byte, len(source))
        b.ResetTimer()
    
        for i := 0; i < b.N; i++ {
            _ = testCopy(source, x)
        }
    }
    

我们发现 for 循环最慢，append 比 for 循环快 2 倍多，copy 方式最快，copy 会受底层库的优化，append
中途发生扩张。在我们准备好内存空间的情况下，append 和 copy 性能是非常接近的。

从这点我们是不是能猜测，如果抛开内存扩张来说，append 和 copy 其实是一回事。这给我们的教训是，当打算使用 append
的时候，最好提前分配足够的空间，这样有助于提升性能，减少由于性能扩张带来的损失。

copy 函数生成的代码最简单的，理论上它的性能也是最好的。因为大部分人使用 append 函数都不会提前分配内存，让 append
函数自己管理内存，进行扩张和赋值，扩张除了分配空间之外，还需要把原来的数据拷贝过来。

同样是字节切片的操作，不同的操作方式有很大的性能差异。所以同样一种东西有不同的操作方式，在什么情况下用什么方式最合理的，同样是 append 和 copy
内置函数，它们究竟有什么不同？什么情况下性能差不多？什么情况下用什么方式最合理？

很显然，对大数据复制的话 copy 性能最好，因为可以提前预估内存给足够的内存。对于 append 操作比较多，最好内存直接提前分配好，因为 copy
行为比较固定，就能提前知道分配多大数据。append 操作另外的意思是，不确定往里面追加多少数据，所以它们使用场景不同，copy
明确知道要复制多少数据；append 有个动态构建行为，并不清楚往里面追加多少东西。

### 避免长时间引用大数组

    
    
    //go:noinline
    func test() []byte {
        s := make([]byte, 0, 100<<20)
        s = append(s, 1, 2, 3, 4)
        return s
        // s2 := make([]byte, len(s))
        // copy(s2, s)
        // return s2
    }
    func main() {
        s := test()
        for {
            fmt.Println(s)
            runtime.GC()
            time.Sleep(time.Second)
        }
    }
    
    
    
    $ go build && GODEBUG=gctrace=1 ./test
    

我们不要长时间地引用一个大数组，因为切片使用上有个麻烦。

比如 test 函数返回一个切片，但是有 100M
底层数组。通过切片指向四个元素返回，这个切片依然会引用这个底层数组，导致这个底层数组生命周期一直被引用。可能会下意识地觉得，只是返回切片复制品，实际上对数组没有任何改变，返回的复制品依然指向底层数组，100M
内存一直存在。从语法上我们可能会以为返回切片，在实现上是不一样的。这很容易造成误解，跟踪垃圾回收我们发现 100M 内存一直会在。

那么怎么改变？创建一个新的切片，这个切片的底层引用了一个新的数组，这个数组长度为 4，把 1234
数据复制下来，把这个切片返回，接下来访问这个复制品，它所引用的数组是新的。原来的数组过了生命周期就会回收掉。

### 什么时候需要使用 copy 创建新切片对象，避免大底层数组浪费内存？

使用切片时候，还有需要注意的、也算是比较常见的麻烦。

很多时候我们会折腾一些操作，操作完返回其中的一些数据，比如 data 从随机文件里取出了 20MB
的数据，但是我们真正需要的其实是数据很小的部分，比如我们需要读出很大的文件，读出文件以后我们通过内存的计算找到我们真正需要的一部分，因为我们可能并不确定我们需要的那部分数据在文件哪个部分。

这时候其实非常小心一点是什么？切片指针还引用底层数组，指针引用某个位置和引用整个数组对 GC 来说是一回事。因为整个数组是一个对象，不可能 GC
只回收数组其中的一个区域，所以切片只是引用其中的一小块，但是 GC 来说整个数组都被你引用了，因此引用就相当于字段的成员引用整个结构体。

同样地，引用整个数组的某个片段，对 GC 来说你就持有了整个数组。

这样一来的问题是，底层创建了 20M 的切片，虽然返回其中的片段只有 10 字节，但是 20MB 的数组是被你持有的，这样一来 20MB 内存释放不掉。

    
    
    func test() []byte {
        data := make([]byte, (1<<20)*20) //20MB
        rand.Read(data)                  // /dev/urandom
    
        return data[10:20] // 10 byte 返回的是20MB的数组，可以读取其中的10个字节
    
        // x := make([]byte, 10) //创建10字节的数组
        // copy(x, data[10:])    //需要的数据拷贝到10字节数组里面
        // return x              //返回10字节的数组
    }
    
    func main() {
        x := test()
    
        for i := 0; i < 60; i++ {
            runtime.GC()
            time.Sleep(time.Second)
        }
        runtime.KeepAlive(x)
    }
    
    
    
    go build && GODEBUG=gctrace=1 ./test
    

测试我们可以看到，20MB 内存垃圾回收根本释放不掉，10 字节被你持有，底层数组都被你持有。

所以这种时候我们应该怎么做呢？如果我们只是返回超大号数据的片段，最好的方式创建一个新的切片，创建完了之后的底层数组和上面底层数组不是一回事了，然后把我需要的数据从大数组里拷贝到大数组里面去。接下来持有的是小数组的指针，那么大数组指针就整个被释放掉了。

这个例子说明不要觉得返回切片没问题，一不小心就会造成很大的内存浪费。返回一个切片并不像想象那么简单，一定得知道切片到底引用了什么。

