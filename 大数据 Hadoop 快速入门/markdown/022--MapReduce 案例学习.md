### 基于日志的简单统计

现有网站访问日志，日志的数据格式如下：

    
    
    93.180.71.3 - - [17/May/2015:08:05:32 +0000] "GET /downloads/product_1 HTTP/1.1" 304 0 "-" "Debian APT-HTTP/1.3 (0.8.16~exp12ubuntu10.21)"
    93.180.71.3 - - [17/May/2015:08:05:23 +0000] "GET /downloads/product_1 HTTP/1.1" 304 0 "-" "Debian APT-HTTP/1.3 (0.8.16~exp12ubuntu10.21)"
    80.91.33.133 - - [17/May/2015:08:05:24 +0000] "GET /downloads/product_1 HTTP/1.1" 304 0 "-" "Debian APT-HTTP/1.3 (0.8.16~exp12ubuntu10.17)"
    

日志的各项数据由空格进行分隔，现在需要统计每个 IP 的访问次数。

所以在数据中，只需要关注 IP 地址。提取到 IP 地址之后，其实就是在做 wordcount 词频统计了。此案例较为简单，可以作为巩固练手项目。在
wordcount 基础之上，进行改造，完成代码编写。

完整的数据如下：

> <https://pan.baidu.com/s/140oXyqA8ViBdIxu0SB4utg>
>
> 提取码：fv4n

首先进行数据上传：

    
    
    // 在 HDFS 中创建作业输入目录
    hadoop fs -mkdir -p /tmp/mr/data/ip_input
    // 为目录赋权
    hadoop fs -chmod 777 /tmp/mr/data/ip_input
    // 上传数据文件 log.txt 到 Linux
    // 将 log.txt 上传到作业输入目录
    hadoop fs -put log.txt /tmp/mr/data/ip_input
    

在 Linux 本地创建 IpCounter.java 文件，代码部分实现如下：

    
    
    import java.io.IOException;
    import java.net.URI;
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.FileSystem;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.Mapper;
    import org.apache.hadoop.mapreduce.Reducer;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
    
    public class IpCounter {
    
        public static class LogMapper extends Mapper<Object, Text, Text, IntWritable> {
            private Text logIp = new Text();
            private final static IntWritable one = new IntWritable(1);
    
            @Override
            public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
    
                String logRecord = value.toString();
                String[] logField = logRecord.split(" ");
                logIp.set(logField[0]);
                context.write(logIp, one);
    
            }
        }
    
        public static class LogReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    
            private IntWritable result = new IntWritable();
    
            @Override
            protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
                int sum = 0;
                for (IntWritable val : values) {
                    sum += val.get();
                }
                result.set(sum);
                context.write(key, result);
            }
        }
    
        public static void main(String[] args) throws Exception {
    
            Configuration conf = new Configuration();
            Job job = Job.getInstance(conf, "log analysis");
            job.setJarByClass(IpCounter.class);
            job.setMapperClass(LogMapper.class);
            job.setReducerClass(LogReducer.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(IntWritable.class);
            FileInputFormat.addInputPath(job, new Path(args[0]));
            FileOutputFormat.setOutputPath(job, new Path(args[1]));
            System.exit(job.waitForCompletion(true) ? 0 : 1);
    
        }
    }
    

代码打包和提交：

    
    
    export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
    hadoop com.sun.tools.javac.Main IpCounter.java
    jar cf ip.jar IpCounter*.class
    hadoop jar ip.jar IpCounter /tmp/mr/data/ip_input /tmp/mr/data/ip_output
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024071557.png)

运行结束后，查看运行结果是否正确：

    
    
    hadoop fs -cat /tmp/mr/data/ip_output/part-r-*
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024071625.png)

### 部门总工资统计

现有两份数据，其中 dept 文件保存部门信息，emp 保存员工信息。数据内容如下：

    
    
    # dept 文件内容：部门 ID 部门名称 部门城市
    10,ACCOUNTING,NEW YORK
    20,RESEARCH,DALLAS
    30,SALES,CHICAGO
    40,OPERATIONS,BOSTON
    
    
    
    # emp 文件内容：ID 姓名 工作 上司 ID 入职时间 工资 提成 部门 ID
    7369,SMITH,CLERK,7902,17-12 月-80,800,,20
    7499,ALLEN,SALESMAN,7698,20-2 月-81,1600,300,30
    7521,WARD,SALESMAN,7698,22-2 月-81,1250,500,30
    7566,JONES,MANAGER,7839,02-4 月-81,2975,,20
    7654,MARTIN,SALESMAN,7698,28-9 月-81,1250,1400,30
    7698,BLAKE,MANAGER,7839,01-5 月-81,2850,,30
    7782,CLARK,MANAGER,7839,09-6 月-81,2450,,10
    7839,KING,PRESIDENT,,17-11 月-81,5000,,10
    7844,TURNER,SALESMAN,7698,08-9 月-81,1500,0,30
    7900,JAMES,CLERK,7698,03-12 月-81,950,,30
    7902,FORD,ANALYST,7566,03-12 月-81,3000,,20
    7934,MILLER,CLERK,7782,23-1 月-82,1300,,10
    

根据数据，需要求得各个部门的总工资。这里因为 dept 文件较小，可以先缓存到 Map 节点，并提取文件中的部门 ID 和部门名称转换为 Map
集合进行存储，避免 Shuffle。在 Map 运算时，从 emp 员工数据中提取工资和部门 ID，并使用部门 ID 与 Map 集合中的 dept
部门内容进行关联，取得部门名称，输出（部门名称,员工工资）。在 Reduce 端对员工工资进行累加运算，求得部门总工资的统计。

其中可以在入口方法中使用 DistributedCache.addCacheFile() 方法，将 dept 文件路径缓存，并绑定到 job 的
Configuration 对象中。

    
    
    // args[0]中传入的便是 dept 文件路径
    DistributedCache.addCacheFile(new Path(args[0]).toUri(), job.getConfiguration());
    

然后在 Map 阶段，从 job 的 Configuration 对象中获取 dept 缓存路径，并读取 dept 文件，缓存到本地使用。

    
    
    // context 为上下文环境
    Path[] paths = DistributedCache.getLocalCacheFiles(context.getConfiguration());
    

首先进行数据上传：

    
    
    // 在 HDFS 中创建作业输入目录
    hadoop fs -mkdir -p /tmp/mr/data/dept_input
    hadoop fs -mkdir -p /tmp/mr/data/emp_input
    // 为目录赋权
    hadoop fs -chmod 777 /tmp/mr/data/dept_input
    hadoop fs -chmod 777 /tmp/mr/data/emp_input
    // 创建文件 dept 和 emp,并添加数据内容
    vi dept
    vi emp
    // 将文件上传到作业输入目录
    hadoop fs -put dept /tmp/mr/data/dept_input
    hadoop fs -put emp /tmp/mr/data/emp_input
    

在 Linux 本地创建 SumDeptSalary.java 文件，整体代码实现如下：

    
    
    import java.io.BufferedReader;
    import java.io.FileReader;
    import java.io.IOException;
    import java.util.HashMap;
    import java.util.Map;
    import org.apache.log4j.Logger;   
    
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.filecache.DistributedCache;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.Mapper;
    import org.apache.hadoop.mapreduce.Reducer;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
    import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
    import org.apache.hadoop.util.GenericOptionsParser;
    
    
    public class SumDeptSalary {
    
    
        public static class MapClass extends Mapper<LongWritable, Text, Text, Text> {
    
            // 用于缓存 dept 文件中的数据
            private Map<String, String> deptMap = new HashMap<String, String>();
            private String[] kv;
    
            // 此方法会在 Map 方法执行之前执行且执行一次
            @Override
            protected void setup(Context context) throws IOException, InterruptedException {
                BufferedReader in = null;
                try {
    
                    // 从当前作业中获取要缓存的文件
                    Path[] paths = DistributedCache.getLocalCacheFiles(context.getConfiguration());
    
                    String deptIdName = null;
                    for (Path path : paths) {
    
                        // 对部门文件字段进行拆分并缓存到 deptMap 中
                        if (path.toString().contains("dept")) {
                            in = new BufferedReader(new FileReader(path.toString()));
                            while (null != (deptIdName = in.readLine())) {
    
                                // 对部门文件字段进行拆分并缓存到 deptMap 中
                                // 其中 Map 中 key 为部门编号，value 为所在部门名称
                                deptMap.put(deptIdName.split(",")[0], deptIdName.split(",")[1]);
                            }
                        }
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    try {
                        if (in != null) {
                            in.close();
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
    
            public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
    
                // 对员工文件字段进行拆分
                kv = value.toString().split(",");
                for (String k : deptMap.keySet()) {
                    System.out.println(k);
                  }
                // map join: 在 map 阶段过滤掉不需要的数据，输出 key 为部门名称和 value 为员工工资
                if (deptMap.containsKey(kv[7].trim())) {
                    if (null != kv[5] && !"".equals(kv[5].toString())) {
                        context.write(new Text(deptMap.get(kv[7].trim())), new Text(kv[5].trim()));
                    }
                }
            }
        }
    
        public static class Reduce extends Reducer<Text, Text, Text, LongWritable> {
    
            public void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
    
                // 对同一部门的员工工资进行求和
                long sumSalary = 0;
                for (Text val : values) {
                    sumSalary += Long.parseLong(val.toString());
                }
    
                // 输出 key 为部门名称和 value 为该部门员工工资总和
                context.write(key, new LongWritable(sumSalary));
            }
        }
    
    
        /**
         * 主方法，执行入口
         * @param args 输入参数
         */
        public static void main(String[] args) throws Exception {
            // 实例化作业对象，设置作业名称、Mapper 和 Reduce 类
            Configuration conf = new Configuration();
            Job job = new Job(conf, "SumDeptSalary");
            job.setJarByClass(SumDeptSalary.class);
            job.setMapperClass(MapClass.class);
            job.setReducerClass(Reduce.class);
    
            // 设置输入格式类
            job.setInputFormatClass(TextInputFormat.class);
    
            // 设置输出格式
            job.setOutputFormatClass(TextOutputFormat.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);
    
            // 第 1 个参数为缓存的部门数据路径、第 2 个参数为员工数据路径和第 3 个参数为输出路径
            // String[] otherArgs = new GenericOptionsParser(job.getConfiguration(), args).getRemainingArgs();
            DistributedCache.addCacheFile(new Path(args[0]).toUri(), job.getConfiguration());
            FileInputFormat.addInputPath(job, new Path(args[1]));
            FileOutputFormat.setOutputPath(job, new Path(args[2]));
    
            System.exit(job.waitForCompletion(true) ? 0 : 1);
        }
    }
    

代码打包和提交：

    
    
    export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
    hadoop com.sun.tools.javac.Main SumDeptSalary.java
    jar cf SumDeptSalary.jar SumDeptSalary*.class
    hadoop jar SumDeptSalary.jar SumDeptSalary /tmp/mr/data/dept_input/dept /tmp/mr/data/emp_input/emp /tmp/mr/data/sum_output
    

运行结束后，查看运行结果是否正确：

    
    
    hadoop fs -cat /tmp/mr/data/sum_output/part-r-*
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201025062800.png)

### 部门平均工资统计

数据内容与上一个案例——部门总工资统计相同。

    
    
    # dept 文件内容：部门 ID 部门名称 部门城市
    10,ACCOUNTING,NEW YORK
    20,RESEARCH,DALLAS
    30,SALES,CHICAGO
    40,OPERATIONS,BOSTON
    
    
    
    # emp 文件内容：ID 姓名 工作 上司 ID 入职时间 工资 提成 部门 ID
    7369,SMITH,CLERK,7902,17-12 月-80,800,,20
    7499,ALLEN,SALESMAN,7698,20-2 月-81,1600,300,30
    7521,WARD,SALESMAN,7698,22-2 月-81,1250,500,30
    7566,JONES,MANAGER,7839,02-4 月-81,2975,,20
    7654,MARTIN,SALESMAN,7698,28-9 月-81,1250,1400,30
    7698,BLAKE,MANAGER,7839,01-5 月-81,2850,,30
    7782,CLARK,MANAGER,7839,09-6 月-81,2450,,10
    7839,KING,PRESIDENT,,17-11 月-81,5000,,10
    7844,TURNER,SALESMAN,7698,08-9 月-81,1500,0,30
    7900,JAMES,CLERK,7698,03-12 月-81,950,,30
    7902,FORD,ANALYST,7566,03-12 月-81,3000,,20
    7934,MILLER,CLERK,7782,23-1 月-82,1300,,10
    

现在要求统计各个部门的平均工资。在统计部门总工资的基础上，在 Reduce
中对遍历部门每个员工的工资列表，累加求得部门员工人数，总工资除以员工人数求得平均工资。

首先进行数据上传（上一个案例已经上传过，便不用重复进行）：

    
    
    // 将 Hadoop 当前用户切换为 hdfs，进行访问授权
    export HADOOP_USER_NAME=hdfs
    // 在 HDFS 中创建作业输入目录
    hadoop fs -mkdir -p /tmp/mr/data/dept_input
    hadoop fs -mkdir -p /tmp/mr/data/emp_input
    // 创建文件 dept 和 emp,并添加数据内容
    vim dept
    vim emp
    // 将文件上传到作业输入目录
    hadoop fs -put dept /tmp/mr/data/dept_input
    hadoop fs -put emp /tmp/mr/data/emp_input
    

在 Linux 本地创建 DeptNumberAveSalary.java 文件，整体代码实现如下：

    
    
    import java.io.BufferedReader;
    import java.io.FileReader;
    import java.io.IOException;
    import java.util.HashMap;
    import java.util.Map;
    
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.conf.Configured;
    import org.apache.hadoop.filecache.DistributedCache;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.Mapper;
    import org.apache.hadoop.mapreduce.Reducer;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
    import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
    import org.apache.hadoop.util.GenericOptionsParser;
    import org.apache.hadoop.util.Tool;
    import org.apache.hadoop.util.ToolRunner;
    
    public class DeptNumberAveSalary extends Configured implements Tool {
    
        public static class MapClass extends Mapper<LongWritable, Text, Text, Text> {
    
            // 用于缓存 dept 文件中的数据
            private Map<String, String> deptMap = new HashMap<String, String>();
            private String[] kv;
    
            // 此方法会在 Map 方法执行之前执行且执行一次
            @Override
            protected void setup(Context context) throws IOException, InterruptedException {
                BufferedReader in = null;
                try {
                    // 从当前作业中获取要缓存的文件
                    Path[] paths = DistributedCache.getLocalCacheFiles(context.getConfiguration());
                    String deptIdName = null;
                    for (Path path : paths) {
    
                        // 对部门文件字段进行拆分并缓存到 deptMap 中
                        if (path.toString().contains("dept")) {
                            in = new BufferedReader(new FileReader(path.toString()));
                            while (null != (deptIdName = in.readLine())) {
    
                                // 对部门文件字段进行拆分并缓存到 deptMap 中
                                // 其中 Map 中 key 为部门编号，value 为所在部门名称
                                deptMap.put(deptIdName.split(",")[0], deptIdName.split(",")[1]);
                            }
                        }
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    try {
                        if (in != null) {
                            in.close();
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
    
        public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
    
                // 对员工文件字段进行拆分
                kv = value.toString().split(",");
    
                // map join: 在 map 阶段过滤掉不需要的数据，输出 key 为部门名称和 value 为员工工资
                if (deptMap.containsKey(kv[7])) {
                    if (null != kv[5] && !"".equals(kv[5].toString())) {
                        context.write(new Text(deptMap.get(kv[7].trim())), new Text(kv[5].trim()));
                    }
                }
            }
        }
    
        public static class Reduce extends Reducer<Text, Text, Text, Text> {
    
        public void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
    
                long sumSalary = 0;
                int deptNumber = 0;
    
                // 对同一部门的员工工资进行求和
                for (Text val : values) {
                    sumSalary += Long.parseLong(val.toString());
                    deptNumber++;
                }
    
                // 输出 key 为部门名称和 value 为该部门员工工资平均值
                context.write(key, new Text("Dept Number:" + deptNumber + ", Ave Salary:" + sumSalary / deptNumber));
            }
        }
    
        @Override
        public int run(String[] args) throws Exception {
    
            // 实例化作业对象，设置作业名称、Mapper 和 Reduce 类
            Job job = new Job(getConf(), "DeptNumberAveSalary");
            job.setJobName("DeptNumberAveSalary");
            job.setJarByClass(DeptNumberAveSalary.class);
            job.setMapperClass(MapClass.class);
            job.setReducerClass(Reduce.class);
    
            // 设置输入格式类
            job.setInputFormatClass(TextInputFormat.class);
    
            // 设置输出格式类
            job.setOutputFormatClass(TextOutputFormat.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);
    
            // 第 1 个参数为缓存的部门数据路径、第 2 个参数为员工数据路径和第 3 个参数为输出路径
            // String[] otherArgs = new GenericOptionsParser(job.getConfiguration(), args).getRemainingArgs();
            DistributedCache.addCacheFile(new Path(args[0]).toUri(), job.getConfiguration());
            FileInputFormat.addInputPath(job, new Path(args[1]));
            FileOutputFormat.setOutputPath(job, new Path(args[2]));
    
            job.waitForCompletion(true);
            return job.isSuccessful() ? 0 : 1;
        }
    
        /**
         * 主方法，执行入口
         * @param args 输入参数
         */
        public static void main(String[] args) throws Exception {
            int res = ToolRunner.run(new Configuration(), new DeptNumberAveSalary(), args);
            System.exit(res);
        }
    }
    

注意：因为直接在 main 方法中完成作业创建，会提示警告信息：

    
    
    WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
    

所以需要在 DeptNumberAveSalary 类中继承 Configured，并实现 Tool 接口，在 run 方法中完成 Job 创建，在
Main 方法中使用 ToolRunner.run 调用即可。

代码打包和提交：

    
    
    export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
    hadoop com.sun.tools.javac.Main DeptNumberAveSalary.java
    jar cf DeptNumberAveSalary.jar DeptNumberAveSalary*.class
    hadoop jar DeptNumberAveSalary.jar DeptNumberAveSalary /tmp/mr/data/dept_input/dept /tmp/mr/data/emp_input/emp /tmp/mr/data/avg_output
    

运行结束后，查看运行结果是否正确：

    
    
    hadoop fs -cat /tmp/mr/data/avg_output/part-r-*
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201025063757.png)

