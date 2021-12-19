### 前言

前面的课程主要学习了 Spring 框架 IoC 容器的相关理论和具体使用，Spring
作为一款非常优秀的框架，其编程思想和规范是非常值得我们去学习的。在自己现有的程度上，多去阅读优秀的源码，学习优秀的编程思想，对于开发者来讲，提升是非常大的。

而学习源码最有效的方式就是仿照源码自己动手写一个类似的框架，学习是要以输出为目标的，如果只是为了读源码而读源码，很可能读完之后脑子还是一片空白，根本没有学到什么有价值的东西。

带着输出的目标去学习就不一样了，你会有方向感，学习也就更具有针对性，如果你能自己动手写一个功能与源码类似的框架，不要求实现全部功能，核心流程实现即可，那么就可以认为真正掌握框架的使用了。

这一讲我们就一起来实现一个 Spring 的 IoC 容器，通过配置文件的形式自动创建对象。

核心技术有两点：XML 解析 + 反射，这两个技术点也是 Java 核心的重中之重，借着写框架的机会，我们也对这两个核心技术点做一个复习和巩固。

具体思路如下：

（1）根据需求编写 XML 文件，配置需要创建的 POJO。

（2）编写程序读取 XML 文件，获取 POJO 的相关信息，比如通过哪个类来创建？有哪些属性？属性值都是什么？

（3）根据第 2 步获取到的数据，结合反射机制动态创建对象，同时完成属性的赋值。

（4）将创建好的 POJO 存入 Map 集合，设置 key-value 存储结构，key 就是 XML 文件中 POJO 对应的 id 值，value
就是动态创建的 POJO。

（5）测试方法中先加载自定义 IoC 容器，然后就可以通过 id 值获取对象。

核心代码如下所示。

（1）XML 配置文件，完全按照 Spring 的配置文件格式来编写，没有任何区别。

    
    
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:context="http://www.springframework.org/schema/context"
           xmlns:p="http://www.springframework.org/schema/p"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd
    ">
    
        <bean id="address" class="com.southwind.entity.Address">
            <property name="id" value="1"></property>
            <property name="name" value="科技路"></property>
        </bean>
    
    </beans>
    

（2）自定义 ApplicationContext 接口，定义 getBean 方法，通过 id 返回 bean。

    
    
    public interface ApplicationContext {
        public Object getBean(String id);
    }
    

（3）自定义 ApplicationContext 接口的实现类 ClassPathXmlApplicationContext，完成以下工作。

  * 创建装置 bean 的集合，这里使用 Map 集合，因为其 key-value 的存储结构方便存取 bean。
  * 在 ClassPathXmlApplicationContext 类的构造函数中完成 IoC 的核心业务，解析 XML 配置文件，通过反射机制创建定义好的 bean，并保存在上一步创建的集合中。
  * 实现抽象方法 getBean，通过 id 从集合中找到对应的 bean 进行返回。

    
    
    public class ClassPathXmlApplicationContext implements ApplicationContext {
        private Map<String,Object> ioc = new HashMap<String, Object>();
        public ClassPathXmlApplicationContext(String path){
            try {
                SAXReader reader = new SAXReader();
                Document document = reader.read("./src/main/resources/"+path);
                Element root = document.getRootElement();
                Iterator<Element> iterator = root.elementIterator();
                while(iterator.hasNext()){
                    Element element = iterator.next();
                    String id = element.attributeValue("id");
                    String className = element.attributeValue("class");
                    //通过反射机制创建对象
                    Class clazz = Class.forName(className);
                    //获取无参构造函数，创建目标对象
                    Constructor constructor = clazz.getConstructor();
                    Object object = constructor.newInstance();
                    //给目标对象赋值
                    Iterator<Element> beanIter = element.elementIterator();
                    while(beanIter.hasNext()){
                        Element property = beanIter.next();
                        String name = property.attributeValue("name");
                        String valueStr = property.attributeValue("value");
                        String methodName = "set"+name.substring(0,1).toUpperCase()+name.substring(1);
                        Field field = clazz.getDeclaredField(name);
                        Method method = clazz.getDeclaredMethod(methodName,field.getType());
                        //根据成员变量的数据类型将 value 进行转换
                        Object value = null;
                        if(field.getType().getName() == "long"){
                            value = Long.parseLong(valueStr);
                        }
                        if(field.getType().getName() == "java.lang.String"){
                            value = valueStr;
                        }
                        if(field.getType().getName() == "int"){
                            value = Integer.parseInt(valueStr);
                        }
                        method.invoke(object,value);
                        ioc.put(id,object);
                    }
                }
            } catch (DocumentException e) {
                e.printStackTrace();
            } catch (ClassNotFoundException e){
                e.printStackTrace();
            } catch (NoSuchMethodException e){
                e.printStackTrace();
            } catch (InstantiationException e){
                e.printStackTrace();
            } catch (IllegalAccessException e){
                e.printStackTrace();
            } catch (InvocationTargetException e){
                e.printStackTrace();
            } catch (NoSuchFieldException e){
                e.printStackTrace();
            }
        }
    
        public Object getBean(String id) {
            return ioc.get(id);
        }
    }
    

这里重点梳理一下解析 spring.xml 并创建 bean 的逻辑。

spring.xml 如下所示。

    
    
    <bean id="address" class="com.southwind.entity.Address">
      <property name="id" value="1"></property>
      <property name="name" value="科技路"></property>
    </bean>
    

  * 加载 spring.xml 文件，用 dom4j 进行解析。

    
    
    SAXReader reader = new SAXReader();
    Document document = reader.read("./src/main/resources/"+path);
    Element root = document.getRootElement();
    Iterator<Element> iterator = root.elementIterator();
    while(iterator.hasNext()){
      ...
    }
    

  * while 循环内部完成对 bean 标签的解析，首先获取 bean 标签的 id 值和 class 值。

    
    
    String id = element.attributeValue("id");
    String className = element.attributeValue("class");
    

  * 通过反射机制获取 className 对应的运行时类，进而获取无参构造函数创建 bean。

    
    
    Class clazz = Class.forName(className);
    //获取无参构造函数，创建目标对象
    Constructor constructor = clazz.getConstructor();
    Object object = constructor.newInstance();
    

  * object 就是 bean 对象，但是现在它只是一个空对象，属性值都为 null 或者 0，因此接下来就是给 object 的各个属性赋值，具体操作方式是继续迭代 bean 标签，获取它的所有子标签 property，每一个 property 标签就对应 object 的一对属性值。

    
    
    Iterator<Element> beanIter = element.elementIterator();
    while(beanIter.hasNext()){
      ...
    }
    

  * while 循环内部完成对 property 标签的解析，获取 name 和 value 值，name 就表示当前所对应的属性名，value 就是对应的属性值，如何来完成赋值呢？思路是先通过 name 获取到属性对应的 setter 方法，然后调用 setter 方法并将 value 作为参数传入来完成赋值，通过反射机制获取到 name 对应属性的 setter 方法。

    
    
    String methodName = "set"+name.substring(0,1).toUpperCase()+name.substring(1);
    Field field = clazz.getDeclaredField(name);
    Method method = clazz.getDeclaredMethod(methodName,field.getType());
    

  * method 就是目标 setter 方法，接下来还有个问题，调用 setter 完成赋值时所传入参数的数据类型必须和方法定义的参数数据类型一致，但是现在我们获取到的 value 全部为 String 类型，这就需要做一个映射：根据当前属性的数据类型，对 value 进行数据类型转换，保证二者类型一致，具体操作如下。

    
    
    //根据成员变量的数据类型将 value 进行转换
    Object value = null;
    if(field.getType().getName() == "long"){
      value = Long.parseLong(valueStr);
    }
    if(field.getType().getName() == "java.lang.String"){
      value = valueStr;
    }
    if(field.getType().getName() == "int"){
      value = Integer.parseInt(valueStr);
    }
    

  * 这里只列举了 long、String、int 类型的映射，其他数据类型的转换逻辑是一样的，实现了数据类型转换。现在完成最后一步，通过反射机制调用动态生成的 setter 方法，完成属性赋值，同时将动态创建的 bean 存入集合中，bean 标签的 id 值作为 key。

    
    
    method.invoke(object,value);
    ioc.put(id,object);
    

  * 通过以上的操作，完成了 spring.xml 的解析，动态创建了定制 bean，同时提供了获取方法。

（4）测试类，使用自定义的 Application 和 ClassPathXmlApplicationContext 来加载 spring.xml，然后通过
id 获取 bean，使用方式与 Spring Framework 完全一致。

    
    
    public class Test {
        public static void main(String[] args) {
            ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
            Address address = (Address) applicationContext.getBean("address");
            System.out.println(address);
        }
    }
    

（5）运行结果如下图所示。

![](https://images.gitbook.cn/da227c90-a83f-11e9-b1f2-a974c71823fb)

### 总结

本讲带领大家一起学习了 Spring IoC 容器的底层机制，并且自己动手模仿 Spring Framework 完成了一个自定义 IoC
容器，功能上只是实现了 IoC
容器解析机制很小的一部分，但是它的核心原理我们应该清楚了。仿照源码自己动手写框架并不是要求我们实现框架的所有功能，这也不现实，主要目的是通过自己动手写代码搞清楚框架的核心机制，从而更好的应用框架。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

