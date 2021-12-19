上一节中我们学习了如何通过 REST Assured 调用接口并查看测试报告，为了后续可以编写复杂接口调用场景，将通过三次课程带领大家学习 Groovy。

Groovy
作为一门语言有大量知识点，接下来的三节课只会讲解接口测试中会使用的知识点，这样既可以保证没有编程基础的小伙伴也能跟上后续课程，也可以保证不花费大量篇幅讲
Groovy 语言。

如果想了解 Groovy更多的信息，可以查看 Groovy官方文档：[http://Groovy-lang.org/](http://Groovy-
lang.org/%EF%BC%89)

为了达到本此课程的目标，我把 Groovy的讲解分为如下 6 个小部分

  * Groovy 语法
  * Groovy 闭包
  * Groovy 集合处理
  * Groovy 操作数据库
  * Groovy 文件操作
  * Groovy 脚本文件

## Groovy 语法

这里列举了 9 个 Groovy语法上的特点，稍后会针对这些特点做实战练习：

  1. 代码结尾处无需使用";"
  2. 可以不用显示定义数据类型，所有数据类型都可以用 def 定义
  3. 方法返回值前无需添加 return 关键字，如果方法不用 void 修饰，方法内的最后一行返回值即函数的返回值
  4. 可以指定方法中参数默认值，方法中的参数可以不指定数据类型
  5. 所有方法默认都是 public，无需添加 public 关键字
  6. 方法的()可以取消
  7. Gstring：字符串中支持变量解析和换行处理
  8. 任何对象都可以被强制转换为布尔值，任何为 null、void 的对象，等同于 0 或空值都会解析为 false，反之则为 true
  9. Groovy 中的“ == ”是 Java 中的 equal，如果需要判断两个对象值是否相等使用“==”

以下是练习的代码片段，为了运行方便请在 `pom.xml` 文件中添加 junit 依赖，这样就可以使用 @Test 注解方式快速运行所写的代码。

    
    
    def add(a, b, c = 1) {    //方法的参数可以不指定数据类型，参数可以指定默认值，例如C的默认值是1 
            a + b + c         //无retrun关键字，代码结尾处无需添加；
        }
    
        @Test                //使用了junit中@Test注解运行代码
        void testAdd() {
            println add(2, 2)   //因为c有默认值，所有调用add方法的时候，可以不传入第三个变量值
            def c = add 1, 1, 3   //调用方法时，方法的（）可以取消掉
            println c
        }
    

以下代码中使用了 Groovy 提供的 Gstring，双引号中添加${变量}，可自动进行变量解析，"""中支持换行符解析

    
    
     @Test
        void testGstring() {
            def c = 100
            println "this c value is ${c}"      //“”支持变量解析
            int[] array = [1, 2, 3]
            array.each() { it ->
                println """the array value is ${it}\n """
            }
        }
    

以下代码实践了 Groovy 中任何对象都可转换为布尔值

    
    
    @Test
        void testIf() {
            def c = 0
            int[] myArray = []
            def myString = ""
            if (!c) {  //0转换为布尔值
                println "c is 0"
            }
            if (!myArray) {   //空数组转换为布尔值
                println "myArray is 空值数组"
            }
            if (!myString) {  //空字符串转换为布尔值
                println "myString is 空值字符串"
            }
        }
    

以下代码实践了 Groovy 中对象值比较使用 ==，对象引用比较使用 is

    
    
    @Test
        void testEqual() {
            def a = [1,2,3]
            def b = [1,2,3]
            def c = a
            if (a == b) {     //值比较
                println "a's value equal to b"
            }
            if (a.is(b)) {
                println "a is b"
            } else {
                println "a is not b"
            }
            if (a.is(c)) {
                println "a is c"
            }
        }
    

运行上述代码，可以看到正常运行，打印的值与期望一致

![](https://images.gitbook.cn/15749147669315)

上面讲解了 Groovy 的一些基础语法，接下来将讲解 Groovy 的闭包。

## Groovy 闭包

Groovy
官方对闭包的定义是“闭包是一个匿名代码块，可接受参数，返回值并可分配给变量”，闭包使用`{},{clouserParameter->statement}`，clouserParameter指闭包接受的参数，多个参数用逗号隔开，statement指闭包中的代码。

下面是一段闭包实践代码：

    
    
    package tl86.ThirdCourse
    
    import org.junit.Test
    class ClosureStudy {
        def firstClosure = {println "hello world"} //把闭包赋值给变量
        def secondClosure= {a,b -> a+b}
        @Test
        void testClosure() {
            firstClosure() //运行闭包
            println secondClosure(1,2)  //返回1+2的值
        }
        //闭包作为方法的参数
        def myFunction(name,closure) {
            closure(name)
        }
        @Test
        void testMyFunction() {
            myFunction("Dave",{name-> println "my name is ${name}"})
            myFunction("Dave",{it-> println "my name is ${it}"})
            myFunction("Dave",{println "my name is ${it}"})
           // 上面三种写法效果一样，闭包的参数只有一个时，可以用it，且可以省略 it->
        }
        //闭包是方法的唯一参数
        def function(closure) {
            def a ="hello"
            closure(a)
        }
        @Test
        void testFunction() {
            function({it -> println it})
            function{it -> println it} //方法的括号可以取消，前面已练习过
            //上面两种写法效果一样
            // 第二种写法在数据集处理部分使用很多,我们讲闭包也是为数据集处理做准备，接口测试中有大量数据集处理的场景
        }
    }
    

运行上述代码如下图所示，可以看到都运行成功，说明闭包使用正确接。下来讲解的数据集处理就用到了闭包，例如 Groovy 语言自带的 find，findAll
等方法就是闭包

![](https://images.gitbook.cn/15749147669337)

## 数据集处理

前面的知识点可以说是学习数据集处理的铺垫，接口测试中有大量对 JSON 对象解析的场景，要掌握好这个首先得学习 Groovy 中数据集的处理。以下是 map
和 list 的 each 方法使用

    
    
    @Test
        void testEach() {
            //数组的each方法，就是一个闭包
            def firstList=["zhangshan",1,2,"lisi"]
            firstList.each{println it}
    
            // 采用each巧妙计算数组的和
            def secondList=[1,3,5,7]
            def a=0
            secondList.each {a = a +it}
            println a
    
            // map对象也有each方法
            def myMap = ["name":"Tom","age":100]
            myMap.each {key,value -> println key + "---" +value}
        }
    

以下是list的find方法使用

    
    
     @Test
        void testFind() {
            def firstList = [1, 3, 5, 8, 9]
            def result = firstList.find{ it -> it > 5 }
            println result //打印出8
            result = firstList.findAll{it -> it>5}
            println result  // 打印出[8,9]
    
        }
    

运行上述代码，全部运行成功且打印的信息与期望一致。说明获取、查找数据集的方式正确。

![](https://images.gitbook.cn/15749147669355)

上面讲了这么多，那在接口测试什么场景会使用上面所学知识呢？假如我们有个待测接口，作用是返回书籍信息，Response 是个 JSON 对象。

对于这个接口，测试场景是：校验返回值中是否包含 bookName 等于“三国演绎”的书籍，并判断该书的价格是否等于期望的价格。

为了测试该场景，我们先用wiremock模拟一个这样的接口,以下是mapping文件

    
    
    {
      "request": {
        "method": "GET",
        "urlPath": "/api/getAllBooks"
      },
      "response": {
        "status": 200,
        "bodyFileName": "bookDetails.json",
        "headers": {
          "Content-Type": "application/json; charset=UTF-8"
        }
      }
    }
    

以下是接口的response文件

    
    
    {
      "books": [
        {
          "name": "西游记",
          "price": 100
        },
        {
          "name": "三国演绎",
          "price": 20
        },
        {
          "name": "水浒传",
          "price": 33
        }
      ],
      "store": "木子书店"
    }
    

以下是测试场景脚本

    
    
    class Case extends Specification {
        //测试场景：期望调用接口获取到名字为"三国演绎的书籍"且判断价格是否等于20
    
        JsonSlurper jsonSlurper = new JsonSlurper()
        //接口返回的是json字符串，jsonSlurper作用是将json字符串转换为 Groovy  的集合对象
    
        def "should book's price correct"() {
            given: ""
            when: "get books response"
            def response = given().baseUri("http://localhost:9090")
                    .when()
                    .get("/api/getAllBooks")
                    .then()
                    .assertThat()
                    .statusCode(200)
                    .extract().response().getBody().asString()
            then: "book's price is correct"
            def bookPrice = jsonSlurper.parseText(response).books.find{ it -> it.name == bookName }.price    //通过将字符串的response转换为数据集，然后调用 Groovy  自带的find方法，轻松找到名称为“三国演绎”的书籍价格
            Assert.assertEquals("bookPrice: ${bookPrice} is not correct",bookPrice, price)   //校验书籍的价格是否等于期望的价格，如果校验失败，会打印填写的错误信息
            where:
            bookName | price
            "三国演绎"| 20
        }
    }
    

执行上述 Case 如下图所示，可以看到如果期望价格设置为 20，Case 运行通过。修改期望价格为 19，Case
运行失败，且打印除了对应的错误信息，说明 Case 脚本达到了校验前面测试场景的目的。

![](https://images.gitbook.cn/15749147669379)

通过上面的例子可以看到利用 Groovy 的闭包可以轻松获取 Response 中需要的信息并进行校验。

至此今天的课程就结束了，本次课程学习了 Groovy 的基本知识，并进行了“接口测试中如何通过 Groovy 的闭包轻松获取response中信息”的练习。

下次课程将介绍如何使用 Groovy操作数据库。

