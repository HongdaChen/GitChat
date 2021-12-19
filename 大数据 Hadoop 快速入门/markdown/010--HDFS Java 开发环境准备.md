为了之后 HDFS Java API 的学习，首先需要配置下 Java 环境，然后安装 IDE 工具。

### 语言环境安装

首先是 Java 语言环境的安装。

#### **安装流程**

Windows 环境和 Linux 环境的安装步骤不同。

对于 Windows 环境，步骤如下：

  1. JDK 1.8 下载安装
  2. 配置环境变量
  3. 验证是否安装成功

但现在 windows 版的 JDK 安装好后，自动会配置环境变量，所以可以省略第 2 步。

对于 Linux 环境，步骤如下：

  1. JDK 1.8 下载、解压
  2. 配置环境变量
  3. 验证是否安装成功

本专栏主要以 Windows 环境配置为主。

#### **Windows 平台安装步骤**

1\. JDK 安装，首先去官网下载 Windows 版的 JDK（版本为 JDK 1.8）：

> <https://www.oracle.com/technetwork/java/javase/downloads/index.html>

在 Windows 平台安装时，使用的是可视化安装，这里不需要讲解太多。

2\. 配置环境变量（默认会配置，可以跳过，但如果之后的步骤执行错误，则需要手动配置）

打开环境变量配置：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012071125.png)

新建 JAVA_HOME 环境变量，JAVA_HOME 配置的值为 JDK 安装路径：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012071143.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012071151.png)

配置 PATH：在 PATH 中添加如下路径。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012071225.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012071233.png)

3\. 测试安装是否成功：运行 CMD 打开命令行窗口，执行

    
    
    java –version
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012071335.png)

#### **Linux 平台安装步骤**

1\. JDK 下载、解压。

2\. 配置环境变量，将以下命令中的 `{path to java}` 更改为 JDK 的安装目录：

    
    
    # 编辑配置文件
    vi /etc/profile
    # 在末尾添加
    export JAVA_HOME={path to java}
    export Path=$Path:$JAVA_HOME/bin
    # 使环境变量生效
    source /etc/profile
    

3\. 测试安装是否成功：

    
    
    java –version
    

### HDFS 开发 jar 包获取

HDFS 开发 jar 包可以使用 Maven 进行管理，也可以直接使用 Haoop 安装目录下提供的 jar 包。专栏以直接导入 jar 包的方式为主，有
Maven 使用经验的直接导入以下依赖。

    
    
    <!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-client -->
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-client</artifactId>
        <version>2.7.7</version>
    </dependency>
    

Hadoop 安装目录下的 share/hadoop 目录下会提供开发者所需的 jar 包，使用脚本安装的话为
/opt/app/hadoop-2.7.7/share/hadoop 目录。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023042705.png)

可以自行从 Hadoop 安装本地进行拷贝，当然也可以直接从网盘下载，链接下附：

> <https://pan.baidu.com/s/14RexcaFbgA5hLVe2iYCMqw>
>
> 提取码：a6d1

从 Hadoop 安装本地的拷贝过程是：首先需要将 common 目录下的 hadoop-common-2.7.7.jar 和其依赖的 lib 下的所有
jar 包下载到本地。

其中 lib 目录下的 jar 包较多，可以使用 zip 命令打包后再进行下载：

    
    
    zip file.zip ./*
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023042821.png)

如果使用的是 XShell，那么可以安装 lrzsz 包，使用 `sz [path]` 命令将文件下载到本地，如果是上传文件，则使用的是 rz 命令。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023043253.png)

然后，将 /opt/app/hadoop-2.7.7/share/hadoop/hdfs 目录下的 hadoop-hdfs-2.7.7.jar
包也下载到本地。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023043422.png)

此时，使用 Java 开发 HDFS 的所需 jar 包就已经获取完成。

当然，需要注意的是，在 Windows 平台运行 HDFS，需要额外的一些文件。将压缩包中的 hadoop.dll 和 winutils.exe 文件拷贝到
c:\windows\system32 目录中即可。

链接：

> <https://pan.baidu.com/s/10DJzC_341ILTb_Y6EshiVw>
>
> 提取码：pun1

### 开发工具

#### **开发工具介绍**

主流的 Java 开发工具有：

  * IntelliJ IDEA：IDEA 是 JVM 企业开发使用最多的 IDE 工具，为最大化的开发效率而设计。提供静态代码检查和沉浸式编程体验。
  * Eclipes：免费的 Java 开发工具，几乎是每个 Java 入门初学者的必备。
  * 其他：Subline Text、Atom、VS Code。

接下来完成 IntelliJ IDEA 的安装。

#### **IntelliJ IDEA 安装 &项目创建**

开发工具 IntelliJ IDEA
安装，首先进入到[官网](https://www.jetbrains.com/idea/download/#section=windows)下载并进行可视化安装。

1\. 安装好之后，创建第一个项目：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012072114.png)

2\. 创建 Java 项目，添加 JDK：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201021073456.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201021073614.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201021073702.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201021073732.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201021073827.png)

3\. 导入必备 jar 包：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012072330.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012072339.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023043905.png)

4\. 创建 Java Class：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012072408.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012072429.png)

4\. 编写 Hello Word 代码，并运行，测试是否成功。

    
    
    public class Test {
        public static void main(String[] args) {
            System.out.println(“Hello World”);
        }
    }
    

