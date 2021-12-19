### 前言

MyBatis 作为一个“半自动化”的 ORM 框架，需要开发者手动定义 SQL。在业务需求比较复杂的情况下，手动拼接 SQL
的工作量就会非常大，为了适用不同的情况，往往需要做很多重复性的工作，这种步骤繁琐的工作对于开发者来讲是很痛苦的，同时也容易出错。

比如我们定义一个方法，通过某些属性来查找 User 对象，User 定义如下所示。

    
    
    public class User{
        private Integer id;
        private String username;
        private String password;
        private Integer age;
    }
    

我们将这些属性封装成一个实例化对象作为参数传入到方法中，定义 3 种查询，分别是：

  * 通过 id 和 username 查询 User。
  * 通过 username 和 password 查询 User。
  * 通过 password 和 age 查询 User。

在 UserRepository 中定义以上 3 个方法，如下所示。

    
    
    public User findByUser1(User user);
    public User findByUser2(User user);
    public User findByUser3(User user);
    

对应的 UserRepository.xml 如下所示。

    
    
    <select id="findByUser1" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
      select * from t_user where id = #{id} and username = #{username}
    </select>
    
    <select id="findByUser2" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
      select * from t_user where username = #{username}
      and password = #{password}
    </select>
    
    <select id="findByUser3" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
      select * from t_user where password = #{password} and age = #{age}
    </select>
    

这样是可以完成需求的，但是我们会发现上述代码过于冗余，复用性很低，出现大量重复代码，3 条 SQL 中 `select * from t_user
where` 是完全一样的，区别在于 where 条件是不同属性的组合，如果能有一种机制可以动态判断参数是否包含某个属性，如果包含，则在 where
条件中追加该属性，否则不追加，开发就会方便很多。这样我们就可以把 3 条 SQL 合并成 1 条，UserRepository 中的 3 个方法也可以合并为
1 个方法，如何实现呢？

这就是我们本节课要学习的 MyBatis 动态 SQL，顾名思义，SQL 不是固定的，可以根据不同的参数信息来动态拼接不同的 SQL，以适应不同的需求。

MyBatis 动态 SQL 极大地简化了开发者的工作，开发者只需要定义一个具体的模版，然后添加特定的业务逻辑，MyBatis 框架就可以自动生成不同的
SQL 语句，以应对各种不同的需求，使用起来非常简单方便。

接下来我们就一起来学习如何使用 MyBatis 动态 SQL，我们通过对 User 对象的操作来举例说明。

1\. 创建数据表

    
    
    CREATE TABLE `t_user` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `username` varchar(11) DEFAULT NULL,
      `password` varchar(11) DEFAULT NULL,
      `age` int(11) DEFAULT NULL,
      PRIMARY KEY (`id`)
    )
    

2\. 创建 User 实体类

    
    
    public class User{
        private Integer id;
        private String username;
        private String password;
        private Integer age;
    }
    

3\. 创建 UserRepository

    
    
    public interface UserRepository {
        public User findByUser(User user);
    }
    

4\. 创建 UserRepository.xml

    
    
    <mapper namespace="com.southwind.repository.UserRepository">
    
        <select id="findByUser" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
            select * from t_user where id = #{id} and username = #{username}
            and password = #{password} and age = #{age}
        </select>
    
    </mapper>
    

5\. 在 resources 路径下创建 config.xml，添加各种配置

    
    
    <configuration>
    
        <!-- 设置 settings -->
        <settings>
            <!-- 打印 SQL -->
            <setting name="logImpl" value="STDOUT_LOGGING"/>
        </settings>
    
        <environments default="development">
            <environment id="development">
                <transactionManager type="JDBC" />
                <dataSource type="POOLED">
                    <property name="driver" value="com.mysql.jdbc.Driver" />
                    <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useUnicode=true&amp;characterEncoding=UTF-8" />
                    <property name="username" value="root" />
                    <property name="password" value="root" />
                </dataSource>
            </environment>
        </environments>
    
        <mappers>
            <mapper resource="com/southwind/repository/UserRepository.xml"/>
        </mappers>
    
    </configuration>
    

6\. 测试

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
            User user = new User();
            user.setId(1);
            user.setUsername("张三");
            user.setPassword("123");
            user.setAge(22);
            User user2 = userRepository.findByUser(user);
            System.out.println(user2);
        }
    }
    

查询结果如下所示。

![WX20190617-172126@2x](https://images.gitbook.cn/13c9db10-b140-11e9-8d9f-5d806c870c68)

参数 User 的属性完全匹配数据表中的记录，是可以查询出结果的。现在对代码进行修改，去掉 User 的 password 属性赋值操作，通过
id、username、age 三个字段去匹配，理想的结果是同样可以查询出数据表中的那条记录。

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream is = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(is);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
            User user = new User();
            user.setId(1);
            user.setUsername("张三");
            user.setAge(33);
            User user2 = userRepository.findByUser(user);
            System.out.println(user2);
        }
    }
    

结果如下图所示，与我们的预期完全不同。

![WX20190617-172214@2x](https://images.gitbook.cn/37d26770-b140-11e9-aadf-b32faf6b5e19)

为什么结果是 null 呢？因为 SQL 语句 where 条件使用的是 and 关键字进行连接，所有条件必须同时满足，我们可以看到此时的 SQL
语句如下所示：

    
    
    select * from t_user where id = 1 and username = "张三" and password = null and age = 33
    

很显然这条 SQL 语句是查询不出任何结果的，现在针对这种情况进行优化，判断 User 对象，如果 password 属性值不为 null，SQL
语句就添加 password 的判断，如果 password 属性为 null，则不添加。

我们可以使用动态 SQL 来完成上述操作。

### if 标签

    
    
    <select id="findByUser" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
          select * from t_user where
          <if test="id!=null">
            id = #{id}
          </if>
          <if test="username!=null">
            and username = #{username}
          </if>
          <if test="password!=null">
            and password = #{password}
          </if>
          <if test="age!=null">
            and age = #{age}
          </if>
    </select>
    

结构非常清晰，就是一个 if 流程控制，再次测试，结果如下图所示。

![WX20190617-172358@2x](https://images.gitbook.cn/55978f10-b140-11e9-aadf-b32faf6b5e19)

可以看到，成功查询出数据表中的对应记录，但是这种方式存在一个漏洞，对代码做如下修改，去掉 id 属性的赋值。

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream is = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(is);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            UserRepository useRepository = sqlSession.getMapper(UserRepository.class);
            User user = new User();
            user.setUsername("张三");
            user.setPassword("123");
            user.setAge(33);
            User user2 = useRepository.findByUser(user);
            System.out.println(user2);
        }
    }
    

运行结果如下图所示。

![WX20190617-172711@2x](https://images.gitbook.cn/5ff29ef0-b140-11e9-aadf-b32faf6b5e19)

程序抛出异常，并且是 SQL 语句错误，为什么会这样呢？我们来分析动态 SQL 代码，现在没有给 id 赋值，即 id==null，所以
"id=#{id}" 这段代码不会添加到 SQL 语句中，那么最终拼接好的动态 SQL 是这样的：

    
    
    select * from t_user where and username = ? and password = ? and age = ?
    

where 后面直接跟 and，很明显的语法错误，所以此时的代码还不够智能，应该根据 User 参数的属性值自动决定是否要添加 and 关键字，即当
"id=#{id}" 不出现在 SQL 语句中时，"and password=#{password}" 应该自动删除 and 关键字，如何做到呢？添加
where 标签即可。

### where 标签

    
    
    <select id="findByUser" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
            select * from t_user
            <where>
                <if test="id!=null">
                    id = #{id}
                </if>
                <if test="username!=null">
                    and username = #{username}
                </if>
                <if test="password!=null">
                    and password = #{password}
                </if>
                <if test="age!=null">
                    and age = #{age}
                </if>
            </where>
    </select>
    

再次测试，结果如下图所示。

![WX20190617-172830@2x](https://images.gitbook.cn/96854350-b140-11e9-83fd-07d7da80e621)

查询成功，SQL 语句自动删除了不需要的 and 关键字，所以一般 if 标签和 where 标签会组合起来使用。

### choose、when 标签

choose、when 标签和 if 标签用法很类似。

    
    
    <select id="findByUser" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
            select * from t_user 
            <where>
                <choose>
                    <when test="id!=null">
                        id = #{id}
                    </when>
                    <when test="username!=null">
                        and username = #{username}
                    </when>
                    <when test="password!=null">
                        and password = #{password}
                    </when>
                    <when test="age!=null">
                        and age = #{age}
                    </when>
                </choose>
            </where>
    </select>
    

### trim 标签

trim 标签中的 prefix 和 suffix 属性会被用于生成实际的 SQL 语句，会和标签内部的语句拼接。如果语句的前面或后面遇到
prefixOverrides 或 suffixOverrides 属性中指定的值，MyBatis
会自动将它们删除。在指定多个值的时候，别忘了每个值后面都要有一个空格，保证不会和后面的 SQL 连接在一起。

    
    
    <select id="findByUser" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
            select * from t_user 
            <trim prefix="where" prefixOverrides="and">
               <if test="id!=null">
                   id = #{id}
               </if>
               <if test="username!=null">
                   and username = #{username}
               </if>
               <if test="password!=null">
                   and password = #{password}
               </if>
               <if test="age!=null">
                   and age = #{age}
               </if>
            </trim>
    </select>
    

### set 标签

set 标签用于 Update 操作，会自动根据参数选择生成 SQL 语句。

UserRepository：

    
    
    public interface UserRepository {
        public int update(User user);
    }
    

UserRepository.xml：

    
    
    <update id="update" parameterType="com.southwind.entity.User">
            update t_user
            <set>
                <if test="username!=null">
                    username = #{username},
                </if>
                <if test="password!=null">
                    password = #{password},
                </if>
                <if test="age!=null">
                    age = #{age}
                </if>
            </set>
            where id = #{id}
    </update>
    

测试：

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream is = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(is);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
            User user = new User();
            user.setId(1);
            user.setUsername("大明明");
            user.setAge(22);
            System.out.println(userRepository.update(user));
            sqlSession.commit();
        }
    }
    

结果如下图所示。

![WX20190617-173006@2x](https://images.gitbook.cn/bb636f30-b140-11e9-9446-7bea49a7ddfe)

可以看到，User 参数只设置了 username 和 age 属性，所以动态生成的 SQL 语句就没有包含对 password 的修改。

### foreach 标签

foreach 标签可以迭代生成一系列值，这个标签主要用于 SQL 的 in 语句。

修改 User 实体类，添加一个 `List<Integer>` 类型的属性 ids。

    
    
    public class User{
        private Integer id;
        private String username;
        private String password;
        private Integer age;
        private List<Integer> ids;
    }
    

UserRepository.xml：

    
    
    <select id="findByUser" parameterType="com.southwind.entity.User" resultType="com.southwind.entity.User">
            select * from t_user 
            <where>
                <foreach collection="ids" open="id in (" close=")" item="id" separator=",">
                    #{id}
                </foreach>
            </where>
    </select>
    

测试：

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream is = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(is);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
            User user = new User();
            List<Integer> ids = new ArrayList<Integer>();
            ids.add(1);
            ids.add(2);
            ids.add(3);
            user.setIds(ids);
            List<User> list = userRepository.findByUser(user);
            for(User item : list){
                System.out.println(item);
            }
        }
    }
    

结果如下图所示。

![WX20190617-173323@2x](https://images.gitbook.cn/d74d4720-b140-11e9-83fd-07d7da80e621)

可以看到，根据 User 参数的 ids 属性中 id 的个数，动态生成了 SQL 语句，查询出结果。

### 总结

本节课我们讲解了 MyBatis 框架的动态 SQL 机制，MyBatis 作为一个“半自动化”的 ORM 框架，每一次操作都需要在 Mapper.xml
中定义 SQL 语句以及 POJO 的映射关系，使用这种方式进行开发，代码的扩展性较差，我们可以通过动态 SQL
机制来解决这一问题，提高代码的灵活性和扩展性。

[请点击这里查看源码](https://github.com/southwind9801/gcmybatis.git)

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

