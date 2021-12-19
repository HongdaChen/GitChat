### 前言

在正式开始学习本讲内容之前，先来思考一个问题，为什么要使用 Spring MVC
表单标签库？答案很简单，使用它是为了简化代码的开发，提高我们的开发效率，其实我们使用任何一款框架或者工具都是出于这个目的，为了实现快捷开发。

Spring MVC 表单标签库是如何来提高我们的开发效率呢？首先来做一个对比，业务场景：控制层返回业务数据到视图层，视图层需要使用 EL 将业务数据绑定到
JSP 页面表单中。

（1）实体类

    
    
    public class Student {
        private int id;
        private String name;
        private int age;
        private String gender;
    }
    

（2）Handler

    
    
    @Controller
    @RequestMapping("/hello")
    public class HelloHandler {
    
        @RequestMapping(value="/get")
        public String get(Model model) {
            Student student = new Student(1,"张三",22,"男");
            model.addAttribute("student", student);
            return "index";
        }
    
    }
    

（3）JSP

    
    
    <h1>修改学生信息</h1>
    <form action="" method="post">
         学生编号：<input type="text" name="id" value=${student.id } readonly="readonly"/><br/>
         学生姓名：<input type="text" name="name" value=${student.name } /><br/>
         学生年龄：<input type="text" name="age" value=${student.age } /><br/>
         学生性别：<input type="text" name="gender" value=${student.gender } /><br/>
         <input type="submit" value="提交"/><br/>
    </form>
    

使用 Spring MVC 表单标签可以直接将业务数据绑定到 JSP 表单中，非常简单。

（1）在 JSP 页面导入 Spring MVC 标签库，与导入 JSTL 标签库的语法非常相似，前缀 prefix 可以自定义，通常定义为 form。

    
    
    <%@ taglib prefix="form" uri="http://www.springframework.org/tags/form"%>
    

（2）将 form 表单与业务数据进行绑定，通过 modelAttribute 属性完成绑定，将 modelAttribute 的值设置为控制器向
model 对象存值时的 name 即可。

![](https://images.gitbook.cn/367c9950-9a1b-11e8-992f-9dfb28d2b53f)

![](https://images.gitbook.cn/3e7b1b90-9a1b-11e8-992f-9dfb28d2b53f)

（3）form 表单与业务数据完成绑定之后，接下来就是将业务数据中的值取出绑定到不同的标签中，通过设置标签的 path 属性完成，将 path
属性的值设置为业务数据对应的属性名即可。

![](https://images.gitbook.cn/47ce5860-9a1b-11e8-bd2f-43e393597943)

完整代码如下。

    
    
    <h1>修改学生信息</h1>
    <form:form modelAttribute="student">
         学生编号：<form:input path="id" readonly=""/><br/>
         学生姓名：<form:input path="name" /><br/>
         学生年龄：<form:input path="age" /><br/>
         学生性别：<form:input path="gender" /><br/>
         <input type="submit" value="提交"/><br/>
    </form:form>
    

重新运行程序，结果如下图所示。

![](https://images.gitbook.cn/57d63250-9a1b-11e8-8334-9bfa28241acd)

完成数据的绑定。

接下来我们详细讲解常用标签的使用方法。

（1）form 标签

    
    
    <form:form modelAttribute="student" method="post">
    

渲染的是 HTML 中的 `<form></form>`，通过 modelAttribute 属性绑定具体的业务数据。

（2）input 标签

    
    
    <form:input path="name" />
    

渲染的是 HTML 中的 `<input type="text"/>`，form 标签绑定的是业务数据，input 标签绑定的就是业务数据中的属性值，通过
path 与业务数据的属性名对应，并支持级联属性。

创建 Address 实体类。

    
    
    public class Address {
        private int id;
        private String name;
    }
    

修改 Student 实体类，添加 Address 属性。

    
    
    public class Student {
        private int id;
        private String name;
        private int age;
        private String gender;
        private Address address;
    }
    

模型对象添加 Address。

    
    
    @RequestMapping(value="/get")
    public String get(Model model) {
        Student student = new Student(1,"张三",22,"男");
        Address address = new Address(1,"科技路");
        student.setAddress(address);
        model.addAttribute("student", student);
        return "index";
    }
    

修改 JSP 页面，设置级联。

    
    
    <h1>修改学生信息</h1>
    <form:form modelAttribute="student">
         学生编号：<form:input path="id" readonly=""/><br/>
         学生姓名：<form:input path="name" /><br/>
         学生年龄：<form:input path="age" /><br/>
         学生性别：<form:input path="gender" /><br/>
         学生住址：<form:input path="address.name" /><br/>
         <input type="submit" value="提交"/><br/>
    </form:form>
    

（3）password 标签

    
    
    <form:password path="password" />
    

渲染的是 HTML 中的 `<input type="password"/>`，通过 path 与业务数据的属性名对应，password
标签的值不会在页面显示。

![](https://images.gitbook.cn/901eeb20-9a1b-11e8-8334-9bfa28241acd)

（4）checkbox 标签：

    
    
    <form:checkbox path="hobby" value="读书" />读书
    

渲染的是 HTML 中的 `<input type="checkbox"/>`，通过 path 与业务数据的属性名对应，可以绑定
boolean、数组和集合。

如果绑定 boolean 类型的变量，该变量值为 true，则表示选择，false 表示不选中。

    
    
    student.setFlag(true);
    
    checkbox：<form:checkbox path="flag" />
    

![](https://images.gitbook.cn/a55fbeb0-9a1b-11e8-992f-9dfb28d2b53f)

如果绑定数组或集合类型，集合中的元素等于 checkbox 的 vlaue 值，则该项选中，否则不选中。

    
    
    student.setHobby(Arrays.asList("读书","看电影","旅行"));
    
    <form:checkbox path="hobby" value="读书" />读书
    <form:checkbox path="hobby" value="看电影" />看电影
    <form:checkbox path="hobby" value="打游戏" />打游戏
    <form:checkbox path="hobby" value="听音乐" />听音乐
    <form:checkbox path="hobby" value="旅行" />旅行<br/>
    

![](https://images.gitbook.cn/b5faff00-9a1b-11e8-8334-9bfa28241acd)

（5）checkboxs 标签：

    
    
    <form:checkboxes items="${student.hobby }" path="selectHobby" />
    

渲染的是 HTML 中的一组 `<input type="checkbox"/>`，这里需要结合 items 和 path 两个属性来使用，items
绑定被遍历的集合或数组，path 绑定被选中的集合或数组，可以这样理解，items 为全部选型，path 为默认选中的选型。

    
    
    student.setHobby(Arrays.asList("读书","看电影","打游戏","听音乐","旅行"));
    student.setSelectHobby(Arrays.asList("读书","看电影"));
    
    <form:checkboxes items="${student.hobby }" path="selectHobby" />
    

需要注意的是 path 可以直接绑定业务数据的属性，items 则需要通过 EL 的方式从域对象中取值，不能直接写属性名。

![](https://images.gitbook.cn/cb1dc340-9a1b-11e8-9724-fb3ff7de9e38)

（6）radiobutton 标签

    
    
    <form:radiobutton path="radioId" value="0" />
    

渲染的是 HTML 中的一个 `<input type="radio"/>`，绑定的数据与标签的 value 值相等为选中状态，否则为不选中状态。

    
    
    student.setRadioId(1);
    
    <form:radiobutton path="radioId" value="0" />男
    <form:radiobutton path="radioId" value="1" />女
    

![](https://images.gitbook.cn/e76153f0-9a1b-11e8-9724-fb3ff7de9e38)

（7）radiobuttons 标签

    
    
    <form:radiobuttons items="${student.grade }" path="selectGrade" />
    

渲染的是 HTML 中的一组 `<input type="radio"/>`，这里需要结合 items 和 path 两个属性来使用，items
绑定被遍历的集合或数组，path 绑定被选中的值，可以这样理解，items 为全部选型，path 为默认选中的选型，用法与 `<form:checkboxs
/>` 一致。

    
    
    Map<Integer,String> gradeMap=new HashMap<Integer,String>();
    gradeMap.put(1, "一年级");
    gradeMap.put(2, "二年级");
    gradeMap.put(3, "三年级");
    gradeMap.put(4, "四年级");
    gradeMap.put(5, "五年级");
    gradeMap.put(6, "六年级");
    student.setGrade(gradeMap);
    student.setSelectGrade(3);
    
    <form:radiobuttons items="${student.grade }" path="selectGrade" />
    

![](https://images.gitbook.cn/fa9e8f50-9a1b-11e8-8334-9bfa28241acd)

（8）select 标签

    
    
    <form:select items="${student.citys }" path="selectCity" />
    

渲染的是 HTML 中的一个 `<select/>`，这里需要结合 items 和 path 两个属性来使用，items 绑定被遍历的集合或数组，path
绑定被选中的值，用法与 `<form:radiobuttons/>` 一致。

    
    
    Map<Integer,String> cityMap=new HashMap<Integer,String>();
    cityMap.put(1, "北京");
    cityMap.put(2, "上海");
    cityMap.put(3, "广州");
    cityMap.put(4, "深圳");
    cityMap.put(5, "西安");
    cityMap.put(6, "武汉");
    student.setCitys(cityMap);
    student.setSelectCity(5);
    
    <form:select items="${student.citys }" path="selectCity" />
    

![](https://images.gitbook.cn/13683b80-9a1c-11e8-9724-fb3ff7de9e38)

（9）form:select 结合 form:options 的使用，form:select 只定义 path 属性，在 form:select
标签内部添加一个子标签 form:options，设置 items 属性，获取被遍历的集合。

    
    
    <form:select path="selectCity">
        <form:options items="${student.citys }"></form:options>
    </form:select>
    

（10）form:select 结合 form:option 的使用，form:select 定义 path 属性，给每一个 form:option 设置
values 属性，path 与哪个 value 相等，该项默认选中。

    
    
    <form:select path="selectCity">
        <form:option value="5">杭州</form:option>
        <form:option value="6">成都</form:option>
        <form:option value="7">南京</form:option>
    </form:select>
    

![](https://images.gitbook.cn/29c26e50-9a1c-11e8-992f-9dfb28d2b53f)

（11）textarea 标签

渲染的是 HTML 中的一个 `<textarea/>`，path 绑定业务数据的属性值，作为文本输入域的默认值。

    
    
    student.setIntroduce("你好，我叫...");
    
    <form:textarea path="introduce"/>
    

![](https://images.gitbook.cn/3af9c790-9a1c-11e8-992f-9dfb28d2b53f)

（12）hidden 标签

渲染的是 HTML 中的一个 `<input type="hidden"/>`，path 绑定业务数据的属性值。

（13）errors 标签

该标签需要结合 Spring MVC 的 Validator 验证器来使用，将验证结果的 error 信息渲染到 JSP 页面中。

创建验证器，实现 Validator 接口。

    
    
    public class StudentValidator implements Validator{
    
        public boolean supports(Class<?> clazz) {
            // TODO Auto-generated method stub
            return Student.class.equals(clazz);
        }
    
        public void validate(Object target, Errors errors) {
            // TODO Auto-generated method stub
            ValidationUtils.rejectIfEmpty(errors, "name", null, "姓名不能为空"); 
            ValidationUtils.rejectIfEmpty(errors, "password", null, "密码不能为空");
        }
    
    }
    

springmvc.xml 中添加 Validator 配置。

    
    
    <mvc:annotation-driven validator="studentValidator"/>
    
    <bean id="studentValidator" class="com.southwind.validator.StudentValidator"/>
    

创建业务方法，第一个 login 方法用来生成业务数据，跳转到 login.jsp，第二个 login 方法用来做验证判断。

    
    
    @GetMapping(value = "/login")
    public String login(Model model){
         model.addAttribute(new Student());
         return "login";
    }
    
    @PostMapping(value="/login")
    public String register(@Validated Student student,BindingResult br) {
        if (br.hasErrors()) {
            return "login";
        }
        return "success";
    }
    

login.jsp

    
    
    <h1>修改学生信息</h1>
    <form:form modelAttribute="student" action="login" method="post">
        学生姓名：<form:input path="name" /><form:errors path="name"/><br/>
        学生密码：<form:password path="password" /><form:errors path="password"/><br/>
        <input type="submit" value="提交"/>
    </form:form>
    

运行，结果如下图所示，不输入学生姓名和学生密码，直接点击提交按钮。

![](https://images.gitbook.cn/5c4ca9d0-9a1c-11e8-8334-9bfa28241acd)

直接将错误信息绑定到 JSP 页面，结果如下图所示。

![](https://images.gitbook.cn/609e29a0-9a1c-11e8-992f-9dfb28d2b53f)

### 总结

本节课我们讲解了 Spring MVC 表单标签库的使用，Spring MVC 表单标签库相较于 JSP 标准标签库 JSTL
更加方便快捷，简化了很多操作，同时可以自动完成数据的绑定，将业务数据绑定到视图层的对应模块进行展示。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

[请单击这里下载源码](https://github.com/southwind9801/springmvc_taglib.git)

