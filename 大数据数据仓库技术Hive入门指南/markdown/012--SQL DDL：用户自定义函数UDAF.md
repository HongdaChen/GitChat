## UDAF的创建与实现

Hive
UDAF有两种实现方式，可以继承UDAF或者AbstractGenericUDAFResolver类，也可以实现GenericUDAFResolver2接口。

其中直接继承UDAF类，功能实现较为简单，但在运行时使用Hive反射机制，导致性能有损失。

在较新版本中org.apache.hadoop.hive.ql.exec.UDAF类已经废弃，但因为其实现方便，在很多开发者中较为流行。

通过AbstractGenericUDAFResolver和GenericUDAFResolver2实现UDAF，更加灵活，性能也更出色，是社区推荐的写法。

而AbstractGenericUDAFResolver是GenericUDAFResolver2接口的实现类，所以一般建议直接继承AbstractGenericUDAFResolver类进行UDAF的编写。

## UDAF实现方式一：继承UDAF类

#### UDAF开发流程

继承UDAF类进行UDAF的开发流程是：

  1. 继承org.apache.hadoop.hive.ql.exec.UDAF类
  2. 以静态内部类方式实现org.apache.hadoop.hive.ql.exec.UDAFEvaluator接口
  3. 实现接口中的init、iterate、terminatePartial、merge、terminate方法

其中UDAFEvaluator接口中的方法具体描述为：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/d799d2150fbfe69e160ec28911654b2c.png#pic_center)

这些方法会在Map、Reduce的不同阶段进行调用执行，首先会执行init方法进行全局初始化，然后Map节点调用iterate方法对数据进行处理，处理结果由terminatePartial
方法返回；如果Map阶段设置了Combiner部分聚合功能，则返回的数据继续交由iterate进行处理，处理结果再由terminatePartial
方法返回。

Reduce节点获取到数据后，执行merge方法对结果进行汇总计算，计算的最终结果会由terminate 方法进行返回。

#### 案例描述

现在通过一个案例，来进行UDAF开发实践。

具体要求为：对传入的数字列表，按照数字大小进行排序后，找出最大的并返回。其实就是在实现内置函数max的功能。

#### UDAF开发

案例内容相对简单，首先创建Java类MaxInt，继承org.apache.hadoop.hive.ql.exec.UDAF，以静态内部类方式实现org.apache.hadoop.hive.ql.exec.UDAFEvaluator接口，重写接口方法。

首先，在静态内部类MaxiNumberIntUDAFEvaluator中定义成员变量result用于结果的保存。然后在init方法中，对result变量进行初始化，这里直接赋值为null。

然后在iterate方法中，对传入的每一行数据与result变量进行比较，取最大值进行保存。terminatePartial方法直接返回result结果即可。

merge方法在这里不需要进行额外逻辑的编写，直接调用iterate方法在reduce端完成最大值的寻找即可。最后terminate返回result，即最终结果。

代码的具体实现如下：

    
    
    import org.apache.hadoop.hive.ql.exec.UDAF;
    import org.apache.hadoop.hive.ql.exec.UDAFEvaluator;
    import org.apache.hadoop.io.IntWritable;
    
    @org.apache.hadoop.hive.ql.exec.Description(name = "MaxInt",
    extended = "示例：select MaxInt(col) from src;",
    value = "_FUNC_(col)-将col字段中寻找最大值，并返回")
    public class MaxInt extends UDAF {
    public  static  class MaxiNumberIntUDAFEvaluator implements UDAFEvaluator{
    private IntWritable result;
    public void init() {
    result = null;
    }
    public boolean iterate(IntWritable value){
    if (value == null){
    return false;
    }
    if (result == null){
    result = new IntWritable(value.get());
    }else {
    result.set(Math.max(result.get(),value.get()));
    }
    return true;
    }
    public IntWritable terminatePartial(){
    return result;
    }
    public boolean merge(IntWritable other){
    return iterate(other);
    }
    public IntWritable terminate(){
    return result;
    }
    }
    }
    

打成Jar包，在hive中创建临时函数进行测试。

    
    
    add jars file:///root/UDFS.jar;
    create temporary function max_int as "MaxInt";
    

创建测试表city_score，并写入测试数据。

    
    
    create table city_score(id int, province string, city string, score int);
    insert into city_score values (1, "GuangDong", "GuangZhou", 95),(2, "GuangDong", "ShenZhen", 90),(3, "GuangDong", "DongGuan", 85);
    

使用udaf函数统计最大值：

    
    
    select max_int(score) from city_score;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/973f904283be29fac7a81c73bcde1505.png#pic_center)

### UDAF实现方式二：继承AbstractGenericUDAFResolver类

继承org.apache.hadoop.hive.ql.udf.generic.GenericUDAFEvaluator.AbstractGenericUDAFResolver类进行UDAF的开发，是社区推荐的写法。相对于直接继承UDAF，实现方式更加灵活，性能更优，支持处理复杂数据。但实现起来会稍微复杂一些。

#### 开发流程

使用AbstractGenericUDAFResolver进行UDAF开发的具体流程为：

  1. 继承AbstractAggregationBuffer类，用于保存中间结果
  2. 新建类，继承GenericUDAFEvaluator，实现UDAF处理流程
  3. 新建类，继承AbstractGenericUDAFResolver，注册UDAF

这里用到的多个类，在用户自定义函数开发中，UDAF的实现最为复杂。其实也比较好理解，AbstractGenericUDAFResolver继承类用于注册UDAF；而UDAF的处理流程实现，使用GenericUDAFEvaluator的继承类完成；在处理流程中使用AbstractAggregationBuffer类的集成类进行中间结果的保存。

首先，对于AbstractGenericUDAFResolver类，需要重写getEvaluator方法，根据根据SQL传入的参数类型，返回正确的GenericUDAFEvaluator对象。而GenericUDAFEvaluator对象，则是UDAF的逻辑实现。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/0dac515dc81fa41cecc08af2b419afd5.png#pic_center)

GenericUDAFEvaluator的继承类，需要重写6个方法，这些方法的具体说明如下：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/a1b38b3e963e55efcdba4d1173f067dd.png#pic_center)

其实除了getNewAggregationBuffer，用于获取存放中间结果的AggregationBuffer对象之外，其它方法与UDAF类是一致的。

但init方法承担了更多的职责，和GenericUDF一样，它需要完成解析器的初始化，并定义返回值类型。
而且因为在mapreduce处理的不同阶段，UDAF传入的数据会有不同的区分，比如Map阶段传入的数据可能为String类型，而Reduce聚合时仅需要的是Double类型，所以init方法要区分不同的阶段，从而生成不同的解析器。

如何区分不同的阶段呢？GenericUDAFEvaluator.Model类中定义了这些阶段：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/4a2ba7a9d10cef876c1cc0f33addf812.png#pic_center)

在init方法中，根据不同的阶段，生成不同的解析器即可。其余方法的实现，基本与UDAF方式一致；只是中间结果的保存，使用了AggregationBuffer对象，需要使用getNewAggregationBuffer方法进行返回。

AggregationBuffer类随着需要保存的中间结果的不同，需要自行继承实现。在继承类中定义变量，用于保存中间结果，可以自行添加get、set、add方法，也可以重写estimate方法，用于缓冲区大小。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/55b96639f454399af8b9b37a839c94b1.png#pic_center)

缓冲区的大小，可以使用org.apache.hadoop.hive.ql.util.JavaDataModel中定义的常量来进行设置。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/827aff4111650bd671d9556b6026b3ff.png#pic_center)

#### GenericUDF实际案例

现在，完成一个UDAF的开发案例来进行实践。这个案例中，需要计算每个省份下的城市名字符串总长度。

在表中，包含两个字段：Province省份、City城市。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/6bb24b384015f5e29144d5f03007634d.png#pic_center)

对Province省份进行Group By操作后，统计每个省份下所有City城市名字符串的 总长度，结果如下：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/821f0607bb680594adc4be5963caea4c.png#pic_center)

首先定义AbstractAggregationBuffer继承类，用于保存计算结果。

    
    
    import org.apache.hadoop.hive.ql.udf.generic.GenericUDAFEvaluator;
    import org.apache.hadoop.hive.ql.util.JavaDataModel;
    
    public class FieldLengthAggregationBuffer extends GenericUDAFEvaluator.AbstractAggregationBuffer {
    private Integer value = 0;
    public Integer getValue() {
    return value;
    }
    
    public void setValue(Integer value) {
    this.value = value;
    }
    
    public void add(int addValue) {
    synchronized (value) {
    value += addValue;
    }
    }
    
    @Override
    public int estimate() {
    return JavaDataModel.PRIMITIVES1;
    }
    
    }
    

这里主要定义了Integer类型的成员变量value用于结果的保存，然后设置了get、set、add方法，重写了estimate设置缓冲区的大小。因为Integer类型，所以，只需要4byte大小的JavaDataModel.PRIMITIVES1即可。

然后定义GenericUDAFEvaluator的继承类FieldLengthUDAFEvaluator，用于实现UDAF计算逻辑：

    
    
    import org.apache.hadoop.hive.ql.metadata.HiveException;
    import org.apache.hadoop.hive.ql.udf.generic.GenericUDAFEvaluator;
    import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
    import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
    import org.apache.hadoop.hive.serde2.objectinspector.PrimitiveObjectInspector;
    
    
    public class FieldLengthUDAFEvaluator extends GenericUDAFEvaluator {
    @Override
    public ObjectInspector init(Mode m, ObjectInspector[] parameters) throws HiveException {…}
    @Override
    public AggregationBuffer getNewAggregationBuffer() throws HiveException {…}
    @Override
    public void reset(AggregationBuffer agg) throws HiveException {…}
    @Override
    public void iterate(AggregationBuffer agg, Object[] parameters) throws HiveException {…}
    @Override
    public Object terminatePartial(AggregationBuffer agg) throws HiveException {…}
    @Override
    public void merge(AggregationBuffer agg, Object partial) throws HiveException {…}
    @Override
    public Object terminate(AggregationBuffer agg) throws HiveException {…}
    }
    

在GenericUDAFEvaluator继承类中首先需要定义成员变量，确定数据的输入、输出类型。

    
    
    public class FieldLengthUDAFEvaluator extends GenericUDAFEvaluator {
    PrimitiveObjectInspector inputOI;
    ObjectInspector outputOI;
    PrimitiveObjectInspector integerOI;
    }
    

在类中，首先重写init方法，初始化输入、输出的数据类型。这里因为UDAF的不同，需要根据Mode m的不同，来初始化不同的输入、输出解析器。

    
    
    @Override
    public ObjectInspector init(Mode m, ObjectInspector[] parameters) throws HiveException {
    super.init(m, parameters);
    // COMPLETE或者PARTIAL1，输入的都是数据库的原始数据
    if(Mode.PARTIAL1.equals(m) || Mode.COMPLETE.equals(m)) {
    inputOI = (PrimitiveObjectInspector) parameters[0];
    } else {
    // PARTIAL2和FINAL阶段，都是基于前一个阶段init返回值作为parameters入参
    integerOI = (PrimitiveObjectInspector) parameters[0];
    }
    outputOI = ObjectInspectorFactory.getReflectionObjectInspector(
    Integer.class,
    ObjectInspectorFactory.ObjectInspectorOptions.JAVA
    );
    // 给下一个阶段用的，即告诉下一个阶段，自己输出数据的类型
    return outputOI;
    }
    

然后，重写getNewAggregationBuffer方法，获取存放中间结果的对象，直接返回FieldLengthAggregationBuffer对象即可。

    
    
    public AggregationBuffer getNewAggregationBuffer() throws HiveException {
    return new FieldLengthAggregationBuffer();
    }
    

重写reset方法，用于UDAF重置，这里只需要将中间结果设置为0即可：

    
    
    public void reset(AggregationBuffer agg) throws HiveException {
    ((FieldLengthAggregationBuffer)agg).setValue(0);
    }
    

重写iterate，在map端对数据进行聚合运算：

    
    
    public void iterate(AggregationBuffer agg, Object[] parameters) throws HiveException {
    if(null==parameters || parameters.length<1) {
    return;
    }
    Object javaObj = inputOI.getPrimitiveJavaObject(parameters[0]);
    ((FieldLengthAggregationBuffer)agg).add(String.valueOf(javaObj).length());
    }
    

重写terminatePartial，返回map、combiner部分聚合结果：

    
    
    public Object terminatePartial(AggregationBuffer agg) throws HiveException {
    return terminate(agg);
    }
    

重写merge方法，在combiner、reduce阶段对部分结果进行汇总：

    
    
    public void merge(AggregationBuffer agg, Object partial) throws HiveException {
    ((FieldLengthAggregationBuffer) agg).add((Integer)integerOI.getPrimitiveJavaObject(partial));
    }
    

重写terminate，将最终运算结果返回：

    
    
    public Object terminate(AggregationBuffer agg) throws HiveException {
    return ((FieldLengthAggregationBuffer)agg).getValue();
    }
    

GenericUDAFEvaluator继承类实现UDAF计算逻辑后，就需要编写AbstractGenericUDAFResolver实现类，实例化Evaluator实现类，注册成UDAF进行使用：

    
    
    import org.apache.hadoop.hive.ql.parse.SemanticException;
    import org.apache.hadoop.hive.ql.udf.generic.AbstractGenericUDAFResolver;
    import org.apache.hadoop.hive.ql.udf.generic.GenericUDAFEvaluator;
    import org.apache.hadoop.hive.ql.udf.generic.GenericUDAFParameterInfo;
    import org.apache.hadoop.hive.serde2.typeinfo.TypeInfo;
    
    @org.apache.hadoop.hive.ql.exec.Description(name = "FieldLength",
    extended = "示例：select FieldLength(city) from src group by province;",
    value = "_FUNC_(col)-将统计当前组中数据的总字符串长度，并返回")
    public class FieldLength extends AbstractGenericUDAFResolver {
    @Override
    public GenericUDAFEvaluator getEvaluator(GenericUDAFParameterInfo info) throws SemanticException {
    return new FieldLengthUDAFEvaluator();
    }
    
    @Override
    public GenericUDAFEvaluator getEvaluator(TypeInfo[] info) throws SemanticException {
    return new FieldLengthUDAFEvaluator();
    }
    }
    

打成Jar包，在hive中创建临时函数进行测试。

    
    
    add jars file:///root/UDFS.jar;
    create temporary function field_length as "FieldLength";
    

使用udaf函数统计最大值：

    
    
    select field_length(city) from city_score group by province;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/a44c5ee7efa36e1daa51bb92b720cd22.png#pic_center)

