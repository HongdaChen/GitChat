本课程使用 VS Code 作为编写 Electron 应用的 IDE，主要原因是 VS Code 对 JavaScript
的调试支持得比较好。在开发正式的项目时，不管是开源的还是私有的，通常都需要通过代码库来管理，最流行的是 Git 代码库。VS Code 已经集成了
Git，能够支持 Git 的基本功能。本节将介绍如何使用 VS Code 将本地代码提交到远程 Git
代码仓库，以及从远程代码仓库克隆（clone）代码到本地。

首先要选择一个远程代码仓库，比较流行的有 GitHub、Bitbucket 等。如果项目是开源的，建议使用 GitHub，如果项目是私有的，建议使用
Bitbucket，当团队人数不大于 5 人，使用 Bitbucket 创建私有项目是完全免费的，而且不限项目数量。

VS Code 本身并不带 Git 工具包，所以要自己安装。如果读者使用的是 Mac OS X 或 Linux 系统，Git
工具包已经集成到系统中了，如果使用的是 Windows 系统，需要通过下载 Git 安装包，[点击这里下载](https://git-
scm.com/download/win)。

安装完 Git 工具包后，在命令提示行输入如下命令查看 Git 的版本。

    
    
    git version
    

VS Code 会自动检测 Git 的版本，要求最低版本是 Git 2.0.0，如果读者下载的是最新的工具包，肯定满足这个要求。

### 4.1 向远程代码库提交源代码

如果项目是新创建的，可以按下面的步骤向远程代码库提交源代码。

（1）创建一个新的目录

需要为新项目准备一个空的目录，本例是 newproject。

（2）初始化本地代码创库

Git 代码库分为本地和远程两种，如果代码发生了变化，首先会将变化保存到本地代码库，然后再将变化提交到远程代码库。由于 newproject 目录中还没有
Git 本地代码库，所以使用下面的命令初始化 Git 本地代码库。

    
    
    git init
    

Cit 本地代码库就是一个 .git 目录，该目录是隐藏的，但使用 cd .git 命令可以进入该目录。

（3）用 VS Code 打开 newproject 目录

（4）在 VS Code 中创建新的文件

也可以将前面编写的 Electron 应用的文件复制到 newproject 目录中，这时 VS Code 左侧的工程文件列表就会显示 newproject
目录中的所有文件。

（5）将 Git 本地代码库与远程代码库绑定

这时只能将修改过的代码提交到本地代码库，如果要将代码提交到远程代码库，需要使用下面的命令指定远程代码库的 URL。

    
    
    git remote add origin https://myuser@bitbucket.org/oriothers/newproject.git
    

其中 origin 是为远程代码库指定一个名字，这个名字可以是任意字符串，通常指定为 origin。

由于本地代码库现在还没有分支，只有第一次提交（commit）才会创建 master 分支，所以需要将 newproject
目录中文件提交到本地代码库。切换到 VS Code，然后单击左侧第 3
个按钮，如下图所示，在右侧会显示所有已经修改的文件，单击文件右侧的加号按钮，将所有修改过的文件暂存更改。

![](https://images.gitbook.cn/f70e8410-85d6-11e9-8d58-614adcc16eab)

然后单击如下图所示的对号按钮将代码提交给本地代码库。

![](https://images.gitbook.cn/fe8f98f0-85d6-11e9-9032-b799f8df8c5d)

在单击对号按钮后，会显示如下图所示的提交消息输入框，输入当前提交的相关消息。此处相当于对当前提交的注释，表明主要对代码做了哪些修改。

![](https://images.gitbook.cn/062a9e70-85d7-11e9-8486-f3b917949bc3)

最后按  键将代码提交给本地代码库。

接下来在终端执行如下的命令查看 Git 分支。

    
    
    git branch
    

如果查询如下图所示的结果，那么说明 master 本地分支已经建立。

![](https://images.gitbook.cn/0d753a00-85d7-11e9-abfb-6dd98f43f841)

现在使用下面的命令将 master 本地分支与名为 origin 的 URL 关联。

    
    
    git push -u origin master
    

（6）向远程代码库提交代码

单击如下图所示的省略号按钮，在弹出菜单中单击“推送”菜单项，就会将本地代码提交到远程代码库。读者可以切换到远程代码库的 Web
管理页面，查看刚才的提交结果。

![](https://images.gitbook.cn/1594ffe0-85d7-11e9-bc1e-f30537d8dd2a)

### 4.2 从远程代码库克隆项目

如果其他人想访问在远程代码库中的项目，需要将代码库中的项目克隆到本地，修改后再提交到远程代码库。

先创建一个目录，这里是 myproject，然后执行下面的命令初始化本地代码库。

    
    
    git init
    

接下来使用下面的命令指定远程代码库的 URL。

    
    
    git remote add origin https://myuser@bitbucket.org/oriothers/newproject.git
    

单击下图所示菜单中的“拉取自”菜单项。

![](https://images.gitbook.cn/1e534fb0-85d7-11e9-b348-99886785a9f8)

显示如下图的输入窗口，选择下方的 URL，以及稍后出来的 master，最后按  键，就会将代码从远程代码库克隆到本地的 myproject 目录。

![](https://images.gitbook.cn/2673fff0-85d7-11e9-a725-a777499fb7c2)

读者也可以直接使用下面的命令从远程代码库克隆代码到本地。

    
    
    git pull origin master
    

### 4.3 文件展示窗口（只针对 Mac OS X）

对于 Mac OS X 系统，通过设置 BrowserWindow 对象的 setRepresentedFilename
方法，来设置窗口所代表的文件的路径名，并且将这个文件的图标放在窗口标题栏上。单击右键，会显示指定文件的目录层次。下面先看一下实现代码。

    
    
    //  index.js
    const {app, BrowserWindow} = require('electron');
     function createWindow () {
        win = new BrowserWindow();
        //  指定 temp 目录下的一个文件
         win.setRepresentedFilename('/temp/centos7-node1（192.168.56.15）.ova')
    
        win.loadFile('./index.html');
        win.on('closed', () => {
          console.log('closed');
          win = null;
        })
      }
    app.on('ready', createWindow)
    

index.html 文件的代码如下：

    
    
    <!DOCTYPE html>
    <html>
    <head>
      <!--  指定页面编码格式  -->
      <meta charset="UTF-8">
      <!--  指定页头信息 -->
      <title>文件展示窗口（针对 Mac OS X）</title>
    
    </head>
    
    <body>
    <button>我的按钮</button>
    </body>
    </html>
    

运行程序，会看到指定文件的图标显示在窗口的标题栏上，如下图所示。

![](https://images.gitbook.cn/2e332f40-85d7-11e9-bfb9-e99dcfb524e2)

右键标题栏，会弹出如下图所示的菜单，依次往下会看到当前文件的目录结构，当前文件在 temp 目录中，而 temp 在“我的硬盘”中。

![](https://images.gitbook.cn/35ca3d20-85d7-11e9-aaa8-11db97ef343e)

单击某个菜单项，会弹出一个系统窗口，列出该目录中所有文件，如下图所示。

![](https://images.gitbook.cn/3ca54e00-85d7-11e9-89d9-450409f4603a)

### 答疑与交流

为了让订阅课程的读者更快更好地掌握课程的重要知识点，我们为每个课程配备了课程学习答疑群服务，邀请作者定期答疑，尽可能保障大家学习效果。同时帮助大家克服学习拖延问题！

请添加小助手伽利略微信 GitChatty6，并将支付截图发给她，小助手会拉你进课程学习群。

