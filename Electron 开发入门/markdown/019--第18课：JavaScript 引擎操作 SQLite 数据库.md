尽管 Electron 应用可以利用 localStorage API 将数据以键值的形式保存在客户端，但 localStorage
并不适合存储大量的数据，而且 localStorage 是通过键值存储数据的，比较适合保存配置信息，而不是海量的结构化数据，如二维表。因此，要想在
Electron 客户端保存更复杂的数据而且便于检索，就需要使用真正的数据库，如 SQLite、MySQL。

由于 Electron 是基于 Node.js 的，因而 Node.js 支持的数据库，Electron 同样也支持。本节将会介绍如何在 Electron
中使用 SQLite 数据库，后面的文章会介绍如何在 Electron 中直接操作 MySQL 数据库。

SQLite 是一款轻量级桌面关系型数据库，[点击这里详见官网](https://www.sqlite.org)。

SQLite 支持多种接口，如 Java、C++、C、Python、JavaScript 等，Node.js 也支持多种方式操作 SQLite
数据库，其中最简单的就是使用 sql.js。sql.js 是一个 JavaScript 脚本文件，是直接从 C 语言版本的 SQLite
操作引擎转换过来的，因此文件尺寸比较大，大概有 2.7 MB，读者在此并不需要打开 sql.js 研究里面的代码，只要知道如何用即可。

由于 sql.js 是使用纯 JavaScript 编写的，因而兼容性非常好，在 Node.js 中也有非常好的表现。本节将会介绍 sql.js
的基本用法，也就是如何利用 sql.js 创建或打开 SQLite 数据库，以及数据库的基本操作（增、删、改、查）。

读者只需要将 sql.js 文件复制到 Electron 工程根目录或其他子目录即可，然后使用下面的代码导入 sql.js。

    
    
    let sql = require('./sql.js');
    

再使用下面的代码创建 SQLite 数据库。

    
    
    let db = new sql.Database();
    

> 注意，这里创建的 SQLite 数据库实际上是在内存中，并没有作为文件保存在硬盘上。

接下来可以使用 run 方法执行 SQL 语句，代码如下。

    
    
    let sql = ...
     db.run(sql)
    

run 方法可以执行多种 SQL 语句，如创建表、插入、更新、删除等；如果要查询记录，可以使用 exec 方法，代码如下。

    
    
    let rows = db.exec("select * from table1");
    

exec 方法会返回一个数组，用于存储查询结果。

要注意的是，前面的所有操作都是在内存中完成的，到现在为止，还没有将 SQLite
数据库保存在硬盘中，因此在执行创建表、插入、更新、删除等任何修改数据的操作后，应该使用下面的代码将内存中的数据库作为文件保存在硬盘中。

    
    
    //获取 SQLite 数据库的二进制数据
    var binaryArray = db.export();
    var fs = require('fs');
    fs.writeFile("test.db", binaryArray, "binary", function (err) {
           ... ...
    });
    

在上面的代码中，db.export( ) 用于获得 SQLite 数据库的二进制数据，然后利用 fs.writeFile 方法将这些数据写到 test.db
文件中。

下面看一个完整的案例，这个案例演示了如何用 sql.js 中的 API 创建数据表，对数据进行增、删、改、查操作，实现步骤如下。

（1）编写主页面文件（index.html）

    
    
    <!DOCTYPE html>
    <html>
    <head>
      <!--指定页面编码格式-->
      <meta charset="UTF-8">
      <!--指定页头信息-->
      <title>SQLite数据库</title>
      <script src="event.js"></script>
    </head>
    <body>
        <button id="button_create" onclick="onClick_CreateDatabase()">创建SQLite数据库</button>
        <p/>
        <button id="button_insert" onclick="onClick_Insert()" disabled>插入记录</button>
        <p/>
        <button id="button_query" onclick="onClick_Query()" disabled>查询记录</button>
        <p/>
        <button id="button_update" onclick="onClick_Update()" disabled>更新记录</button>
        <p/>
        <button id="button_delete" onclick="onClick_Delete()" disabled>删除记录</button>
        <p/>
        <label id="label_rows"></label>
    </body>
    </html>
    

在 index.html 页面中引用了 event.js 文件，并放置了 5
个按钮和一个标签，按钮用了完成数据库的常用操作，在查询记录后，会将查询结果显示在这个标签中。

页面的效果如下图所示。

Mac OS X 的效果。

![](https://images.gitbook.cn/35a8ac60-85cc-11e9-8685-619d8143eac4)

Windows 的效果。

![](https://images.gitbook.cn/3fdc1000-85cc-11e9-aff2-1b677b2c941e)

（2）编写 event.js 脚本文件

在 event.js 脚本文件中包含了本例的核心代码，也就是对 SQLite 数据库的各种操作。

    
    
    let sql = require('./sql.js');
    //在内存中创建数据库
    let db = new sql.Database();
    var fs = require('fs');
    //将内存中的数据库写到 test.db 文件中，callback 是回到函数
    //用于处理写文件成功或失败的事件
    function writeDBToDisk(callback) {
        var binaryArray = db.export();
        fs.writeFile("test.db", binaryArray, "binary", function (err) {
            if (err) {
                if(callback != undefined)
                    callback(err);
            } else {
                if(callback != undefined)
                    callback('成功保存数据库文件');
            }
        });
    }
    //创建数据库
    function onClick_CreateDatabase() {
        //如果 test.db 文件存在，则删除该文件
        fs.exists('test.db',function(exists) {
            if (exists) {
                fs.unlinkSync('test.db');
            }
            //用于创建 t_products 表的 SQL 语句
            let createTableSQL = `create table if not exists t_products(
                              id integer primary key autoincrement,
                              product_name varchar(100) not null,
                              price float not null  )`;
            //运行 SQL 语句创建 t_products 表
            db.run(createTableSQL);
           //将数据库写到 test.db 文件中，并将相应的按钮置为可用状态
            writeDBToDisk((msg)=>{
                button_create.disabled = true;
                button_insert.disabled = false;
                alert(msg);
            });
    
        });
    }
    //插入记录
    function onClick_Insert() {
        if(db == undefined) return;
        let insertSQL = 'insert into t_products(product_name,price) select "iPhone10",10000 union all select "Android手机",8888 union all select "特斯拉",888888;'
        //向 t_products 表中插入 2 条记录
        db.run(insertSQL);
       //将插入记录后的数据库写到 test.db 文件中
        writeDBToDisk((msg)=>{
            alert(msg);
            button_insert.disabled = true;
            button_query.disabled = false;
            button_update.disabled = false;
            button_delete.disabled = false;
        });
    }
    //查询 t_products 表中所有的记录
    function onClick_Query() {
        if(db == undefined) return;
        let selectSQL = 'select * from t_products';
        var rows = db.exec("select * from t_products WHERE id<5");
        label_rows.innerText = '';
        //将查询结果显示在标签中
        for(var i = 0; i < rows[0].values.length;i++) {
            label_rows.innerText += '\r\n产品ID:' + rows[0].values[i][0] +
                '\r\n产品名称:' + rows[0].values[i][1] +
                '\r\n产品价格:' + rows[0].values[i][2] + '\r\n';
        }
    }
    //将 id 等于 3 的记录中的 price 字段值更新为 999999
    function onClick_Update() {
        if(db == undefined) return;
        let updateSQL = 'update t_products set price = 999999 where id = 3';
        db.exec(updateSQL);
        writeDBToDisk((msg)=>{
            alert(msg);
        });
    }
    //删除 id 等于 2 的记录
    function onClick_Delete() {
        if(db == undefined) return;
        let deleteSQL = 'delete from t_products where id = 2';
        db.exec(deleteSQL);
        writeDBToDisk((msg)=>{
            alert(msg);
        });
    }
    

运行程序，单击“创建 SQLite 数据库”按钮，会在工程根目录中创建一个名为 test.db 的数据库文件，然后单击“插入记录”按钮，会向
t_products 表中插入两条记录，再单击“查询记录”按钮，会将 t_products 表中所有的记录显示在标签中，效果如下图所示。

Mac OS X 的效果。

![](https://images.gitbook.cn/4a9d82d0-85cc-11e9-934b-1d6f440edeb3)

Windows 的效果。

![](https://images.gitbook.cn/53451650-85cc-11e9-9df7-175fb72c7dd4)

读者也可以单击其他按钮测试本例。

[点击这里下载源代码](https://github.com/geekori/electron_gitchat_src)

### 答疑与交流

为了让订阅课程的读者更快更好地掌握课程的重要知识点，我们为每个课程配备了课程学习答疑群服务，邀请作者定期答疑，尽可能保障大家学习效果。同时帮助大家克服学习拖延问题！

请添加小助手伽利略微信 GitChatty6，并将支付截图发给她，小助手会拉你进课程学习群。

