现在整个数据仓库的流程已经打通，并且所有脚本也已经封装完成。但从业务数据库抽取数据，一般选择在夜间进行，而且数据仓库的整个处理流程是有先后关系的，所以需要使用自动化调度工具来进行定时、控制依赖关系。

现使用 Azkaban 作为数据仓库的调度工具。接下模拟新一天的数据生成，并使用 Azkaban 完成整个数据处理过程的自动调度。

### 测试数据生成

在 Node02，即 MySQL 安装节点中执行 SQL，生成新一天的数据

    
    
    mysql -uroot -pDBa2020*
    use mall;
    CALL init_data('2020-06-12',300,200,300,FALSE);
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200824083957.png)

### 编写 Azkaban Job

因为 Azkaban 可以通过 Web 界面来进行作业调度，所以，Azkaban 的 Job 文件需要在本地计算机中编写。

Azkaban 运行的 Job，包括
import.job、ods.job、dwd.job、dws.job、ads.job、export.job；之后需要使用工具打包成压缩文件。

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
    

ads.job

    
    
    type=command
    do_date=${dt}
    dependencies=dws
    command=/home/warehouse/shell/ads_sale.sh ${do_date}
    

export.job

    
    
    type=command
    do_date=${dt}
    dependencies=ads
    command=/home/warehouse/shell/sqoop_export.sh all ${do_date}
    

使用压缩软件，将 6 个 job 文件打包成 mall-job.zip。或者可以直接使用已经压缩好的文件，文件下载方式如下：

  * 链接：<https://pan.baidu.com/s/1V2L1DacADGT0nm8H68x8hA>
  * 提取码：i5fp

### 使用 Azkaban 进行任务调度

1\. 在 3 台虚拟机中同时启动 Azkaban：

    
    
    azkaban-executor-start.sh
    

![](https://images.gitbook.cn/2603c230-f04d-11ea-b27a-6f83744fa302)

2\. 在存放 shell 脚本的虚拟机（Node03）上启动 Azkaban Web 服务器：

    
    
    cd /opt/app/azkaban/server
    azkaban-web-start.sh
    

![](https://images.gitbook.cn/3ad58040-f04d-11ea-8755-9ff4c3d0bc34)

3\. 访问 Azkaban Web 界面，端口 8443，账号密码均为 admin。

![](https://images.gitbook.cn/4aa86780-f04d-11ea-80b6-61caae27bd5a)

![](https://images.gitbook.cn/5aebeae0-f04d-11ea-b27a-6f83744fa302)

3\. 上传并运行 job，运行时指定 executor 为 shell 脚本存放的服务器，并配置脚本参数：

![](https://images.gitbook.cn/71cd2800-f04d-11ea-affc-a54214209ff7)

![](https://images.gitbook.cn/7f0407a0-f04d-11ea-8755-9ff4c3d0bc34)

![](https://images.gitbook.cn/8c12ece0-f04d-11ea-8755-9ff4c3d0bc34)

![](https://images.gitbook.cn/98ce5af0-f04d-11ea-affc-a54214209ff7)

![](https://images.gitbook.cn/a4f96fe0-f04d-11ea-bc58-d7943dd51ce7)

![](https://images.gitbook.cn/b1e11e10-f04d-11ea-b27a-6f83744fa302)

![](https://images.gitbook.cn/bdf26970-f04d-11ea-b91c-6d7fbdb6f8f7)

![](https://images.gitbook.cn/cb497b40-f04d-11ea-9212-f1aa28746d87)

图中的参数列表如下：

    
    
    useExecutor node03
    dt 2020-06-12
    

4\. 可以在 Web 界面查看整体流程的执行情况：

![](https://images.gitbook.cn/db900be0-f04d-11ea-b2cc-b183c5e37897)

![](https://images.gitbook.cn/eb180770-f04d-11ea-affc-a54214209ff7)

5\. 运行成功后，在 Node02 节点，查看 MySQL 数据库的结果，验证数据是否成功。

    
    
    mysql -uroot -pDBa2020*
    use mall;
    SELECT * FROM ads_sale_tm_category1_stat_mn;
    

![](https://images.gitbook.cn/018f5f30-f04e-11ea-8755-9ff4c3d0bc34)

