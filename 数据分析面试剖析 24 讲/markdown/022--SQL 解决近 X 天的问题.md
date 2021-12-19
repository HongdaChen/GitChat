在面试和实际项目中，我们经常会遇到这样两类问题，即以时间为轴线，沿着时间轴分析过去一段时间的用户特征或者行为。

一类是根据用户第一次访问的时间统计最近 N 天的行为特征，称之为近 X 天问题；另一类是根据用户第一次访问的时间统计连续 N 天的行为特征，称之为连续 X
天问题。

下面，先讲下近 X 天问题的解决方法。一般在 BI
报表里面，关于用户分析时，用户留存是个不可缺少的分析，而业界比较成熟的判断标准就是计算一些具体的指标，包括计算用户次日、3 日、7 日、30 日和 90
日的留存率。

这些指标的通俗定义如下：

  * 次日留存：当日登录后，第二天也登录了，比如 2021.2.10 登录过，2021.2.11 登录的算作次日留存。
  * 三日留存：当日登录后，第三天也登录了，比如 2021.2.10 登录过，2021.2.12 登录的算作 3 日留存。
  * 七日留存：当日登录后，第七天也登录了，比如 2021.2.10 登录过，2021.2.16 登录的算作 7 日留存。
  * 以此类推……

现假设，有一张用户登录表 t_user_login，字段 user_id 和 login_time 分别表示新增用户 id
和登陆时间。接下来，我们计算用户次日、3 日、7 日、30 日和 90 日的留存率。

一般解决此类近 X 天问题的思路如下：

  * 第一步，获取新增用户最早的登陆时间，标记为新的一列 first_login_time；
  * 第二步，计算最早登陆时间和所有登陆时间之前的天数差值，即 login_time - first_login_time，标记为新的一列 login_by_day；
  * 第三步，列转行，找到 login_by_day 对应为 1、3、7、30 和 90 的个数，并分别标记为新的列 day_1、day_3、day_7、day_30 和 day_90;
  * 第四步，计算留存率，对不同留存日期的 user_id 进行汇总就是留存人数，除以首日登录人数，就得到了不同留存时间的留存率。

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
    

按照上面的解决思路，进行操作。

步骤一：从表中提取出新增用户的 user_id 和 login_time，并格式化 login_time。

    
    
    SELECT
        user_id,
        DATE_FORMAT(login_time,'%Y/%m/%d') as login_time
    FROM t_user_login
    GROUP BY user_id,login_time;
    

步骤二：增加一列 first_login_time，存储每个用户 ID 最早登录时间。

    
    
    SELECT
        b.user_id,
        b.login_time,
        c.first_login_time
    FROM 
        (SELECT
        user_id,
        DATE_FORMAT(login_time,'%Y/%m/%d') as login_time
        FROM t_user_login
        GROUP BY user_id,login_time) b
    LEFT JOIN
        (SELECT       
            a.user_id,
            min(a.login_time) as first_login_time
         FROM 
           (SELECT
            user_id,
            DATE_FORMAT(login_time,'%Y/%m/%d') as login_time
            FROM t_user_login
            GROUP BY user_id,login_time) a
         GROUP BY a.user_id) c
    ON b.user_id = c.user_id
    ORDER BY b.user_id,b.login_time;
    

步骤三：用登录时间-最早登录时间得到一列 login_by_day。

    
    
    SELECT 
        d.user_id,
        d.login_time,
        d.first_login_time,
        DATEDIFF(d.login_time,d.first_login_time) as login_by_day
    FROM
      (SELECT
        b.user_id,
        b.login_time,
        c.first_login_time
       FROM 
        (SELECT
        user_id,
        DATE_FORMAT(login_time,'%Y/%m/%d') as login_time
        FROM t_user_login
        GROUP BY user_id,login_time) b
       LEFT JOIN
        (SELECT       
            a.user_id,
            min(a.login_time) as first_login_time
         FROM 
           (SELECT
            user_id,
            DATE_FORMAT(login_time,'%Y/%m/%d') as login_time
            FROM t_user_login
            GROUP BY user_id,login_time) a
         GROUP BY a.user_id) c
      ON b.user_id = c.user_id
      ORDER BY b.user_id,b.login_time) d
    ORDER BY d.user_id,d.login_time;
    

步骤四：找到 login_by_day 对应为 1、3、7、30 和 90 的个数，并分别标记为新的列 day_1、day_3、day_7、day_30 和
day_90。

    
    
    SELECT
        e.first_login_time,
        sum( CASE WHEN e.login_by_day = 1 THEN 1 ELSE 0 END ) day_1,
        sum( CASE WHEN e.login_by_day = 3 THEN 1 ELSE 0 END ) day_3,
        sum( CASE WHEN e.login_by_day = 7 THEN 1 ELSE 0 END ) day_7 
    FROM
        (
        SELECT
            d.user_id,
            d.login_time,
            d.first_login_time,
            DATEDIFF( d.login_time, d.first_login_time ) AS login_by_day 
        FROM
            (
            SELECT
                b.user_id,
                b.login_time,
                c.first_login_time 
            FROM
                ( SELECT user_id, DATE_FORMAT( login_time, '%Y/%m/%d' ) AS login_time FROM t_user_login GROUP BY user_id, login_time ) b
                LEFT JOIN (
                SELECT
                    a.user_id,
                    min( a.login_time ) AS first_login_time 
                FROM
                    ( SELECT user_id, DATE_FORMAT( login_time, '%Y/%m/%d' ) AS login_time FROM t_user_login GROUP BY user_id, login_time ) a 
                GROUP BY
                    a.user_id 
                ) c ON b.user_id = c.user_id 
            ORDER BY
                b.user_id,
                b.login_time 
            ) d 
        ORDER BY
            d.user_id,
            d.login_time 
        ) e 
    GROUP BY
        e.first_login_time 
    ORDER BY
        e.first_login_time
    

通过上面的 SQL，我们计算出新增用户在不同时间段能的留存数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210224135638321.JPG#pic_center)

根据最后得到的数据，我们直接对应除以当日新增用户数，就能算出来留存率。

    
    
    SELECT
        e.first_login_time,
        count(DISTINCT e.user_id) as new_person,
        sum( CASE WHEN e.login_by_day = 1 THEN 1 ELSE 0 END )/count(DISTINCT e.user_id) day_1,
        sum( CASE WHEN e.login_by_day = 3 THEN 1 ELSE 0 END )/count(DISTINCT e.user_id) day_3,
        sum( CASE WHEN e.login_by_day = 7 THEN 1 ELSE 0 END )/count(DISTINCT e.user_id) day_7,
        sum( CASE WHEN e.login_by_day = 30 THEN 1 ELSE 0 END )/count(DISTINCT e.user_id) day_30,
        sum( CASE WHEN e.login_by_day = 90 THEN 1 ELSE 0 END )/count(DISTINCT e.user_id) day_90 
    FROM
        (
        SELECT
            d.user_id,
            d.login_time,
            d.first_login_time,
            DATEDIFF( d.login_time, d.first_login_time ) AS login_by_day 
        FROM
            (
            SELECT
                b.user_id,
                b.login_time,
                c.first_login_time 
            FROM
                ( SELECT user_id, DATE_FORMAT( login_time, '%Y/%m/%d' ) AS login_time FROM t_user_login GROUP BY user_id, login_time ) b
                LEFT JOIN (
                SELECT
                    a.user_id,
                    min( a.login_time ) AS first_login_time 
                FROM
                    ( SELECT user_id, DATE_FORMAT( login_time, '%Y/%m/%d' ) AS login_time FROM t_user_login GROUP BY user_id, login_time ) a 
                GROUP BY
                    a.user_id 
                ) c ON b.user_id = c.user_id 
            ORDER BY
                b.user_id,
                b.login_time 
            ) d 
        ORDER BY
            d.user_id,
            d.login_time 
        ) e 
    GROUP BY
        e.first_login_time 
    ORDER BY
        e.first_login_time
    

上面求用户留存率的问题就解决了，解决问题按照上面的思路即可。至于 SQL 的写法，还有多种写法，不一定按照上面的 SQL 来做，下面提供一种更好理解的
SQL 写法。

    
    
    SELECT
        first_login_time '日期',
        count(user_id_d0) '新增数量',
        count(user_id_d1) / count(user_id_d0) '次日留存',
        count(user_id_d3) / count(user_id_d0) '3 日留存',
        count(user_id_d7) / count(user_id_d0) '7 日留存',
        count(user_id_d30) / count(user_id_d0) '30 日留存',
        count(user_id_d90) / count(user_id_d0) '90 日留存'
    FROM
        (
            SELECT DISTINCT
                first_login_time,
                a.user_id_d0,
                b.user_id AS user_id_d1,
                c.user_id AS user_id_d3,
                d.user_id AS user_id_d7,
                f.user_id AS user_id_d30,
                g.user_id AS user_id_d90
            FROM
                (
                    SELECT       
            user_id as user_id_d0,
            min(DATE_FORMAT(login_time,'%Y/%m/%d')) as first_login_time
            FROM t_user_login
            GROUP BY user_id
                ) a
            LEFT JOIN t_user_login b ON DATEDIFF(DATE(b.login_time),a.first_login_time) = 1
            AND a.user_id_d0 = b.user_id
            LEFT JOIN t_user_login c ON DATEDIFF(DATE(c.login_time),a.first_login_time) = 3
            AND a.user_id_d0 = c.user_id
            LEFT JOIN t_user_login d ON DATEDIFF(DATE(d.login_time),a.first_login_time) = 7
            AND a.user_id_d0 = d.user_id
            LEFT JOIN t_user_login f ON DATEDIFF(DATE(f.login_time),a.first_login_time) = 30
            AND a.user_id_d0 = f.user_id
            LEFT JOIN t_user_login g ON DATEDIFF(DATE(g.login_time),a.first_login_time) = 90
            AND a.user_id_d0 = g.user_id
        ) AS temp
    GROUP BY
    first_login_time
    

结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210224150426798.JPG#pic_center)

这个 SQL 和上面的 SQL 相比，可能更好计算出结果，但是表关联非常多，遇到数据量很大的时候，查询比较耗时，SQL 可能不是最优的。

### 总结

关于用户留存的问题，是数据分析师面试的时候经常遇到的一类问题，建议自己动手实践和手写一遍，做到更好的理解其解决思路。

