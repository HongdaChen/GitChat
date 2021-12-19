### 包安装和镜像源

先来区分几个小白容易混淆的概念：Python 解释器、PyCharm、Anaconda、Conda 安装、pip 安装。

  * PyCharm 是 Python 常用的集成开发环境，全称 Integrated Development Environment，简称 IDE，它本身无法执行 Python 代码。
  * Python 解释器负责执行 Python 代码。可去 Python 官网下载指定版本的 Python，如常用的 Python 3.7 或 Python 3.8 版本；如果安装过 Anaconda，它里面也包括某版本的 Python 解释器。PyCharm 里可选择配置指定版本的 Python 解释器。
  * Anaconda：组装 Python 常用包和环境在一起，开发者使用 Conda 命令，可以非常方便地安装各种 Python 包。
  * Conda 安装：安装 Anaconda 软件后，能够使用 Conda 命令下载。Anaconda 源，常用的清华、中科大镜像源。Conda 安装不仅能装 Python 相关的包，还能安装 C++ 相关的包。
  * pip 安装：也是一种类似于 Conda 安装的 Python 安装方法，用于从 Python Package Index 安装包的工具，只能安装 Python 相关的包。

**镜像源**

使用 Conda 安装某些包会出现慢或安装失败问题，最有效方法是修改镜像源为国内镜像源。

先查看已经安装过的镜像源，Windows 系统在 CMD 窗口中执行命令：

    
    
    conda config --show
    

查看配置项 channels，如果显示带有 tsinghua，则说明已安装过清华镜像。

    
    
    channels:
    - https://mirrors.tuna.tsinghua.edu.cn/tensorflow/linux/cpu/
    - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/msys2/
    - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
    - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
    - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
    

还可以添加中科大镜像源：

    
    
    conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
    

并设置搜索时显示通道地址：

    
    
    conda config --set show_channel_urls yes
    

确认是否安装镜像源成功，执行 `conda config --show`，找到 channels 值为如下：

    
    
    channels:
      - https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
      - defaults
    

如果要移除镜像源，使用 `conda config --remove channels url 地址`，比如要删除清华的某个镜像，使用以下命令：

    
    
    conda config --remove channels https://mirrors.tuna.tsinghua.edu.cn/tensorflow/linux/cpu/
    

或者使用下面命令删除所有所有镜像源：

    
    
    conda config --remove-key channels
    

举一个安装深度学习库 PyTorch 的例子，安装使用 Conda 方法，命令如下所示：

    
    
    conda install pytorch torchvision cudatoolkit=10.1
    

需要安装的包共有两个，每行后面的链接就是要从清华下载的镜像源。

![image-20200127233201258](https://images.gitbook.cn/2020-02-05-014810.png)

特别注意：不要使用如下命令，多了 `-c pytorch` 后不会再从清华镜像源下载，属于画蛇添足！

    
    
    conda install pytorch torchvision cudatoolkit=10.1 -c pytorch
    

环境搭建和包安装是学习的第一步，千里之行始于足下。

第一步很重要，在后面会单独有一个讨论环境搭建安装的两个案例。

### 搭建 Python 执行环境

Python 的执行环境分为两类：交互环境和调试环境。下面分别介绍两种环境，搭建 Python 的方法。

#### **Jupyter NoteBook**

Jupyter NoteBook 是一个交互式笔记本，提供随时可运行代码的交互式环境。

![1574568966613](https://images.gitbook.cn/2020-02-05-014811.png)

Jupyter NoteBook 是一种“实时”调试工具。每写一行代码，同时按下 `Alt + Enter` 便立即执行此行代码，并得到结果。

安装完 Anaconda 后，Jupyter NoteBook 默认便已安装上。接下来，演示如何入门使用 Jupyter NoteBook。

调出 CMD 命令窗口，并输入 `jupyter NoteBook`，服务启动，并弹出 Jupyter NoteBook 前端界面。

![1574569473435](https://images.gitbook.cn/2020-02-05-014813.png)

点击 New，选择 Python 3 选项，生成第一个 NoteBook。

默认输入第一行代码 `a,b=1,3`，第二行输入文字，并设置为“标题”类型。

![1574569782243](https://images.gitbook.cn/2020-02-05-014814.png)

另起一行输入 `b,a = 1,3`，按下 `Alt + Enter` 执行，实现交换两个元素取值。

如下所示，分别输入 a、b 回车，确认 a、b 交换成功。

![1574570199127](https://images.gitbook.cn/2020-02-05-14815.png)

安装 Jupyter NoteBook 的扩展插件包 jupyter _contrib_ nbextensions。安装后多一个 NBextensions
选项卡，里面有各种插件可供选择。

推荐安装下面 3 个有用的插件：

  * Autopep 8：安装 Autopep 8 库，实现自动格式化代码。
  * Highlighter：选中的文字高亮显示。
  * Hinterland：代码自动提示，无需按下 Tab 键。

#### **IPython**

IPython 是一个增强的可交互式 Python 编程工具。基于它衍生出的 Jupyter NoteBook，其 Python 内核正是
IPython。IPython 提供便捷的 Shell 特性，命令历史查询机制，输出结果缓存功能。

输入 `ipython` 命令，自动弹出窗口。它更加精简，也提供随时可执行代码的交互环境。

![1574581951936](https://images.gitbook.cn/2020-02-05-014815.png)

输入一行命令，可立即 Enter 执行此行：

    
    
    In [3]: a,b=1,3
    In [4]: b,a=1,3
    In [5]: a
    Out[5]: 3
    In [6]: b
    Out[6]: 1
    

提供 Linux 系统 Shell 工具的标准命令，比如查看历史输入命令，使用 `history` 便可查看：

    
    
    In [9]: history
    ?
    a,b=1,3
    b,a=1,3
    a
    b
    ls
    history
    

IPython 的一个主要使用场景，在拿不准某个函数使用时，可立即启动 `ipython` 验证。

比如，忘记 NumPy 的 randint、rand 函数使用时，执行如下代码：

    
    
    In [10]: import numpy as np
    
    In [11]: np.random.randint(2,3)
    Out[11]: 2
    In [12]: np.random.rand(2,3)
    Out[12]:
    array([[0.98391545, 0.40438994, 0.30680514],
           [0.92961344, 0.02093899, 0.62911785]])
    

#### **VS Code**

VS Code 全称为 Visual Studio Code，一款轻量级，但功能强大的源代码编辑器，跨 Windows、macOS 和 Linux。

![1574583831323](https://images.gitbook.cn/2020-02-05-014816.png)

<https://code.visualstudio.com/> 官网下载安装包，大小约为 53M，安装过程非常简单。

VS Code 相比于 Jupyter NoteBook 和 IPython，更适合做代码“调试”工作。

启动 VS Code，左侧为资源管理器，右侧代码编辑界面：

![1574583091789](https://images.gitbook.cn/2020-02-05-014817.png)

其中编辑窗口可为多屏：

![1574583205924](https://images.gitbook.cn/2020-02-05-014818.png)

按下调试虫，再按下 F5，启动代码调试：

![1574583309200](https://images.gitbook.cn/2020-02-05-014819.png)

VS Code 中的调试界面如下：

左侧为当前变量的值，右侧上为调试键，F10 键 step over 调试，F11 键进入 step into 调试，右侧下为断点（F9
为此行增加断点）停靠行：

![1574583655843](https://images.gitbook.cn/2020-02-05-14820.png)

安装插件也非常方便，点击最左侧的插件按钮，输入想安装的插件，目前已安装 14 个插件。

![1574584135450](https://images.gitbook.cn/2020-02-05-014820.png)

安装 Autopep8 插件，实现代码自动格式化；Pyright 插件于 2019 年微软发布的一款 Python
静态类型自动检查的工具，关于此工具的使用方法会在后面章节中详细介绍。

#### **PyCharm**

PyCharm 是一款专业的 Python IDE 工具，官网：

> <https://www.jetbrains.com/pycharm/>

Community 版本完全免费，Professional 版可试用。一般下载 Community 版就够用：

![1574585117692](https://images.gitbook.cn/2020-02-05-014823.png)

安装完成后，新建一个项目，看到 PyCharm 自动生成一个 venv 私有包环境文件夹。

![1574593247566](https://images.gitbook.cn/2020-02-05-14824.png)

下一步，新建一个 Book.py 文件，并输入以上代码。

接下来做 Python 解释器的配置，选择 File，选择 setting。如下，Python 解释器选择为已经安装好的 Anaconda 里的
python.exe：

![1574593412464](https://images.gitbook.cn/2020-02-05-014824.png)

为 Book.py 运行配置参数，如下选择 Edit Configurations：

![1574593639622](https://images.gitbook.cn/2020-02-05-014826.png)

弹出配置界面后，依次选择左侧 `+`，脚本路径（script path）选择 Book.py 所在目录。

Python 解释器选择 Anaconda 下安装的 Python 解释器。如下所示：

![1574593701944](https://images.gitbook.cn/2020-02-05-014829.png)

在指定行打上断点，点击调试按钮后。程序运行到断点处：

![1574593851061](https://images.gitbook.cn/2020-02-05-014831.png)

调试窗口如下，看到各种系统变量的取值：

![1574593876061](https://images.gitbook.cn/2020-02-05-14832.png)

按下 F7 进入 Book 类的 __init__ 函数，观察此时 Book 类的各个变量的取值：

![1574593930332](https://images.gitbook.cn/2020-02-05-014832.png)

如果想在 PyCharm 内安装扩展包，依次选择 File，选择 settings，选择右侧的
`+`，弹出的窗口中输入想要安装的扩展包，比如“tensorflow”，然后点击“install package”后开始安装。

![1574594084081](https://images.gitbook.cn/2020-02-11-143602.png)

以上就是 Python 常见的执行环境搭建过程。

### 小结

今天，一起学习了 ：

  * Python 环境搭建，镜像源安装包；
  * 4 款常用的 Python 开发环境：Jupyter NoteBook、IPython、VS Code、PyCharm。

