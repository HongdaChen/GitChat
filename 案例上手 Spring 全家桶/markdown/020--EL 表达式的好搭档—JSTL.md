### 前言

上一讲介绍了 EL 表达式的使用，可以简化 JSP 页面的代码开发，但实际上 EL
表达式也有自己的缺陷，只能做展示，不能编写动态功能，比如集合的遍历，为了解决这一问题，JSP 提供了 JSTL 组件供开发者使用，因此通常情况下 JSP
页面的开发为 EL + JSTL，这一讲我们就来详细学习 JSTL 的使用。

### 什么是 JSTL

JSP Standard Tag Library，简称 JSTL，JSP 标准标签库，提供了一系列的标签供开发者在 JSP 页面中使用，编写各种动态功能。

### 常用 JSTL 标签库

JSTL 提供了很多标签，以库为单位进行分类，实际开发中最常用的标签库有 3 种。

  * 核心标签库
  * 格式化标签库
  * 函数标签库

### 如何使用 JSTL

（1）pom.xml 添加 JSTL 依赖。

    
    
    <dependency>
        <groupId>jstl</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>
    

（2）JSP 页面导入 JSTL 标签库，prefix 设置前缀，相当于别名，在 JSP 页面中可以直接通过别名使用标签，uri 设置对应的标签库。

    
    
    <%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
    

（3）使用标签完成功能。

### 核心标签库

引入核心标签库。

    
    
    <%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
    

set：向域对象中添加一个数据。

var 为变量名，value 为变量值，scope 为保存的域对象，默认为 pageContext。

    
    
    <c:set var="message" value="张三" scope="request"/>
    ${requestScope.message }
    

运行结果如下图所示。

![](https://images.gitbook.cn/cb1925c0-9ad3-11e8-b37c-dd4feba3837e)

修改对象的属性值。

property 为属性名，value 为属性值，target 为对象名，注意这里需要使用 EL 表达式。

    
    
    <%
        Reader reader = new Reader();
        request.setAttribute("reader", reader);
     %>
     <c:set value="张三" property="name" target="${reader}"></c:set>
     <c:set value="1" property="id" target="${reader}"></c:set>
     ${reader }
    

运行结果如下图所示。

![](https://images.gitbook.cn/d9aada70-9ad3-11e8-a178-519d5b470954)

out：输出域对象中的数据。

value 为变量名，default 的作用是若变量不存在，则输出 default 的值。

    
    
    <c:set var="message" value="张三" scope="request"/>
    <c:out value="${message }" default="未定义"></c:out>
    

运行结果如下图所示。

![](https://images.gitbook.cn/e60563d0-9ad3-11e8-b37c-dd4feba3837e)

    
    
    <c:out value="${message }" default="未定义"></c:out>
    

运行结果如下图所示。

![](https://images.gitbook.cn/f3e04ba0-9ad3-11e8-b37c-dd4feba3837e)

remove：删除域对象中的数据。

var 为要删除的变量名。

    
    
    <c:set var="info" value="Java"></c:set>
    <c:remove var="info"/>
    <c:out value="${info }" default="未定义"/>
    

运行结果如下图所示。

![](https://images.gitbook.cn/07da6e60-9ad4-11e8-831e-0180aea56660)

catch：捕获异常。

若 JSP 的 Java 脚本抛出异常，会直接将异常信息打印到浏览器。

    
    
    <%
       int num = 10/0;
    %>
    

运行结果如下图所示。

![](https://images.gitbook.cn/1d8611b0-9ad4-11e8-a178-519d5b470954)

使用 catch 可以将异常信息保存到域对象中。

    
    
    <c:catch var="error">
    <%
        int num = 10/0;
    %>
    </c:catch>
    <c:out value="${error }"></c:out>
    

运行结果如下图所示。

![](https://images.gitbook.cn/30052d30-9ad4-11e8-8cbe-ad3f3badcc18)

if ：流程控制。

test 为判断条件，如果条件成立，会输出标签内部的内容，否则不输出。

    
    
    <c:set value="1" var="num1"></c:set>
    <c:set value="2" var="num2"></c:set>
    <c:if test="${num1>num2}">${num1 } > ${num2 }</c:if>
    <c:if test="${num1<num2}">${num1 } < ${num2 }</c:if>
    

运行结果如下图所示。

![](https://images.gitbook.cn/46d0e680-9ad4-11e8-831e-0180aea56660)

choose：流程控制。

相当于 if-else 的用法，when 相当于 if，otherwise 相当于 else。

    
    
    <c:set value="1" var="num1"></c:set>
    <c:set value="2" var="num2"></c:set>
    <c:choose>
         <c:when test="${num1>num2}">${num1 }</c:when>
         <c:otherwise>${num2 } </c:otherwise>   
    </c:choose>
    

运行结果如下图所示。

![](https://images.gitbook.cn/5e8cf340-9ad4-11e8-b37c-dd4feba3837e)

forEach：迭代集合。

    
    
    <%
       Reader reader1 = new Reader();
       reader1.setId(1);
       reader1.setName("张三");
       Reader reader2 = new Reader();
       reader2.setId(2);
       reader2.setName("李四");
       Reader reader3 = new Reader();
       reader3.setId(3);
       reader3.setName("王五");
       List<Reader> list = new ArrayList<Reader>();
       list.add(reader1);
       list.add(reader2);
       list.add(reader3);
       request.setAttribute("list", list);
       Map<Integer,Reader> map = new HashMap<Integer,Reader>();
       map.put(1, reader1);
       map.put(2, reader2);
       map.put(3, reader3);
       request.setAttribute("map", map);
       Set<Reader> set = new HashSet<Reader>();
       set.add(reader1);
       set.add(reader2);
       set.add(reader3);
       request.setAttribute("set", set);
    %>
    <c:forEach items="${list }" var="reader">
         ${reader }<br/>
    </c:forEach>
    <hr/>
    <c:forEach items="${map }" var="reader">
           ${reader }<br/>
    </c:forEach>
    <hr/>
    <c:forEach items="${set}" var="reader">
           ${reader }<br/>
    </c:forEach>
    

运行结果如下图所示。

![](https://images.gitbook.cn/9530aea0-9ad4-11e8-8cbe-ad3f3badcc18)

### 格式化标签库

可以将日期和数字按照一定的格式进行格式化输出。

引入格式化标签库：

    
    
    <%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
    

日期格式化：

    
    
    <%
       pageContext.setAttribute("date", new Date());
    %>
    <c:out value="${date }"></c:out><br/>
    <fmt:formatDate value="${date }"/><br/>
    <fmt:formatDate value="${date }" pattern="yyyy-MM-dd hh:mm:ss" />
    

运行结果如下图所示。

![](https://images.gitbook.cn/b35d98c0-9ad4-11e8-b37c-dd4feba3837e)

数字格式化：

value 为数值，maxIntegerDigits 为整数位，maxFractionDigits 为小数位。

    
    
    <fmt:formatNumber value="32165.23434" maxIntegerDigits="2" maxFractionDigits="3"></fmt:formatNumber>
    

运行结果如下图所示。

![](https://images.gitbook.cn/be8ee460-9ad4-11e8-b37c-dd4feba3837e)

### 函数标签库

引入函数标签库。

    
    
    <%@ taglib prefix="fn" uri="http://java.sun.com/jsp/jstl/functions" %>
    

函数标签库的使用与其他标签稍有区别，类似与 EL 表达式。

    
    
    <%
       pageContext.setAttribute("info", "Java,C");
    %>
    判断是否包含"Ja"：${fn:contains(info,"Java") }<br/>
    判断是否以"Ja"开始：${fn:startsWith(info,"Ja") }<br/>
    判断是否以"C"结尾：${fn:endsWith(info,"C") }<br/>
    求"va"的下标：${fn:indexOf(info,"va") }<br/>
    "Java"替换为"JavaScript"：${fn:replace(info,"Java","JavaScript") }<br/>
    截取：${fn:substring(info,2,3) }<br/>
    以","分割：${fn:split(info,",")[1] }
    

运行结果如下图所示。

![](https://images.gitbook.cn/d6262b10-9ad4-11e8-a178-519d5b470954)

### 总结

本节课我们讲解了 JSTL（JSP Standard Tag Library，JSP 标准标签库），它和 EL
表达式一样也是作用于视图层的组件，通常情况下两个组件会配合起来使用，JSTL 负责完成模式数据的逻辑处理，如遍历集合、判断等，EL
只负责展示结果，二者相结合的方式可以大大简化 JSP 的代码开发。

### 分享交流

我们为本课程付费读者创建了微信交流群，以方便更有针对性地讨论课程相关问题。入群方式请添加小助手的微信号：GitChatty5，并注明「全家桶」。

阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

> 温馨提示：需购买才可入群哦，加小助手微信后需要截已购买的图来验证~

