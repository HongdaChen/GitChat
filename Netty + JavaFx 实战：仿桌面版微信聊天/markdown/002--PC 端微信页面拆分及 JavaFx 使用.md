## 一、前言

对于一个知识点的学习过程来说，往往使用自己熟悉的工具或方式才更易于上手。因为所有同类型的知识点在抛出复杂的流程拨云见日后，所能得到的几乎都是同样的设计思想和实现理论。

在 Java 语言桌面版开发中，直至目前共提供了三套 UI 开发方式；Awt、Swing、JavaFx，以及一些扩展组件 SWT 等。在这三套 UI
组件中，JavaFx 是最新也是最为好用的，因为他提供了丰富的功能，以及 XML 定义、CSS 设计，因此这也是我们这次选择 JavaFx 开发 UI
的原因。以下是给出的对比图；

UI 组件 | 比喻 | 产品 | 描述  
---|---|---|---  
awt | 石头 | 早期的 Eclipse(不过后来人家优化成 SWT 了) | SUN 在 1996 年推出的 UI
框架，整体框架较重，不适合作为桌面开发的解决方案  
swing | 80 年代文具小刀 | JetBrain | 更轻、更快，更加丰富，但 swing 也有毛病，不过都是可以解决和回避的，并且有很多成熟的方案  
javaFx | 地摊军刀 | 暂无 | 功能强大、简单易用，也是官网推荐的，但是目前使用的人不多，所以遇到问题较难弄
(主要还是来的太晚)。文档；[http://javafxchina.net/main](http://javafxchina.net/main/)  
  
* * *

那么仿照 PC
端微信界面开发，我们需要分析下这个窗体的框架结构，以方便我们进行整体功能的设计。例如，哪一块区域是被反复填充隐藏和展现的、哪一块功能是需要拆解实现的等等。接下来我们就开始分析
UI 界面，为后续使用 JavaFx 开发做准备。

## 二、PC 端微信界面拆分

在本次 UI 中需要实现的核心功能包括；登陆、对话栏、好友栏，所以我们会着重分析这部分内容，以方便我们使用 JavaFx 实现 UI。

### 1、登陆主窗体

PC 端微信的登陆是通过手机扫描二维码的方式进行验证登陆，而我们这部分是没用手机端做验证的，所以需要一个类似 QQ 一样的用户名、密码登陆框。

#### 1.1、登陆窗体分析

![0m9jC8](https://images.gitbook.cn/0m9jC8)

### 2、会话主窗体

会话主窗体包括；对话框体、好友框体，这部分窗体内容我们仿照 PC 端微信进行拆解分析，如下；

#### 2.1、对话框体拆解分析

![l1YEd8](https://images.gitbook.cn/l1YEd8)

#### 2.2、好友框体拆解分析

![evxdfh](https://images.gitbook.cn/evxdfh)

综上，是我们对 UI 部分的拆解分析，主要部分包括了；登陆、对话框选中会话 (好友、群组)、会话消息展示 (个人、好友)、搜索添加好友、选中好友会话等。

## 三、JavaFx 使用

在本文的最开始我们对 JavaFx 进行了简单的介绍，在通过分析 PC 端微信界面 UI
后，我们大概可以知道实现这些页面会用到哪些组件。以下我们将列出核心的组件、事件、方法和 CSS，以方便后续编码开发时使用。

### 1、组件

序号 | 名称 (En) | 名称 (Ch) | 功能  
---|---|---|---  
1 | Pane | 面板 | 用于承载其他元素  
2 | Label | 标签 | 可以添加文字、图片等内容  
3 | TextField | 明文输入框 | 单行文字内容  
4 | PasswordField | 密文输入框 | 单行密码内容  
5 | TextArea | 多行输入框 | 会话内容编辑框  
6 | Button | 按钮 | 登陆、发送消息、退出等  
7 | ListView | 列表 | 好友栏、对话栏，也可以 ListView 里面填充 ListView，这个用在好友栏多种内容的实现  
  
### 2、事件

序号 | 名称 | 功能 | 用途  
---|---|---|---  
1 | setOnAction | Button 按钮类点击事件 | 登陆、最小化、退出等  
2 | setOnMouseEntered | 鼠标进入 | 变换背景色  
3 | setOnMouseExited | 鼠标移除 | 恢复背景色  
4 | setOnMousePressed | 鼠标摁下 | 发送消息 (S)  
5 | setOnMouseClicked | 鼠标点击 | 删除对话框中元素  
5 | setOnKeyPressed | 键盘事件 | 登陆、发送消息和一些快捷键操作  
  
### 3、方法

序号 | 名称 | 功能 | 用途  
---|---|---|---  
1 | setPrefSize | 设置元素宽、高 | 基本用途  
2 | setLayoutX | 设置位置，X 轴 | 基本用途  
3 | setLayoutY | 设置位置，Y 轴 | 基本用途  
4 | setText | 设置文本 | 基本用途  
5 | children.add | 添加元素 | 基本用途  
6 | getStyleClass().add | css 样式类 | 预定一些 CSS 样式，也可以在过程中更改  
7 | setStyle | 单个 CSS 样式 | 更好头像常用  
8 | setUserData | 设置数据 (对象) | 这个方法很 **重要** ，在初始化元素时候可以填充数据，在事件操作时候可以取出数据  
9 | ViewList.getSelectionModel().getSelectedItem() | 获取选中元素 | 列表选中判断  
10 | ViewList.getSelectionModel().select(talkElement) | 选中某个元素 |
从好友点击发送消息时跳转到会话框需要选中  
11 | ViewList.getItems().get(0) | 获取某个元素 | 用在元素比对时  
12 | ViewList.scrollTo(left) | 滚动条位置 | 发送消息滚动条自动滚动，这是一种定位滚动方法  
13 | setVisible | true/false 是否可见设置 | 用在展示类元素 / 面板的切换  
14 | ViewList.getItems().clear() | 清空列表框 | 好友、会话、消息等清空操作  
15 | listView.getSelectionModel().clearSelection() | 清空选中元素 | ListView 中嵌套
ListView，需要做清空选中操作  
16 | lookup | 查找元素 | 从某个元素、面板或者主面板中寻找某个 id 的元素  
  
### 4、样式 (CSS)

序号 | 名称 | 功能 | 用途  
---|---|---|---  
1 | @import | 引入 CSS | 为了设计的干净，我们需要将非同类的 css 单独写文件再引入  
2 | -fx-background-color | 背景色 | 基本用途，另外属性；transparent 透明色  
3 | -fx-background-image: url("/fxml/chat/img/system/more_0.png") | 背景图 | 基本用途  
4 | -fx-background-radius:1px | 背景圆角 | 设置圆角图时使用  
5 | -fx-padding | 内框距离位置，顺时针；上、右、下、左 | 基本用途  
6 | -fx-border-color: rgb(180,180,180); | 边框颜色 | 基本用途  
7 | -fx-border-width: 1px; | 边框大小也就是宽度 | 基本用途  
8 | -fx-border-radius: 4px; | 边框圆角度，100 会成一个圆 | 基本用途  
9 | -fx-alignment: center; | 居中；center-right、center-left | 文字居中操作  
10 | -fx-font-family: "微软雅黑"; | 字体，这里可以选择很多 | 基本用途  
11 | -fx-text-fill: red; | 字体颜色 | 基本用途  
12 | -fx-font-size: 12px; | 字体大小 | 基本用途  
13 | -fx-overflow: hidden; | 越界隐藏 | 基本用途  
14 | -fx-text-overflow: ellipsis; | 越界展示点点点 | 基本用途  
15 | -fx-text-align: left; | 文字位置；居左、居右、居中 | 基本用途  
16 | derive(#c7c6c6, 10%); | 颜色透明度 | 基本用途  
17 | :pressed、:hover | 事件类 CSS | 基本用途  
  
### 5、JavaFx Demo

JavaFx 的初始化工程非常简单，按照如下操作即可。

#### 5.1、环境配置

  1. JDK 1.8
  2. IntelliJ IDEA 2018 开源版，(你可以使用自己的版本)

#### 5.2、创建工程

模板情况下 JavaFx 的创建非常简单，如下图；

![](https://images.gitbook.cn/OUpnFi)

#### 5.3、工程结构

    
    
    itstack-test-01
    └── src
        └── main
            └── java
                └── sample
                    ├── Controller.java    
                    ├── Main.java    
                    └── sample.fxml
    

#### 5.4、代码简述

> Controller.java & 控制操作，默认生成代码为空
    
    
    

> Main.java & 主窗体页面
    
    
    public class Main extends Application {
    
        @Override
        public void start(Stage primaryStage) throws Exception{Parent root = FXMLLoader.load(getClass().getResource("sample.fxml"));
            primaryStage.setTitle("Hello World");
            primaryStage.setScene(new Scene(root, 300, 275));
            primaryStage.show();}
    
        public static void main(String[] args) {launch(args);
        }
    
    }
    

> sample.fxml & 配置
    
    
    <?import javafx.geometry.Insets?>
    <?import javafx.scene.layout.GridPane?>
    
    <?import javafx.scene.control.Button?>
    <?import javafx.scene.control.Label?>
    <GridPane fx:controller="sample.Controller"
              xmlns:fx="http://javafx.com/fxml" alignment="center" hgap="10" vgap="10">
    </GridPane>
    

#### 5.5、效果演示

**点击运行 Main**

![](https://images.gitbook.cn/xJlLGu)

**好** ！不出意外你已经看到了一个空白的窗体，就像一个空白的娃娃。

### 6、Javafx Maven Demo

可以看到上面的工程已经很简单，但在实际使用中如果将业务代码与页面逻辑都放在一起写，对于小工程来说还好，但是如果是随着项目不断迭代不断拓展的话，那么将越来越难以维护。为此我们需要将工程
UI 独立出来，用 Maven 的方式进行管理。以下是我们的一个 Maven 管理的 Demo，为后续我们开发整体的页面打下基础。

#### 6.1、环境配置

  1. JDK 1.8
  2. IntelliJ IDEA 2018 开源版，(你可以使用自己的版本)

#### 6.2、创建 Maven 工程

Maven 工程如我们正常开发的一样，点击创建如下图；

![](https://images.gitbook.cn/N2j9so)

#### 6.3、工程结构

    
    
    itstack-test-02
    └── src
        ├── main
        │   ├── java
        │   │   └── org.itstack.test
        │   │       └── Application.java
        │   └── resources    
        │       └── fxml.demo
        │           ├── css
        │           │   └── demo.css    
        │           ├── img        
        │           │   └── tortoise.png    
        │           └── demo.fxml
        └── test
            └── java
                └── org.itstack.test
                    └── ApiTest.java
    

#### 6.4、代码简述

> Application.java & 与上面例子 Main.java 类似
    
    
    public class Application extends javafx.application.Application{
    
        @Override
        public void start(Stage primaryStage) throws Exception {Parent root = FXMLLoader.load(getClass().getResource("/fxml/demo/demo.fxml"));
            primaryStage.setTitle("Hello World");
            primaryStage.setScene(new Scene(root, 300, 275));
            primaryStage.show();}
    
        public static void main(String[] args) {launch(args);
        }
    }
    

  * 这里的 fxml 资源从 resources 中获取
  * 后续这个方法类只作为启动，不会直接开发页面

> resources/fxml/demo/css/demo.css & 样式设置
>
> resources/fxml/demo/img/tortoise.png & 图片资源
>
> resources/fxml/demo/demo.xml & 配置文件
    
    
    <?import javafx.geometry.Insets?>
    <?import javafx.scene.control.*?>
    <?import javafx.scene.layout.Pane?>
    <?import javafx.scene.text.Font?>
    <Pane id="demo" maxHeight="-Infinity" maxWidth="-Infinity" minHeight="-Infinity" minWidth="-Infinity"
          prefWidth="540" prefHeight="415" stylesheets="@css/demo.css" xmlns="http://javafx.com/javafx/8.0.121"
          xmlns:fx="http://javafx.com/fxml/1">
    
    </Pane>
    

#### 6.5、Maven 配置

    
    
    <plugin>
        <groupId>com.zenjava</groupId>
        <artifactId>javafx-maven-plugin</artifactId>
        <version>8.8.3</version>
        <configuration>
            <mainClass>org.itstack.test.Application</mainClass>
        </configuration>
    </plugin>
    

  * 当需要使用 maven 管理的时候，我们需要将指定的启动类配置到 Maven 中，这样才可以正常运行

#### 6.6、效果演示

**点击运行 Application**

![](https://images.gitbook.cn/ZRld05)

好！不出意外你已经看到了一个同样的空白窗体，与上面一样

## 四、总结

  * 在这一篇中我们介绍了 JavaFx 以及基本功能、事件等使用，还分析了 PC 端的微信页面，方便我们后续的实现。另外在最后我们做了 2 个例子，分别使用不同的方式来创建一个 JavaFX 程序。这些都是为了后续的功能实现在铺设基础道路。
  * JavaFx 虽然很强大，但是由于来的太晚了，以至于使用人并不多，但如果随着谷歌公司的重视，可能将来也会越来越重要。但，目前由于文档、资料内容较少，所以遇到问题要多研究解决。另外本套专栏中的开发也会教会你很多 JavaFx 的技巧。
  * 一个新知识体系的学习，往往会经历；搜集资料运行 Helloworld、熟练使用 API 做业务开发、深度研究重要知识点和源码设计、领会思想运用技能建设自己的知识栈。

