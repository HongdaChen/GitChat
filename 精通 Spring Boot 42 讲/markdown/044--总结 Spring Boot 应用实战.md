我们一起回顾下本系列课程，该课程其实是 Spring Boot 应用实战的四十多个常用案例，这些案例提炼于工作中经常使用的场景，目标是通过实践真实案例来学习
Spring Boot ，以帮助大家快速掌握如何在工作中使用 Spring Boot 技术。

在准备这系列课程之前，我在博客上连载了一系列关于 Spring Boot
的文章，期间得到了广大读者的不少反馈。根据这些反馈信息我在思考，怎样才能比较全面快速地掌握 Spring Boot，以及如何进行日常项目的开发。

通过多年的项目研发和读者反馈，让我总结出： **唯有真正的实践才是学习的最佳路径**
，因此在本系列课程中，几乎每课有对应的示例项目，强烈建议大家根据课程内容，一边学习一边敲代码动手练习，把课程中的每一个示例都一一实践，这样全程学习下来相信对
Spring Boot 会有一个比较深刻的理解。

本课程共包含五大部分，通过不同的示例讲解了 Spring Boot 技术栈的使用场景和实践方式。

**第一部分** ： **从零开始认识 Spring Boot** ，主要介绍了什么是 Spring Boot、诞生的背景和设计理念，Spring Boot
2.X 主要更新了哪些内容，我们从 1.X 升级到 2.X 的时候需要注意哪些内容，最后介绍了如何去开发一个 Hello World 版的 Spring
Boot 项目，方便大家了解 Spring Boot 项目的开发流程。

**第二部分** ： **主要介绍了项目中 Spring Boot Web 的相关技术** ，如今所有研发的项目中 Web 项目占据了 90 %
以上市场份额，Web 开发技术是我们最常使用的。因此这部分内容介绍了如何使用 Spring Boot 创建常见的 Web 应用，包括 Spring Boot
整合 JSP；模板引擎 Thymeleaf 的基础使用、高阶使用、页面布局；Web 开发过程中有一些常见的使用场景，如上传文件、构建 RESTful
服务、Swagger 2 的使用，以及如何使用 WebSocker 技术创建一个多人聊天室等。

**第三部分** ： **Spring Boot 和数据库技术实践**
，数据是公司最重要的资产，在项目开发中数据库操作是永远无法绕过的一步，也是最高频最重要的功能操作。这部分内容介绍了数据库操作的三大 ORM
框架：JDBC、MyBatis、Spring Boot JPA，演示了如何在 Spring Boot 项目中集成操作、构建多数据源、集成 Druid
连接池等，最后使用 JPA 和 Thymeleaf 综合实践。

**第四部分** ： **Spring Boot 集成 MQ、缓存、NoSQL 等中间件**
，中间件是互联网公司支撑高并发业务的必备组件，常用的组件有缓存、消息中间件、NoSQL
数据库、定时任务等。这部分内容介绍了项目中这些中间件的使用方式，以及如何使用 Spring Boot 设计一个邮件系统。

**第五部分** ： **综合实践** ，最后一部分主要关注的是 Spring Boot 项目的安全控制、应用监控、集群监控、测试部署、Docker
打包部署，最后用一个简单的用户管理系统回顾了课程中的相关技术点。

这里我画了一个思维导图，帮助大家整理整个课程的知识点：

![](http://www.ityouknow.com/assets/images/2018/springboot/42.png)

一种新框架的诞生往往伴随着一种思维方式的提升，约定优于配置是一种设计思想，在整个指导思想之下，Spring Boot 重构了 Spring
的使用，让其焕发出新春。同样依赖于此思想，Spring Boot 整合了庞大的技术生态，融合了主流性的系统框架，让开发者几乎零配置的使用各种开源软件。

在使用 Spirng Boot 设计研发时，也可以参考这样的设计思路，去定制我们自己的 Starter
包、定制公司内部的集成方案，让外部人员使用我们的服务时，也可以“零”配置，以默认的方式来实现 80% 日常功能需求，特殊的场景需使用其他方式来实现。

如果把 Spring Boot 放到整个微服务架构的生态中，其实 Spring Boot
扮演着底层开发的角色，利用它来快速开发的特点创建微服务，同时还有极大的开放性可以和微服务其他组件无缝结合，特别是 Spirng Cloud 正是利用
Spirng Boot 相关特性来研发的。

大家都知道 Spring Boot 的推出是为了更好的使用 Spring，Spring Boot 是在强大的 Spring
框架基础上构建而来，因此该框架所具备的功能，Spring Boot 都具备并且更好用，我们也可以通过了解 Spring 技术栈来了解 Spring Boot
的能力范围。

Spring 技术栈所包含的技术框架图如下：

![](http://www.ityouknow.com/assets/images/2018/springboot/spring-stack.png)

在课程的最后，推荐一些 Spring Boot 的学习资源：

  * [Spring Boot 参考指南](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)，是 Spring 官方提供学习 Spring Boot 的文档，比较详细地讲述了 Spring Boot 的使用；
  * [Spring Boot 中文索引](http://springboot.fun/)，这是一个专业收集 Spring Boot 学习资源的开源项目，全网有最全的 Spring Boot 开源项目和文章；
  * [云收藏](https://github.com/cloudfavorites)，是一个使用 Spring Boot 2.0 相关技术栈构建的个人收藏类开源项目，使用了课程中的大部分技术，目前 GitHub 上面的 Star 数超过了 2600；
  * [spring-boot-examples](https://github.com/ityouknow/spring-boot-examples)，GitHub 上关于 Spring Boot 使用的各种小案例，Star 数量超过了 7300，是 GitHub 上 Star 最多的个人 Spring Boot 开源项目之一；
  * [公号“纯洁的微笑”](http://www.ityouknow.com/assets/images/keeppuresmile.jpg)，这个公号会定期发布业内优秀的 Spring Boot 使用案例；
  * [我的博客](http://www.ityouknow.com)，会持续跟踪 Spring Boot 技术的最新进展，欢迎大家关注。

Spring Boot 是一个非常伟大的创新，也给开发带来了全新的模式和挑战，使用 Spring Boot 多年，深深感觉到 Spring Boot
化繁为简、快速便捷开发的魅力。也希望大家通过 42 节课程的学习，能对其有更进一步的了解，在工作中有所帮助。

最后感谢大家对我的关注，我将一如既往的继续分享我对技术的理解。

