在 2019 年 1 月 5 日，Electron 最新的 4.0.1 版发布了，尽管在写作本系列课程内容时，Electron 4.0
稳定版还没有发布，不过经过测试，本课程中的例子在 Electron 4.0 中仍然可以正常使用。Electron 4.0 及以上版本只是修正了一些
bug，同时还加了一些功能（主要增加了事件、一些方法），大的功能并没有增加什么。

![avatar](https://images.gitbook.cn/FhcdSyh54S_XjHCzpfpj-V_P41CJ)

如果希望升级到 Electron 4.0 或更高版本，可以按下面的方式去做。

如果机器上已经安装了 Electron 3.x 或更低版本，不要直接使用下面的代码升级。

    
    
    npm update electron -g
    

例如，机器上安装了 Electron 3.0.1，使用上面的命令只能升级到 Electron 3.1.0，跨大版本的升级，如从 2 升级到 3，或从 3
升级到 4，需要先使用下面的命令卸载 Electron。

    
    
    npm uninstall electron -g
    

然后使用下面的命令重新安装 Electron。

    
    
    npm install electron -g
    

安装完后，输入 electron --version 命令，如果输出 v4.0.1，说明已经安装成功，Good Luck。

