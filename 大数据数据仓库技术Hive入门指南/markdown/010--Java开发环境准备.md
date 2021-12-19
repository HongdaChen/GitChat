为了之后Hive用户自定义函数（UDF）的学习，首先需要配置下Java环境，然后安装IDE工具。

## 语言环境安装

首先是Java语言环境的安装。

### 安装流程

Windows环境和Linux环境的安装步骤不同。

对于Windows环境，步骤如下：

    
    
    1. JDK1.8下载安装
    
    2. 配置环境变量
    
    3. 验证是否安装成功
    

但现在windows版的JDK安装好后，自动会配置环境变量，所以可以省略第2步。

对于Linux环境，步骤如下：

    
    
    1. JDK1.8下载、解压
    
    2. 配置环境变量
    
    3. 验证是否安装成功
    

本专栏主要以Windows环境配置为主

### Windows平台安装步骤

  1. JDK安装，首先去官网下载Windows版的JDK（版本： JDK1.8）https://www.oracle.com/technetwork/java/javase/downloads/index.html，在Windows平台安装时，使用的是可视化安装，这里不需要讲解太多。
  2. 配置环境变量（默认会配置，可以跳过，但如果之后的步骤执行错误，则需要手动配置）

①打开环境变量配置

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/c6b8eb5bc44e6b07175874e607b13f95.png#pic_center)

②新建JAVA _HOME环境变量，JAVA_ HOME配置的值为JDK安装路径

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/68698546783a73f01b7e3a13a296b6da.png#pic_center)

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/792cec748530e7404758453606d0a96a.png#pic_center)

③配置PATH：在PATH中添加如下路径

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/4e7afe348e6c076ca674882a8e6c2e22.png#pic_center)

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/eb502f482fab2917579cee426b79923e.png#pic_center)

  1. 测试安装是否成功：运行CMD打开命令行窗口，执行java –version

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/2ea72a1ea7f0ce868864119f432a081b.png#pic_center)

### Linux平台安装步骤

  1. JDK下载、解压
  2. 配置环境变量，将以下命令中的{path to java}更改为JDK的安装目录

    
    
    # 编辑配置文件
    vi /etc/profile
    # 在末尾添加
    export JAVA_HOME={path to java}
    export Path=$Path:$JAVA_HOME/bin
    # 使环境变量生效
    source /etc/profile
    

  1. 测试安装是否成功

    
    
    java –version
    

## Hive开发Jar包获取

Hive开发Jar包可以使用Maven进行管理，也可以直接使用Hive安装目录下提供的jar包。专栏以直接导入jar包的方式为主，有Maven使用经验的直接导入以下依赖。

    
    
    <!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-client -->
    <dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>2.7.7</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.hive/hive-exec -->
    <dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-exec</artifactId>
    <version>2.3.7</version>
    </dependency>
    

Hive安装目录下的lib目录下会提供开发所需的Jar包，使用脚本安装的话为/opt/app/apache-
hive-2.3.7-bin/lib目录。找到hive-exec-2.3.7.jar。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/e8ad2607dd77815f951b05e88496fcaf.png#pic_center)

除了Hive的开发Jar包之外，还需要依赖Hadoop公共开发包。这些Jar包，可以在Hadoop安装目录下的share/hadoop/common目录下获取。将hadoop-
common-2.7.7.jar和其依赖的lib下的所有jar包下载到本地。

其中lib目录下的jar包较多，可以使用zip命令打包后再进行下载：zip file.zip ./*

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/52ade054203595bd3e1388b5e3a9b795.png#pic_center)

如果使用的是XShell，那么可以安装lrzsz包，使用sz [path]命令将文件下载到本地，如果是上传文件，则使用的是rz命令。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/0966582368c3ea49ea991be5dc91a5d1.png#pic_center)

所依赖的Jar包，可以按照上述步骤，自行从集群安装目录进行拷贝，当然也可以直接从网盘下载，链接下附：

    
    
    链接：https://pan.baidu.com/s/1XpZOVpff53ACloKb9sk9Yg
    提取码：ppkm
    复制这段内容后打开百度网盘手机App，操作更方便哦
    

## 开发工具

### 开发工具介绍

主流的Java开发工具有：

  1. IntelliJ IDEA：IDEA是JVM企业开发使用最多的IDE工具，为最大化的开发效率而设计。提供静态代码检查和沉浸式编程体验
  2. Eclipes：免费的Java开发工具，几乎是每个Java入门初学者的必备
  3. 其他：Subline Text、Atom、VS Code

接下来完成IntelliJ IDEA的安装。

### IntelliJ IDEA安装&项目创建

开发工具IntelliJ
IDEA安装，首先进入到[官网](https://www.jetbrains.com/idea/download/#section=windows)下载并进行可视化安装。

  1. 安装好之后，创建第一个项目

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/2d62c816a35814ca53d12b5586e1d74e.png#pic_center)

  1. 创建Java项目，添加JDK

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/044703f83e610a2825b8fc96f963125a.png#pic_center)

![图片](https://uploader.shimo.im/f/C97KOK20ZuMJHLd0.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxNX0.jUS2w7nKys8gjuu1Mh_f2IS_O7TsFT2DVL8BFQeWGKA&fileGuid=FXlnqFRZklstjeZ5)

![图片](https://uploader.shimo.im/f/i0Tdwd1y5VTGcwDc.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxNX0.jUS2w7nKys8gjuu1Mh_f2IS_O7TsFT2DVL8BFQeWGKA&fileGuid=FXlnqFRZklstjeZ5)

![图片](https://uploader.shimo.im/f/3OLj3WS2ftuIZ06O.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxNX0.jUS2w7nKys8gjuu1Mh_f2IS_O7TsFT2DVL8BFQeWGKA&fileGuid=FXlnqFRZklstjeZ5)

![图片](https://uploader.shimo.im/f/kEMyvOTKa55qTYeb.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxNn0.OFg3yolT7X5azi5SSiK81r4wj8Gyls7zshCTGmJv1Hw&fileGuid=FXlnqFRZklstjeZ5)

  1. 导入必备Jar包

![图片](https://uploader.shimo.im/f/crOsObvED1nQqDVQ.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxNn0.OFg3yolT7X5azi5SSiK81r4wj8Gyls7zshCTGmJv1Hw&fileGuid=FXlnqFRZklstjeZ5)

![图片](https://uploader.shimo.im/f/wTi93Ll6vwIXLZPO.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxNn0.OFg3yolT7X5azi5SSiK81r4wj8Gyls7zshCTGmJv1Hw&fileGuid=FXlnqFRZklstjeZ5)

![图片](https://uploader.shimo.im/f/r6xUDANgU6gwuLTH.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxN30.Q5Ho1PRvnVImk0k4VXP7bx7tPa9MiSidATJAmhIQDwU&fileGuid=FXlnqFRZklstjeZ5)

  1. 创建Java Class

![图片](https://uploader.shimo.im/f/7AnG9pXLVrjHI9Gi.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxN30.Q5Ho1PRvnVImk0k4VXP7bx7tPa9MiSidATJAmhIQDwU&fileGuid=FXlnqFRZklstjeZ5)

![图片](https://uploader.shimo.im/f/1RMpPCVeCPJoFh8r.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxN30.Q5Ho1PRvnVImk0k4VXP7bx7tPa9MiSidATJAmhIQDwU&fileGuid=FXlnqFRZklstjeZ5)

  1. 编写Hello Word代码，并运行，测试是否成功

    
    
    public class Test {
    public static void main(String[] args) {
    System.out.println("Hello World");
    }
    }
    

![图片](https://uploader.shimo.im/f/bUvkz01OjIiHMIhO.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxN30.Q5Ho1PRvnVImk0k4VXP7bx7tPa9MiSidATJAmhIQDwU&fileGuid=FXlnqFRZklstjeZ5)

![图片](https://uploader.shimo.im/f/JgXbVQr9rKK0akaH.png!thumbnail?download=1&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHQiOiJleHBvcnQiLCJ1c2VySWQiOjAsImV4cCI6MTYyMDIzMTAxN30.Q5Ho1PRvnVImk0k4VXP7bx7tPa9MiSidATJAmhIQDwU&fileGuid=FXlnqFRZklstjeZ5)

