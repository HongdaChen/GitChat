上一篇内容，通过用户留存率的案例，讲解了解决近 X 天问题的思路，接下来，在本篇内容来看看关于连续 X
天的问题，该类问题是面试和实际业务中经常需要解决的问题。

首先对连续指标、做个定义，如下：

  * 1 日连续：当日登录后，第二天也登录了，比如 2021.2.10 登录过，2021.2.11 登录的算作 1 日连续。 
  * 3 日连续：当日登录后，第二和三天也登录了，比如 2021.2.10 登录过，2021.2.11 和 2021.2.12 登录的算作 3 日连续。
  * 以此类推……

现假设，有一张用户登录表 t_user_login，字段 user_id 和 login_time 分别表示用户 id 和登陆时间。接下来，我们通过 SQL
来求连续 3 日、7 日登陆的用户。

一般，解决此类连续 X 天问题的思路如下：

  * 第一步，可以使用窗口函数对 user_id 分组排序 rn； 
  * 第二步，用登陆时间减去排序后的序号 rn；
  * 第三步，如果日期连续的话，则得到的这个日期会相同。

下面，我们通过一个案例来进行说明。

首先，我们先建 t_user_login 表。

    
    
    DROP TABLE IF EXISTS `t_user_login`;
    CREATE TABLE `t_user_login` (
      `id` int NOT NULL AUTO_INCREMENT COMMENT '主键',
      `user_id` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '用户 id',
      `login_time` datetime DEFAULT NULL COMMENT '登陆时间',
      PRIMARY KEY (`id`,`user_id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=31 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
    

然后，插入测试数据，如下：

    
    
    INSERT INTO t_user_login VALUES ('1', 'test01', '2020-02-02 15:17:08');
    INSERT INTO t_user_login VALUES ('2', 'test01', '2020-02-03 18:17:08');
    INSERT INTO t_user_login VALUES ('3', 'test01', '2020-02-04 17:17:08');
    INSERT INTO t_user_login VALUES ('4', 'test01', '2020-02-05 19:17:08');
    INSERT INTO t_user_login VALUES ('5', 'test01', '2020-02-07 12:17:08');
    INSERT INTO t_user_login VALUES ('6', 'test01', '2020-02-10 15:17:08');
    INSERT INTO t_user_login VALUES ('7', 'test01', '2020-02-11 15:17:08');
    INSERT INTO t_user_login VALUES ('8', 'test01', '2020-02-12 15:17:08');
    INSERT INTO t_user_login VALUES ('9', 'test01', '2020-02-15 15:17:08');
    INSERT INTO t_user_login VALUES ('10', 'test01', '2020-02-20 15:17:08');
    INSERT INTO t_user_login VALUES ('11', 'test02', '2020-02-03 15:17:08');
    INSERT INTO t_user_login VALUES ('12', 'test02', '2020-02-05 15:17:08');
    INSERT INTO t_user_login VALUES ('13', 'test02', '2020-02-06 15:17:08');
    INSERT INTO t_user_login VALUES ('14', 'test02', '2020-02-09 15:17:08');
    INSERT INTO t_user_login VALUES ('15', 'test02', '2020-02-10 15:17:08');
    INSERT INTO t_user_login VALUES ('16', 'test02', '2020-02-11 15:17:08');
    INSERT INTO t_user_login VALUES ('17', 'test02', '2020-02-12 15:17:08');
    INSERT INTO t_user_login VALUES ('18', 'test02', '2020-02-15 15:17:08');
    INSERT INTO t_user_login VALUES ('19', 'test02', '2020-02-18 15:17:08');
    INSERT INTO t_user_login VALUES ('20', 'test02', '2020-02-21 15:17:08');
    INSERT INTO t_user_login VALUES ('21', 'test03', '2020-02-05 15:17:08');
    INSERT INTO t_user_login VALUES ('22', 'test03', '2020-02-06 15:17:08');
    INSERT INTO t_user_login VALUES ('23', 'test03', '2020-02-08 15:17:08');
    INSERT INTO t_user_login VALUES ('24', 'test03', '2020-02-09 15:17:08');
    INSERT INTO t_user_login VALUES ('25', 'test03', '2020-02-12 15:17:08');
    INSERT INTO t_user_login VALUES ('26', 'test03', '2020-02-14 15:17:08');
    INSERT INTO t_user_login VALUES ('27', 'test03', '2020-02-22 15:17:08');
    INSERT INTO t_user_login VALUES ('28', 'test03', '2020-02-23 15:17:08');
    INSERT INTO t_user_login VALUES ('29', 'test03', '2020-02-25 15:17:08');
    INSERT INTO t_user_login VALUES ('30', 'test03', '2020-02-28 15:17:08');
    

按照上面的解决思路，进行操作：

步骤一：可以使用窗口函数对 user_id 分组排序 rn。

    
    
    SELECT
     user_id,
     login_time, 
     ROW_NUMBER() OVER(partition by user_id order by login_time) as rn
    FROM t_user_login
    

步骤二：用登陆时间减去排序后的序号 rn，标记为新的一列 tmp_time。

    
    
    SELECT
        a.user_id,
        DATE_FORMAT(a.login_time,'%Y/%m/%d' ) AS login_time,
        a.rn,
        DATE_FORMAT( date_sub( a.login_time, INTERVAL a.rn DAY ), '%Y/%m/%d' ) AS tmp_time 
    FROM
        ( SELECT
           user_id, 
           login_time, 
           ROW_NUMBER() OVER ( PARTITION BY user_id ORDER BY login_time ) AS rn 
         FROM t_user_login ) a
    

通过上面的 SQL，我们来看下，对应的结果如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225121510771.JPG?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzMwNjQz,size_16,color_FFFFFF,t_70#pic_center)

步骤三：如果日期连续的话，则得到的这个日期会相同，通过统计相同日期的个数，来计算出连续登陆的用户。

比如下面我们计算各用户的最大连续登陆天数。

    
    
    SELECT
    b.user_id,
    max(b.max_count_day) as '最大连续登陆天数'
    FROM
    (SELECT
        a.user_id,
        DATE_FORMAT(a.login_time,'%Y/%m/%d' ) AS login_time,
        a.rn,
        DATE_FORMAT( date_sub( a.login_time, INTERVAL a.rn DAY ), '%Y/%m/%d' ) AS tmp_time,
        count(1) as max_count_day 
    FROM
        ( SELECT
           user_id, 
           login_time, 
           ROW_NUMBER() OVER ( PARTITION BY user_id ORDER BY login_time ) AS rn 
         FROM t_user_login ) a
     GROUP BY a.user_id,tmp_time) b
     GROUP BY b.user_id
    

结果如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/202102251401068.JPG#pic_center)

### 总结

以上是关于连续 X 天问题的解决思路，主要是对窗口函数的灵活运用，类似的案例网上很多，可以多动手实践几个。

下面提示一个 mysql 执行 SQL 语句常见的问题，错误提示：

    
    
    this is incompatible with sql_mode=only_full_group_by
    

上面的错误，一般出现在新安装的 MySQL 或者 MySQL 进行版本升级之后，在对 GROUP BY 聚合操作时，如果在 SELECT 中的列，没有在
GROUP BY 中出现，那么这个 SQL 是不合法的，因为列不在 GROUP BY 从句中。简而言之，就是 SELECT 后面接的列必须被 GROUP
BY 后面接的列所包含。

解决问题可以参考下面两篇博客：

  * <https://blog.csdn.net/qq_42175986/article/details/82384160>
  * <https://www.imooc.com/article/294753>

