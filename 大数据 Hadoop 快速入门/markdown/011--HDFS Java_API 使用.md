### 环境初始化

首先完成 Java 开发环境准备，创建工程并导入开发所需的 jar 包。之后在准备好的工程中完成以下步骤。

1\. 在 IDE 中新建一个类，类名为 HDFSApp：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012073100.png)

2\. 在类中添加成员变量保存公共信息：

    
    
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.*;
    import org.apache.hadoop.fs.permission.FsPermission;
    import org.apache.hadoop.io.IOUtils;
    
    import java.io.BufferedInputStream;
    import java.io.File;
    import java.io.FileInputStream;
    import java.io.InputStream;
    import java.net.URI;
    
    // 将代码中的 {HDFS_HOST}:{HDFS_PORT} 替换为 HDFS 的 IP 与端口，如 192.168.31.41:9000
    public class HDFSApp {
        public static final String HDFS_PATH="hdfs://{HDFS_HOST}:{HDFS_PORT}";
        FileSystem fileSystem = null;
        Configuration configuration = null;
    }
    

3\. 在类中新增构造函数，初始化运行环境：

    
    
    public HDFSApp() throws Exception{
        this.configuration = new Configuration();
        this.fileSystem = FileSystem.get(new URI(HDFS_PATH), configuration, "hadoop");
    }
    

#### **API 基本使用**

**1\. 创建目录**

任务：在 HDFS 上创建目录“/tmp/java_data”。

    
    
    // 添加方法 mkdir()，方法中实现目录的创建
    public void mkdir() throws Exception {
        fileSystem.mkdirs(new Path("/tmp/java_data"));
    }
    

在 main 函数中执行测试：

    
    
    // 创建 Main 函数，对方法进行测试
    public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.mkdir();
    }
    

回到 shell 工具中，使用 shell 命令查看是否执行成功。

    
    
    hadoop fs -ls /tmp/
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023044857.png)

**2\. 更改目录权限**

任务：将 HDFS 目录“/tmp/java_data”的权限改为“rwxrwxrwx”。

    
    
    // 添加方法 setPathPermission，方法中实现对目录的授权
    public void setPathPermission() throws Exception {
            fileSystem.setPermission(new Path("/tmp/java_data"), new FsPermission("777"));
        }
    

在 main 函数中执行测试：

    
    
    // 在 Main 函数中，对方法进行测试
    public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.setPathPermission();
    }
    

回到 shell 工具中，使用 shell 命令查看是否执行成功。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023045304.png)

**3\. 上传文件**

任务：将本地文件“file.txt”上传到 HDFS 目录“/tmp/hdfs_data”目录中。

    
    
    // 在本地创建 file.txt 文件，文件中内容为 hello word
    // 添加方法 copyFromLocalFile，方法中完成本地文件 file.txt 的上传
    public void copyFromLocalFile() throws Exception {
            Path localPath = new Path("path to local file.txt");
            Path hdfsPath = new Path("/tmp/java_data/");
            fileSystem.copyFromLocalFile(localPath, hdfsPath);
        }
    

在 main 函数中执行测试：

    
    
    // 在 Main 函数中，对方法进行测试
    public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.copyFromLocalFile();
    }
    

回到 shell 工具中，使用 shell 命令查看是否执行成功。

    
    
    hadoop fs -ls /tmp/java_data
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023051102.png)

**4\. 查看目录内容**

任务：查看 HDFS 目录“/tmp/java_data”的内容。

    
    
    // 添加方法 listFiles，方法中查看“/tmp/java_data”目录下的内容
    public void listFiles(String dir) throws Exception {
            FileStatus[] fileStatuses = fileSystem.listStatus(new Path(dir));
            for(FileStatus fileStatus : fileStatuses) {
                String isDir = fileStatus.isDirectory() ? "文件夹" : "文件";
                short replication = fileStatus.getReplication();
                long len = fileStatus.getLen();
                String path = fileStatus.getPath().toString();
                System.out.println(isDir + "\t" + replication + "\t" + len + "\t" + path);
            }
        }
    

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.listFiles("/tmp/java_data");
        }
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023051511.png)

**5\. 查看文件内容**

任务：查看 HDFS 文件“/tmp/java_data/file.txt”的内容。

    
    
    // 添加方法 cat，方法中实现对文件 file.txt 的查看
    public void cat(String path) throws Exception {
            FSDataInputStream in = fileSystem.open(new Path(path));
            IOUtils.copyBytes(in, System.out, 1024);
            in.close();
        }
    

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.cat("/tmp/java_data/file.txt");
        }
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023051740.png)

**6\. 下载文件**

任务：从 HDFS 中将“/tmp/java_data/file.txt”文件下载到本地。

    
    
    // 添加方法 copyToLocalFile，方法中实现对文件 file.txt 的下载
    public void copyToLocalFile() throws Exception {
            Path localPath = new Path("path to save file");
            Path hdfsPath = new Path("/tmp/java_data/file.txt");
            fileSystem.copyToLocalFile(hdfsPath, localPath);
        }
    

下载文件到本地，需要先将 hadoop.dll 文件拷贝到 c:\windows\system32 目录中，否则会报错：

    
    
    java.io.IOException: (null) entry in command string: null chmod 0644
    

链接：

> <https://pan.baidu.com/s/10DJzC_341ILTb_Y6EshiVw>
>
> 提取码：pun1

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.copyToLocalFile();
        }
    

**7\. 创建文件**

任务：在 HDFS “/tmp/java_data”目录下创建新文件 word.txt，文件内容为 hello hadoop。

    
    
    // 添加 create 方法，在方法中实现 word.txt 的创建，并写入 hello hadoop 字符串
    public void create() throws Exception {
            FSDataOutputStream output = fileSystem.create(new Path("/tmp/java_data/word.txt"));
            output.write("hello hadoop".getBytes());
            output.flush();
            output.close();
        }
    

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.create();
            hdfsApp.cat("/tmp/java_data/word.txt");
        }
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023054141.png)

**8\. 文件追加**

任务：对“/tmp/java_data/word.txt”文件追加内容。

    
    
    // 1. 在本地创建文件 word_append.txt，内容为 hello world append
    // 2. 添加 append 方法，方法中实现对 word.txt 文件的追加
    public void append() throws Exception {
            FSDataOutputStream output = fileSystem.append(new Path("/tmp/java_data/word.txt"));
            InputStream in = new BufferedInputStream(
                    new FileInputStream(
                            new File("path to word_append.txt")));
            IOUtils.copyBytes(in, output, 4096);
        }
    

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.append();
        }
    

因为 HDFS 会有一定的延迟，所以无法使用之前编写的 cat 方法立即查看结果，所以需要到命令行终端中使用 shell 命令查看。

    
    
    hadoop fs -cat /tmp/java_data/word.txt
    

**9\. 文件合并**

任务：将“/tmp/java_data/”目录下的 file.txt 文件合并到 word.txt 文件中。

    
    
    // 添加方法 concat，方法中将 file.txt 文件合并到 word.txt 文件中
    public void concat() throws Exception {
            Path[] srcPath = {new Path("/tmp/java_data/file.txt")};
            Path trgPath = new Path("/tmp/java_data/word.txt");
            fileSystem.concat(trgPath,srcPath);
        }
    

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.concat();
            hdfsApp.cat("/tmp/java_data/word.txt");
        }
    

**10\. 文件改名**

任务：将 HDFS 中的“/tmp/java_data/word.txt”改名为 word_new.txt。

    
    
    // 添加方法 rename，方法中将 word.txt 文件改名为 word_new.txt
    public void rename() throws Exception {
            Path oldPath = new Path("/tmp/java_data/word.txt");
            Path newPath = new Path("/tmp/java_data/word_new.txt");
            fileSystem.rename(oldPath, newPath);
        }
    

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.rename();
            hdfsApp.listFiles("/tmp/java_data/");
        }
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023055649.png)

**11\. 清空文件**

任务：清空 HDFS 文件“/tmp/java _data/word_ new.txt”内容。

    
    
    // 添加方法 truncate，方法中将文件 word_new.txt 清空
    public void truncate() throws Exception {
            fileSystem.truncate(new Path("/tmp/java_data/word_new.txt"), 0);
        }
    

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.truncate();
            hdfsApp.cat("/tmp/java_data/word_new.txt");
        }
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023055853.png)

**12\. 删除文件**

任务：将 HDFS 文件“/tmp/rest _data/word_ new.txt”删除。

    
    
    // 添加方法 delete，方法中将文件 word_new.txt 删除
    public void delete() throws Exception{
            fileSystem.delete(new Path("/tmp/java_data/word_new.txt"), true);
        }
    

在 main 函数中执行测试：

    
    
        // 在 Main 函数中，对方法进行测试
        public static void main(String[] args) throws Exception{
            HDFSApp hdfsApp = new HDFSApp();
            hdfsApp.delete();
            hdfsApp.listFiles("/tmp/java_data/");
        }
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023060004.png)

