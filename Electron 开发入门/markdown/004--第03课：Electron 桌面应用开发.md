开发一款功能完整的桌面应用，通常不会使用记事本去完成所有的开发工作，核心的开发工作一般会在 IDE（集成开发环境，Integrated
Development Environment）中完成。由于 Electron 应用主要使用 Web 技术（HTML、CSS 和
JavaScript）开发，因此只要支持编辑 JavaScript、HTML、CSS 代码的 IDE，都可以开发 Electron
应用。本节课将介绍两款常用的 IDE：WebStorm 和 VS Code，这两款 IDE 都支持 Web 开发，不过要想开发 Electron
应用，还需要做一个配置和调整。

### 3.1 用 Webstorm 开发 Electron 应用

可以直接在 WebStorm 中编辑 Electron 应用的代码，不过运行 Electron 应用需要执行如下的命令。

    
    
    electron .
    

当然可以在终端输入上述命令，不过比较麻烦，比较好的做法是在 WebStorm 中直接运行 Electron 应用，实现方法有两个，我们逐一介绍。

#### 1\. 使用扩展工具

首先在 WebStorm 中先创建一个空的工程，然后将前面编写的 Electron 应用的 3 个文件复制到 WebStorm 工程中。

再将需要执行的命令添加到 WebStorm 的扩展工具中，打开 WebStorm 的扩展工具设置窗口，按下图进行设置，最后单击 OK 按钮关闭 Create
Tool 对话框。

![avatar](https://images.gitbook.cn/FpUJeSj7_bwj9lc_zO58yYQ3SSr7)

创建完运行 Electron 应用的扩展工具后，选择工程中的文件，然后在右键菜单中单击 External Tools |
electron（如下图所示），就会在 WebStorm 中运行 Electron 应用；或直接单击 WebStorm 中的 Tools |
External Tools | electron 菜单项，也可以运行 Electron 应用。

![avatar](https://images.gitbook.cn/FncYMl6MKyy-hkFLrdeQa9RMtXsL)

> 注意：在 Windows、Program 中要输入 electron.cmd。

#### 2\. 使用脚本文件

使用第一种方式运行 Electron 应用，每次都需要单击 electron
菜单项，比较麻烦。为了更简单，可以在工程源代码文件目录（一般为工程根目录）创建一个脚本文件，如 run.js，并输入下面的代码：

    
    
    var exec = require('child_process').exec;
    free = exec('electron .');
    

这两行代码使用了 Node.js 中 child_process 模块的响应 API 执行 electron 命令。直接在 WebStorm 中运行
run.js 文件即可，在第一次运行该文件后，下一次运行可以直接单击 WebStorm 右上角的运行按钮，如下图所示。

![avatar](https://images.gitbook.cn/Fo_sLSDujHKJSK1gqvySXcRsK-U5)

### 3.2 用 VS Code 开发 Electron 应用

VS Code 是微软公司开发的开源的代码编辑工具，支持插件，它本身其实就是用 Electron 开发的，因此使用 VS Code 开发 Electron
应用更能说明 Electron 的强大。

![avatar](https://images.gitbook.cn/Fr2pOxKRrNGqT2OrUKcJK-JHJuJw)
![avatar](https://images.gitbook.cn/FpjGwpXUeUr5WN1kc9m-Hf4xSSt-)

### 3.3 用设置断点的方式调试 Electron 应用

调试程序是开发 Electron 应用必不可少的步骤，最简单的调试方式就是执行下面的代码，在主窗口右侧会显示调试窗口，如下图所示。

    
    
     win.webContents.openDevTools()
    

![avatar](https://images.gitbook.cn/FsUESXjFmySjej_Lidbcg_g7RnuT)

使用 console.log() 方法可以在调试窗口中输出信息。不过这种调试方式比较笨，而且不能观察变量等资源的状态，因此本节课推荐另外一种调试方式，使用
VS Code 设置断点的方式调试 Electron 应用。

切换到 first.js 文件，在 createWindow() 函数中添加如下两行代码。

    
    
    var n = 20;
    console.log(n);
    

然后在 var n = 20; 的序号前面单击设置断点，如下图所示。

![avatar](https://images.gitbook.cn/Fmp-4by8zCn_RKqGkXQKhixXLN8B)

切换到调试窗口（左侧第 4 个按钮），如下图所示。

![avatar](https://images.gitbook.cn/Fv6EvF6pmUbjghsm3nSLYxa8wa1D)

接下来设置 launch.json 文件，代码如下：

    
    
    {
        "version": "0.2.0",
        "configurations": [
            {
                "type": "node",
                "request": "launch",
    
                "cwd": "${workspaceRoot}",            
                "name": "Electron Main",
                "runtimeExecutable": "electron",
                "runtimeArgs": [
                    ".",
                    "--enable-logging"
                ],
    
                "protocol": "inspector"
            }
        ]
    }
    

单击上方的 Debug Main Process 按钮，会用调试的方式运行 Electron
应用，这时并没有显示窗口，而是程序停到了设置断点的那一行。单击代码窗口上方的 Step Over、Step Into
按钮，会一步一步执行程序，在左侧的变量监视区域会显示相关变量值的变化。双击变量，还可以在调试状态修改变量的值，这样可以更方便地观察不同值的变量对程序的影响。

> [点击了解更多《Electron
> 开发入门》](https://gitbook.cn/m/mazi/comp/column?columnId=5c3168154fcd483b02710425&utm_source=lnsd002)

