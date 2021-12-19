### 前言

用户管理的服务端功能准备完毕后，就可以实现 Web 端的效果了。本篇带领大家实现用户管理功能的页面展示及将用户数据导出成 Excel 文件。

### 知识准备

本篇涉及到的新知识点为在页面上导出 Excel，这里用到了 vue-json-excel 组件。

#### **vue-json-excel**

vue-json-excel 文档地址：

> <https://www.npmjs.com/package/vue-json-excel>

![xlsx](https://images.gitbook.cn/4f649210-951e-11ea-a9dc-ef5cf20ceba2)

在该网站上是这样介绍的：

> Download your JSON data as an excel file directly from the browser. This
> component it's based on the solution proposed on this thread
> https://stackoverflow.com/questions/17142427/javascript-to-export-html-
> table-to-excel

也就是说，它是可以实现将数据转换成 Ecxel 文件并从浏览器中下载到本地的功能，使用比较简单，仅需安装一个依赖包即可：

    
    
    npm install vue-json-excel
    

vue-json-excel 的特点有：

  * 支持 JSON 的数据格式
  * 支持直接绑定数据源及下载前获取数据源
  * 支持自定义表头及列数据
  * 支持列数据的自定义处理
  * 支持自定义工作表

### 搭建后台管理页面结构

#### **功能分析**

和用户主页相同，后台管理页面也要先进行登录操作，和用户登录页面不同的是，后台管理页面无法注册。首个系统管理员数据需要在数据库中手动添加，如果需要其他系统管理员，可以在管理员页面功能中设计
**添加管理员** 的功能。

后台管理系统的登录页面效果如下：

![登录](https://images.gitbook.cn/90a2b040-951e-11ea-82b3-6111407489cd)

登录成功后进入后台管理页面，该页面使用经典的后台管理页面布局方式，左侧为导航栏，右侧为页面内容，效果如下：

![概览](https://images.gitbook.cn/a7f0e000-951e-11ea-b359-c776f1823b74)

#### **流程设计**

  * 登录 
    * 在后台管理系统登录页面中输入账号密码后，请求登录
      * 使用和用户登录同样的接口，区别在于 type 参数固定传值 100
    * 登录成功后，保存服务端返回的 JWT，进入后台管理页面
      * 由于管理员操作具有风险性，将 JWT 保存在 sessionStorage 中，而不是 localStorage
      * 后续管理员相关操作也是从 sessionStorage 中取出 JWT 
  * 进入后台管理页面
    * 管理页面左侧的导航栏要根据 **Router** 动态生成

#### **代码实现**

后台管理系统的登录页面布局方式和用户登录页面相同，这里就不再啰嗦了，主要讲解如何动态生成管理页面中的导航栏。

**1\. 添加页面**

首先新建几个空的 Vue 文件，Admin.vue、AdminCount.vue、AdminIndex.vue、AdminUser.vue
等。其中：Admin.vue 页面作为父页面，定义后台管理功能的页面布局；其他页面作为具体的功能页面。如下图：

![空文件](https://images.gitbook.cn/ddfe9cf0-951e-11ea-89e8-d3e59c640cba)

**2\. 定义路由**

在 router.js 中为新添加的页面设置路由，如：

    
    
    //后台管理登录页面
        {
            path: '/AdminLogin',
            component: ()=> import("./views/admin/AdminLogin.vue"),
            name: '管理员登录'
        },
        //后台管理页面
        {
            path: '/Admin0',
            component: Admin,
            name: '概览',
            iconCls: 'el-icon-menu',
            children: [{
                path: '/AdminIndex',
                component: () => import("./views/admin/AdminIndex.vue"),
                name: '系统概览'
            }],
            show: true
        },
        {
            path: '/Admin1',
            component: Admin,
            name: '用户管理',
            iconCls: 'el-icon-user-solid',
            children: [{
                path: '/AdminUser',
                component: () => import("./views/admin/AdminUser.vue"),
                name: '用户列表'
            }],
            show: true
        }
    

路由参数说明：

  * compnent：页面模板，在拥有 **children** 的路由上，会将 **children** 的 compnent 指定的页面挂载在该参数指定的父页面中 `<router-view>` 标签的位置
  * iconCls：导航菜单中要显示的图标，该值的设置参照 element UI 中的图标库 <https://element.eleme.cn/#/zh-CN/component/icon>
  * show：是否显示在导航菜单中
  * children：子页面，也就是实际的功能页面

**3\. 构建导航菜单**

在 Admin.vue 中读取路由表，过滤出要在导航菜单中显示的页面：

    
    
    <!--导航菜单-->
                        <el-menu :default-active="$route.path" class="el-menu-vertical-demo" @open="handleopen($route.path)" @close="handleclose($route)"
                            @select="handleselect" unique-opened router >
                            <template v-for="(item,index) in $router.options.routes">
                                <div :key="item.path" v-if="item.show">
                                    <el-submenu :index="index+''" v-if="!item.leaf" :key="index">
                                        <template slot="title">
                                            <i :class="item.iconCls"></i>
                                            {{item.name}}
                                        </template>
                                        <el-menu-item v-for="child in item.children" :index="child.path" :key="child.path">{{child.name}}</el-menu-item>
                                    </el-submenu>
                                    <el-menu-item v-if="item.leaf&&item.children.length>0" :index="item.children[0].path" :key="index">
                                        <i :class="item.iconCls"></i>
                                        {{item.children[0].name}}
                                    </el-menu-item>
                                </div>
                            </template>
                        </el-menu>
    

### 实现用户列表功能

#### **功能分析**

点击导航菜单中的 **用户列表** 菜单项后，进入用户列表页面，该页面主要提供用户列表的展示操作，效果如下：

![用户列表](https://images.gitbook.cn/ff0cfc20-951e-11ea-b103-15b42d8d0dac)

#### **流程设计**

  * 使用 `<el-form>` 布局查询模块
  * 使用 `<el-pagination>` 布局分页模块
  * 使用 `<el-table>` 展示用户列表

#### **代码实现**

**1\. 定义接口**

展示用户列表，需要调用后台接口获取数据，在 api.js 中按照后台接口的格式要求定义好请求方法，如下：

    
    
    /**
     * 获取用户列表 5002
     */
    export const req_getUserList = (page,size,email) => { 
        return axios.get(api, {
            params:{
                page:page,
                size:size,
                email:email
            },
            headers:{
                'method':'admin/userList',
                'token' : window.sessionStorage.getItem("adminToken")
            }
        }).then(res => res.data).catch(err => err);  
    };
    
    /**
     * 获取用户总数 5007
     */
    export const req_getUserCounter = (email) => { 
        return axios.get(api, {
            params:{
                email:email
            },
            headers:{
                'method':'admin/userCounter',
                'token' : window.sessionStorage.getItem("adminToken")
            }
        }).then(res => res.data).catch(err => err);  
    };
    

**2\. 分页展示**

使用 Element UI 的分页控件实现分页效果，在 AdminUser.vue 中引用 `<el-pagination>` 控件，如下：

    
    
    <el-pagination layout="prev, pager, next" @current-change="handleCurrentChange" :page-size="pagination.size" :total="pagination.total"
                        style="float:right;">
                    </el-pagination>
    

设置 layout，表示需要显示的内容，用逗号分隔，布局元素会依次显示。prev 表示上一页，next 为下一页，pager 表示页码列表，除此以外还提供了
jumper 和 total、size 和特殊的布局符号`->`。`->` 后的元素会靠右显示，jumper 表示跳页元素，total
表示总条目数，size 用于设置每页显示的页码数量。需要指定的参数如下：

    
    
                //分页参数
                    pagination: {
              // 总记录数
                        total: 0,
              // 当前页码
                        page: 1,
              // 每页显示的记录数
                        size: 10
                    },
    

将查询到的用户列表展示，注意将状态值、日期等字段进行格式转换：

    
    
    <!--列表-->
            <el-table :data="list" highlight-current-row stripe style="width: 100%;">
                <el-table-column prop="id" label="ID">
                </el-table-column>
                <el-table-column prop="email" label="用户名" sortable>
                </el-table-column>
          <!--通过 formatter 属性绑定自定义的格式转换方法-->
                <el-table-column prop="type" label="用户类型" :formatter="formatType" sortable>
                </el-table-column>
                <el-table-column prop="createTime" label="注册时间" sortable>
                    <template slot-scope="scope">
                        <i class="el-icon-time"></i>
                        <!--通过直接调用自定义函数实现格式转换-->
                        <span style="margin-left: 10px">{{ format.formatDate(scope.row.createTime) }}</span>
                    </template>
                </el-table-column>
            </el-table>
    

自定义的格式转换方法如下：

    
    
            formatType(row) {
                    return row.type == 0 ? '普通用户' : row.type == 100 ? '管理员' : '未知';
                }
    

**3\. 模糊查询**

设置值查询条件为按用户名（当前项目中将注册邮箱作为用户名）查询：

    
    
    <el-form :inline="true" :model="filters">
                        <el-form-item>
                            <el-input v-model="filters.name" placeholder="用户名"></el-input>
                        </el-form-item>
                        <el-form-item>
                            <el-button type="primary" v-on:click="handleGetUserList">查询</el-button>
                        </el-form-item>
                    </el-form>
    

查询数据时，传递查询条件：

    
    
    getUserList() {
                    req_getUserList(this.pagination.page, this.pagination.size, this.filters.name).then(response => {
                        let {
                            success,
                            data,
                            message
                        } = response;
                        if (success !== 0) {
                            this.$message.error(message);
                        } else {
                            this.list = data;
                        }
                    });
                }
    

### 导出用户列表

#### **功能分析**

用户管理页面的功能主要是显示用户列表，支持以不同条件查询用户列表，并将其导出为 Excel 表格。其中的关键点就是导出 Excel。目前导出 Excel
常用的有两种解决方案：Export2Excel.js 和 vue-json-excel，两种方案的特点如下：

**方案一** ：file-saver + xlsx + script-loader + Blob.js + Export2Excel.js

优点：

  * 支持多种 Excel 格式的读写
    * xls
    * xlsx
    * cvs
    * xlsm
  * 支持超大文件的读写
    * FireFox 浏览器下最大可支持 800MB
    * Chrome 浏览器下最大可支持 2GB
  * file-saver 作为单独的文件保存组件，可支持多种文件格式的保存
    * 从 URL 中保存文件
    * 将 canvas 保存为图片
    * 将 js 对象保存为文本文件

缺点：

  * 依赖多，安装繁琐（需要安装 5 个依赖包）

**方案二** ：vue-json-excel

优点：

  * 依赖少，仅需一个依赖包即可
  * 可满足基本的 Excel 导出需要，支持多种自定义操作

缺点：

  * 仅支持 JSON 导出为 Excel 功能，不支持读取 Excel 的功能

根据 KellerNotes
项目的特点，选择使用方案二实现，并详细讲解。同时，会在本篇源码中附上第一种方案的例子，但不做详细讲解，下一篇源码中将删除该例子。

#### 代码实现

**1\. 导出本页数据**

引入 vue-json-excel 组件：

    
    
    import JsonExcel from 'vue-json-excel';
    
        export default {
            components: {
                JsonExcel
            }
    }
    

导出本页数据时，定义 data 属性即可完成数据源的绑定，使用 name 属性定义要导出的文件名，如下：

    
    
    <!-- 导出本页数据时，数据已经存在，直接取 list 即可 -->
                    <JsonExcel :data="list" :fields="json_fields" name="用户信息.xls">
                        <el-button type="primary"> 导出本页数据</el-button>
                    </JsonExcel>
    

fields 属性定义要导出的列，格式如下：

    
    
    // 定义要导出的字段与数据格式，通过回调函数的方式对数据格式进行处理
                    json_fields: {
                        "用户ID": "id",
                        "注册邮箱": "email",
                        "用户类型": {
                            field: "type",
                            callback: (value) => {
                                return value === 0 ? "普通用户" : value === 100 ? "管理员" : "未知";
                            }
                        },
                        "注册时间": {
                            field: "createTime",
                            callback: (value) => {
                                return this.format.formatDate(value);
                            }
                        }
                    }
    

**2\. 导出所有数据**

导出全部数据时，需要从服务端获取数据，使用 fetch 回调方法指定获取数据的方法，并等待异步请求结束后返回获取到的数据：

    
    
    <JsonExcel :fetch="getUserListForDownLoad" :fields="json_fields" name="用户信息.xls" type="xls">
            <el-button type="primary"> 导出全部数据</el-button>
    </JsonExcel>
    

在 getUserListForDownLoad() 方法中读取数据：

    
    
    // 获取用户列表，获取到数据后返回给 JsonExcel 组件，用于导出成 excel，在这里将最大行数限制为 1000
                async getUserListForDownLoad() {
                    let userList = await req_getUserList(1, 1000, this.filters.name).then(response => {
                        let {
                            success,
                            data,
                            message
                        } = response;
                        if (success !== 0) {
                            this.$message.error(message);
                            return [];
                        } else {
                            return data;
                        }
                    });
                    return userList;
                }
    

### 效果测试

分页查询：

![分页查询](https://images.gitbook.cn/9418e3e0-9521-11ea-89e8-d3e59c640cba)

用户名模糊查询：

![用户名模糊查询](https://images.gitbook.cn/a9c21630-9521-11ea-b103-15b42d8d0dac)

导出的数据：

![导出的数据](https://images.gitbook.cn/bf1e39f0-9521-11ea-ba95-3ddc66291c5d)

### 源码地址

本篇完整的源码地址：

> <https://github.com/tianlanlandelan/KellerNotes>

### 小结

本篇带领大家实现了基本的用户管理功能，包括分页展示用户数据、按用户名模糊查询、导出为 Excel 表格等功能。

