### 前言

前面的内容集中对 Spring IoC 进行了详细讲解，之前说过，Spring 的两大核心机制是 IoC 和 AOP，本讲我们就一起来学习 AOP
的相关知识。

本章节源码：<https://github.com/southwind9801/gcspringaop>

AOP（Aspect Oriented
Programming）意为面向切面编程，我们所熟悉的是面向对象编程（OOP），将程序中所有参与模块都抽象成对象，然后通过对象之间的相互调用关系来完成需求。

AOP 是对 OOP
的一个补充，是在另外一个维度上抽象出对象，具体是指程序运行时动态地将非业务代码切入到业务代码中，从而实现代码的解耦合，将非业务代码抽象成一个对象，对该对象进行编程这就是面向切面编程思想，如下图所示。

![enter image description
here](https://images.gitbook.cn/f7ba69f0-a850-11e9-b2f7-c3e9ddc8f21b)

AOP 的优点：

  * 可大大降低模块之间的耦合性
  * 提高代码的维护性
  * 提高代码的复用性
  * 集中管理非业务代码，便于维护
  * 业务代码不受非业务代码的影响，逻辑更加清晰

说概念太空泛，不好理解，我们还是通过代码来直观感受什么是 AOP。

（1）创建一个计算器接口 Cal，定义四个方法：加、减、乘、除。

    
    
    public interface Cal {
        public int add(int num1,int num2);
        public int sub(int num1,int num2);
        public int mul(int num1,int num2);
        public int div(int num1,int num2);
    }
    

（2）创建接口实现类 CalImpl，实现四个方法。

    
    
    public class CalImpl implements Cal{
        @Override
        public int add(int num1, int num2) {
            // TODO Auto-generated method stub
            int result = num1+num2;
            return result;
        }
    
        @Override
        public int sub(int num1, int num2) {
            // TODO Auto-generated method stub
            int result = num1-num2;
            return result;
        }
    
        @Override
        public int mul(int num1, int num2) {
            // TODO Auto-generated method stub
            int result = num1*num2;
            return result;
        }
    
        @Override
        public int div(int num1, int num2) {
            // TODO Auto-generated method stub
            int result = num1/num2;
            return result;
        }
    }
    

（3）在测试方法中创建 CalImpl 对象，调用方法。

    
    
    public class Test {
        public static void main(String[] args) {
            Cal cal = new CalImpl();
            cal.add(10, 3);
            cal.sub(10, 3);
            cal.mul(10, 3);
            cal.div(10, 3);
        }
    }
    

以上这段代码很简单，现添加功能，在每一个方法执行的同时，打印日志信息：该方法的参数列表和该方法的计算结果。这个需求很简单，我们只需要在每一个方法体中，运算执行之前打印参数列表，运算结束之后打印计算结果即可，对代码做出如下修改。

    
    
    public class CalImpl implements Cal{
    
        @Override
        public int add(int num1, int num2) {
            // TODO Auto-generated method stub
            System.out.println("add方法的参数是["+num1+","+num2+"]");
            int result = num1+num2;
            System.out.println("add方法的结果是"+result);
            return result;
        }
    
        @Override
        public int sub(int num1, int num2) {
            // TODO Auto-generated method stub
            System.out.println("sub方法的参数是["+num1+","+num2+"]");
            int result = num1-num2;
            System.out.println("sub方法的结果是"+result);
            return result;
        }
    
        @Override
        public int mul(int num1, int num2) {
            // TODO Auto-generated method stub
            System.out.println("mul方法的参数是["+num1+","+num2+"]");
            int result = num1*num2;
            System.out.println("mul方法的结果是"+result);
            return result;
        }
    
        @Override
        public int div(int num1, int num2) {
            // TODO Auto-generated method stub
            System.out.println("div方法的参数是["+num1+","+num2+"]");
            int result = num1/num2;
            System.out.println("div方法的结果是"+result);
            return result;
        }
    
    }
    

再次运行代码，成功打印日志信息。

![enter image description
here](https://images.gitbook.cn/5636f390-a851-11e9-b2f7-c3e9ddc8f21b)

功能已经实现了，但是我们会发现这种方式业务代码和打印日志代码的耦合性非常高，不利于代码的后期维护。

如果需求改变，需要对打印的日志内容作出修改，那么我们就必须修改 4 个方法中的所有相关代码，如果是 100 个方法呢？每次就得需要手动去改 100
个方法中的代码。

换个角度去分析，会发现 4
个方法中打印日志信息的代码基本相同，那么有没有可能将这部分代码提取出来进行封装，统一维护呢？同时也可以将日志代码和业务代码完全分离开，实现解耦合。

按照这思路继续向下走，我们希望做的事情是把这 4
个方法的相同位置（业务方法执行前后）提取出来，形成一个横切面，并且将这个横切面抽象成一个对象，将所有的打印日志代码写到这个对象中，以实现与业务代码的分离。

这就是 AOP 的思想。

如何实现？使用动态代理的方式来实现。

我们希望 CalImpl 只进行业务运算，不进行打印日志的工作，那么就需要有一个对象来替代 CalImpl 进行打印日志的工作，这就是代理对象。

代理对象首先应该具备 CalImpl 的所有功能，并在此基础上，扩展出打印日志的功能，具体代码实现如下所示。

（1）删除 CalImpl 方法中所有打印日志的代码，只保留业务代码。

    
    
    public class CalImpl implements Cal{
    
        @Override
        public int add(int num1, int num2) {
            // TODO Auto-generated method stub
            int result = num1+num2;
            return result;
        }
    
        @Override
        public int sub(int num1, int num2) {
            // TODO Auto-generated method stub
            int result = num1-num2;
            return result;
        }
    
        @Override
        public int mul(int num1, int num2) {
            // TODO Auto-generated method stub
            int result = num1*num2;
            return result;
        }
    
        @Override
        public int div(int num1, int num2) {
            // TODO Auto-generated method stub
            int result = num1/num2;
            return result;
        }
    
    }
    

（2）创建 MyInvocationHandler 类，并实现 InvocationHandler 接口，成为一个动态代理类。

    
    
    public class MyInvocationHandler implements InvocationHandler{
        //委托对象
        private Object obj = null;
    
        //返回代理对象
        public Object bind(Object obj){
            this.obj = obj;
            return Proxy.newProxyInstance(obj.getClass().getClassLoader(), obj.getClass().getInterfaces(), this);
        }
    
        @Override
        public Object invoke(Object proxy, Method method, Object[] args)
                throws Throwable {
            // TODO Auto-generated method stub
            System.out.println(method.getName()+"的参数是:"+Arrays.toString(args));
            Object result = method.invoke(this.obj, args);
            System.out.println(method.getName()+"的结果是:"+result);
            return result;
        }
    
    }
    

bind 方法是 MyInvocationHandler 类提供给外部调用的方法，传入委托对象，bind 方法会返回一个代理对象，bind
方法完成了两项工作：

（1）将外部传进来的委托对象保存到成员变量中，因为业务方法调用时需要用到委托对象。

（2）通过 Proxy.newProxyInstance 方法创建一个代理对象，解释一下 Proxy.newProxyInstance 方法的参数：

  * 我们知道对象是 JVM 根据运行时类来创建的，此时需要动态创建一个代理对象的运行时类，同时需要将这个动态创建的运行时类加载到 JVM 中，这一步需要获取到类加载器才能实现，我们可以通过委托对象的运行时类来反向获取类加载器，obj.getClass().getClassLoader() 就是通过委托对象的运行时类来获取类加载器的具体实现；
  * 同时代理对象需要具备委托对象的所有功能，即需要拥有委托对象的所有接口，因此传入obj.getClass().getInterfaces()；
  * this 指当前 MyInvocationHandler 对象。

以上全部是反射的知识点。

  * invoke 方法：method 是描述委托对象所有方法的对象，agrs 是描述委托对象方法参数列表的对象。
  * method.invoke(this.obj,args) 是通过反射机制来调用委托对象的方法，即业务方法。

因此在 method.invoke(this.obj, args)
前后添加打印日志信息，就等同于在委托对象的业务方法前后添加打印日志信息，并且已经做到了分类，业务方法在委托对象中，打印日志信息在代理对象中。

测试方法中执行如下代码。

    
    
    public class Test {
        public static void main(String[] args) {
            //委托对象
            Cal cal = new CalImpl();
            MyInvocationHandler mh = new MyInvocationHandler();
            //代理对象
            Cal cal2 = (Cal) mh.bind(cal);
            cal2.add(10, 3);
            cal2.sub(10, 3);
            cal2.mul(10, 3);
            cal2.div(10, 3);
        }
    }
    

运行结果如下图所示。

![enter image description
here](https://images.gitbook.cn/1f919510-a852-11e9-b1f2-a974c71823fb)

成功，并且现在已经做到了代码分离，CalImpl 类中只有业务代码，打印日志的代码写在 MyInvocationHandler 类中。

以上就是通过动态代理实现 AOP 的过程，我们在使用 Spring 框架的 AOP 时，并不需要这么复杂，Spring
已经对这个过程进行了封装，让开发者可以更加便捷地使用 AOP 进行开发。

接下来学习 Spring 框架的 AOP 如何使用。

在 Spring 框架中，我们不需要创建 MyInvocationHandler 类，只需要创建一个切面类，Spring
底层会自动根据切面类以及目标类生成一个代理对象。

（1）创建切面类 LoggerAspect。

    
    
    @Aspect
    @Component
    public class LoggerAspect {
    
        @Before("execution(public int com.southwind.aspect.CalImpl.*(..))")
        public void before(JoinPoint joinPoint){
            //获取方法名
            String name = joinPoint.getSignature().getName();
            //获取参数列表
            String args = Arrays.toString(joinPoint.getArgs());
            System.out.println(name+"的参数是:"+args);
        }
    
        @After("execution(public int com.southwind.aspect.CalImpl.*(..))")
        public void after(JoinPoint joinPoint){
            //获取方法名
            String name = joinPoint.getSignature().getName();
            System.out.println(name+"方法结束");
        }
    
        @AfterReturning(value="execution(public int com.southwind.aspect.CalImpl.*(..))",returning="result")
        public void afterReturn(JoinPoint joinPoint,Object result){
            //获取方法名
            String name = joinPoint.getSignature().getName();
            System.out.println(name+"方法的结果是"+result);
        }
    
        @AfterThrowing(value="execution(public int com.southwind.aspect.CalImpl.*(..))",throwing="ex")
        public void afterThrowing(JoinPoint joinPoint,Exception ex){
            //获取方法名
            String name = joinPoint.getSignature().getName();
            System.out.println(name+"方法抛出异常："+ex);
        }
    
    }
    

LoggerAspect 类名处添加两个注解：

  * @Aspect，表示该类是切面类；
  * @Component，将该类注入到 IOC 容器中。

分别来说明类中的 4 个方法注解的含义。

    
    
    @Before("execution(public int com.southwind.aspect.CalImpl.*(..))")
    public void before(JoinPoint joinPoint){
      //获取方法名
      String name = joinPoint.getSignature().getName();
      //获取参数列表
      String args = Arrays.toString(joinPoint.getArgs());
      System.out.println(name+"的参数是:"+args);
    }
    

  * @Before：表示 before 方法执行的时机。execution(public int com.southwind.aspect.CalImpl.*(..))：表示切入点是 com.southwind.aspect 包下 CalImpl 类中的所有方法。即 CalImpl 所有方法在执行之前会首先执行 LoggerAspect 类中的 before 方法。
  * after 方法同理，表示 CalImpl 所有方法执行之后会执行 LoggerAspect 类中的 after 方法。
  * afterReturn 方法表示 CalImpl 所有方法在 return 之后会执行 LoggerAspect 类中的 afterReturn 方法。
  * afterThrowing 方法表示 CalImpl 所有方法在抛出异常时会执行 LoggerAspect 类中的 afterThrowing 方法。

因此我们就可以根据具体需求，选择在 before、after、afterReturn、afterThrowing 方法中添加相应代码。

（2）目标类也需要添加 @Component 注解，如下所示。

    
    
    @Component
    public class CalImpl implements Cal{
    
        @Override
        public int add(int num1, int num2) {
            // TODO Auto-generated method stub
            return num1+num2;
        }
    
        @Override
        public int sub(int num1, int num2) {
            // TODO Auto-generated method stub
            return num1-num2;
        }
    
        @Override
        public int mul(int num1, int num2) {
            // TODO Auto-generated method stub
            return num1*num2;
        }
    
        @Override
        public int div(int num1, int num2) {
            // TODO Auto-generated method stub
            return num1/num2;
        }
    
    }
    

（3）在 spring.xml 中进行配置。

    
    
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:aop="http://www.springframework.org/schema/aop"
        xmlns:context="http://www.springframework.org/schema/context"
        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd
            http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd">
    
        <!-- 自动扫描 -->
        <context:component-scan base-package="com.southwind.aspect"></context:component-scan>
    
        <!-- 使 Aspect 注解生效，为目标类自动生成代理对象 -->
        <aop:aspectj-autoproxy></aop:aspectj-autoproxy>
    
    </beans>
    

  * 将 com.southwind.aspect 包中的类扫描到 IoC 容器中。
  * 添加 aop:aspectj-autoproxy 注解，Spring 容器会结合切面类和目标类自动生成动态代理对象，Spring 框架的 AOP 底层就是通过动态代理的方式完成 AOP。

（4）测试方法执行如下代码，从 IoC 容器中获取代理对象，执行方法。

    
    
    public class Test {
        public static void main(String[] args) {
            //加载 spring.xml
            ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
            //获取代理对象
            Cal cal = (Cal) applicationContext.getBean("calImpl");
            //执行方法
            cal.add(10, 3);
            cal.sub(10, 3);
            cal.mul(10, 3);
            cal.div(10, 3);
        }
    }
    

运行结果如下图所示。

![enter image description
here](https://images.gitbook.cn/168f41f0-a853-11e9-a83a-d91106bfb0e3)

成功，结合代码，回过头来说几个概念更好理解。

  * **切面对象** ：根据切面抽象出来的对象，即 CalImpl 所有方法中需要加入日志的部分，抽象成一个切面对象 LoggerAspect。
  * **通知** ：切面对象的具体代码，即非业务代码，LoggerAspect 对象打印日志的操作。
  * **目标** ：被横切的对象，即 CalImpl 实例化对象，将通知加入其中。
  * **代理** ：切面对象、通知、目标混合之后的内容，即我们用 JDK 动态代理机制创建的对象。
  * **连接点** ：需要被横切的位置，即通知要插入业务代码的具体位置。

### 总结

本讲我们讲解了 Spring 的另外一个核心机制：AOP 的基本概念和具体使用，相比于 IoC，AOP
的概念更加抽象不好理解，但是它很重要，在实际开发中如事务管理就可以通过 AOP 来实现；同时还具体介绍了两种 AOP 的实现，一是通过 JDK
动态代理的方式自己来完成复杂的业务组装，二是借助于 Spring Framework 提供的 AOP 来更加简便的完成面向切面编程业务。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

