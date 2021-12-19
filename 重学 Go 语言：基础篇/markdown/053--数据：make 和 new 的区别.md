### make 和 new 的区别

我们知道，在 Go 里面创建引用对象有几种方式。

    
    
    func makeMap() map[int]int {
        return make(map[int]int)
    }
    
    func newMap() map[int]int {
        //new返回的是一个指针，我们取指针的值
        return *new(map[int]int)
    }
    
    func main() {
        m1 := makeMap() // m1(*hmap) -> heap(hmap)
        m2 := newMap()  // m2(*hmap) -> 00 00 00 00
    
        fmt.Println(m1, m2)
    
        m1[1] = 100
        m2[1] = 100 //nil map?
    
        p1 := (*uintptr)(unsafe.Pointer(&m1))
        p2 := (*uintptr)(unsafe.Pointer(&m2))
    
        fmt.Println("%x, %x\n", *p1, *p2)
    }
    

这两种到底有什么不同？就字典来说在行为上有什么不同？

任何复合结构的内存都是延迟分配的，也就是当你往里面赋值的时候才去分配。make 和 new 都不会对 key、value 提前分配内存，除非指定容量。首先
hash 表在没有往里面添加东西的时候我们怎么知道分配多大合适，也许对方只分配 6 个 keyValue 对，分配 100
个都不合理。所以这种复合结构的内存分配，多数时候会采取的策略就是延迟分配，在需要的时候进行扩容。例如切片在很小时候两倍扩容，很大时候按照四分之一扩容，越大就缩小扩容阈值。

反汇编看到 makeMap 通过 runtime.makemap 函数来实现字典创建。make 是个语法糖，最后都会被编译器翻译成具体的函数调用。

hashmap.go 文件中 makemap 函数很显然返回的是通用型指针
*hmap，不管是什么类型的字典，它使用的是同一种数据结构，而且这个函数返回的其实是一个指针。

    
    
    func main() {
        m1 := make(map[string][100]byte) // *hmap -> map[key]value
        m2 := make(map[byte]byte)        // *hmap -> map[key]value
    
        fmt.Println(unsafe.Sizeof(m1), unsafe.Sizeof(m2))
    }
    

换句话说例如我们创建不同的字典，可能 value 数据集有很长，很显然从表面上看，这两种字典的 key 和 value 差别非常的大，从源码上看 m1、m2
其实都应该是个指针。也就是说 keyValue 的数据类型和字典本身的内部数据结构没有关系，我们输出 m1、m2 的长度和 keyValue
的数据类型有没有关系，例如切片结构就是 {ptr, len, cap}，也就是说，底层存什么和当前对象本身并不是一致的关系。

m1、m2 其实都是 *hmap 指针，只不过在使用的时候，编译器会把它对应到 keyValue 数据类型上去，但是从内存布局上说 m1、m2
就是指针。字典类型本身就是个指针。

makemap 函数：

    
    
    buckets := bucket
    if B !=0 {
        buckets = newarray(t.bucket, 1<<B)
    }
    

可以看到，真正的数据桶是有可能创建也有可能不创建的，也就意味着这是一种运行期行为，数据桶用来存 keyvalue 数据的。makemap
函数创建新的字典对象，相关东西全部初始化状态。很显然 makemap 函数并没有我们想象当中那么复杂，实际上的核心就是创建字典对象。

所以当我们执行 make 调用的时候，实际上会被翻译成 runtime.makemap 函数调用，它在堆上创建了 hmap
的对象，接下来返回这个对象的指针。也就是说 make 函数其实我们拿到的是两个对象，一个是堆上初始化状态的字典，m1 存储的实际上是对象指针。

我们现在知道了 map[int]int 本身是个指针，那么 new(map[int]int) 和 new(uintptr) 是一回事。所以
new(map[int]int) 是在堆上创建了一个 8 字节指针内存，因为 new 的话会初始化为零值，就是 8 字节 0。接下来返回这个指针，所以 m2
是在堆上创建了 hmap 指针，但是里面没有指向任何东西，是零值。

也就是说 make 函数其实创建了两个对象，一个指针对象一个字典结构体对象。new 只是创建了指针对象，这就是差别。当我们访问的时候，m1 找到对应的
hmap 然后进行赋值操作，m2 去找的话都是零值，实际上就是 nil pointer。

这就解释了 m1 和 m2 都是合法的，但是 m1 有个真实的目标指向，m2 没有，m2 虽然是个指针，但是没有指向真实的目标数据，只是为指针类型创建 8
字节内存，但是并没有让这个指针指向有效的对象，我们知道一个指针如果没有指向有效的对象是没有用的。赋值的时候需要透过指针找到目标对象，然后对目标对象进行操作，因为赋值是
mapassign1 函数，它必须拿到 hmap 对象。问题是 m2 没有所以输出错误。

    
    
    (gdb) info locals
    (gdb) ptype m1
    (gdb) ptype m2
    (gdb) p *m1
    (gdb) p *m2
    

所以我们同样是持有指针，用 make 持有的指针是指向堆上的对象，有明确的指向。new 虽然分配了内存，内存存储指针，但是没有指向。

new 计算你所提供数据类型的长度，然后在堆上按照提供的类型的长度分配内存空间，同时保证这段内存是被初始化过的，然后把内存的指针返回出来。

