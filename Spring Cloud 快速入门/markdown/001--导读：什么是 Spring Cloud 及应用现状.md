### Spring Cloud 是什么？

在学习本课程之前，读者有必要先了解一下 Spring Cloud。

Spring Cloud 是一系列框架的有序集合，它利用 Spring Boot
的开发便利性简化了分布式系统的开发，比如服务发现、服务网关、服务路由、链路追踪等。Spring Cloud
并不重复造轮子，而是将市面上开发得比较好的模块集成进去，进行封装，从而减少了各模块的开发成本。换句话说：Spring Cloud
提供了构建分布式系统所需的“全家桶”。

### Spring Cloud 现状

目前，国内使用 Spring Cloud 技术的公司并不多见，不是因为 Spring Cloud 不好，主要原因有以下几点：

  1. Spring Cloud 中文文档较少，出现问题网上没有太多的解决方案。
  2. 国内创业型公司技术老大大多是阿里系员工，而阿里系多采用 Dubbo 来构建微服务架构。
  3. 大型公司基本都有自己的分布式解决方案，而中小型公司的架构很多用不上微服务，所以没有采用 Spring Cloud 的必要性。

但是，微服务架构是一个趋势，而 Spring Cloud 是微服务解决方案的佼佼者，这也是作者写本系列课程的意义所在。

### Spring Cloud 优缺点

其主要优点有：

  1. 集大成者，Spring Cloud 包含了微服务架构的方方面面。
  2. 约定优于配置，基于注解，没有配置文件。
  3. 轻量级组件，Spring Cloud 整合的组件大多比较轻量级，且都是各自领域的佼佼者。
  4. 开发简便，Spring Cloud 对各个组件进行了大量的封装，从而简化了开发。
  5. 开发灵活，Spring Cloud 的组件都是解耦的，开发人员可以灵活按需选择组件。

接下来，我们看下它的缺点：

  1. 项目结构复杂，每一个组件或者每一个服务都需要创建一个项目。
  2. 部署门槛高，项目部署需要配合 Docker 等容器技术进行集群部署，而要想深入了解 Docker，学习成本高。

Spring Cloud 的优势是显而易见的。因此对于想研究微服务架构的同学来说，学习 Spring Cloud 是一个不错的选择。

### Spring Cloud 和 Dubbo 对比

Dubbo 只是实现了服务治理，而 Spring Cloud 实现了微服务架构的方方面面，服务治理只是其中的一个方面。下面通过一张图对其进行比较：

![这里写图片描述](https://images.gitbook.cn/66335f70-4633-11e9-8193-db0ca9692d09)

（图片引自：[程序猿DD](http://blog.didispace.com/microservice-framework/)，作者：翟永超）

可以看出，Spring Cloud 比较全面，而 Dubbo 由于只实现了服务治理，需要集成其他模块，需要单独引入，增加了学习成本和集成成本。

### Spring Cloud 学习

Spring Cloud 基于 Spring Boot，因此在研究 Spring Cloud 之前，本课程会首先介绍 Spring Boot
的用法，方便后续 Spring Cloud 的学习。

本课程不会讲解 Spring MVC 的用法，因此学习本课程需要读者对 Spring 及 Spring MVC 有过研究。

本课程共分为四个部分：

  * 第一部分初识 Spring Boot，掌握 Spring Boot 基础知识，为后续入门 Spring Cloud 打好基础 。

  * 第二部分 Spring Cloud 入门篇，主要介绍 Spring Cloud 常用模块，包括服务发现、服务注册、配置中心、链路追踪、异常处理等。

  * 第三部分 Spring Cloud 进阶篇，介绍大型分布式系统中事务处理、线程安全等问题，并以一个实例项目手把手教大家搭建完整的微服务系统。

  * 第四部分 Spring Cloud 高级篇，解析 Spring Cloud 源码，并讲解如何部署基于 Spring Cloud 的大型分布式系统。

> 本课程的所有示例代码均可在：https://github.com/lynnlovemin/SpringCloudLesson 下载。
>
> [点击了解《Spring Cloud
> 快速入门》，解决更多实际问题](https://gitbook.cn/gitchat/column/5af108d20a989b69c385f47a?utm_source=lysd001)。

