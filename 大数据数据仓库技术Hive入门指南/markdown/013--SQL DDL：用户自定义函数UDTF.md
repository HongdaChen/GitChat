## UDTF开发要点

Hive
UDTF只有一种实现方式，需要继承org.apache.hadoop.hive.ql.udf.generic.GenericUDTF类，并重写initialize,
process, close三个方法。

这三个方法的具体描述为：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/3a590970861aea5eb3f38941ba24c9b8.png#pic_center)

因为UDTF是将一行数据拆分为多行，所以在处理过程中按照一定规则拆分出的每一行数据，在遍历过程中，会交由forward方法传递给收集器，从而完成多行数据的生成。

## UDTF开发案例

### 字符串拆分

#### 案例描述

现在通过一个案例，来进行UDTF开发实践。

具体要求为：实现个人信息的字符串拆分，拆分为多行，并解析成name、age字段。

案例数据为：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/61b90ca306c817ca1faa0e71a3f99319.png#pic_center)

其中每行输入数据包含多条个人信息，由#进行分隔，个人信息中name、age使用:分隔。

最终的计算结果为：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/27154ace352f36a8cbcaf717ed5515cd.png#pic_center)

#### UDTF开发

案例内容相对简单，首先创建Java类StringSpilt，继承org.apache.hadoop.hive.ql.udf.generic.GenericUDTF，重写initialize,
process, close三个方法。

    
    
    import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
    import org.apache.hadoop.hive.ql.metadata.HiveException;
    import org.apache.hadoop.hive.ql.udf.generic.GenericUDTF;
    import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
    import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
    import org.apache.hadoop.hive.serde2.objectinspector.PrimitiveObjectInspector;
    import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
    import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;
    import java.util.ArrayList;
    
    @org.apache.hadoop.hive.ql.exec.Description(name = "StringSpilt",
    extended = "示例：select StringSpilt(info) from src;",
    value = "_FUNC_(col)-将col字段值先按照#进行拆分，然后再使用:将每个子串拆分为name、age两个字段值")
    public class StringSpilt extends GenericUDTF {
    @Override
    public StructObjectInspector initialize(ObjectInspector[] objectInspectors) throws UDFArgumentException {...}
    @Override
    public void process(Object[] record) throws HiveException {...}
    @Override
    public void close() {...}
    }
    

类创建好之后，先将解析器设置为类属性。

    
    
    private PrimitiveObjectInspector stringOI = null;
    

然后，重写initiable方法，进行参数检查，解析器初始化，并定义UDTF的返回值类型。

    
    
    @Override
    public StructObjectInspector initialize(ObjectInspector[] objectInspectors) throws UDFArgumentException {
    if (objectInspectors.length != 1)
    {
    throw new UDFArgumentException("SplitString only takes one argument");
    }
    if (objectInspectors[0].getCategory() != ObjectInspector.Category.PRIMITIVE)
    {
    throw new UDFArgumentException("SplitString only takes String as a parameter");
    }
    stringOI = (PrimitiveObjectInspector) objectInspectors[0];
    ArrayList<String> fieldNames = new ArrayList<String>();
    ArrayList<ObjectInspector> fieldOIs = new ArrayList<ObjectInspector>();
    fieldNames.add("name");
    fieldNames.add("age");
    fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
    fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
    
    return ObjectInspectorFactory.getStandardStructObjectInspector(fieldNames,fieldOIs);
    }
    

在initiable方法中，定义返回值时需要注意；因为案例要求返回两列，所以返回值使用getStandardStructObjectInspector方法返回结构体类型。方法需要传入两个参数，均为List类型，第一个List中定义每一列的列名，这里即为name、age，第二个List中则对应每一列的数据类型，案例中两列均为String类型，但这里需要封装为javaStringObjectInspector。

接下来重写process方法，进行具体的数据处理过程：

    
    
    @Override
    public void process(Object[] record) throws HiveException {
    final String input = stringOI.getPrimitiveJavaObject(record[0]).toString();
    String[] inputSplits = input.split("#");
    for (int i = 0; i < inputSplits.length; i++) {
    try{
    String[] result = inputSplits[i].split(":");
    forward(result);
    }catch (Exception e){
    continue;
    }
    }
    }
    

这里处理过程比较简单，直接按照#对数据进行切分，遍历切分后的数据，再继续对数据按照:切分，解析为name、age。输出类型因为在initiable方法中已经定义，拆分为2列进行存储，所以需要将解析后的数据保存为String数组，使用forward方法进行返回。

于是String数组中的第一个值会保存到name字段中，第二个值则保存到age字段中。

最后重写close方法，进行清理工作，因为本身案例比较简单，没有使用额外资源，所以这部分代码留空。

    
    
    @Override
    public void close() {
    }
    

打成Jar包，上传到/root/目录下，在hive中创建临时函数进行测试。

    
    
    add jars file:///root/UDFS.jar;
    create temporary function str_split as "StringSpilt";
    

创建测试表log_info，并写入测试数据。

    
    
    create table log_info(id int, log string);
    insert into log_info values (1, "ZhangSan:18#LiSi:20#WangWu:30"),(2, "LiBai:18#DuFu:20");
    

使用udtf函数对字符串进行拆分：

    
    
    select str_split(log) from log_info;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/12176ea7c1bb668f47176080677fa19f.png#pic_center)

### JSON解析

#### 案例描述

在第一个案例的字符串解析基础上，增加一些难度，进行JSON的解析开发。

具体要求为：将JSON字符串中的多个(key，value)对拆分成多行进行存储，每行的(key，value)数据解析成name、value两个字段进行保存。

案例数据为：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/cd851508244041d8e79422fb2b0b2d2d.png#pic_center)

将每行的JSON最终的计算结果为：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/6dcaefed8f81fe5388332834eb74a65b.png#pic_center)

当然，现在这个案例没有实用价值，只是对JSON字符串做了简单解析、拆分，在理解代码的基础上，体会UDTF的开发流程。

#### UDTF开发

有了第一个案例的基础，实现起来其实也比较简单，主要是JSON解析这里，需要单独封装一个方法来进行。首先创建Java类JsonParse，继承org.apache.hadoop.hive.ql.udf.generic.GenericUDTF，重写initialize,
process, close三个方法。

    
    
    import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
    import org.apache.hadoop.hive.ql.metadata.HiveException;
    import org.apache.hadoop.hive.ql.udf.generic.GenericUDTF;
    import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
    import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
    import org.apache.hadoop.hive.serde2.objectinspector.PrimitiveObjectInspector;
    import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
    import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;
    import org.json.JSONObject;
    import org.mortbay.util.ajax.JSON;
    
    import java.util.ArrayList;
    import java.util.Iterator;
    import java.util.List;
    
    public class JsonParse extends GenericUDTF {
    @Override
    public StructObjectInspector initialize(ObjectInspector[] objectInspectors) throws UDFArgumentException {...}
    @Override
    public void process(Object[] record) throws HiveException {...}
    @Override
    public void close() {...}
    }
    

类创建好之后，先将解析器设置为类属性。

    
    
    private PrimitiveObjectInspector stringOI = null;
    

然后，重写initiable方法，进行参数检查，解析器初始化，并定义UDTF的返回值类型。

    
    
    @Override
    public StructObjectInspector initialize(ObjectInspector[] objectInspectors) throws UDFArgumentException {
    if (objectInspectors.length != 1) {
    throw new UDFArgumentException("NameParserGenericUDTF() takes exactly one argument");
    }
    if (objectInspectors[0].getCategory() != ObjectInspector.Category.PRIMITIVE &&
    ((PrimitiveObjectInspector) objectInspectors[0]).getPrimitiveCategory() != PrimitiveObjectInspector.PrimitiveCategory.STRING) {
    throw new UDFArgumentException("NameParserGenericUDTF() takes a string as a parameter");
    }
    //初始化输入解析器
    stringOI = (PrimitiveObjectInspector) objectInspectors[0];
    //初始化输出解析器
    List<String> fieldNames = new ArrayList<String>(2);
    List<ObjectInspector> fieldOIs = new ArrayList<ObjectInspector>(2);
    // 输出列名
    fieldNames.add("name");
    fieldNames.add("value");
    fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
    fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
    return ObjectInspectorFactory.getStandardStructObjectInspector(fieldNames, fieldOIs);
    }
    

接下来就需要新增JSON解析方法，完成JSON的解析：

    
    
    private ArrayList<Object[]> parseInputRecord(String feature) {
    ArrayList<Object[]> resultList = null;
    
    if (feature != null && feature.startsWith("\ufeff")) {
    feature = feature.substring(1);
    }
    
    try {
    JSONObject json = new JSONObject(feature);
    resultList = new ArrayList<Object[]>();
    
    for (String key : json.keySet()) {
    Object[] item = new Object[2];
    item[0] = key;
    item[1] = json.get(key);
    resultList.add(item);
    }
    } catch (Exception e) {
    e.printStackTrace();
    }
    return resultList;
    }
    

这里将JSON解析成一个List集合，List中的每个元素都是一个数组item，其中item[0] [1]存储value值。

然后重写process方法，完成Json数据的处理过程：

    
    
    @Override
    public void process(Object[] record) throws HiveException {
    
    final String feature = stringOI.getPrimitiveJavaObject(record[0]).toString();
    ArrayList<Object[]> results = parseInputRecord(feature);
    Iterator<Object[]> it = results.iterator();
    while (it.hasNext()) {
    Object[] r = it.next();
    forward(r);
    }
    }
    

这里处理过程比较简单，直接调用parseInputRecord方法，解析完成JSON后，得到存储所有（Key，Value）对的List集合。

然后遍历List集合，将每个Key，Value取出，然后使用forward进行返回。

每次遍历，会新生成一行数据，其中key值会保存到name字段中，value值则保存到value字段中。

最后重写close方法，进行清理工作，因为本身案例比较简单，没有使用额外资源，所以这部分代码留空。

    
    
    @Override
    public void close() throws HiveException {
    }
    

打成Jar包，上传到/root/目录下，在hive中创建临时函数进行测试。

    
    
    add jars file:///root/UDFS.jar;
    create temporary function json_parse as "JsonParse";
    

创建测试表json_info，并写入测试数据。

    
    
    create table json_info(id int, json string);
    insert into json_info values (1, '{"name":"LiBai","age":30,"hobby":"drink"}'),(2, '{"name":"DuFu","age":28,"hobby":"play"}');
    

使用udtf函数对字符串进行拆分：

    
    
    select json_parse(json) from json_info;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/de546bdb33c21d393c22af9c10c8ab39.png#pic_center)

