### 前言

在前后端分离模式开发中，前端开发主要依据的是接口文档，本篇带领大家按照接口文档完成注册功能的开发。由于是首次进行业务功能开发，需要先配置好项目必须的组件：路由管理、界面
UI、Ajax 请求等。

### 知识准备

#### **Vue Router**

Vue Router 是 Vue.js 官方的路由管理器。它和 Vue.js
的核心深度集成，让构建单页面应用变得易如反掌，本项目中使用它来完成路由的管理功能。

  * 嵌套的路由/视图表
  * 模块化的、基于组件的路由配置
  * 路由参数、查询、通配符
  * 基于 Vue.js 过渡系统的视图过渡效果
  * 细粒度的导航控制
  * 带有自动激活的 CSS class 的链接
  * HTML5 历史模式或 hash 模式，在 IE9 中自动降级
  * 自定义的滚动条行为

Vue Router 官方文档地址：

> <https://router.vuejs.org/zh/>

![vue-router](https://images.gitbook.cn/35083790-72a6-11ea-aa9c-4b75f9a497aa)

在项目中使用如下命令安装 vue-router：

    
    
    npm install --save vue-router
    

#### **Element UI**

Element，一套为开发者、设计师和产品经理准备的基于 Vue 2.0
的桌面端组件库，提供了页面布局、表单、表格、进度条、消息提示、对话框、弹出框等各种规范的格式与简便易用的 UI 组件。本项目中使用它来完成界面 UI。

Element 官方文档地址：

> <https://element.eleme.io/#/zh-CN>

![element](https://images.gitbook.cn/5c70dee0-72a6-11ea-9d30-373ba420c527)

在项目中使用如下命令安装 Element UI：

    
    
    npm i element-ui -S
    

#### **Axios**

Axios 是一个基于 Promise 的 HTTP 库，可以用在浏览器和 Node.js 中。本项目使用它来完成 Ajax 请求的发送和处理。

Axios 官方文档地址：

> <http://www.axios-js.com/>

![axios](https://images.gitbook.cn/835e62c0-72a6-11ea-9c78-13c8e455a47a)

在项目中使用如下命令安装 Axios：

    
    
    npm install axios
    

安装好以上三个组件后，项目配置文件 **package.json** 内的依赖项 dependencies 应该是这个样子的：

    
    
      "dependencies": {
        "axios": "^0.19.2",
        "core-js": "^3.4.4",
        "element-ui": "^2.13.0",
        "vue": "^2.6.10",
        "vue-router": "^3.1.6"
      }
    

### 项目路由管理

#### **新建页面**

在 Web 项目 src 文件夹下新建目录 views，所有 Vue 页面将存放在这个目录下。然后在 views 目录下新建 Vue 文件，命名为
Login.vue 和 Register.vue 分别用作登录页面和注册页面，代码如下：

    
    
    <!-- 登录页面 -->
    <template>
        <div>
            Login
        </div>
    </template>
    <script>
        export default{
            data(){
                return{}
            },
            methods:{},
        mounted(){}
        }
    </script>
    <style scoped>
    </style>
    

这是一个标准的 Vue 页面代码布局：

  * templat：HTML 模板文件
  * script：Vue.js 脚本
  * style：CSS 样式

新建一个 Vue 文件，命名为 404.vue，用做错误提示页面，代码如下：

    
    
    <template>
        <p class="page-container">404 page not found</p>
    </template>
    <style scoped>
        .page-container {
            font-size: 20px;
            text-align: center;
            color: rgb(192, 204, 218);
        }
    </style>
    

#### **配置路由表**

在项目的 main.js 同级目录中新建一个 JS 文件，命名为 routes.js，用于路由配置，代码如下：

    
    
    import Login from './views/Login.vue'
    import Register from './views/Register.vue'
    import NotFound from "./views/404.vue"
    
    let routes = [
        {   path: '/Login',component: Login,name:"Login"},
      { path: '/Register',component: Register, name: 'Register'},
        { path: '/404',component: NotFound, name: 'NotFound'},
        {   path: '*', redirect: { path: '/404' }}
    ];
    
    export default routes;
    

这个文件将 Login.vue、Register.vue、404.vue 三个页面引入，并将其配置为不同的路由（语法格式参照 Vue Router
中文文档）：

  * 浏览器访问 /Login 时，路由到 Login.vue，即登录页面
  * 浏览器访问 /Register 时，路由到 Register.vue，即注册页面
  * 浏览器访问其他地址时，路由到 404.vue，即错误提示页面

#### **引入组件**

将项目的 main.js 文件修改为如下所示：

    
    
    import Vue from 'vue'
    import App from './App.vue'
    //引入 vue-router 组件
    import VueRouter from 'vue-router'
    //引入 element-ui 组件
    import ElementUI from 'element-ui'
    //引入 element-ui 样式表
    import 'element-ui/lib/theme-chalk/index.css'
    //引入路由表
    import routes from './routes'
    Vue.use(ElementUI)
    Vue.use(VueRouter)
    
    const router = new VueRouter({
        //使用history模式来避免页面输入路由参数后自动加 # 
        mode: 'history',
        routes
    })
    
    new Vue({
        router,
      render: h => h(App)
    }).$mount('#app')
    

通过以上操作，就完成了路由功能的配置。

验证路由功能：`npm run serve` 启动项目后，在浏览器访问 http://localhost:8080/Login，就访问到了
Login.vue 页面，如下：

![Login](https://images.gitbook.cn/ff6491a0-72a6-11ea-9d12-3d2bd05cf9bc)

在浏览器访问 http://localhost:8080/User，因为 /User 在路由中没有配置，就被重定向到了 404 页面，如下：

![User](https://images.gitbook.cn/0bf76910-72a7-11ea-9e02-c110b4a4e660)

### 注册功能的实现

Vue 页面的开发流程是页面结构 -> 数据 -> 交互：

  1. 先声明接口，接口规定了传入参数、返回数据的数量和格式，页面按照规定格式组织数据才能实现数据的传输。
  2. 然后实现页面结构，会用到基于 Vue.js 的 UI 框架（如：Element UI），涉及到 `<template>` 标签中的内容。

  * 需要自定义样式的地方，在 `<style>` 标签中书写 CSS 样式，也可以使用 SCSS 语法。
  * 需要数据支持的地方，在 `<script>` 标签中的 data() 方法中定义数据。

  1. 最后实现用户交互，在 `<script>` 标签中的 `methods` 对象中实现需要交互的方法。

#### **声明接口**

页面功能开发过程中，需要发送 HTTP 请求。建议每次开发页面前先将 HTTP 接口声明完毕，这样做的好处在于：

  * 尚未进行页面开发，此时能排除其他干扰，按照接口文档声明，保证接口的正确性和规范性
  * 声明好的接口有助于合理定义页面数据格式，避免因格式不一致导致的操作复杂化

在 **main.js** 同级目录下新建一个 **api.js** 文件，将所有发送 HTTP 请求的方法都写在这个文件中，代码如下：

    
    
    import axios from 'axios';
    /** 需要先登录再访问的路由 */
    let api = 'api';
    let form = "form"
    
    /** 不需要登录就能访问的路由 */
    let base = 'base';
    /**
     * 获取注册验证码
     */
    export const req_getCodeForRegister = (email) => { 
        return axios.get(base + '/getCodeForRegister', {
            params:{
                email:email
            }
        }).then(res => res.data).catch(err => err);  
    };
    /**
     * 注册用户
     */
    export const req_register = (user) => { 
        return axios.post(base + '/register', {
            email:user.email,
            password:user.password,
            code:user.code
        }).then(res => res.data).catch(err => err); 
    };
    

这里写了两个请求：获取注册验证码、注册用户，写的依据就是前期梳理的接口文档，具体的接口定义参见第 4 篇《接口文档设计》中 **1001** 、
**1002** 接口的描述。

将请求统一管理，所有接口都在同一个文件中，很容易查找和管理，同时也有利于保持统一的代码风格和数据格式。

#### **页面设计**

注册页面效果图如下：

![](https://images.gitbook.cn/47a1c370-72a7-11ea-b3fa-5b760b39398d)

根据效果图，从上到下逐行对页面进行分析：

  * 第一部分：页面标题 **KellerNotes 注册** ，居中显示
  * 第二部分：步骤条，使用 Element UI 中的步骤条组件 `<el-steps>` 实现
  * 第三部分：表单，标签、文本框、按钮等
  * 第四部分：按钮或超链接，点击后跳转到登录页面

在 **Register.vue** 页面中，涉及到的新知识就是 Element UI 中的 **步骤条** 组件，代码如下：

    
    
            <!-- 步骤条 -->
            <el-steps :active="step" finish-status="success">
                <el-step title="验证邮箱"></el-step>
                <el-step title="设置密码"></el-step>
                <el-step title="完善资料"></el-step>
            </el-steps>
    ... ....
    <script>
      export default {
        data() {
          return {
            step: 0
          };
        }
      }
    </script>
    

Steps 步骤条：引导用户按照流程完成任务的分步导航条，可根据实际应用场景设定步骤，步骤不得少于 2 步。在该页面中的步骤条用到了一个变量
step，用来控制步骤条的进度，初始值为 0，没进行一步加一。

页面其他内容按照正常的 HTML 标签格式布局就可以了，拿第一步举例，代码如下：

    
    
    <el-row v-if="step == 0">
          <el-form>
                    <el-form-item>
                        <!--邮箱输入框-->
                        <span class="ColorCommon font-bold">邮箱</span>
                        <span class="ColorDanger" v-show="!user.emailChecked"> (请填写正确的邮箱地址)</span>
                        <el-input type="text" v-model="user.email" placeholder="kellerNotes@foxmail.com" autofocus="autofocus"></el-input>
                    </el-form-item>
                    <el-form-item style="width:100%;">
                        <el-button type="primary" style="width:100%;" :disabled="getCodeButtonDisabled" @click="handleGetCode">获取验证码</el-button>
                        <!-- 跳转到登录页面的按钮 -->
                        <el-row>
                            <el-col class="alignRight">
                                <el-button type="text" @click="handleGoLogin()">已有账号，直接登录</el-button>
                            </el-col>
                        </el-row>
                    </el-form-item>
         </el-form>
    </el-row>
    

页面的代码比较多，就不全部贴出来了，在文章末尾的 **源码地址**
中有本篇完整的项目代码。在步骤条中，我们设计了三个步骤：验证邮箱、设计密码、完善资料，其中，前两步就可以完成用户注册功能，也就是本篇的内容；最后一步是创建用户名片，暂时不实现，留待后续实现用户名片功能时完成。

#### **实现交互**

交互过程中，离不开数据和交互方法，以获取验证码功能为例，所需的数据如下：

    
    
        //是否禁用获取验证码按钮
        getCodeButtonDisabled: false,
        //注册用户数据
        user: {
        //邮箱
            email:null,
        //邮箱是否验证通过
            emailChecked:true
            },
        //注册界面步骤条当前步骤Index
        stepsActive: 0
    

交互方法是：用户输入邮箱后点击 **获取验证码** 按钮，点击后处理流程如下：

  1. 通过正则表达式验证输入的是否是邮箱地址：如果是，开始发送 HTTP 请求；否则，提示邮箱格式不正确
  2. 发送 HTTP 请求，判断返回的业务状态是不是 success：如果是，提示用户验证码发送成功，进入下一步操作；否则，提示服务端返回的错误信息。

实现代码如下：

    
    
    handleGetCode() {
                  //调用 data.js 中的方法验证邮箱
                    if(format.isEmail(this.user.email)){
                        this.user.emailChecked = true;
                    }else{
                        this.user.emailChecked = false;
                        return;
                    }
                  //将获取验证码按钮禁用，避免用户重复点击
                    this.getCodeButtonDisabled = true;
                    let email = this.user.email;
                  //调用获取验证码的接口发送 Http 请求
                    req_getCodeForRegister(email).then(response => {
                        //解析接口应答的json串
                        let {
                            message,
                            success
                        } = response;
                        //应答不成功，提示错误信息
                        if (success !== 0) {
                            this.$message({
                                message: message,
                                type: 'error'
                            });
                        } else {
                            //应答成功，提示验证码发送成功
                            this.$notify.success({
                                title: email,
                                message: "验证码已发送到您的邮箱，请及时查收"
                            });
                //控制步骤条状态，进入下一步操作
                            this.step++;
                        }
                    });
                }
    

该方法中的第一行代码用到了 data.js 中的方法验证一个字符串是不是邮箱地址，该 JS
是自己写的工具类，其中验证邮箱的代码使用正则表达式实现，具体代码如下：

    
    
    //使用正则表达式定义邮箱格式
    emailFormat : new RegExp("^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+@((\\[[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\])|(([a-zA-Z\\-0-9]+\\.)+[a-zA-Z]{2,}))$"),
        isEmail(email){
          //使用正则表达式验证一个字符串是否是邮箱地址
            return this.emailFormat.test(email);
        }
    

### 跨域问题处理

注册功能开发完成后，进行测试，会发现访问失败，是因为浏览器同源策略不允许跨域访问。在前端开发中，跨域问题的处理是避免不了的，Vue CLI
官网上对跨域的配置由详细的说明，官网地址：

> <https://cli.vuejs.org/zh/config/#vue-config-js>

![vue.config](https://images.gitbook.cn/e9a28ab0-72a7-11ea-ad55-4b5f9f0c99bb)

在项目 package.json 同级目录下新建 vue.config.js 文件，按如下格式配置：

    
    
    module.exports = {
        outputDir: 'dist', //build 输出目录
        assetsDir: 'assets', //静态资源目录（js、css、img）
        lintOnSave: false, //是否开启 ESLint
        devServer: {
            open: true, //是否自动弹出浏览器页面
            host: "127.0.0.1",
            port: 8088, //指定启动端口，必须是大于 1024 的值才生效
            https: false, //是否使用 HTTPS 协议
            hotOnly: false, //是否开启热更新
            proxy: { //跨域代理地址
                '/api': {
                    target: 'http://127.0.0.1:8080',
                    changeOrigin: true,
                    secure: false
                },
          '/base': {
                    target: 'http://127.0.0.1:8080',
                    changeOrigin: true,
                    secure: false
                }
            },
        }
    }
    

这样指定了项目运行的端口号和请求代理地址，在使用 Ajax 访问 `/base` 时会将请求转发到
http://127.0.0.1:8080，也就是服务端运行的地址。

### 效果测试

测试时要保证前后端项目都是正常运行的，同时检查后端项目运行的端口与前端代理的端口保持一致，比如本项目中都是 8080。

#### **前端效果**

填写邮件地址，发送验证码，页面提示 **验证码已发送到您的邮箱，请及时查收** ：

![验证码发送成功](https://images.gitbook.cn/2716f6b0-72a8-11ea-9d12-3d2bd05cf9bc)

填写收到的验证码并设置密码，注册用户，注册完成后提示 **注册成功** ：

![注册成功](https://images.gitbook.cn/2fd0d9b0-72a8-11ea-ad55-4b5f9f0c99bb)

这是邮箱中收到的邮件：

![](https://images.gitbook.cn/3b4744a0-72a8-11ea-803c-711d330d13a5)

#### **后端日志记录**

服务端会在每次收到请求和查询数据库时在控制台打印日志，观察日志可以看到每一次请求的处理过程。

收到发送验证码的请求，校验用户是否已经注册：

![发送验证码日志](https://images.gitbook.cn/47f5e170-72a8-11ea-80ec-dd597353a0eb)

验证码发送成功后，记录在数据库中：

![邮件记录日志](https://images.gitbook.cn/584d8910-72a8-11ea-8b3a-93665903b876)

收到注册请求，校验验证码，校验成功后注册用户，将用户信息写在数据库中：

![注册日志](https://images.gitbook.cn/63e0cad0-72a8-11ea-aa9c-4b75f9a497aa)

### 源码地址

本篇完成源码地址：

> <https://github.com/tianlanlandelan/KellerNotes/tree/master/12.注册2>

