### 前言

前面的课程我们已经系统地学习了 Spring Boot 技术栈核心组件的使用，这节课我们就用这部分所学的技术完成一个电商系统的项目实战开发。

### 需求简介

  1. 账户管理：用户登录、退出登录、用户注册
  2. 商品模块：商品展示、商品检索、商品分类、商品详情
  3. 购物车模块：展示购物车商品、添加商品到购物车、修改购物车商品、删除购物车商品
  4. 订单模块：选择地址、订单结算
  5. 用户模块：查看我的订单、查看我的信息、地址管理、资讯查看

### 开发环境

  * JDK 10.0.1
  * Maven 3.5.3
  * Tomcat 9.0.8
  * IDEA 2019.1

### 技术选型

  * Spring Boot 2.1.5.RELEASE
  * MyBatis 3.4.5
  * Thymeleaf
  * MySQL 8.0.11

### 代码实现

1\. 创建数据表

    
    
    DROP TABLE IF EXISTS `easybuy_cart`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
     SET character_set_client = utf8mb4 ;
    CREATE TABLE `easybuy_cart` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `productid` int(11) DEFAULT NULL,
      `quantity` int(11) DEFAULT NULL,
      `cost` int(11) DEFAULT NULL,
      `userid` int(11) DEFAULT NULL,
      PRIMARY KEY (`id`),
      KEY `productId` (`productid`),
      KEY `userId` (`userid`),
      CONSTRAINT `easybuy_cart_ibfk_1` FOREIGN KEY (`productid`) REFERENCES `easybuy_product` (`id`),
      CONSTRAINT `easybuy_cart_ibfk_2` FOREIGN KEY (`userid`) REFERENCES `easybuy_user` (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=101 DEFAULT CHARSET=utf8;
    
    DROP TABLE IF EXISTS `easybuy_news`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
     SET character_set_client = utf8mb4 ;
    CREATE TABLE `easybuy_news` (
      `id` int(10) NOT NULL AUTO_INCREMENT COMMENT '主键',
      `title` varchar(40) NOT NULL COMMENT '标题',
      `content` varchar(1000) NOT NULL COMMENT '内容',
      `createtime` varchar(10) NOT NULL COMMENT '创建时间',
      PRIMARY KEY (`id`),
      UNIQUE KEY `PK__EASYBUY___C63B5EE724927208` (`id`),
      UNIQUE KEY `UQ__EASYBUY___C12AD09D276EDEB3` (`title`)
    ) ENGINE=InnoDB AUTO_INCREMENT=708 DEFAULT CHARSET=utf8;
    
    DROP TABLE IF EXISTS `easybuy_order`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
     SET character_set_client = utf8mb4 ;
    CREATE TABLE `easybuy_order` (
      `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
      `userid` int(255) DEFAULT NULL COMMENT '用户主键',
      `loginname` varchar(255) DEFAULT NULL,
      `useraddress` varchar(255) DEFAULT NULL COMMENT '用户地址',
      `createtime` datetime DEFAULT NULL COMMENT '创建时间',
      `cost` float DEFAULT NULL COMMENT '总消费',
      `serialnumber` varchar(255) DEFAULT NULL COMMENT '订单号',
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=77 DEFAULT CHARSET=utf8;
    
    DROP TABLE IF EXISTS `easybuy_order_detail`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
     SET character_set_client = utf8mb4 ;
    CREATE TABLE `easybuy_order_detail` (
      `id` int(10) NOT NULL AUTO_INCREMENT COMMENT '主键',
      `orderid` int(10) NOT NULL COMMENT '订单主键',
      `productid` int(10) NOT NULL COMMENT '商品主键',
      `quantity` int(10) NOT NULL COMMENT '数量',
      `cost` float NOT NULL COMMENT '消费',
      PRIMARY KEY (`id`),
      UNIQUE KEY `PK__EASYBUY___66E1BD8E2F10007B` (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=69 DEFAULT CHARSET=utf8;
    
    DROP TABLE IF EXISTS `easybuy_product`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
     SET character_set_client = utf8mb4 ;
    CREATE TABLE `easybuy_product` (
      `id` int(10) NOT NULL AUTO_INCREMENT COMMENT '主键',
      `name` varchar(200) NOT NULL COMMENT '名称',
      `description` varchar(1024) DEFAULT NULL COMMENT '描述',
      `price` float NOT NULL COMMENT '价格',
      `stock` int(10) NOT NULL COMMENT '库存',
      `categoryleveloneid` int(10) DEFAULT NULL COMMENT '分类1',
      `categoryleveltwoid` int(10) DEFAULT NULL COMMENT '分类2',
      `categorylevelthreeid` int(10) DEFAULT NULL COMMENT '分类3',
      `filename` varchar(200) DEFAULT NULL COMMENT '文件名称',
      PRIMARY KEY (`id`),
      UNIQUE KEY `PK__EASYBUY___94F6E55132E0915F` (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=777 DEFAULT CHARSET=utf8;
    
    DROP TABLE IF EXISTS `easybuy_product_category`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
     SET character_set_client = utf8mb4 ;
    CREATE TABLE `easybuy_product_category` (
      `id` int(10) NOT NULL AUTO_INCREMENT COMMENT '主键',
      `name` varchar(20) NOT NULL COMMENT '名称',
      `parentid` int(10) NOT NULL COMMENT '父级目录id',
      `type` int(11) DEFAULT NULL COMMENT '级别(1:一级 2：二级 3：三级)',
      PRIMARY KEY (`id`),
      UNIQUE KEY `PK__EASYBUY___9EC2A4E236B12243` (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=697 DEFAULT CHARSET=utf8;
    
    DROP TABLE IF EXISTS `easybuy_user`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
     SET character_set_client = utf8mb4 ;
    CREATE TABLE `easybuy_user` (
      `id` int(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
      `loginname` varchar(255) NOT NULL COMMENT '登录名',
      `username` varchar(255) NOT NULL COMMENT '用户名',
      `password` varchar(255) NOT NULL COMMENT '密码',
      `sex` int(2) NOT NULL DEFAULT '1' COMMENT '性别(1:男 0：女)',
      `identitycode` varchar(60) DEFAULT NULL COMMENT '身份证号',
      `email` varchar(80) DEFAULT NULL COMMENT '邮箱',
      `mobile` varchar(11) DEFAULT NULL COMMENT '手机',
      `type` int(2) DEFAULT '0' COMMENT '类型（1：后台 0:前台）',
      `filename` varchar(255) DEFAULT NULL,
      PRIMARY KEY (`id`),
      UNIQUE KEY `PK__EASYBUY___C96109CC3A81B327` (`loginname`)
    ) ENGINE=InnoDB AUTO_INCREMENT=27 DEFAULT CHARSET=utf8;
    
    DROP TABLE IF EXISTS `easybuy_user_address`;
    /*!40101 SET @saved_cs_client     = @@character_set_client */;
     SET character_set_client = utf8mb4 ;
    CREATE TABLE `easybuy_user_address` (
      `id` int(20) NOT NULL AUTO_INCREMENT COMMENT '主键id',
      `userid` int(255) DEFAULT NULL COMMENT '用户主键',
      `address` varchar(255) DEFAULT NULL COMMENT '地址',
      `createtime` datetime DEFAULT NULL COMMENT '创建时间',
      `isdefault` int(2) DEFAULT '0' COMMENT '是否是默认地址（1:是 0否）',
      `remark` varchar(255) DEFAULT NULL COMMENT '备注',
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=42 DEFAULT CHARSET=utf8;
    

2\. 创建 Maven 工程，pom.xml 添加相关依赖。

    
    
    <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-parent</artifactId>
      <version>2.1.5.RELEASE</version>
    </parent>
    
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
      </dependency>
    
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
      </dependency>
    
      <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>1.3.1</version>
      </dependency>
    
      <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.15</version>
      </dependency>
    
      <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
      </dependency>
    </dependencies>
    

3\. 创建 Handler、Service、Repository、Entity、Mapper.xml 这里统统略过，具体实现直接查看源码。

![](https://images.gitbook.cn/6b7aeb80-ccdc-11e9-8d89-4fa271cb1633)

![](https://images.gitbook.cn/79c1b480-ccdc-11e9-9f23-07a3e2a236db)

![](https://images.gitbook.cn/84879420-ccdc-11e9-9f23-07a3e2a236db)

4\. 创建配置文件 application.yml，添加数据源、视图解析、MyBatis 相关配置。

    
    
    spring:
      datasource:
        url: jdbc:mysql://localhost:3306/easybuy?useUnicode=true&characterEncoding=UTF-8
        username: root
        password: root
        driver-class-name: com.mysql.cj.jdbc.Driver
      thymeleaf:
        prefix: classpath:/static/
        suffix: .html
    mybatis:
      mapper-locations: classpath:/mapping/*.xml
      type-aliases-package: com.southwind.entity
    

5\. 在 resources 路径下创建 static 文件夹，将视图层相关静态资源（包括
HTML、JavaScript、jQuery、CSS、图片）放置在此路径下。

![](https://images.gitbook.cn/905a01c0-ccdc-11e9-9a11-bbb3551196dc)

6\. 创建启动类 Application。

    
    
    @SpringBootApplication
    @MapperScan("com.southwind.repository")
    public class Application {
        public static void main(String[] args) {
            SpringApplication.run(Application.class,args);
        }
    }
    

7\. 启动 Application，项目截图如下所示。

![](https://images.gitbook.cn/9c2124c0-ccdc-11e9-9f23-07a3e2a236db)

![](https://images.gitbook.cn/a8883140-ccdc-11e9-8d89-4fa271cb1633)

![](https://images.gitbook.cn/b3955310-ccdc-11e9-9f23-07a3e2a236db)

![](https://images.gitbook.cn/bf448140-ccdc-11e9-beb5-a53251e30de8)

![](https://images.gitbook.cn/c9253100-ccdc-11e9-beb5-a53251e30de8)

![](https://images.gitbook.cn/d4ebded0-ccdc-11e9-beb5-a53251e30de8)

![WX20190621-160405@2x](https://images.gitbook.cn/e07d72e0-ccdc-11e9-beb5-a53251e30de8)

### 总结

本节课为项目实战，这次我们选择的是 Spring Boot + MyBatis 3.4.5 + Thymeleaf + MySQL
的技术选型，目的是对本阶段课程 Spring Boot 的所学知识点进行系统性梳理，通过实战的形式让大家掌握 Spring Boot
技术栈在实际开发中的使用。

[请点击这里查看源码](https://github.com/southwind9801/shopping.git)

[点击这里获取 Spring Boot
视频专题](https://pan.baidu.com/s/1K2cNTk6JmZa50RYSKwvwGA)，提取码：e4wc

