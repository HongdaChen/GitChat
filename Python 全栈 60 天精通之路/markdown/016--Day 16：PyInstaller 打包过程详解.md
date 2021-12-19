### 介绍

PyInstaller 能将一个应用程序打包为独立可执行的文件，比如 Windows 下打包为 exe 文件，详情可参考：

> <http://www.pyinstaller.org/>

![1574592883830](https://images.gitbook.cn/2020-02-14-080721.png)

> PyInstaller is a program that **freezes (packages) Python programs** into
> **stand-alone executables, under Windows, Linux, Mac OS X, FreeBSD, Solaris
> and AIX**.

PyInstaller 相比于同类的优势：

  * 支持 Python 2.7、Python 3.3-3.6；
  * 生成的可执行文件字节数更小；
  * 对第三方包的支持非常好，只需要将它们放到 Python 的解释器对应的文件夹中，PyInstaller 便可自动打包到最终生成的可执行文件中。

> As an example, libraries like PyQt, Django or matplotlib are fully
> supported, without having to handle plugins or external data files manually.
> Check our compatibility list of Supported Packages for details.

### 安装 PyInstaller

使用下面命令：

    
    
    PyInstaller from PyPI
    

以上是官网给出的安装方式，pip 安装会更简捷，因为它会自动安装 PyInstaller 的第三方库地依赖。

但是笔者在安装时，不是走的这种方式，而是下载 PyInstaller 的源文件：

> <http://www.pyinstaller.org/downloads.html>

命令行界面中 cd 到 PyInstaller 的目录下，执行：

    
    
    python seteup.py install
    

应用这种方式的需要自行先下载安装 pywin32 库，需要注意它的版本一定要与 Python 的版本一致，两方面：

  * Python 版本
  * Python 是 32 位还是 64 位

如果 pywin32 的版本与 Python 不一致，不会安装成功。总结，安装 PyInstaller 推荐使用 pip 安装方法。

### PyInstaller 打包

打包最重要的一步，也是第一步，梳理程序用到的第三方库有哪些，比如用到了：

    
    
    numpy，
    pandas，
    matplotlib
    xlrd
    

一定要确保程序用到的 Python 解释器所在的物理安装路径下，在 site-packages 文件夹下有了以上这些库，并且要与自己的程序用到的一致。

如果做不好，打包会提示找不到第三方库的引用等。

第二步，将自己的程序代码放到 PyInstaller 的源文件根目录下。

第三步，执行以下命令：

    
    
    pyinstaller yourprogram.py
    

说明：如果想打包不带命令窗口，前面加参数：

    
    
    pyinstaller -w -F yourprogram.py
    

参数意义解释：

  * -w：去掉命令窗口
  * -F：打包成一个可执行文件

### 预置的文件如何发布

程序代码中往往使用一些提前预置的文件，比如窗口图片、配置文件等，那么如何将这些文件发布出来呢。

笔者使用的方法是将这些文件 copy 到最终生成的可执行文件目录下，按照自己想要的文件系统组织。

注意这种方法系统中 **不能出现绝对路径** 。

### 其他问题

打包过程中，如果出现问题，需要首先知道问题是什么，因此，建议使用命令中不要带有 `-w`，这样可以看到命令窗口中的错误，等完全测试好了后，再添加 `-w`。

遇到的一个问题：

![image-20200205165206662](https://images.gitbook.cn/2020-02-14-080730.png)

解决方法：在 Python 解释器文件目录

    
    
    Python36-32\Lib\site-packages\PyInstaller-3.3+4e8e0ff7a-py3.6.egg\PyInstaller\hooks
    

下添加一个 hook-pandas.py 文件：

    
    
     hiddenimports=[
       \#all your previous hidden imports
       'pandas', 'pandas._libs.tslibs.timedeltas'
     ]
    

以上，便是 PyInstaller 的完整打包过程。

### 小结

使用 PyInstaller 可打包 Python 文件为 .exe
文件发布部署，今天与大家一起学习这个打包过程，包括里面常见的问题，比如图片显示不出来，打包时提示某个包找不到，这些问题都能在这篇文章中找到解决方法。

