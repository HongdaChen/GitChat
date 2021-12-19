最后，将完成的 Shell 脚本交由 Azkaban 进行自动化调度。具体步骤讲解如下。

### MySQL 数据生成

在 Node02 节点的 MySQL 中执行 SQL，生成数据：

    
    
    mysql -uroot -pDBa2020*
    use mall;
    CALL init_data('2020-06-13',300,200,300,FALSE);
    

### Azkaban 任务编写

编写 Azkaban 运行 job，其中包
import.job、ods.job、dwd.job、dws.job、gmv_ads.job、gmv_export.job。

文件内容如下：

import.job

    
    
    type=command
    do_date=${dt}
    command=/home/warehouse/shell/sqoop_import.sh all ${do_date}
    

ods.job

    
    
    type=command
    do_date=${dt}
    dependencies=import
    command=/home/warehouse/shell/ods_db.sh ${do_date}
    

dwd.job

    
    
    type=command
    do_date=${dt}
    dependencies=ods
    command=/home/warehouse/shell/dwd_db.sh ${do_date}
    

dws.job

    
    
    type=command
    do_date=${dt}
    dependencies=dwd
    command=/home/warehouse/shell/dws_db.sh ${do_date}
    

gmv_ads.job

    
    
    type=command
    do_date=${dt}
    dependencies=dws
    command=/home/warehouse/shell/ads_gmv.sh ${do_date}
    

gmv_export.job

    
    
    type=command
    do_date=${dt}
    dependencies=gmv_ads
    command=/home/warehouse/shell/sqoop_gmv_export.sh all ${do_date}
    

使用压缩软件，将 6 个 job 文件打包成 mall-gmv-job.zip，提供现成的脚本文件如下：

  * 链接：<https://pan.baidu.com/s/15p2eVMj92Qd5P-i1T6vkOg>
  * 提取码：zco0 

### 使用 Azkaban 进行任务调度

1\. 在 3 台虚拟机中同时启动 Azkaban：

    
    
    azkaban-executor-start.sh
    

2\. 在存放 shell 脚本的虚拟机（Node03）上启动 Azkaban Web 服务器：

    
    
    cd /opt/app/azkaban/server
    azkaban-web-start.sh
    

3\. 访问 Azkaban Web 界面，端口 8443，账号密码均为 admin，进入后完成项目的创建：

![](https://images.gitbook.cn/84bbd160-f051-11ea-a374-77fb7954ed83)

4\. 上传并运行 job，运行时指定 executor 为 Shell 脚本存放的服务器，并配置运行参数：

![](https://images.gitbook.cn/9664d830-f051-11ea-8755-9ff4c3d0bc34)

运行参数如下：

    
    
    useExecutor node03
    dt 2020-06-13
    

5\. 运行结束后，查看 MySQL 中的结果，确认执行成功。

    
    
    mysql -uroot -pDBa2020*
    use mall;
    SELECT * FROM ads_gmv_sum_day;
    

![](https://images.gitbook.cn/ac2ae150-f051-11ea-889f-9de3f88b5342)

