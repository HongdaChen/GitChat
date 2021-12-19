### 前言

上一篇，我们完成了 Spring Boot + MyBatis 项目的搭建，本篇将对 MyBatis 做深度的解析，构建自己的 MyBatis
工具类，为快捷高效的代码开发做准备。

#### **现状分析**

从这一篇开始正式写代码，按照既定的接口文档和数据库设计文档实现相应的业务逻辑。那么是不是接到开发任务就要赶紧写业务代码呢？

在实际开发项目时切忌闷头苦干，要思考如何快速高效地完成任务，一个资深程序员往往都有自己完整的开发工具，这些基于平时大量的项目积累。有了这些工具，看似繁杂的劳动可能他分分钟就做完了。接下来的几个小节我带大家构建属于自己的工具包，让你在以后的工作能高效、优质、优雅地完成自己的代码。

结合我们前边刚搭好服务端框架来说，Spring Boot + MyBatis 灵活方便，足够我们项目的需要，但MyBatis 在灵活的同时，有大量的 SQL
语句要写，这就成为部分程序员的痛点了，以后每个 Mapper 的代码都可能是这样的：

    
    
    @Mapper
    public interface UserMapper {
        @Select("select id, type, email, password, createTime, status, isDelete, updateTime, updateUserId from user_info")
        List<UserInfo> selectAll();
    
        @Select("select id, type, email, password, createTime, status, isDelete, updateTime, updateUserId from user_info where id = #{id}")
        UserInfo selectById(int id);
    
        @Select("Select id, type, email, password, createTime, status, isDelete, updateTime, updateUserId from user_info where email = #{email} ")
        UserInfo selectByEmail(String email);
    
        @Insert("Insert into user_info (id, type, email, password, createTime, status, isDelete) values(#{id}, #{type}, #{email}, #{password}, #{createTime}, #{status}, #{isDelete})")
        @Options(useGeneratedKeys=true,keyProperty = "id", keyColumn = "id")
        Integer insertAndReturnKey(UserInfo userInfo);
    }
    

#### **期望结果**

为了避免书写大量的 SQL，你可能在网上也查过如何用 MyBatis 通用 Mapper，然后也能找到一些第三方 jar
包。如果你的目标是初级工程师，以后想边写代码边百度的话，直接拿来用就可以。如果你想在技术上深入发展的话，还是要明白其原理，写一个属于自己的工具包比较好。

因此，我们期望有一个自己量身定制的通用 Mapper，在使用时，只需要新建一个 Mapper 继承通用 Mapper，即可使用常用的增删改查功能，如：

    
    
    @Mapper
    public interface UserInfoMapper extends BaseMapper<UserInfo> {
    
    }
    
    
    
    public void getAll(){
            List<UserInfo> list = userMapper.baseSelectAll(new UserInfo());
            for(UserInfo userInfo : list){
                Console.println(userInfo.getId() + "",userInfo);
            }
        }
    

本小节，我带大家实现一个功能强大、用法简单的通用 Mapper。

### 知识准备

MyBatis 的实现主要依靠注解、反射和泛型，通用 Mapper 也一样。另外，无论是什么样的通用 Mapper，实现原理都是一样的：

  1. 在实体类上通过注解标注表名和对应的数据库字段
  2. 运行时通过 Java 反射机制解析传入的实体类，解析出字段名、表名，构建 SQL 脚本并运行
  3. 使用泛型机制，将 SQL 脚本执行的结果映射成对象

下面简单介绍一下本小节用到的知识点。

#### **注解**

> Java 注解（Annotation）又称 Java 标注，是 JDK5.0 引入的一种注释机制。Java
> 语言中的类、方法、变量、参数和包等都可以被标注。和 Javadoc 不同，Java 标注可以通过反射获取标注内容。——来自百度百科

Spring Boot 和 MyBatis 中大量用到了注解，在 MyBatis 的 `package
org.apache.ibatis.annotations` 包中你会看到大量属性的身影：

![](https://images.gitbook.cn/f84e0000-7246-11ea-964d-61a29639fe46)

重点关注里面这几个注解，这些是完成通用 Mapper 的关键：

@InsertProvider：方法注解，通过该注解可以将一个方法 A 中的参数传入注解中指定类的方法 B 中，生成 SQL，执行 Insert
操作，并返回执行成功的条数。利用这个 InsertProvider 可以实现通用的 Insert 方法，如：

    
    
    @InsertProvider(type = BaseInsertProvider.class,method = "insert")
    <T extends BaseEntity> Integer baseInsert(T entity) throws DuplicateKeyException;
    

@Options：方法注解，能够设置缓存时间，能够为对象生成自增的主键值，配合 InsertProvider 注解可以实现插入记录并返回自增主键的功能。如：

    
    
        @InsertProvider(type = BaseInsertProvider.class,method = "insertAndReturnKey")
        @Options(useGeneratedKeys=true,keyProperty = "id", keyColumn = "id")
        <T extends BaseEntity> Integer baseInsertAndReturnKey(T entity) throws DuplicateKeyException;
    

@UpdateProvider：方法注解，通过该注解可以将一个方法 A 中的参数传入注解中指定类的方法 B 中，生成 SQL，执行 Update
操作，并返回更新成功的条数。如：

    
    
    @UpdateProvider(type = BaseUpdateProvider.class,method = "updateById")
    <T extends BaseEntity> Integer baseUpdateById(T entity);
    

@SelectProvider：方法注解，通过该注解可以将一个方法 A 中的参数传入注解中指定类的方法 B 中，生成 SQL，执行 Select
操作，并将结果映射成方法 A 指定的返回类型。如：

    
    
    @SelectProvider(type = BaseSelectProvider.class,method = "selectPageListByCondition")
    <T extends BaseEntity> List<K> baseSelectPageListByCondition(T entity);
    

@DeleteProvider：方法注解，通过该注解可以将一个方法 A 中的参数传入注解中指定类的方法 B 中，生成 SQL，执行 Delete
操作，并返回删除成功的条数。如：

    
    
    @DeleteProvider(type = BaseDeleteProvider.class,method = "deleteById")
    <T extends BaseEntity> Integer baseDeleteById(T entity);
    

#### **反射**

> 在 Java
> 程序运行状态中，对于任何一个类，都可以获得这个类的所有属性和方法；对于给定的一个对象，都能够调用它的任意一个属性和方法。这种动态获取类的内容以及动态调用对象的方法称为反射机制。——来自百度百科

通过反射机制，我们可以通过一个对象获取到它的如下数据：

  * 类：类名、类的完整路径、类上的注解……
  * 成员变量：变量名、变量类型、变量值、变量上的注解……
  * 方法：方法名、方法上的注解……

#### **泛型**

> Java 泛型是 J2SE 1.5 中引入的一个新特性，其本质是参数化类型，也就是说所操作的数据类型被指定为一个参数（type
> parameter）这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法。——来自百度百科

注解和反射的使用都离不开泛型，泛型给我们提供了在不知道传入参数具体类型的情况下进行后续操作的可能性。

### 方案分析

#### **Insert 语句**

MyBatis 常用的 Insert 语句格式如下：

    
    
    INSERT INTO user_info (email,password) VALUES (#{email},#{password});
    

对 Insert 语句进行拆解分析，可以发现，只要知道了表名、字段名，就可以使用 @InsertProvider 的方式生成一个对应的 SQL 脚本，如图：

![insert](https://images.gitbook.cn/0d2e15f0-7247-11ea-af73-378397689c85)

#### **Delete 语句**

MyBatis 常用的 Delete 语句格式如下：

    
    
    DELETE FROM user_info WHERE email = #{email}；
    

对 Delete 语句进行拆解分析，可以发现，想使用 @InsertProvider 的方式生成一个 Delete 脚本，需要知道表名和查询条件，如图：

![delete](https://images.gitbook.cn/15b63a90-7247-11ea-af73-378397689c85)

#### **Update 语句**

MyBatis 常用的 Update 语句格式如下：

    
    
    UPDATE user_info SET email = #{email} WHERE id = #{id};
    

对 Update 语句进行拆解分析，可以看到：要生成 Update 语句，需要知道表名、字段名、查询条件，如图：

![update](https://images.gitbook.cn/1dd256f0-7247-11ea-a207-7f4534b95cc3)

#### **Select 语句**

MyBatis 常用的 Select 语句格式如下：

    
    
    SELECT id,email FROM user_info WHERE  type = #{type} ORDER BY createTime DESC;
    

对 Select 语句进行拆解分析，可以看到：生成 Selete 语句需要知道表名、字段名、查询条件、排序条件。如图：

![select](https://images.gitbook.cn/26127610-7247-11ea-8377-13f07d2f46fb)

#### **总体方案**

经过以上分析发现，获取到表名、字段名、查询条件、排序条件后，就可以生成大多数常用的 SQL 语句了。

本小节开始时讲了到通用 Mapper 的实现原理，下面按照相同的实现原理，分 4 步实现 **MyBatis 通用 Mapper** ：

  1. 定义 **注解** ，在实体类中分别标注表名、字段名等关键信息
  2. 在使用时通过 **Java 反射机制** 获取到注解信息并对其进行解析，从而获取到表名、字段名等信息
  3. 使用获取到的关键信息生成 SQL 语句
  4. 执行 SQL 语句并返回执行结果

在实现过程中注意通过 **Java 泛型机制** 严格控制输入、输出的类型。

### 代码实现

#### **定义注解**

  * @TableAttribute：自定义一个类注解 @TableAttribute，用于标注表名
  * @FieldAttribute：自定义一个属性注解 @FieldAttribute，用于标注字段名
  * @IndexAttribute：查询条件通常要设置为索引，自定义一个属性注解 @IndexAttribute，用于标注索引字段，如果这个字段有值，就当做查询条件
  * @SortAttribute：自定义一个属性注解 @SortAttribute，用于标注排序字段

示例代码如下：

    
    
    /**
     * 表注解，类型为类注解，用在类的定义之前
     */
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface TableAttribute {
        /**
         * 表名
         * @return
         */
        String name() ;
    }
    
    
    
    /**
     * 字段名注解，属性注解，用于要查询的字段名之前
     */
    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface FieldAttribute {
    
    }
    

#### **引入注解**

注解定义完毕后，我们就可以使用反射机制读取到一个对象所对应的类中的所有注解，从而解析出实体类对应的表名、字段名等信息。在这里还存在一些问题：多个查询条件用
AND 还是 OR？排序方式是 DESC 还是 ASC？这些条件对于每个对象来说都是不一样的。解决方法就是——继承。

新建一个 BaseEntity
类，作为所有实体类的父类，该类定义了多条件查询的连接方式、排序方式、分页查询必须的参数等，实体类继承该类后，就可以使用这些参数完成复杂的查询了。BaseEntity
类具体代码如下：

    
    
    /**
     * @author yangkaile
     * @date 2019-07-17 16:59:52
     * BaseEntity，使用复杂查询（带条件的增删改查和分页查询）时需要继承的父类
     * 该类提供了可供选择的多条件查询方式、排序方式、分页查询相关参数等
     * 数据实体类继承该类即可使用
     *
     */
    public class BaseEntity {
        /**
         * 是否查询明细字段
         */
        private boolean baseKyleDetailed = true;
        /**
         * 多个查询条件是否用And连接
         */
        private Boolean baseKyleUseAnd = true;
        /**
         * 是否按排序关键字升序排列
         */
        private Boolean baseKyleUseASC = true;
        /**
         * 页面大小
         */
        private int baseKylePageSize = 10;
        /**
         * 要查询的页码
         */
        private int baseKyleCurrentPage = 1;
        /**
         * 根据页面大小和要查询的页码计算出的起始行号
         */
        private int baseKyleStartRows ;
     }
    

实体类继承 BaseEntity 类，并将相应的注解引入，代码如下：

    
    
    //引入 @TableAttribute 注解，标注 UserInfo 类对应的是 user_info 表
    @TableAttribute(name = "user_info")
    public class UserInfo extends BaseEntity{
    
        // 标注 id 是对应一个数据库字段，字段名和属性同名
        @FieldAttribute
        private int id;
    
        @FieldAttribute
          // 标注 email 字段是索引，可以作为查询条件使用
        @IndexAttribute
        private String email;
    
        @FieldAttribute
        // 标注 createTime 字段可以作为排序条件使用
        @SortAttribute
        private Date createTime = new Date();
        ......
    }
    

#### **解析注解**

新建一个类 SqlFieldReader.java 用于读取实体类上的注解。首先解析表名：要求传入一个继承了 BaseEntity
类的实体对象，通过反射，获取到该对象对应的类，解析出类上的 @TableAttribute 注解里的 name 参数，也就是表名。具体代码如下：

    
    
    /**
     * 读取表名，要求类上有@TableAttribute注解
     * @param entity 实体对象
     * @return tableName
     */
    public static <T extends BaseEntity> String getTableName(T entity){
        TableAttribute table = entity.getClass().getAnnotation(TableAttribute.class);
        if(table == null){
            return null;
        }
        return table.name();
    }
    

接下来解析字段名，同样是使用泛型和反射机制。由于实体类的字段可能很多，所以在设计时可以考虑：如果一个实体类中的所有属性都是数据库字段，允许该类的属性不使用
@FieldAttribute 注解，这时将所有属性名都当做数据库字段处理。在这种考虑下，就有了如下的代码：

    
    
    /**
     * 获取所有字段列表
     * 读取类中带@FieldAttribute注解的字段，如果都没有带该注解，则返回类中所有字段
     * @param cls 实体类型
     * @return {id,name}
     */
    public static <T extends BaseEntity> String getFieldStr(T entity){
            Class cls = entity.getClass();
            Field[] fields = cls.getDeclaredFields();
            //带@FieldAttribute注解的属性名
            StringBuilder builder = new StringBuilder();
            //所有属性名
            StringBuilder allFields = new StringBuilder();
            for(Field field:fields){
                allFields.append(field.getName()).append(",");
                if(field.getAnnotation(FieldAttribute.class) != null){
                    FieldAttribute fieldAttribute = field.getAnnotation(FieldAttribute.class);
                    //如果查询明细字段，返回明细字段
                    if(entity.isBaseKyleDetailed()){
                        builder.append(field.getName()).append(",");
                        //如果不查询明细字段，不返回明细字段
                    }else {
                        if(!fieldAttribute.detailed()){
                            builder.append(field.getName()).append(",");
                        }
                    }
    
                }
            }
            if(builder.length() > 0){
                return builder.substring(0,builder.length() - 1);
            }else if(allFields.length() > 0){
                return allFields.substring(0,allFields.length() - 1);
            }else {
                return  null;
            }
        }
    

getFieldStr 方法将字段以英文逗号拼接成一个字符串，用于 Select 场景，还有一个 getFieldList 方法返回的是
list。在这个方法设计时要考虑它的使用场景：可能是 SelectById 这种查询单条记录的场景，也可能是 SelectAll
这种查询列表的场景。在查询列表时，有些字段可能不适合显示，如：敏感字段（密码等）、大字段（文章详情等），因此可以在 @FieldAttribute
注解上设计一个属性 detailed，不适合显示的字段就标注一下。在 BaseEntity 类中添加属性
baseKyleDetailed，在进行查询时，可以控制是否查询这些字段。这样就很大程度上增加了通用 Mapper
的灵活性。当然，灵活的同时不能设计过度，如果需要特殊查询某几个字段，就单独书写 SQL 语句就好了，通用 Mapper 要简单易用，不能让使用太过复杂。

其他几个注解的解析方式和 @FieldAttribute 的相同，在文末的链接中有完整的源码，在这里就不一一赘述了。

#### **生成 SQL**

下面以 SelectById 方法为例，说明如何将解析出来的注解拼装成完整的 SQL 语句。

新建一个 BaseSelectProvider.java 类，该类负责所有 Select 语句的生成。SelectById 语句格式：

    
    
    SELECT 字段名 FROM 表名 WHERE id=#{id}
    

分析语句可以发现，`SELECT 字段名 FROM 表名` 可以作为所有 Select 语句的前缀，可以为此单独写一个方法
**getSelectPrefix** 。

另外，Select 方法相对来说比较常用，且对于一个实体类来说，Select
前缀是固定的，为此，可以设计缓存用来存储前缀，以提高程序运行效率，避免每次都重新解析注解。在这里设计了两个 ConcurrentHashMap
分别存储查询所有字段的前缀和查询一般字段（排除掉不显示字段后的所有字段）的前缀，SelectById 完整代码如下：

    
    
     // 查询所有字段的前缀
     public static Map<String,String> selectPrefixWithDetailedMap = new ConcurrentHashMap<>(16);
     // 查询一般字段的前缀
     public static Map<String,String> selectPrefixMap = new ConcurrentHashMap<>(16);
    
    /**
     * 根据ID 查询数据
     * @param entity 实体对象
     * @param <T> 实体类型
     * @return SELECT id,name... FROM route WHERE id = #{id}
     */
    public static <T extends BaseEntity> String selectById(T entity){
        String sql = getSelectPrefix(entity) + " WHERE id = #{id}";
        Console.info("selectById",sql,entity);
        return sql;
    }
    
    /**
    * 获取通用查询前缀
    * @param entity 实体类类型
    * @return SELECT 所有字段 FROM 表名
    */
    private static <T extends BaseEntity> String getSelectPrefix(T entity){
       String className = entity.getClass().getName();
       String sql;
       // 如果要显示明细字段，查询中包含明细字段
       if(entity.isBaseKyleDetailed()){
           sql = selectPrefixWithDetailedMap.get(className);
       // 如果不显示明细字段，查询中不包含明细字段  
       }else {
           sql = selectPrefixMap.get(className);
       }
       if(StringUtils.isEmpty(sql)){
           sql = "SELECT " + SqlFieldReader.getFieldStr(entity) + " FROM " + SqlFieldReader.getTableName(entity) + " ";
          if(entity.isBaseKyleDetailed()){
               selectPrefixWithDetailedMap.put(className,sql);
           }else {
               selectPrefixMap.put(className,sql);
           }
       }
       return sql;
    }
    

#### **映射结果**

新建 BaseMapper.java 接口类，代码如下：

    
    
    // 使用泛型接口，指定要实现的实体类必须继承 BaseEntity 类
    public interface BaseMapper<T extends BaseEntity> {
        // 使用 @SelectProvider 注解将 BaseSelectProvider 类中的 selectById 方法绑定到 baseSelectById 方法上
        @SelectProvider(type= BaseSelectProvider.class,method = "selectById")
        // baseSelectById 方法接收一个实体类对象，
        T baseSelectById(T entity);
    }
    

至此，通用的 SelectById
方法创建完毕，可以在项目中使用了。在文末的源码中，有完成的增删改查，包括分页查询、条件查询等完整的实现代码，希望大家能将其作为参考，自己实现一遍。

### 效果测试

将 UserMapper 继承 BaseMapper，并指定具体类型为 UserInfo，代码如下：

    
    
    @Mapper
    // 接口 UserMapper 继承 BaseMapper ,并指定具体类型为 UserInfo
    public interface UserMapper extends BaseMapper<UserInfo> {
    
    }
    

新建测试类 UserMapperTest.java，测试通用 Mapper 的执行情况，代码如下：

    
    
    @SpringBootTest
    class UserMapperTest {
    
        @Resource
        private UserMapper userMapper;
    
        @Test
        void contextLoads() {
        }
    
       /**
       * 测试通用 Mapper 的 baseInsertAndReturnKey 方法
       */
        @Test
        public void initTestData(){
            for(int i = 0 ; i < 10 ; i++){
                UserInfo userInfo = new UserInfo();
                userInfo.setType(1);
                userInfo.setEmail(StringUtils.getAllCharString(10));
                userInfo.setPassword("123456");
                userMapper.baseInsertAndReturnKey(userInfo);
                Console.println(userInfo.getId() + "",userInfo);
            }
        }
    
       /**
       * 测试通用 Mapper 的 baseSelectAll 方法
       */
        @Test
        public void getAll(){
            List<UserInfo> list = userMapper.baseSelectAll(new UserInfo());
            for(UserInfo userInfo : list){
                Console.println(userInfo.getId() + "",userInfo);
            }
        }
    }
    

运行结果如下图：

![getAll](https://images.gitbook.cn/73dd0310-7247-11ea-964d-61a29639fe46)

![insertUserInfo](https://images.gitbook.cn/7add9d50-7247-11ea-b39b-4d2cde13353b)

### 源码地址

本篇完成的源码地址如下：

>
> <https://github.com/tianlanlandelan/KellerNotes/tree/master/7.MyBatis通用Mapper/server>

### 小结

本篇通过对常用 SQL 语句的分析总结及 Java 相关的知识点，提出了自己设计通用 Mapper
的可行性，并对其进行了详细的技术介绍与实现。完成本篇的学习，可以让大家对通用 Mapper 有深入的理解，并可以灵活设计开发属于自己的 MyBatis
工具包。

