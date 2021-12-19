想学机器学习或 Web 前端，然后下载的包怎么都装不上，这是最让人头疼的事。下面通过两个实际的较复杂的安装包的案例，介绍安装 Python 包的常见问题。

掌握下面这两个包的安装方法后，其他包安装可能都不是问题。

### 实际案例一：安装 PyTorch

这是安装 Python 包时最常见的问题之一，如何解决？又有哪些注意事项？

#### **安装包时，慢到装不上**

无论 pip 还是 Conda 安装 PyTorch 都太慢，都是安装官方文档去做的，就是超时装不上。

无法开展下一步，卡脖子的感觉太不好受。

按照 PyTorch 官档提示，选择好后，完整复制上面命令：

    
    
    conda install pytorch torchvision cudatoolkit=10.1 -c pytorch
    

到 CMD 中，系统是 Windows。

![](https://images.gitbook.cn/2020-02-05-015000.png)

接下来提示，Conda 需要安装的包，选择 y。

继续安装，但是在安装时，发现进度条几乎一动不动。反复尝试，就是这样。

有些无奈，还感叹怎么深度学习的路一开始就这么难！

#### **解决方法**

无论是安装 CPU 版还是 CUDA 版，网上关于这些的参考资料太多。

无非就是 CUDA 硬件和 CUDA 开发包的版本要对应、Python 版本要对应等，这些问题基本都容易解决。

棘手的还是，如何解决慢到无法装的问题。

最有效方法是，添加镜像源，常见的是来自清华或中科大。

先查看，是否已经安装相关镜像源，Windows 系统在 CMD 窗口中，执行命令：

    
    
    conda config --show
    

如果显示为如下：

    
    
    channels:
      - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
      - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/menpo/
      - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/bioconda/
      - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/msys2/
      - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
      - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
      - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
    

说明已经安装好清华的镜像源。

如果没有安装，请参考下面命令，安装清华的源：

    
    
    conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
    

依次安装上面所有的源。设置搜索时显示通道地址，执行下面命令：

    
    
    conda config --set show_channel_urls yes
    

#### **最关键一步**

安装好镜像源后，使用下面命令，安装为什么还是龟速？

    
    
    conda install pytorch torchvision cudatoolkit=10.1 -c pytorch
    

命令最后是 `-c pytorch`，这样默认还是从 Conda 源下载。所以，新安装的清华等源都没有用上。

正确命令：

    
    
    conda install pytorch torchvision cudatoolkit=10.1
    

也就是去掉 `-c pytorch`。在安装时，可以看到使用清华源。同时，安装速度直线提升，顺利完成。

#### **测试是否安装成功**

结合官档，执行下面代码：

    
    
    torch.cuda.is_available()
    

返回 True，说明安装 CUDA 成功。

    
    
    In [1]: import torch
    
    In [2]: torch.cuda
    Out[2]: <module 'torch.cuda' from 'D:\\Programs\\anaconda\\lib\\site-packages\\torch\\cuda\\__init__.py'>
    
    In [3]: torch.cuda.is_available()
    Out[3]: True
    
    In [4]: from __future__ import print_function
    
    In [5]: x = torch.rand(5,3)
    
    In [6]: print(x)
    tensor([[0.0604, 0.1135, 0.2656],
            [0.5353, 0.9246, 0.3004],
            [0.4872, 0.9592, 0.2215],
            [0.2598, 0.5031, 0.6093],
            [0.2986, 0.1599, 0.5862]])
    

### 实际案例二：安装 TensorFlow

Windows 上安装 TensorFlow-gpu 版，这个安装案例遇到了各种常见问题。该如何一个一个击破这些问题？

#### **确认 GPU 型号**

  * 右键点击桌面上的“此电脑”图标，在弹出菜单中选择“属性”菜单项；
  * 点击左侧边栏的“设备管理器”菜单项，找到“显卡适配器”菜单项；
  * 点击前面的展开按钮，就可以看到电脑中安装的显卡驱动程序；
  * 右键查看的显卡驱动程序，选择“属性”菜单项；
  * 点击详细信息”标签”。

#### **方法一：pip 安装**

安装 CUDA Toolkit + cuDNN。注意，需要和电脑的 GPU 的型号匹配：

> <https://developer.nvidia.com/cuda-toolkit-archive>

![](https://images.gitbook.cn/df9c40a0-5552-11ea-ad74-0783a5f0ad5e)

GTX 1060 型号，根据官网上的提示，对应找到合适的 CUDA 和 cuDNN。

分别是 Toolkit CUDA 9 版本、cuDNN 7 版本。

进入下载界面，选择匹配的版本之后，点击下载：

![](https://images.gitbook.cn/addff190-5554-11ea-8ccf-d385e6cc64ec)

下载 cuDnn7.0，需要在 Nvidia 上注册账号，使用邮箱注册就可以，免费。

登陆账号后可以下载。cuDNN 网址下载：

> <https://developer.nvidia.com/rdp/cudnn-archive>

![](https://images.gitbook.cn/fbdf2320-5554-11ea-987f-4debc547abef)

完成下载 CUDAToolkit 9.0 和 cuDNN 7.0 后，下面开始安装。

![](https://images.gitbook.cn/2020-02-05-115536.png)

安装失败的可能一种情况：已经安装 nvidia 显卡驱动，再安装 CUDAToolkit 时，会因二者版本不兼容，导致 CUDA 无法正常使用。

安装之前，需要先卸载显卡驱动。

![](https://images.gitbook.cn/2020-02-05-115538.png)

这一步，检测显卡是否支持 CUDA：

![](https://images.gitbook.cn/2020-02-05-115539.png)

![](https://images.gitbook.cn/2020-02-05-115541.png)

按照提示，一步步通过，即可安装成功。

![](https://images.gitbook.cn/2020-02-05-115542.png)

将这三个文件，拷贝到 CUDA 安装的根目录下，替换掉原始的文件。

![](https://images.gitbook.cn/2020-02-05-115543.png)

将这四个路径添加到环境变量中去，一般是默认路径安装。如果不是默认路径的，要找到对应的路径再添加。

![](https://images.gitbook.cn/2020-02-05-115544.png)

使用 `pip install tensorflow-gpu` 安装，然后，测试是否成功安装 CUDA：

![](https://images.gitbook.cn/2020-02-05-115545.png)

安装成功！

#### **方法二：Conda 安装**

  * 第二种方法：conda install tensorflow-gpu
  * 环境：Anaconda

**安装 Anaconda**

装好 Anaconda 后，然后将 conda install tensorflow-gpu 根据
[Anaconda](https://www.anaconda.com/tensorflow-in-anaconda/) 官网的提示，当使用 conda
通过命令 `conda installtensorflow-gpu` 安装 GPU 加速版本的 TensorFlow 时，这些库会自动安装。其版本已知与
tensorflow-gpu 软件包兼容。

此外，Conda 将这些库安装到一个位置，这个位置中它们不会干扰可能通过其他方法安装的这些库的其他实例。

Conda 一下，经过了一段漫长时间的等待，终于安装完成。预想应该成功，但是报错了，报错提示：

    
    
    tensorflow.python.framework.errors_impl.InternalError:cudaGetDevice() failed. 
    Status: CUDA driver version is insufficient for CUDAruntime version
    

原来，CUDA 驱动版本对于 CUDA 运行时版本，是不够的。既然是版本不够，那么就：

    
    
    conda install cudatoolkit==9.0
    

成功！

如果在使用 tensorflow-gpu 版本运行代码的时候，出现：

    
    
    Blas GEMM launch failed
    

通过设定 config 为使用的显存按需自动增长，避免显存被耗尽，可进行有效的预防。

> <https://blog.csdn.net/feixiang7701/article/details/81515447>

### 小结

今天学习了两个安装 Python 包的案例，这是比较难以安装的 Python 包。

了解这两个案例后，相信即便是新手，以后再安装 Python 包也会得心应手。

