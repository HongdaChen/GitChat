### 前言

上一篇我们完成了 MyBatis 通用 Mapper 的开发，在掌握了 Java 反射、泛型机制后，发现我们能做的远远不止于此……

#### **现状分析**

使用通用 Mapper 能节省大量的 SQL 书写时间，能满足开发过程中对于持久层大多数的需求。但是作为一个全栈开发工程师，目标不仅仅于此。

工作中，往往需要自己设计数据表，写实体类；修改数据表，修改实体类，总感觉花费很多时间在这样重复的事情上。如果你喜欢先设计数据表，然后根据数据表生成 Java
代码，可以使用代码生成器，或者根据项目需要手写一个代码生成器。如果你更喜欢写 Java 代码而不喜欢写建表脚本，就可以根据 Java 实体类生成建表脚本。

#### **期望结果**

如果在上一篇《MyBatis 通用
Mapper》的基础上进行适当的扩展，就能达到调用一个方法即可生成一段标准的建表脚本或者直接创建好数据表，岂不是很不错的选择，比如说这样子：

    
    
    public void createUserInfoTable(){
        userMapper.baseCreate(new UserInfo());
    }
    

本篇就带大家实现这样的功能，是不是感觉酷酷的 ^_^。

### 知识准备

#### **类型转换**

要想实现根据 Java 实体类生成 SQL 脚本，重点和难点是进行合适的数据类型转换，下面作者总结了在 Java 实体类中常用的类型与 MySQL
类型间的对应关系：

Java 类型 | MySQL 类型 | 说明  
---|---|---  
String | varchar/text/mediumtext/longtext | varchar/text 最大 64Kb，mediumtext
最大长度 16Mb，longtext 最大 4Gb  
int/Integer | int |  
long/Long | bigint |  
float/Float | decimal |  
double/Double | decimal |  
byte[]/Byte[] | varbinary/blob | varbinary 允许最大长度为 65535，超过后自动转换为 blob 类型  
Date | datetime |  
  
根据这样的对应关系，我们尝试着用代码实现 Java 类型到 MySQL 类型间的转换。

新建类 TypeCaster.java，专门处理类型转换。考虑到有些类型是固定的对应关系，如：Java 的 int 类型对应 MySQL 的 int
类型，因此在设计时，使用一个 HashMap 做缓存，将这些固定不变的对应关系缓存起来。

    
    
    //新建 HashMap 用来存储 Java 到 MySQL 间数据格式的对应关系
    private static Map<String,String> map = new HashMap<>(16);
    
    private static final String STRING = "string";
    private static final String INT = "int";
    private static final String INTEGER = "integer";
    private static final String LONG = "long";
    private static final String DATE = "date";
    private static final String BYTE_ARRAY = "byte[]";
    private static final String FLOAT = "float";
    private static final String DOUBLE = "double";
    static {
            //HashMap 初始化
        map.put(STRING,"varchar(50)");
        map.put(INT,"int");
        map.put(INTEGER,"int");
        map.put(LONG,"bigint");
        map.put(DATE,"datetime");
        map.put(BYTE_ARRAY,"varbinary(50)");
        map.put(FLOAT,"decimal(10,2)");
        map.put(DOUBLE,"decimal(10,2)");
    }
    

在这里有一点默认值设置的考虑：

  1. Java 中的 String 类型可转换为 MySQL 中的 varchar 类型，varchar 类型要求长度在 65535 以内，超过之后可以转换为 text，更大的有 mediumtext 和 longtext。可以根据实际项目的使用场景，设置 varchar 的默认长度。在本项目中将其默认长度设置为 50，存储昵称、邮箱、签名等信息。
  2. Java 中的 byte[] 类型可转换为 MySQL 中的 varbinary 类型，varbinary 类型要求长度在 65535 以内，超过之后可以转换为 blob。可以根据实际项目的使用场景，设置 varbinary 的默认长度。在本项目中将其默认长度设置为 50。
  3. Java 中的 float、double 类型可转换为 MySQL 中的 decimal 类型，decimal 类型要求有两个长度值：数据总长度和浮点数的长度。可以根据实际项目的使用场景，设置 decimal 的默认长度。

默认转换规则设置完毕后，需要对特殊设置长度的字段进行转换，代码如下：

    
    
            //varchar/varbinary 类型，允许最大长度为 65535，在这里限制：如果超过 3000，转换为 text/blob
        private static final int MAX = 3000;
        /**
         * TINYTEXT     256 bytes
         * TEXT     65,535 bytes    ~64kb
         * MEDIUMTEXT      16,777,215 bytes   ~16Mb
         * LONGTEXT     4,294,967,295 bytes     ~4Gb
         */
        private static final int TEXT_MAX = 65535;
        //decimal 类型的最大长度为 65，根据平时使用的需要，设置为 20，足够大多数场景使用了
        private static final int DECIMAL_MAX = 20;
    public static String getType(String key,int length){
        if(StringUtils.isEmpty(key)){
         return null;
        }
        if(length <= 0){
            return map.get(key.toLowerCase());
        }
        /*
        float/Float/double/Double 类型判断设置的长度是否符合规则，如果超长，将长度设置为允许的最大长度
         */
        if(FLOAT.equalsIgnoreCase(key)
                || DOUBLE.equalsIgnoreCase(key)){
            length = length > DECIMAL_MAX ? DECIMAL_MAX:length;
            return "decimal(" + length + ",2)";
        }
        //String 根据长度，转换为 varchar 或 text
        if(STRING.equalsIgnoreCase(key)){
            if(length < MAX){
                return "varchar(" + length + ")";
            }
            if(length < TEXT_MAX){
                return "text";
            }
            return "mediumtext";
        }
        //byte[] 根据长度，转换为 varbinary 或 blob
        if(BYTE_ARRAY.equalsIgnoreCase(key)){
            if(length < MAX){
                return "varbinary(" + length + ")";
            }
            return "blob";
        }
        return map.get(key.toLowerCase());
    }
    

至此，Java 到 MySQL 间的类型转换规则设置完毕，可以写个测试类对转换方法进行测试。然后就可以分析建表脚本的格式，开始进行脚本的构建了。

### 方案分析

#### **分析建表脚本**

    
    
    create table user_info
    (
     id int auto_increment primary key,
     type int not null comment '用户类型',
     email varchar(200) not null comment '邮箱',
     password varchar(200) null comment '密码',
     createTime datetime null,
     status int null comment '用户账号状态',
     isDelete int null,
     updateTime datetime null,
     updateUserId int null
    )comment '用户信息表';
    create index user_info_index_email on user_info (email);
    create index user_info_index_status on user_info (status);
    create index user_info_index_type on user_info (type);
    

创建的数据表，含表名、数据表描述、字段名、字段描述、字段约束、主键、自增主键、索引，结合通用 Mapper 分析这些所需的信息：

**1\. 数据表名、数据表描述信息**

  * @TableAttribute 注解可以获取到表名
  * 可以在 @TableAttribute 注解中新增方法用来设置数据表的描述信息

**2\. 字段名、字段描述、字段类型、长度、是否必填、是否唯一**

  * @FieldAttribute 注解标注一个属性是数据表的字段
  * 可以通过反射机制获取到属性的类型，写一个 Java 类型到 MySQL 类型的转换规则即可完成字段类型的设置
  * 可以在 @FieldAttribute 注解中新增若干方法分别标注字段描述、长度、是否必填、是否唯一等信息 

**3\. 主键、自增主键**

  * @KeyAttribute 注解可以标注主键
  * 可以在 @KeyAttribute 注解中新增一个方法表示是否是自增主键

**4\. 索引**

  * @IndexAttribute 注解可以标注索引

#### **总体方案**

经过以上分析，自动建表的整体方案设计如下：

  1. 扩展 **MyBatis 通用 Mapper** 中原有的几个注解，以实现建表过程中需要的字段信息
  2. 通过 **Java 反射机制** 解析类和注解信息，获取到建表脚本需要的所有信息
  3. 拼接建表脚本并执行

### 代码实现

#### **扩展注解**

通过分析我们可以知道，只需要在通用 Mapper 相关的注解上添加一些属性，即可满足建表脚本所需的信息。具体代码改动如下。

@TableAttribute：

    
    
    //表说明
    String comment() default "";
    

@FieldAttribute：

    
    
            //字段描述
        String value() default "";
            //是否必填
        boolean notNull() default false;
        //字段长度限制。String 、byte[] 类型分别对应 mysql 中 varchar、varbinary 类型，需要设置长度，默认 50
        int length() default 0;
            //是否唯一
        boolean unique() default false;
    

在这里要注意，由于 @FieldAttribute 注解定义的属性比较多，一定要设置默认值。这样在使用时，很多不用做特殊设置的字段就不用再单独赋值了。

@KeyAttribute：

    
    
            //是否自增
        boolean autoIncr() default false;
    

#### **引用注解**

分别对几个注解进行扩充后，在 UserInfo 中对引用的注解进行字段填充：

    
    
    @TableAttribute(name = "user_info",comment = "用户信息表")
    public class UserInfo   extends BaseEntity {
        @KeyAttribute(autoIncr = true)
        @FieldAttribute
        private int id;
    
        @FieldAttribute(value = "用户类型",notNull = true)
        @IndexAttribute
        private Integer type;
    
        @FieldAttribute(value = "密码",length = 200)
        private String password;
    
        @FieldAttribute(value = "邮箱",notNull = true,length = 200)
        @IndexAttribute
        private String email;
    
        @FieldAttribute
        private Date createTime = new Date();
    
        @FieldAttribute("用户账号状态")
        @IndexAttribute
        private Integer status ;
    
        @FieldAttribute("是否删除，1 表示删除")
        @IndexAttribute
        private Integer isDelete;
    
        @FieldAttribute("最后一次修改时间")
        private Date updateTime = new Date();
    
        @FieldAttribute("修改人")
        private Integer updateUserId;
    }
    

至此，注解信息修改完毕，相关的信息也已经填充完毕，接下来只要在使用时通过反射机制读取到相应信息即可完成创建数据表的 SQL 脚本。

#### **解析字段信息**

实体类的注解信息设置完成之后，就可以对注解进行解析了，和上一节解析的方法一样。先解析字段信息，从属性上获取到字段名、字段对应的 Java 类型；从
@FieldAttribute 注解上获取字段描述、是否非空、是否唯一等约束信息。从而拼接出 SQL 语句中的字段描述，如：`email
varchar(200) not null comment '邮箱'`，完整代码如下：

    
    
    public static <T extends BaseEntity> String getAddFieldSql(T entity){
        //通过反射读取到实体类的字段列表
        Field[] fields = entity.getClass().getDeclaredFields();
        //创建一个 StringBuilder 用于拼接 SQL 语句
        StringBuilder builder = new StringBuilder();
        //遍历字段列表
        for(Field field:fields){
                //获取到 @FieldAttribute 注解信息
            FieldAttribute fieldAttribute = field.getAnnotation(FieldAttribute.class);
            if(fieldAttribute != null){
                //拼接字段名、字段类型和长度限制
                builder.append(field.getName()).append(" ")
    .append(TypeCaster.getType(field.getType().getSimpleName(),fieldAttribute.length()));
                            //添加非空约束
                if(fieldAttribute.notNull()){
                    builder.append(" not null ");
                }
                //添加唯一约束
                if(fieldAttribute.unique()){
                    builder.append(" unique ");
                }
                //如果有字段说明，添加字段说明
                if(StringUtils.isNotEmpty(fieldAttribute.value())) {
                    builder.append(" comment '")
                            .append(fieldAttribute.value()).append("'");
                }
                //逗号隔开下一个字段
                builder.append(", \n");
            }
        }
        //全部拼接完毕后去掉最后一个逗号
        builder.deleteCharAt(builder.lastIndexOf(","));
        //返回 SQL 字符串
        return builder.toString();
    }
    

#### **添加主键**

设置完普通字段后，单独对主键进行设置，参考的 SQL 语句格式如下：

    
    
    alter table user_info change id id int auto_increment  primary key comment '用户Id'; 
    

根据这个格式，我们解析 @KeyAttribute 注解，拼写 SQL 语句：

    
    
    private static <T extends BaseEntity> String getCreateKeySql(T entity){
        //通过反射获取实体类的字段列表
          Field[] fields = entity.getClass().getDeclaredFields();
          //创建一个 StringBuilder 用于拼接 SQL 语句
        StringBuilder builder = new StringBuilder();
        for(Field field:fields){
                //获取到 @KeyAttribute 注解信息
            KeyAttribute keyAttribute = field.getAnnotation(KeyAttribute.class);
            if(keyAttribute != null){
                FieldAttribute fieldAttribute = field.getAnnotation(FieldAttribute.class);
                if(fieldAttribute == null){
                    return "";
                }
                //拼写 SQL 语句
                builder .append("alter table ")
                        .append(getTableName(entity))
                        .append(" change ")
                        .append(field.getName())
                        .append(" ")
                        .append(field.getName())
                        .append(" ")
    .append(TypeCaster.getType(field.getType().getSimpleName(),fieldAttribute.length()));
                            //如果是自增，设置为自增主键
                if(keyAttribute.autoIncr()){
                    builder.append(" auto_increment ");
                }
                builder.append(" primary key comment '")
                        .append(fieldAttribute.value())
                        .append("'; \n");
    
                break;
            }
        }
        //返回拼接好的 SQL 语句
        return builder.toString();
    }
    

#### **添加索引**

最后为数据表添加索引，参考的添加索引的 SQL 格式如下：

    
    
    alter table user_info add index user_info_index_type (type); 
    

在这里有两点说明

  1. 设置的索引名称的格式为：`表名_index_索引字段名`，这样能保证添加的所有索引不会有命名冲突
  2. 在这里暂时不考虑设置联合索引的场景，每个索引都是单字段的

实现代码参考如下：

    
    
    public static <T extends BaseEntity> String getCreateIndexSql(T entity){
            //获取到表名
        String tableName = getTableName(entity);
        StringBuilder builder = new StringBuilder();
            //使用反射机制获取到字段列表
        Field[] fields = entity.getClass().getDeclaredFields();
        for(Field field:fields){
                //只处理带 @IndexAttribute 注解的字段
            if(field.getAnnotation(IndexAttribute.class) != null){
                            //拼写 SQL
                builder.append("alter table ")
                        .append(tableName)
                        .append(" add index ")
                        .append(tableName)
                        .append("_index_")
                        .append(field.getName())
                        .append(" (")
                        .append(field.getName())
                        .append("); \n");
            }
        }
        //返回拼写好的 SQL 
        return builder.toString();
    }
    

#### **拼接脚本**

字段定义、主键定义、索引定义的脚本都拼写完毕后，将这些脚本拼接在一起就是一个完整的建表脚本：

    
    
    public static <T extends BaseEntity> String getCreateTableSql(T entity){
        //解析 @TableAttribute 注解，获取到表名和表说明
          TableAttribute table =  entity.getClass().getAnnotation(TableAttribute.class);
        if(table == null){
            throw new BaseException("要解析表名，未发现@TableAttribute注解");
        }
        //获取表名
        String tableName = table.name();
        //获取表说明
        String tableComment = table.comment();
        StringBuilder builder = new StringBuilder();
        //拼写 create table 表名
        builder.append("create table ")
                .append(tableName)
                .append("( \n");
        // 添加字段
        builder.append(getAddFieldSql(entity));
        builder.append(") ");
        // 如果有表说明，添加表说明
        if(StringUtils.isNotEmpty(tableComment)){
            builder.append("comment '")
                    .append(tableComment)
                    .append("'; \n");
        }else {
            builder.append("; \n");
        }
        //添加主键
        builder.append(getCreateKeySql(entity));
        //添加索引
        builder.append(getCreateIndexSql(entity));
        //输出脚本
        Console.print("",builder.toString());
        //返回拼写好的 SQL 语句
        return builder.toString();
    }
    

至此，就可以通过 Java 实体类生成完整的建表脚本了，可以手动执行生成的建表脚本，也可以参照上一篇的写法，将添加到通用 Mapper 中。

#### **将自动建表的方法添加至通用 Mapper**

新建一个类 BaseCreateProvider，调用 **getCreateTableSql** 方法：

    
    
    /**
     * 创建表的Provider
     * @author yangkaile
     * @date 2019-09-12 15:07:09
     */
    public class BaseCreateProvider {
    
        /**
         *
         * 创建表的同时要创建索引，会执行多条语句，在application.properties中要设置 allowMultiQueries=true
         * spring.datasource.url = jdbc:mysql://localhost:3306/my_core
         * ?useUnicode=true&characterEncoding=utf-8&useSSL=false&allowMultiQueries=true
         * @param entity
         * @param <T>
         * @return
         */
        public static <T extends BaseEntity> String create(T entity){
            return SqlFieldReader.getCreateTableSql(entity);
        }
    }
    

在这里要注意一点，因为建表脚本包括创建数据表、设置主键、添加索引等多条 SQL 语句，要设置数据库连接为允许执行多条语句。

在 BaseMapper 中添加映射：

    
    
    /**
     * 创建表
     * @param entity
     */
    @UpdateProvider(type = BaseCreateProvider.class , method = "create")
    void baseCreate(T entity);
    

### 效果测试

在测试类 UserMapperTest 中新增一个测试方法用于测试自动建表的运行情况：

    
    
    @Test
    public void createUserInfoTable(){
        userMapper.baseCreate(new UserInfo());
    }
    

运行效果如下：

![createUserInfo](https://images.gitbook.cn/2e4dabd0-724a-11ea-b39b-4d2cde13353b)

### 源码地址

本篇完整源码地址如下：

>
> <https://github.com/tianlanlandelan/KellerNotes/tree/master/8.自动生成建表脚本/server>

### 小结

本篇带大家在通用 Mapper 的基础上实现根据 Java
实体类创建数据表，创建的数据表包含字段约束、主键、索引、字段描述、表描述等，能满足大多数使用场景，对于不喜欢写 SQL 语句的同学能提供相当友好的操作方式。

