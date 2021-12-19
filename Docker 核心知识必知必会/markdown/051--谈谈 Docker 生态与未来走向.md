> 本篇是第八部分“生态篇”的第五篇，也是本专栏的最后一篇。在这个部分，我会为你介绍 Docker 生态中的相关项目，以及如何参与到 Docker
> 项目中，最后会聊聊 Docker 未来的走向，上一篇，我为你介绍了如何参与到 Docker 项目的开发中。本篇，我来跟你聊聊 Docker
> 生态与未来走向。

谈及 Docker 生态及未来走向，不可避免的需要谈谈 Docker 其背后的公司 Docker Inc.（下文中统一称之为 Docker 公司）。

Docker 公司最早叫做 dotCloud Inc.，在 2013 年 10 月正式宣布将公司更名为 Docker Inc.。这项决定是由于 Docker
公司将把绝大多数资源都用于发展 Docker 和扩展 Docker 生态上。

之后 Docker 公司发布了 Docker 企业版（即 Docker EE），其中包含了不少企业用户所需的高级特性。之后推出了容器编排服务 Docker
Swarm，但是在容器编排领域的角逐中，算是落败于 Kubernetes 了。

此外，Docker EE 的商业收益并不那么好，终于在 2019 年年底，Docker 公司将其企业服务进行了出售，卖给了一家名为 Mirantis
的公司。

此次交易之后，Docker 公司将自己的重点重新放在了开源和助力开发者体验上，主要产品是 Docker Desktop 和 Docker Hub 等。

### Docker Desktop

在 2020 年年初， Docker 公司相继发布了 Docker Desktop for Windows/Mac 的新版本，提供了统一的可交互式
UI，和基本一致的体验。对于开发者而言，不需要有太多的基础知识，通过统一的交互式 UI 便可操作 Docker 完成基本的操作。

同时， **在 Docker 公司也正计划推出 Docker Desktop for Linux** ，致力于在 Windows/Mac/Linux
这三大主流操作系统上提供一致的操作体验。

更进一步，Docker Desktop 中有内置
Kubernetes，更有利于工程师开发云原生应用，在本地实现开发、部署、调试这个完整的生命周期。直接将应用进行交付。

今年 6 月份将举办 Docker Desktop 的第 4 个生日，届时 Docker 公司将与终端用户共同改进 Docker Desktop
的操作体验，为下一个大版本做准备。

### Docker Hub

Docker Hub 作为目前全球最大的公共镜像仓库，也是 Docker 继续发力的一个重点。

Docker 公司为了满足当前用户的需求，计划在之后提供年度订阅的功能，直接按年进行付费。这样可以使得用户能更好的做出预算，也省去了一些续费的麻烦。

之前 Docker Hub 提供了双因素认证，对于企业或者组织而言，如果能够强制组织内成员都开启双因素认证，将会大大提高安全性。所以 Docker Hub
也正在计划加入该功能，更好地服务于企业或组织。

此外，Docker Hub 中也计划加入日志审计功能，为用户提供更多的安全保障。

### Docker Compose

在 2020 年 4 月 7 日，Docker 宣布将其 Compose 规范（[Compose
Specification](http://www.compose-
spec.io/)）使用一个独立组织进行开源，并计划将它捐给一个中立的基金会进行治理。（我个人看法是：不出意外的话，会捐给 CNCF。）

Compose 在本专栏中用一篇文章做了基本的介绍，你会发现 Compose 是非常简单的。现在 Compose 有数百万开发者在使用，在 GitHub
上有超过 650000 个文件。它的简单易用，极大地简化了应用的部署、编排和调试。

Docker 公司在之前推出过一个 [Compose on Kubernetes](https://github.com/docker/compose-
on-kubernetes) 的工具，可将用 Compose 编排的应用，无缝部署至 Kubernetes 上。

现在开源其 Compose 规范，目的是与开源社区更好地协作，利用 Compose 的简洁来更好地助力开发者的体验。

在 Docker 公司的发展规划中，计划使用 Go 语言进行重写（原先是用 Python 编写的），以便于能更好地利用在容器领域其他项目的经验。

### 其他

在 3 月份， Docker 有发布了其首个[官方 GitHub
Action](https://github.com/marketplace/actions/build-and-push-docker-images)
，通过 GitHub Action 的 workflow 将镜像构建、分发等环节都简单地管理起来，通过这种方式也可以间接地推广 Docker 或者
Docker Hub 的使用。

整体而言，对于 Docker 生态（或者说容器生态），在接下来的几年中仍然是一种主流趋势。并且伴随着 Kubernetes/Service
Mesh/Serverless 等技术和概念的落地，对于 Docker 容器的使用，也将会更加普及。

3 月底是 Docker 的生日周，Docker 社区举办了一场特殊的在线活动，在这次活动上，也分享了一些关于 Docker
未来发展的内容。感兴趣的读者可以访问 <https://dockr.ly/7birthday-event> 查看详情。

### 总结

本篇，我为你分享了一些 Docker 生态及其未来走向，可以说 Docker 公司将其企业服务出售后，做了极大的改变。这对 Docker 而言是非常好的。

Docker 公司致力于提升开发者体验上，所以 Docker 相关的工具链将会更加的完善，也会更容易与 Kubernetes 等项目进行结合。未来会更好！

### 写在最后

自 2019 年 10 月本专栏上线，至今已经整整 6 个月了，感谢你的支持！

