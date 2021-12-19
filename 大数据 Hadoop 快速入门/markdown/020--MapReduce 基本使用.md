### 基础知识

MapReduce 框架只对 `<key, value>` 形式的键值对进行处理。MapReduce 会将任务的输入当成一组 `<key, value>`
键值对，最后也会生成一组 `<key, value>` 键值对作为结果。常见的输入为文件，此时读取的行偏移量会作为 key，文件内容作为 value。

key 和 value 的类必须由框架来完成序列化，所以需要实现其中的可写接口（Writable）。如果需要进行数据排序，还必须实现
WritableComparable 接口。MapReduce 已经提供了基本数据类型的 Writable 实现类，自定义类需要自行实现接口。

常见的基本数据类型的 Writable 有 IntWritable、LongWritable、Text 等等。

MapReduce 任务由 Map 和 Reduce 两个过程，所以需要分别进行编写。Map 的实现需要继承 Mapper 类，实现 map
方法完成数据处理；Reduce 则要继承 Reduer 类，实现 reduce 方法完成数据聚合。

    
    
    /*
     * KEYIN：输入 kv 数据对中 key 的数据类型
     * VALUEIN：输入 kv 数据对中 value 的数据类型
     * KEYOUT：输出 kv 数据对中 key 的数据类型
     * VALUEOUT：输出 kv 数据对中 value 的数据类型
     * 数据类型为 Writable 类型
     */
    public static class MyMapper extends Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT>{
        // Context 为 MapReduce 上下文，在 Map 中通常用于将数据处理结果输出
        public void map(KEYIN key, VALUEIN value, Context context) throws IOException, InterruptedException {
            // Map 功能的实现
        } 
    }
    
    public static class MyReducer extends Reducer<KEYIN,VALUEIN,KEYOUT,VALUEOUT> {
        // 这里 reduce 方法的输入的 Value 值是可迭代 Iterable 类型，因为 Reduce 阶段会将 Key 值相同的数据放置在一起
        public void reduce(KEYIN key, Iterable<VALUEIN> values, Context context ) throws IOException, InterruptedException {
            // Reduce 功能的实现
        }
      }
    

除了 MapReduce，为了提高 Shuffle 效率，减少 Shuffle 过程中传输的数据量，在 Map 端可以提前对数据进行聚合：将 Key
相同的数据进行处理合并，这个过程称为 Combiner。Combiner 需要在 Job 中进行指定，一般指定为 Reducer 的实现类。

Map 和 Reduce 的功能编写完成之后，在 main 函数中创建 MapReduce 的 Job 实例，填写 MapReduce
作业运行所必要的配置信息，并指定 Map 和 Reduce 的实现类，用于作业的创建。

    
    
     public static void main(String[] args) throws Exception {
         // 配置类
        Configuration conf = new Configuration();
        // 创建 MapReduce Job 实例
        Job job = Job.getInstance(conf, "Job Name");
        // 为 MapReduce 作业设置必要的配置
        // 设置 main 函数所在的入口类
        job.setJarByClass(WordCount.class);
        // 设置 Map 和 Reduce 实现类，并指定 Combiner
        job.setMapperClass(MyMapper.class);
        job.setCombinerClass(MyReducer.class);
        job.setReducerClass(IntSumReducer.class);
        // 设置结果数据的输出类
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        // 设置结果数据的输入和输出路径
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        // 作业运行，并输出结束标志
        System.exit(job.waitForCompletion(true) ? 0 : 1);
      }
    

除了基本的设置外，还可以指定 Reduce 的个数：

    
    
    job.setNumReduceTasks(int)
    

MapReduce 提供的常见类，除 Mapper、Reduer 之外，还有 Partitioner 和 Counter。其中 Partitioner
可以自定义 Map 中间结果输出时对 Key 的 Partition 分区，其目的是为了优化并减少计算量；如果不做自定义实现，HashPartitioner
是 MapReduce 使用的默认分区程序。

Counter（计数器）是 MapReduce 应用程序报告统计数据的一种工具。在 Mapper 和 Reducer 的具体实现中，可以利用 Counter
来报告统计信息。

### WordCount

接下来，实现最经典的入门案例，词频统计。编写 MapReduce 程序，统计单词出现的次数。

数据样例：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201020061404.png)

首先准备数据，并上传到 HDFS 中：

    
    
    // 在 HDFS 中创建作业输入目录
    hadoop fs -mkdir -p /tmp/mr/data/wc_input
    // 为目录赋权
    hadoop fs -chmod 777 /tmp/mr/data/wc_input
    // 在本地创建词频统计文件
    echo -e "hello hadoop\nhello hdfs\nhello yarn\nhello mapreduce" > wordcount.txt
    // 将 wordcount.txt 上传到作业输入目录
    hadoop fs -put wordcount.txt /tmp/mr/data/wc_input
    

在 linux 本地创建 WordCount.java 文件，编辑 MapReduce 程序，完成词频统计功能。

注意：使用 Vim 打开 WordCount.java，进行复制时，可能会出现格式问题，最好使用 Vi。

    
    
    import java.io.IOException;
    import java.util.StringTokenizer;
    
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.Mapper;
    import org.apache.hadoop.mapreduce.Reducer;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
    
    public class WordCount {
    
      /*
     * 实现 Mapper，文件的每一行数据会执行一次 map 运算逻辑
     * 因为输入是文件，会将处理数据的行数作为 Key，这里应为 LongWritable，设置为 Object 也可以；Value 类型为 Text：每一行的文件内容
     * Mapper 处理逻辑是将文件中的每一行切分为单词后，将单词作为 Key，而 Value 则设置为 1，<Word,1>
     * 因此输出类型为 Text，IntWritable
     */
      public static class TokenizerMapper
           extends Mapper<Object, Text, Text, IntWritable>{
    
        // 事先定义好 Value 的值，它是 IntWritable，值为 1
        private final static IntWritable one = new IntWritable(1);
        // 事先定义好 Text 对象 word，用于存储提取出来的每个单词
        private Text word = new Text();
    
        public void map(Object key, Text value, Context context
                        ) throws IOException, InterruptedException {
          // 将文件内容的每一行数据按照空格拆分为单词
          StringTokenizer itr = new StringTokenizer(value.toString());
          // 遍历单词，处理为<word,1>的 Key-Value 形式，并输出（这里会调用上下文输出到 buffer 缓冲区）
          while (itr.hasMoreTokens()) {
            word.set(itr.nextToken());
            context.write(word, one);
          }
        }
      }
    
      /*
     * 实现 Reducer
     * 接收 Mapper 的输出，所以 Key 类型为 Text，Value 类型为 IntWritable
     * Reducer 的运算逻辑是 Key 相同的单词，对 Value 进行累加
     * 因此输出类型为 Text，IntWritable，只不过 IntWritable 不再是 1，而是最终累加结果
     */
      public static class IntSumReducer
           extends Reducer<Text,IntWritable,Text,IntWritable> {
        // 预先定义 IntWritable 对象 result 用于存储词频结果
        private IntWritable result = new IntWritable();
    
        public void reduce(Text key, Iterable<IntWritable> values,
                           Context context
                           ) throws IOException, InterruptedException {
          int sum = 0;
          // 遍历 key 相同单词的 value 值，进行累加
          for (IntWritable val : values) {
            sum += val.get();
          }
          result.set(sum);
          // 将结果输出
          context.write(key, result);
        }
      }
    
      // 实现 Main 方法
      public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");
        job.setJarByClass(WordCount.class);
        job.setMapperClass(TokenizerMapper.class);
        job.setCombinerClass(IntSumReducer.class);
        job.setReducerClass(IntSumReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
      }
    }
    

接下来将代码编译为 jar 包：

    
    
    export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
    hadoop com.sun.tools.javac.Main WordCount.java
    jar cf wc.jar WordCount*.class
    

当然也可以使用 IDE 进行编译打包。

打包完成之后，便可以提交作业了，在 main 函数中，定义了两个参数：输入路径和输出路径，所以调用作业时需要指定参数。

    
    
    hadoop jar wc.jar WordCount /tmp/mr/data/wc_input /tmp/mr/data/wc_output
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024070656.png)

运行结束后，查看运行结果是否正确：

    
    
    hadoop fs -cat /tmp/mr/data/wc_output/part-r-*
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024070728.png)

