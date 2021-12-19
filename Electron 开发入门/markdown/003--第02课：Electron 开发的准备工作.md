### 2.1 搭建 Electron 开发环境

在使用 Electron 开发应用之前，要先安装 Electron，而 Electron 需要依赖 Node.js，因此在安装之前，要先安装
Node.js。Node.js 允许使用 JavaScript 开发服务端以及命令行程序，读者可以到 [Node.js
的官网](https://nodejs.org)下载最新版本的安装程序。

Node.js 是跨平台的，建议读者下载长期维护版本（LTS），然后双击安装程序开始安装即可。

安装完 Node.js 后，进入终端（Windows 下是命令提示符窗口），运行如下命令安装 Electron。

    
    
    npm install electron -g
    

如果安装成功，会显示如下图所示的信息（Windows）。

![](https://images.gitbook.cn/b650e220-ac2d-11e8-8c28-1fcd9a8f1993)

安装完 Electron 后，可以输入下面的命令查看 Electron 版本。

    
    
    electron -v
    

如果想删除 Electron，可以使用下面的命令。

    
    
    npm uninstall electron
    

如果想升级 Electron，则可以使用这个命令。

    
    
    npm update electron -g
    

直接执行 electron 命令，会显示如下图所示的窗口，该窗口包含了与 Electron 相关的信息，如 Electron 的版本号、Node.js
的版本号、API Demo 的下载链接等。

![](https://images.gitbook.cn/f09a4680-ace9-11e8-9c45-adc0fa12a28f)

### 2.2 开发第一个 Electron 应用

在开发 Electron 应用之前，需要创建一个 Electron 工程。Electron 工程必须要有一个 package.json 文件，创建
package.json 文件最简单的方式就是使用下面的命令。

    
    
    npm init
    

在执行上面命令之前，最好先建立一个工程目录，在执行命令的过程中，会要求输入一些信息，输入过程如下图所示，如果不想输入，一路回车即可。本例输入了
package name（first）、entry point：(first.js），前者是包名，也可以认为是工程名，默认是
electron；后者是入口点，也就是运行 Electron 应用第一个要运行的 JavaScript 文件名，默认是 index.js。

![](https://images.gitbook.cn/97942c60-ac33-11e8-afe5-6ba901a27e1b)

通过上面的命令自动创建 package.json 文件的内容如下：

    
    
    {
      "name": "first",
      "version": "1.0.0",
      "description": "",
      "main": "first.js",
      "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1"
      },
      "author": "",
      "license": "ISC"
    }
    

其实 package.json 文件中大多数内容都不是必须的，可以将 package.json 文件的内容精简为如下形式：

    
    
    {
      "name": "first",
      "version": "1.0.0",
      "main": "first.js"
    }
    

接下来编写一个最简单的 Electron 程序，该程序除了 package.json 文件是必须的，还有另外两个文件也是必须的，一个就是在
package.json 文件中定义的入口点文件，本例是 first.js；另外一个就是要显示在主窗口中的页面文件。由于 Electron 应用与 Web
应用使用同样的技术，因而这个页面文件就是 HTML 文件，本例是 index.html。因此，一个最基本的 Electron 应用由下面 3 个文件组成：

  * package.json
  * first.js
  * index.html

这里的 index.html 文件就是普通的网页文件，下面给出简单的文件内容。

    
    
    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        <title>Hello World!</title>
    </head>
    <body>
    <h1>这是第一个Electron应用</h1>
    </body>
    </html>
    

first.js 文件的基本任务就是创建一个窗口，并将 index.html 文件显示在这个窗口上，现在先给出 first.js 文件的代码。

    
    
    const {app, BrowserWindow} = require('electron')
     function createWindow () {   
        // 创建浏览器窗口
        win = new BrowserWindow({width: 800, height: 600})
        // 然后加载应用的 index.html
        win.loadFile('index.html')
      }
    app.on('ready', createWindow)
    

其中，electron 是 Electron 的模块，必须引用，该模块导出了一个 app 对象和一个 BrowserWindow 类，app
对象包含一些方法，如 on 方法用于将事件绑定到事件函数中。在代码的最后，将 createWindow() 函数绑定在 ready 事件上，该事件会在
Electron 应用运行时执行，通常在 ready 事件中创建主窗口，以及完成一些初始化工作。

在 createWindow() 函数中创建了 BrowserWindow 对象，一个 BrowserWindows 对象表示一个窗口，通过
BrowserWindow 类构造方法参数指定窗口的尺寸（800 × 600），然后通过 loadFile 方法装载 index.html 文件。

最后使用下面的命令运行 Electron 应用。

    
    
    electron .
    

> 注意：运行上面命令时，终端（或命令提示符）应该在 Electron 工程目录下。

程序运行效果如下图所示。

![](https://images.gitbook.cn/ee56d250-ac3d-11e8-91e0-0f47f5fddd18)

### 2.3 响应事件

编写 GUI 应用要做的最重要的事情就是响应事件，如单击按钮事件、窗口关闭事件等。对于 Electron 应用来说，事件分为如下两类：

  * 原生事件
  * Web 事件

由于 Electron 在创建窗口时需要依赖本地 API，因而有一部分事件属于本地 API 原生的事件。但 Electron 主要使用 Web
技术开发应用，因而用的最多的还是 Web 事件，这些事件的使用方法与传统的 Web 技术完全相同。

Electron 的原生事件有很多，比如窗口关闭事件 close、Electron 初始化完成后的事件
ready（这个在前面已经讲过了）、当全部窗口关闭后触发的事件 window-all-
closed（通常在这个事件中完成最后的资源释放工作）、Electron 应用激活后触发的事件（activate，在 macOS 上，当单击 dock
图标并且没有其他窗口打开时，通常在应用程序中重新创建一个窗口，因此，一般在该事件中判断窗口对象是否为 null，如果是，则再次创建窗口）。

下面完善 first.js 文件的代码，添加了监听上述窗口事件的代码。

    
    
    const {app, BrowserWindow} = require('electron');
    
     function createWindow () {   
        //创建浏览器窗口
        win = new BrowserWindow({width: 800, height: 600});
    
        //然后加载应用的 index.html
        win.loadFile('index.html');
        //关闭当前窗口后触发 closed 事件
        win.on('closed', () => {
          console.log('closed');
          win = null;
        })
      }
     //Electron 初始化完成后触发 ready 事件 
    app.on('ready', createWindow)
    //  所有的窗口关闭后触发 window-all-closed 事件
    app.on('window-all-closed', () => {
        console.log('window-all-closed');
        //非 Mac OS X 平台，直接调用 app.quit() 方法退出程序
        if (process.platform !== 'darwin') {
          app.quit();
        }
      })
      //窗口激活后触发 activate 事件
      app.on('activate', () => {
        console.log('activate');
        if (win === null) {
          createWindow();
        }
      })
    

首先在 Windows 10 上测试 Electron 应用，运行 Electron 应用，会显示 Electron
窗口，然后关闭窗口，会在命令提示符中显示如下图所示的信息。

![](https://images.gitbook.cn/1d683780-acfe-11e8-9c45-adc0fa12a28f)

很明显，window-all-closed 事件先于 closed 触发，不过并没有触发 activate 事件，这个事件需要在 Mac OS X
上触发。现在切换到 Mac OS X 系统，用同样的方法运行 Electron
应用，然后最小化窗口，再让窗口获得焦点，最后关闭窗口，会看到终端输出如下图所示的信息。

这说明在 Mac OS X 系统下，当窗口最小化后再获得焦点，会触发 activate 事件，然后关闭窗口，会触发 window-all-closed 和
closed 事件，不过当关闭最后一个窗口后，Mac OS X
下的应用并不会真正退出，而是应用的一部分仍然驻留内存，这主要是为了提高再次运行应用的效率，这也是为什么 Mac
机器的内存占用率会越来越大的原因，因为应用一旦启动，就不会真正完全退出，iOS 系统也有这个问题。

![](https://images.gitbook.cn/f40b1dc0-acfe-11e8-afe5-6ba901a27e1b)

### 2.4 Electron 应用的特性

到现在为止，我们已经完成了第一个 Electron 应用，再来看一下使用 Electron 开发的应用可以拥有哪些特性：

  * 支持创建多窗口应用，而且每个窗口都有自己独立的 JavaScript 上下文；
  * 可以通过屏幕 API 整合桌面操作系统的特性，也就是说，使用 Web 技术编写的桌面应用的效果与使用本地编程语言（如 C++）开发的桌面应用的效果类似；
  * 支持获取计算机电源状态；
  * 支持阻止操作系统进入省电模式（对于演示文稿类应用非常有用）；
  * 支持创建托盘应用；
  * 支持创建菜单和菜单项；
  * 支持为应用增加全局键盘快捷键；
  * 支持通过应用更新来自动更新应用代码，也就是热更新技术；
  * 支持汇报程序崩溃；
  * 支持自定义 Dock 菜单项；
  * 支持操作系统通知；
  * 支持为应用创建启动安装器。

我们刚刚看到，Electron 支持大量的特性，上述列出的只是其中一部分。其中，程序崩溃汇报是 Electron 独特的特性，NW.js
目前并不支持该特性。Electron 最近还发布了用于应用测试和调试的工具：Spectron 和 Devtron，在后面的课程内容中将会对它们进行详细介绍。

> [点击了解更多《Electron
> 开发入门》](https://gitbook.cn/m/mazi/comp/column?columnId=5c3168154fcd483b02710425&utm_source=lnsd002)

