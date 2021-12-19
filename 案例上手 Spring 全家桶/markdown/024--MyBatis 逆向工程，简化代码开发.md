### 前言

通过前面课程的学习，我们了解到 MyBatis 是一个“半自动”的 ORM 框架，SQL 语句需要开发者自定义，这样做的好处是代码更加灵活，SQL
语句需要我们单独在 Mapper.xml 中定义，与 Mapper 接口相对应，使用 MyBatis 进行开发的基本配置是：

  * 实体类
  * Mapper 接口
  * Mapper.xml

如下图所示。

![](https://images.gitbook.cn/82cd2cc0-b26b-11e9-8e6b-6f3263a19060)

这种方式的缺陷是如果参与业务的表太多，每张表的业务都需要自定义 SQL、创建实体类、Mapper 接口，难免会很麻烦。

而且除了定义 SQL，创建实体类和 Mapper 接口的工作难度并不大，没有涉及太多的业务逻辑，属于简单的重复性工作，那么作为一个主流的 ORM
框架，MyBaits 有没有为我们提供解决方案呢？也就是说 MyBatis 能否自动根据数据表，帮我们生成实体类、Mapper 接口、Mapper.xml？

有的，通过我们本讲所要学习的逆向工程，就可以完成上述需求。

首先来了解一下什么是逆向工程。

逆向工程是 MyBatis 提供的一种自动化配置方案，针对数据表自动生成 MyBatis 所需要的各种资源，即实体类、Mapper
接口、Mapper.xml，但是逆向工程只针对于单表使用，如果数据表之间有级联，逆向工程无法自动生成级联关系，也就是说下面这种结构，无法用逆向工程自动创建。

    
    
    public class Student {
        private Long id;
        private String name;
        private Classes classes;
    }
    
    
    
    public class Classes {
        private Long id;
        private String name;
        private List<Student> students;
    }
    

如上所示，逆向工程可以帮助我们生成 Student 和 Classes，但是无法自动建立级联映射，即逆向工程自动生成的实体类只能是下面这种形式。

    
    
    public class Student {
        private Long id;
        private String name;
    }
    
    
    
    public class Classes {
        private Long id;
        private String name;
    }
    

因此实际开发中，我们需要结合 MyBatis 逆向工程自动生成的单表资源，手动补全级联映射，如下图所示，这也是 MyBatis 逆向工程的弊端之一。

![](https://images.gitbook.cn/929f9ed0-b26b-11e9-8c0d-bd8bdf7559ac)

同时 MyBatis
逆向工程的灵活性较差，它可以根据当前的数据表结构自动生成相关资源，但如果需求发生改变，需要对数据表结构进行修改，则之前自动创建的各种资源就不可再用，需要开发者手动删除，然后重新执行一次逆向工程的代码。

但是瑕不掩瑜，使用逆向工程，让 MyBatis 自动生成数据表对应的各种资源（实体类、Mapper 接口、Mapper.xml），可大大减少开发者的工作量。

了解完逆向工程的基本概念，接下来我们来学习具体操作。

MyBatis 实现逆向工程的组件是 MyBatis Generator，简称 MBG，是专为 MyBatis 框架定制的代码自动生成解决方案，MBG
可以根据数据表结构快速生成对应的 Mapper.xml、Mapper 接口以及实体类，并支持基本的 CRUD
操作，但是业务逻辑相对复杂的操作需要开发者手动完成。

MBG 具体使用步骤如下。

（1）创建 Maven 工程，在 pom.xml 中添加相关依赖。

    
    
    <dependencies>
      <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.4.5</version>
      </dependency>
    
      <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.11</version>
      </dependency>
    
      <dependency>
        <groupId>org.mybatis.generator</groupId>
        <artifactId>mybatis-generator-core</artifactId>
        <version>1.3.2</version>
      </dependency>
    </dependencies>
    

（2）创建目标数据表 t_account，MBG 根据该表结构自动生成相关资源。

![](https://images.gitbook.cn/9dfcdc70-b26b-11e9-8c0d-bd8bdf7559ac)

SQL 语句如下所示。

    
    
    CREATE TABLE `t_account` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `username` varchar(11) DEFAULT NULL,
      `password` varchar(11) DEFAULT NULL,
      `age` int(11) DEFAULT NULL,
      PRIMARY KEY (`id`)
    ) 
    

（3）创建 MBG 配置文件 generatorConfig.xml，核心配置有
jdbcConnection、javaModelGenerator、sqlMapGenerator、javaClientGenerator、table，作用如下所示。

  * jdbcConnection 配置数据库连接信息
  * javaModelGenerator 配置 JavaBean 的生成策略
  * sqlMapGenerator 配置 SQL 映射文件生成策略
  * javaClientGenerator 配置 Mapper 接口的生成策略
  * table 配置要逆向解析的数据表（tableName：表名，domainObjectName：对应的 JavaBean 名）

generatorConfig.xml 完整代码如下所示。

    
    
    <generatorConfiguration>
        <context id="testTables" targetRuntime="MyBatis3">
            <jdbcConnection
                    driverClass="com.mysql.cj.jdbc.Driver"
                    connectionURL="jdbc:mysql://localhost:3306/mybatis?useUnicode=true&amp;characterEncoding=UTF-8"
                    userId="root"
                    password="root"
            ></jdbcConnection>
            <javaModelGenerator targetPackage="com.southwind.entity" targetProject="./src/main/java"></javaModelGenerator>
            <sqlMapGenerator targetPackage="com.southwind.repository" targetProject="./src/main/java"></sqlMapGenerator>
            <javaClientGenerator type="XMLMAPPER" targetPackage="com.southwind.repository" targetProject="./src/main/java"></javaClientGenerator>
            <table tableName="t_account" domainObjectName="Account"></table>
        </context>
    </generatorConfiguration>
    

（4）创建 GeneratorMain 类，执行自动生成资源的代码。

    
    
    public class GeneratorMain {
        public static void main(String[] args) {
            List<String> warings = new ArrayList<String>();
            boolean overwrite = true;
            String genCig = "/generatorConfig.xml";
            File configFile = new File(Main.class.getResource(genCig).getFile());
            ConfigurationParser configurationParser = new ConfigurationParser(warings);
            Configuration configuration = null;
            try {
                configuration = configurationParser.parseConfiguration(configFile);
            } catch (IOException e) {
                e.printStackTrace();
            } catch (XMLParserException e) {
                e.printStackTrace();
            }
            DefaultShellCallback callback = new DefaultShellCallback(overwrite);
            MyBatisGenerator myBatisGenerator = null;
            try {
                myBatisGenerator = new MyBatisGenerator(configuration,callback,warings);
            } catch (InvalidConfigurationException e) {
                e.printStackTrace();
            }
            try {
                myBatisGenerator.generate(null);
            } catch (SQLException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    

（5）运行逆向工程代码之前，工程目录结构是这样的。

![](https://images.gitbook.cn/83d46600-acf9-11e9-b015-0198f673a736)

（6）运行 GeneratorMain 类的 main 方法，刷新工程，结构如下。

![](https://images.gitbook.cn/961232c0-acf9-11e9-bbe3-b33cf3f8d702)

（7）红框标注的部分为自动生成的资源，包括实体类，Mapper 接口以及 Mapper.xml，使用逆向工程自动生成代码测试成功，具体资源如下所示。

实体类：

    
    
    public class Account {
        private Integer id;
        private String username;
        private String password;
        private Integer age;
    }
    

MBG 还会为实体类配套生成一个 Example。

    
    
    public class AccountExample {
        protected String orderByClause;
        protected boolean distinct;
        protected List<Criteria> oredCriteria;
    }
    

该类的作用是用来简化查询操作的，比如我们要在查询操作中添加一些特性，就可以通过设置 Example 来实现，比如我们要执行如下所示的 SQL。

    
    
    select * from t_account where username = "张三" order by age desc;
    

使用 Example 来实现的代码如下所示。

    
    
    AccountExample accountExample = new AccountExample();
    accountExample.setOrderByClause("username desc");
    accountExample.setDistinct(false);
    AccountExample.Criteria criteria = accountExample.createCriteria();
    criteria.andUsernameEqualTo("张三");
    List<Account> accounts = accountMapper.selectByExample(accountExample);
    

Mapper 接口：

    
    
    public interface AccountMapper {
        int countByExample(AccountExample example);
        int deleteByExample(AccountExample example);
        int deleteByPrimaryKey(Integer id);
        int insert(Account record);
        int insertSelective(Account record);
            List<Account> selectByExample(AccountExample example);
        Account selectByPrimaryKey(Integer id);
        int updateByExampleSelective(@Param("record") Account record, @Param("example") AccountExample example);
        int updateByExample(@Param("record") Account record, @Param("example") AccountExample example);
        int updateByPrimaryKeySelective(Account record);
        int updateByPrimaryKey(Account record);
    }
    

Mapper.xml：

    
    
    <mapper namespace="com.southwind.repository.AccountMapper" >
      <resultMap id="BaseResultMap" type="com.southwind.entity.Account" >
        <id column="id" property="id" jdbcType="INTEGER" />
        <result column="username" property="username" jdbcType="VARCHAR" />
        <result column="password" property="password" jdbcType="VARCHAR" />
        <result column="age" property="age" jdbcType="INTEGER" />
      </resultMap>
      <sql id="Example_Where_Clause" >
        <where >
          <foreach collection="oredCriteria" item="criteria" separator="or" >
            <if test="criteria.valid" >
              <trim prefix="(" suffix=")" prefixOverrides="and" >
                <foreach collection="criteria.criteria" item="criterion" >
                  <choose >
                    <when test="criterion.noValue" >
                      and ${criterion.condition}
                    </when>
                    <when test="criterion.singleValue" >
                      and ${criterion.condition} #{criterion.value}
                    </when>
                    <when test="criterion.betweenValue" >
                      and ${criterion.condition} #{criterion.value} and #{criterion.secondValue}
                    </when>
                    <when test="criterion.listValue" >
                      and ${criterion.condition}
                      <foreach collection="criterion.value" item="listItem" open="(" close=")" separator="," >
                        #{listItem}
                      </foreach>
                    </when>
                  </choose>
                </foreach>
              </trim>
            </if>
          </foreach>
        </where>
      </sql>
    </mapper>
    

### 总结

本节课我们讲解了 MyBatis
逆向工程的具体使用，合理地使用逆向工程可以大大简化开发者的工作量，使代码变得更加简单，但是逆向工程也有自己的不足之处，扩展性不好，如果修改了数据表结构，所有自动生成的组件需要删除重新创建，实际工作中需要结合具体情况来选择使用。

[请点击这里查看源码](https://github.com/southwind9801/gcmbg.git)

