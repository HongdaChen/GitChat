接口开发完成并且测试通过后，就可以进行发布，系统发布可以有很多方式，本文将目前主要的发布方式一一列举出来，供大家参考。

### Java 命令行启动

这种方式比较简单，由于 Spring Boot 默认内置了 Tomcat，我们只需要打包成 Jar，即可通过 Java 命令启动 Jar
包，即我们的应用程序。

首先，news 下面的每个子工程都加上（Client 除外）：

    
    
    <packaging>jar</packaging>
    

此表示我们打包成 Jar 包。

其次，我们在每个 Jar 工程（除去 Commmon）的 pom.xml 中都加入以下内容：

    
    
    <build>
            <!-- jar包名字，一般和我们的工程名相同 -->
            <finalName>user</finalName>
            <sourceDirectory>${project.basedir}/src/main/java</sourceDirectory>
            <testSourceDirectory>${project.basedir}/src/test/java</testSourceDirectory>
            <resources>
                <resource>
                    <directory>src/main/resources</directory>
                    <filtering>true</filtering>
                </resource>
            </resources>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <configuration>
                        <fork>true</fork>
                        <mainClass>com.lynn.${project.build.finalName}.Application</mainClass>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>2.5</version>
                    <configuration>
                        <encoding>UTF-8</encoding>
                        <useDefaultDelimiters>true</useDefaultDelimiters>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>2.18.1</version>
                    <configuration>
                        <skipTests>true</skipTests>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <source>1.8</source>
                        <target>1.8</target>
                    </configuration>
                    <executions>
                        <!-- 替换会被 maven 特别处理的 default-compile -->
                        <execution>
                            <id>default-compile</id>
                            <phase>none</phase>
                        </execution>
                        <!-- 替换会被 maven 特别处理的 default-testCompile -->
                        <execution>
                            <id>default-testCompile</id>
                            <phase>none</phase>
                        </execution>
                        <execution>
                            <id>java-compile</id>
                            <phase>compile</phase>
                            <goals> <goal>compile</goal> </goals>
                        </execution>
                        <execution>
                            <id>java-test-compile</id>
                            <phase>test-compile</phase>
                            <goals> <goal>testCompile</goal> </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    

然后执行 `maven clean package` 命名打包：

![enter image description
here](https://images.gitbook.cn/61566270-83e9-11e8-9df7-79b51aa97c78)

![enter image description
here](https://images.gitbook.cn/69173f70-83e9-11e8-81e8-9d5996b3268d)

第一次运行可能需要花点时间，因为需要从 Maven 仓库下载所有依赖包，以后打包就会比较快，等一段时间后，打包完成：

![enter image description
here](https://images.gitbook.cn/a3a27160-83ed-11e8-ab7e-29061e94f1ab)

最后，我们将 Jar 包上传到服务器，依次启动
register.jar、config.jar、gateway.jar、article.jar、comment.jar、index.jar、user.jar
即可，启动命令是：

    
    
    nohup java -server -jar xxx.jar &
    

用 nohup 命令启动 Jar 才能使 Jar 在后台运行，否则 shell 界面退出后，程序会自动退出。

### Tomcat 启动

除了 Spring Boot 自带的 Tomcat，我们同样可以自己安装 Tomcat 来部署。

首先改造工程，将所有 `<packaging>jar</packaging>` 改为 `<packaging>war</packaging>`，去掉内置的
Tomcat：

    
    
    <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
                <scope>provided</scope>
            </dependency>
    

修改 build：

    
    
    <build>
            <!-- 文件名 -->
            <finalName>register</finalName>
            <resources>
                <resource>
                    <directory>src/main/resources</directory>
                    <filtering>true</filtering>
                </resource>
            </resources>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                </plugin>
                <plugin>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>2.5</version>
                    <configuration>
                        <encoding>UTF-8</encoding>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>2.18.1</version>
                    <configuration>
                        <skipTests>true</skipTests>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    

然后修改启动类 Application.java：

    
    
    public class Application extends SpringBootServletInitializer{
    
        public static void main(String[] args) {
            SpringApplication.run(Application.class,args);
        }
    
        @Override
        protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
            return application.sources(Application.class);
        }
    }
    

这样打包后就会生成 War 包，打包方式同上。

我们将 War 上传到服务器的 Tomcat 上即可通过 Tomcat 启动项目。

### Jenkins 自动化部署

我们搭建的是一套微服务架构，真实环境可能有成百上千个工程，如果都这样手动打包、上传、发布，工作量无疑是巨大的。这时，我们就需要考虑自动化部署了。

Jenkins 走进了我们的视野，它是一个开源软件项目，是基于 Java
开发的一种持续集成工具，用于监控持续重复的工作，旨在提供一个开放易用的软件平台，使软件的持续集成变成可能。

下面，我们就来看看如果通过 Jenkins 实现系统的自动化部署。

#### 安装

Jenkins 的安装方式请自行百度，本文不做详细说明。

**注：** 安装好后，至少需要安装 Maven、SSH、Git、SVN 插件。

#### 创建任务

安装好后，访问 Jenkins，登录后，即可看到如下界面：

![enter image description
here](https://images.gitbook.cn/6c202290-83fd-11e8-a776-a97be0899301)

（1）点击系统管理 -> 系统设置，添加服务器 SSH 信息：

![enter image description
here](https://images.gitbook.cn/4f665b00-83fe-11e8-9149-dd330c2a1b31)

（2）点击系统管理 -> 全局工具配置，配置好 JDK 和 Maven：

![enter image description
here](https://images.gitbook.cn/93292840-83fe-11e8-aac1-63307eeea99f)

（3）点击新建任务，输入任务名，选择构建一个 Maven 风格的软件：

![enter image description
here](https://images.gitbook.cn/a5f8f140-83fd-11e8-81e8-9d5996b3268d)

（4）点击确定，进入下一步：

![enter image description
here](https://images.gitbook.cn/b5a0db70-83fe-11e8-ab7e-29061e94f1ab)

这里以 SVN 为例说明（如果代码在 Git上，操作类似），将源码的 SVN 地址、SVN 账号信息依次填入文本框。

![enter image description
here](https://images.gitbook.cn/f6865250-83fe-11e8-889e-a3559e13e7b0)

Build 下面填入 Maven 的构建命令。

在“构建后操作”里按要求填入如图所示内容：

![enter image description
here](https://images.gitbook.cn/4c960780-83ff-11e8-9df7-79b51aa97c78)

其中，启动脚本示例如下：

    
    
    kill -9 $(netstat -tlnp|grep 8080|awk '{print $7}'|awk -F '/' '{print $1}')
    cd /app/hall
    java -server -jar hall.jar &
    

点击保存。

#### 手动构建

任务创建好后，点击“立即构建”即可自动构建并启动我们的应用程序，并且能够实时看到构建日志：

![enter image description
here](https://images.gitbook.cn/aa466780-83ff-11e8-ab7e-29061e94f1ab)

#### 自动构建

我们每次都手动点击“立即构建”也挺麻烦，程序猿的最高进阶是看谁更懒，我都不想点那个按钮了，就想我提交了代码能自动构建，怎么做呢？很简单，进入任务配置界面，找到构建触发器选项：

![enter image description
here](https://images.gitbook.cn/0c6ee220-8400-11e8-9df7-79b51aa97c78)

保存后，Jenkins 会每隔两分钟对比一下 SVN，如果有改动，则自动构建。

### 总结

系统发布方式很多，我们可以根据自身项目特点选择适合自己的方式，当然还有很多方式，比如 K8S、Docker 等等，这里就不再赘述了 ，关于
K8S+Docker 的方式，我会在第20课讲解。

