### 利用匿名函数重构作用域

以便 defer 能在合适时机执行。

    
    
    // 错误的例子
    func example() {
        var m sync.Mutex
    
        for i := 0; i < 5; i++ {
            m.Lock()
            defer m.Unlock()
            println(i)
        }
    }
    

这地方有个问题，example 函数是一个加锁操作。m.Unlock() 只有在 example 函数结束的时候才执行，那么每次循环实际上是执行
m.Lock() 语句，这个时候解锁操作会被延迟到函数结束。很显然这个逻辑并不是我们想要的，我们当时写的意思是加锁，defer
为了保证锁会被释放，defer 就是语句后面就算出错也会执行解锁。

显然现在逻辑出错了，解锁被延长了。

很显然，它会认为程序死锁，为什么死锁，是因为你不停的加锁，解锁没有执行。

怎么重构呢？最简单的做法用匿名函数 func(){} 把 for 循环里面逻辑包起来，因为这个时候 defer
是在匿名函数执行完执行，所以你每次循环都可以保证加锁解锁都可以被执行。

    
    
    func example() {
        var m sync.Mutex
    
        for i := 0; i < 5; i++ {
            func(){
                m.Lock()
                defer m.Unlock()
                println(i)
            }()
        }
    }
    

所以利用匿名函数缩小作用域，因为不用匿名函数这个作用域相当于 example。如果做了匿名函数重构，当前作用域就变成作用域了。

### 使用 defer 改善错误处理

有很多种方式去改善代码，就是写代码的能力，怎么控制让代码写得更干净。例如下面例子，很多函数需要判断
error。重构思路构建一个类似方法表或者函数表，如果参数签名不一样，直接把它打包成匿名函数。

    
    
    func test1() (int, error) {
        return 1, io.EOF
    }
    
    func test2() (int, error) {
        return 1, io.EOF
    }
    
    func test3(x int) (int, error) {
        return x, io.EOF
    }
    
    func main() {
        fns := []func() (int, error){test1, test2, func() (int, error) { return test3(1) }}
        for _, fn := range fns {
            n, err := fn()
            if err != nil {
                log.Println(err)
            }
            log.Println(n)
        }
    }
    

函数重构方式，演进步骤：

  * 重构的一个核心是解耦，而且使得框架基于组装模式（mock、service）；
  * 各部分可独立测试（缩小变化范围），隐藏细节。

重构是否使用闭包：

  * 闭包环境变量的前置要求
  * 闭包匿名函数定义限制
  * 闭包侵入式控制
  * 闭包共享内存限制

延迟调用做得不是特别优雅，例如结构化异常流程控制相对来说比较干净一点，当然要付出相应的代价，因为结构化异常在编译时候需要在调用堆栈上维持一定的状态，否则流程跳转会有问题。

我们用一个例子说明。这段代码先打开数据库可能出错，接下来立即判断是否出错；然后启动事务，启动事务有可能失败，选择的数据库万一不支持事务呢；接下来执行插入操作的时候又可能失败，不但要记录错误信息，事务还要回滚，写下来一半的代码就是判断。如果把错误信息全部去掉，真正的代码就很少，使用结构化异常就很干净。

因为 Go 对于错误处理是基于 C 语言的，我们知道 C
语言没有结构化异常都是返回一个错误。这个错误有两种方式返回，第一种是函数显式的返回，第二种是有个全局变量记录错误，如果出错检查全局变量。其实是一回事，无非是全局变量还是本地变量，相对来说本地变量理论上更好一点，尤其现在高并发状态下，全局变量不知道什么地方出错的。

所以结构化异常和错误返回值判断是两种不同的风格。错误返回值好处是对于编译器来说麻烦更少而且相对来说比较轻量级，它就是普通的返回值不需要做特别的处理，包括对调用堆栈的跟踪变得非常简单，因为所有的返回值都会返回到调用方内存里面去。缺点在于源码上特别难看。结构化异常的好处是源码上干净，缺点是实现结构化异常需要专门的机制去支持，相对来说付出更大的代价，因为结构化异常需要安装调用堆栈跟踪。

    
    
    func testdb() {
        db, err := sql.Open("sqlite3", "./test.db")
        if err != nil {
            log.Fatalln(err)
        }
    
        defer db.Close()
    
        tx, err := db.Begin()
        if err != nil {
            log.Fatalln(err)
        }
    
        _, err = tx.Exec("INSERT INTO user(name, age) VALUES (?,?)", "userx", 90)
        if err != nil {
            tx.Rollback()
            log.Fatalln(err)
        }
        var id int
        err = tx.QueryRow("SELECT id FROM user LIMIT 1").Scan(&id)
        if err != nil {
            tx.Rollback()
            log.Fatalln(err)
        }
        println(id)
    
        if err := tx.Commit(); err != nil && err != sql.ErrTxDone {
            log.Fatalln(err)
        }
    }
    

Go 没有结构化异常，只能错误返回值方式，这种方式真的有必要这样写么？我们尝试用匿名函数对它重构。

首先数据库那块不打算重构，因为整个进程就唯一的一份，大家都共有一个，程序启动时候打开数据库连接池。

启动事务代码块不需要重构，因为启动事务出错的话，后面代码就不需要了。启动事务不属于业务逻辑而是属于底层机制的。

我们把业务逻辑独立出来，用匿名函数包起来，返回错误值。后面只要出错就立即终止。最后延迟调用捕获这个错误，一旦错误不为空就回滚。

用延迟调用做事务回滚记录错误，因为事务回滚记录错误和业务逻辑无关，属于执行底层机制。

    
    
    func testdb2(db *sql.DB) {
        tx, err := db.Begin()
        if err != nil {
            log.Fatalln(err)
        }
    
        func() (err error) {
            defer func() {
                if err != nil {
                    tx.Rollback()
                    log.Fatalln(err)
                }
            }()
    
            _, err = tx.Exec("INSERT INTO user(name, age) VALUES (?,?)", "userx", 90)
            if err != nil {
                return
            }
            var id int
            err = tx.QueryRow("SELECT id FROM user LIMIT 1").Scan(&id)
            if err != nil {
                return
            }
            println(id)
            return
        }()
    
        if err := tx.Commit(); err != nil && err != sql.ErrTxDone {
            log.Fatalln(err)
        }
    }
    
    func main() {
        db, err := sql.Open("sqlite3", "./test.db")
        if err != nil {
            log.Fatalln(err)
        }
    
        defer db.Close()
        testdb2(db)
    }
    

框架和业务分离：

    
    
    func framework(db *sql.DB, bi func(tx *sql.Tx) error) {
        //启动事务
        tx, err := db.Begin()
        if err != nil {
            log.Fatalln(err)
        }
    
        //执行业务逻辑
        func() (err error) {
            //保证出错回滚
            defer func() {
                if err != nil {
                    tx.Rollback()
                    log.Fatalln(err)
                }
            }()
            err = bi(tx) //panic
            return
        }()
    
        if err := tx.Commit(); err != nil && err != sql.ErrTxDone {
            log.Fatalln(err)
        }
    }
    
    func bi(tx *sql.Tx) (err error) {
        _, err = tx.Exec("INSERT INTO user(name, age) VALUES (?,?)", "userx", 90)
        if err != nil {
            return
        }
        var id int
        err = tx.QueryRow("SELECT id FROM user LIMIT 1").Scan(&id)
        if err != nil {
            return
        }
        println(id)
        return
    }
    
    func main() {
        db, err := sql.Open("sqlite3", "./test.db")
        if err != nil {
            log.Fatalln(err)
        }
    
        defer db.Close()
    
        framework(db, bi)
    }
    

